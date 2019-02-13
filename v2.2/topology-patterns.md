---
title: Cluster Topology Patterns
summary: An illustration of common topology patterns.
toc: true
---

This is doc covers common cluster topology patterns with setup examples and performance considerations.

## Considerations

When selecting a pattern for your cluster, the following must be taken into consideration:

- Understand the function of a CockroachDB [leaseholder](architecture/replication-layer.html#leases).
- Understand the impacts of the [leaseholder](architecture/life-of-a-distributed-transaction.html#leaseholder-node) on read and write activities.
- The leaseholders are local to reader and writers within the datacenter.
- The leaseholder migration among the datacenters is minimized by using the [partitioning feature](partitioning.html).
- The follow-the-workload tunable setting is set properly for the workload.
- Whether the application is designed to use the [partitioning feature](partitioning.html) or not.

Before you select a candidate pattern for your cluster, use the following broad patterns as a starting point for discussion and discuss trades-off. The below describes some of the characteristics of existing catalog of patterns.

{{site.data.alerts.callout_info}}
This doc does not factor in hardware differences.
{{site.data.alerts.end}}


## Single (local) datacenter clusters

Local deployment is a single datacenter deployment. The network latency among the nodes is expected to be the same, around 1ms.

### Basic structure for minimum resilience

In the diagrams below:

- `App` is an application that accesses CockroachDB
- `HA-Proxy` is a software based load balancer
- 1, 2, and 3 each represents a CockroachDB node
- The nodes are all running in a single datacenter.

Normal operating mode:

~~~
      App
       |
    HA-Proxy
  /    |    \
 /     |     \
1------2------3
 \___________/
~~~

If Node 1 goes down, the database and app are still fully operational.

~~~
      App
       |
    HA-Proxy
  /    |    \
 /     |     \
x------2------3
 \___________/
~~~

If node 2 is down, the database and app are still fully operational.

~~~
      App
       |
    HA-Proxy
  /    |    \
 /     |     \
1------x------3
 \___________/
~~~

If node 3 is down, the database and app are still fully operational.

~~~
      App
       |
    HA-Proxy
  /    |    \
 /     |     \
1------2------x
 \___________/
~~~

### More resilient structure

Three or more nodes are recommended to provide high availability, share the load, and spread the capacity. Dynamically scaling out the nodes from 3 to 4, 4 to 5, or any other intervals is supported, and there are no constraints on the server increments.

The diagram below depicts each node as a letter (A, B, C, D, E):

~~~
A------C         A-----C           A-----C
 \    /  online  |     |  online   | \ / |
  \  /   ------> |     |  ------>  |  E  |
   \/            |     |           | / \ |
   B             B-----D           B-----D
~~~

## Multi-region clusters

The sample patterns in this section indicate a broad placement of datacenters with “West”, “East” and “Central”. The latency numbers (e.g., 60ms) represent network round-trip from one datacenter to another.

### Basic structure for minimum resilience

Each region defines an availability zone. Three or more regions are recommended. Similar to the [local](#single-local-datacenter) topology, more regions can be added dynamically. It is recommended to have homogenous configuration among the regions for simplified operations. For sophisticated workloads, each region could have different node count and node specification. This heterogeneous configuration could better handle regional specific concurrency and load characteristics. The network latency among the regions is expected to be linear to the distance among the nodes.

The diagram below depicts asymmetrical setup where the Central is closer to the West than the East. This configuration will provide better write latency to the write workloads in the West and Central.

~~~
A---C         A---C
 \ /           \ /
  B             B
  West---80m---East
     \         /  
    20ms    60ms  
       \    /
        \  /
         \/
       Central
        A---C   
         \ /
          B
~~~

<!-- ### More resilient / different structure

_Add description_ -->

### Locality-aware load balancing

When locality is enabled, `haproxy` should be setup to load balance on the database nodes within the same locality as the app servers first. If all of the nodes this preferred locality is down, then try databases in other localities.

For a regional topology depicted in the below diagram:

- the West app servers should connect to the West CockroachDB servers.
- the Central app servers should connect to the Central CockroachDB servers.
- the East app servers should connect to the East CockroachDB servers.

Modern n-tier architecture is simplified as App and DB layers in the below diagram. A client connects to geographically close app server via GSLB. The app servers connect to one of the CockroachDB nodes within their geography via local balancer. A software-based load balancer, HAProxy, located on the app server, the configuration is provided by CockroachDB. A Network-based load balancer can also be used.

~~~
         Clients
           |
          GSLB
           |
  +--------+---------+
West    Central    East
  |        |         |
 App  |   App   |   App
 ---  |   ---   |   ---    
  LB  |    LB   |    LB
      |         |

  West---Central ---East
    \               /
     \ CockroachDB /  
      -------------
~~~

### High-Performance

Some applications have high-performance requirements. NJ and NY in the below diagram depict two separate datacenters that are connected by a high bandwidth low latency network. In this topology, NJ and NY have the performance characteristics of the local topology, but the benefit of Zero RPO and near Zero RTO disaster recovery SLA. CA and NV have been set up with a network capability. The central region serves as the quorum.

~~~
   NJ ---1ms--- NY
     \       /  
     20ms  20ms  
        \  /
         \/  
       Central
         /\  
        /  \
     20ms  20ms  
     /       \  
  CA ---1ms--- NV
  ~~~

## Global clusters

### Basic structure for minimum resilience

The global topology connects the multiple regional topologies together to form a single database that is globally distributed. Transactions are globally consistent.

~~~
    West-----East            West-------East
       \      /                \        /
        \Asia/                  \Europe/
         \  /                    \    /
          \/                      \  /
        Central                 Central
                Asia------Europe
                   \     /  
                    \   /  
                     \ /  
                   Americas

               West---------East
                  \          /
                   \Americas/  
                    \      /
                    Central
~~~

<!-- ### More resilient / different structure

_Add description_ -->

## Partitioned clusters

The sample patterns in this section assume the usage of the geo-partitioning feature, with “West”, “East” and “Central” indicating broad placement of datacenters. The latency numbers (e.g., 60ms) represent network round-trip from one datacenter to another.


Topology                                              | West                         | Central                | East
------------------------------------------------------+------------------------------+------------------------+------------------------
[Symmetrical](#symmetrical-clusters)                  | Read local, Write 60ms       | Read local, Write 60ms | Read local, Write 60ms
[Dual East](#dual-east-datacenters)                   | East	Read local, Write 60ms |                        | Read local, Write 5ms
[Dual East and West](#dual-east-and-west-datacenters) | Read local, Write 5ms        |	                      |

### Symmetrical clusters

During normal operations:

~~~
App                App
 \                  /
West ---60ms--- East
     \          /  
    60ms     60ms  
        \    /
         \  /  
       Central
          |
         App
~~~

#### Replicas and leaseholders

- Tables are partitioned at row-level by locality.
- Rows with the West partition have their Leaseholder in the West datacenter, rows with the Central partition have their Leaseholder in the Central datacenter, and rows with the East partition have their Leaseholder in the East datacenter.
- Replicas are evenly distributed among the three datacenters.

Availability expectations

- The cluster can survive a single datacenter failure.

#### Performance expectations

- Reads respond in a few milliseconds.
- Writes respond in 60 milliseconds.
- Symmetrical 60 milliseconds between datacenters.

#### Application expectations

- West App servers connect to the West CockroachDB nodes.
- Central App servers connect to the Central CockroachDB nodes.
- East App servers connect to the East CockroachDB nodes.

CockroachDB configuration

- Abbreviated startup flag for each datacenter:

    ~~~
    --loc=Region=East
    --loc=Region=Central
    --loc=Region=West
    ~~~

### Dual East datacenters

During normal operations:

~~~
App                App
 \                  /
West ---60ms--- East1
     \           |
       \        5ms
        60ms     |
           \____East2
~~~

#### Replicas and leaseholders

- rows with the West partition will have the leaseholder in the West DC, the East partition in the East DC1
- the 3 replicas will be evenly distributed among the three datacenters

#### Availability expectations

- Can survive a single datacenter failure

#### Performance expectations

- The reader can expect to have a couple of ms response time.
- The East writers can expect to have 5ms response time.
- The West writers can expect to have 60ms response time.

#### Application expectations

- West App servers connect to the West CockroachDB nodes
- East App servers connect to the East1 CockroachDB nodes

#### CockroachDB configuration

- Abbreviated startup flag for each datacenter:

    ~~~
    --loc=Region=East,DC=1
    --loc=Region=East,DC=2
    --loc=Region=West
    ~~~

### Dual East and West datacenters

During normal operations:

~~~
App                App
 \                  /
West1 ---60ms--- East1
  |   \        /   |
  5ms  - 60ms -    5ms
  |   /        \   |
West2----60ms----East2
~~~

#### Replicas and leaseholders

- Rows with the West partition will have the leaseholder in the West DC1, the East partition in the East DC1
- West partitions will have 1 replica each in West1 and West2, then 1 replica in East1 or East2
- East partitions will have 1 replica each in East1 and East2, then 1 replica in West1 or West2

#### Availability expectations

- Can survive a single datacenter failure

#### Performance expectations

- The reader can expect to have a couple of ms response time.
- The writers can expect to have 5ms response time.

#### Application expectations

- West App servers connect to the West CockroachDB nodes
- East App servers connect to the East CockroachDB nodes

#### CockroachDB configuration

- Abbreviated startup flag for each datacenter:

    ~~~
    --loc=Region=West,DC=1
    --loc=Region=West,DC=2
    --loc=Region=East,DC=1
    --loc=Region=East,DC=2
    ~~~

## Anti-patterns

_Do we want to add a section for bad patterns (i.e., two datacenters, even # of replicas)?_
