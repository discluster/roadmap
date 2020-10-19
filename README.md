# Discluster

Discluster is a (still theoretical) standalone cluster management system for large Discord bots, written in pure TypeScript.<br>
Discluster is designed to be easily paired with complete client systems, which include a rest client and a command handler.<br>

## Who This Is For

Discluster is designed to be an extremely scalable sharding solution for Discord bots. Its dynamic design makes it suitable for any medium (10,000 guilds) to very large (1,000,000+ guilds) bot. For bots smaller than this, the amount of processing that takes place within the management of clusters may not make it worthwhile, but this solution should theoretically be able to handle any number of guilds.

## Basic Design

Discluster is split into three different types of process:

    - The centralised MASTER server. Responsible for the managing of CONTROL processes.
    - CONTROL processes. Responsible for higher-level event handling and management of cluster processes.
    - Cluster processes. Responsible for the managing of shards that are in it, and base event handling.

### MASTER

The MASTER server is the main controlling process for the entire system. It is abstracted from all communication with the Discord API. <br>
MASTER can send control signals to CONTROL processes, including instructions to re-balance the amount of clusters in the system, as well as instruct the CONTROL to change the amount of shards each cluster holds (most useful when combined with max_concurrency when identifying to the Discord gateway).<br>
MASTER can also recieve events from CONTROL processes, usually cluster state information, so that MASTER can re-load-balance in case a CONTROL, or a cluster, fails.<br>

MASTER is also responsible for load-balancing the system when it is started. The system can still operate while the MASTER is not running, but if MASTER is down for an extended period of time it may introduce issues with load balancing and, in addition, instructions to CONTROL processes cannot be properly broadcasted.

The MASTER server does **NOT** handle any communication with the Discord API (with the exception of fetching recommended shard count) or with individual clusters. These are handled by the clusters and the CONTROL processes respectively.

### CONTROL

CONTROL processes are responsible for the management of clusters that exist on an individual system, i.e. there is one CONTROL process per machine.<br>
CONTROL processes can be started before the system is booted, the processes will wait for MASTER to provide instructions on how many shards to handle in its clusters.

After recieving instructions to spawn clusters, the CONTROL process is also responsible for the management of the clusters. The clusters themselves exist as child processes to CONTROL, and communicate with CONTROL via IPC (as opposed to MASTER, where CONTROL communicates through a WebSocket).

### Clusters

Clusters are the most numerous process type in a Discluster system. Clusters are responsible for serving shards to the Discord gateway, as well as handling events from the Discord gateway.<br>
Clusters may serve differing amounts of shards, although the amount of shards per cluster will be the same for each machine, i.e. machine A could have 10 per cluster and machine B could have 7 per cluster.

Each shard in a cluster is essentially just a WebSocket connection. Each socket will pass events up to the cluster for handling, mainly dispatch events (such as message creations).<br>
From there, a custom packet handling* solution can be 'attached' to the cluster. More detail on this on the cluster repository (soon).

*'packet handling' in this case refers to anything beyond recieving events from the gateway, which could include command handling, caching, etc.

## Prerequisites

In terms of hosting capability, Discluster can run on a single machine or many machines. The minimum amount of processes it will spam is 3 - one MASTER, one CONTROL and one cluster.
Discluster will be written almost exclusively in TypeScript, so running Discluster will require TypeScript. The only other required dependencies should be `ws` and `node-fetch` (for requesting recommended shards), subject to change.