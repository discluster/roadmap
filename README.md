![](https://i.jacher.io/discluster.png)

# Discluster

Discluster is a (still theoretical) standalone cluster management system for large Discord bots, written in pure TypeScript.<br>
Discluster is designed to be easily paired with complete client systems, which include a rest client and a command handler.<br>

## Who This Is For

Discluster is designed to be an extremely scalable sharding solution for Discord bots. Its dynamic design makes it suitable for any medium (10,000 guilds) to very large (1,000,000+ guilds) bot. For bots smaller than this, the amount of processing that takes place within the management of clusters may not make it worthwhile, but this solution should theoretically be able to handle any number of guilds.

## Why Does This Exist?

Most Discord bots operate by connecting to the [Discord gateway](https://discord.com/developers/docs/topics/gateway) via [WebSocket connections](https://en.wikipedia.org/wiki/WebSocket). Most bots that are in less than about 5000 guilds can operate via a single connection. However, above this count, the amount of packets sent by the gateway to the client can become too much for the connection to handle. This is where [sharding](https://discord.com/developers/docs/topics/gateway#sharding) comes in.

Normally, when a bot implements sharding, all shards are present within a single process, known as a shard cluster (or just 'cluster'). As a bot grows, the amount of shards it requires also grows (at a linear rate). Eventually, the amount of shards required may be too much for an individual process, in which case a multi-cluster system may be implemented, where a bot is broken up into several process, where each one handles a certain number of shards.

Thinking even larger, eventually a bot may grow to a size so large that it may require multiple machines to service the amount of events it recieves. This is where Discluster steps in.<br>
Discluster is designed to be an easily extensible, multi-machine clustering solution for the largest of Discord bots, allowing for automatic load-balancing and redundancy, all in one elegant solution.

## Key Objectives

    - Provide a complete and painless clustering solution.
    - Allow easy extensibility of all components, to create a complete clustered Discord bot solution.
    - Create an easily maintained and futureproof codebase.
    - Use minimal third-party dependencies (within reason).

## Basic Design

Discluster is split into three different types of process:

    - The centralised MASTER server. Responsible for the managing of CONTROL processes.
    - CONTROL processes. Responsible for per-machine management of cluster processes.
    - Cluster processes. Responsible for the managing of individual shards, and event handling.

![](https://i.jacher.io/server_layout.png)
![](https://i.jacher.io/machine_layout.png)

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