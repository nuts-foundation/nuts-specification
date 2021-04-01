# RFC005 Distributed Network using gRPC

|  |  |
| :--- | :--- |
| Nuts foundation | R.G. Krul |
| Request for Comments: 005 | Nedap |
|  | February 2021 |

## Distributed Network using gRPC
### Abstract

This RFC describes a protocol to build a distributed network for creating a shared Directed Acyclic Graph as described by
[RFC004 Verifiable Transactional Graph](rfc004-verifiable-transactional-graph.md) over [gRPC](https://grpc.io/),
which is an RPC framework that uses [Google Protocol Buffers](https://developers.google.com/protocol-buffers).

### Status

This document is currently a draft.

### Copyright Notice
![](../.gitbook/assets/license.png)

This document is released under the [Attribution-ShareAlike 4.0 International \(CC BY-SA 4.0\) license](https://creativecommons.org/licenses/by-sa/4.0/).

## 1. Introduction

When care organizations want to exchange public information (e.g. registry) using a decentralized network,
they need a protocol that builds a distributed network allowing them to freely query and publish that information.
This RFC describes a protocol for building such a distributed network using gRPC to achieve that goal.

## 2. Terminology

* **Node**: local Nuts software system acting (a.k.a. _Nuts Node_).
* **Peer**: remote software system connected to the local node using the protocol described here.
* **Transaction**: self-contained unit of application data on the DAG. 
* **DAG** (Directed Acyclic Graph): graph formed of all transactions that. It provides casual ordering for transactions and means to efficiently compare the local DAG with those of peers. 
* **Heads**: latest transactions of the DAG with no succeeding transactions that refer to it as previous transaction,

## 3. Goals

The protocol aims to synchronize the local DAG with that of peers by;

1. making sure transactions produced by the local node are retrieved by its peers, and
2. retrieving transactions produced by peers.

## 4. Network Topology

The network is a full mesh peer-to-peer network: all participants in the network try to connect to any peer they discover.

The full mesh topology is expected to be performant up to ca. 20 nodes (source?). When a production network is expected
surpass that number of nodes, the protocol should be adjusted to form a partial mesh where nodes only connect to a
maximum number of peer. For instance, an IPFS node tries to connect to 10 other nodes which are randomly distributed.

## 5. Operation

The protocol generally operates as follows:

1. The local node broadcasts at a set interval:
   - the heads of the current block (today) `T`,
   - the heads of the 2 previous blocks `T-1` and `T-2`,
   - XOR of the heads of historic blocks leading up to `T-2`.
   Heads of a block are either transactions that have to other transactions referring to it as `prev` or the last
   transaction of the block before midnight (the next transaction in the branch falls in the next day).
2. When receiving a peer's broadcast, compare it to the local DAG and add missing transaction's to the local DAG:
   - When all head hashes are known in the local DAG and the historic hash equals: no action required.
   - Peer has unknown head: query the block's transactions to find out which transactions are missing.
   - Peer's historic hash differs: DAGs have diverted severely which might indicate a network split, or a peer that
     tries to attack the network by injecting transactions with old timestamps. Ignore the peer's broadcast and report
     to the local node operator.
3. After adding a peer's transaction to the DAG (making sure its cryptographic signature is valid), query the payload from the
   peer if it's missing. 

The sections below specify the details of the protocol operation and maps it to the gRPC messages. See the section
"Protobuf Definition" for a full specification of the protobuf/gRPC contract.

### 5.1. Blocks

Many distributed ledgers (DLT) group transactions into blocks for optimization and consensus. This protocol uses blocks as well,
as optimization for:

- DAG comparison by deriving a single hash from (potentially) many transactions, which can be shared with peers, and
- consensus about up to which point the DAG is immutable (no new transactions can be added by branching).

In most block-based DLTs blocks are immutable, preventing new transactions from being added once the block is created
("mined" in Bitcoin terms). This protocol differs in that Nuts' DAG transactions don't hold transferable value
(in contrast to e.g. Bitcoin or Ethereum) and thus doesn't need consensus about recent transactions, and thus can have
mutable blocks (up to some point).

In this protocol each day is a block. Every day at midnight (UTC timezone) a new block starts.
The signing time of a transaction determines which block it belongs to. As such the signing time of the DAG's root transaction determines the first block.

### 5.1.1. History hash

The _history hash_ is a hash over all transactions leading up to a certain point. It is used for quickly comparing (large) DAGs.
It is calculated by sorting the head transaction references (ascending, so low-high order) and XOR-ing them. For example, if
there are 3 heads:

```
T1=a5b7485b33d485cc4744a63e3273e581e0e7d0fd1b3f020b19c3b913bd5465dc
T2=c0dc584345da8a0e1e7a584aa4a36c30ebdb79d907aff96fe0e90ee972f58a17
T3=f81228d88006ea4949cd6d1c8cbeaf9e51d37df15e9e4b2064f06155b9af4ae3

blockHash = xor(xor(T1, T2), T3) 
blockHash = 9d7938c0f608e58b10f393681a6e262f5aefd4d5420eb0449ddad6af760ea528
```

Each last transaction of every branch is considered a head for history hash calculation.

### 5.2. Broadcasting

The local node's DAG heads MUST be broadcast at an interval using the `AdvertHashes` message, by default every 2 seconds.
The interval MAY be adjusted by the node operator but MUST conform to the limits (min/max interval) defined by the network.
It is advised to keep it relatively short because it directly influences the speed by which new transactions are propagated to peers.  

### 5.3. Querying Peer's DAG

When the local node decides to query a peer's DAG because it differs from its own, it uses the `TransactionListQuery` message.
It MUST specify the block for which to retrieve the transactions using a Unix timestamp that falls within the requested
block. The receiving peer MUST respond with the `TransactionList` message containing all transactions (without payload) from its DAG.

### 5.4. Resolving Transaction Payload

When the local node is missing a transaction's payload, it SHOULD query the peer that provided the transaction
for the payload using the `TransactionPayloadQuery` message. The peer MUST respond with the `TransactionPayload` message,
providing the actual payload in the `data` field. If the peer doesn't have the payload the `data` field MUST be left empty.

## 6. Connections

The protocol uses [gRPC](https://grpc.io/) over [HTTP/2](https://tools.ietf.org/html/rfc7540). Since the protocol uses a
bi-directional stream over which peers can send and receive messages, there needs to be only a single connection between
two peers. For instance, if node _A_ connects to node _B_, _B_ can send messages back to _A_ without having to connect
back to node _A_. Nodes MUST avoid maintaining duplicate connections to their peers to minimize traffic and system load. 

### 6.1. Peer Identification

Since nodes might be operating behind reverse proxies, NAT routers and/or load balancers there needs to be a way to
identify a peer that's not dependent on the remote host/port of the connection. Instead, a node MUST generate a globally
unique identifier (the node's _peer ID_) and provide it as gRPC connection metadata on incoming or outgoing connections.
A peer ID MAY be changed every time a node (re)starts but MUST be the same for all connections of that node. 

### 6.2. Peer Discovery

The protocol doesn't provide a way to discover new peers automatically and is relying on the system to provide it with
new peers to connect to.

### 6.3. Unresponsive Peers

When (re)connecting to a peer that's unresponsive the node MUST take measures to avoid flooding it, since that only
adds more load to a system possibly under stress. A back-off strategy SHOULD be used which only reconnects after an
every increasing waiting period.

### 6.4. Protocol Version

When connecting, the node and peer MUST exchange their protocol version as `version` metadata header. When either side
received a version that's not equal to `1` it MUST close the connection.

### 6.5. Security

Connections MUST be secured using TLS v1.2 (or higher) with both client- and server X.509 certificates.
Refer to [RFC008 Certificate Structure](rfc008-certificate-structure.md) for requirements regarding these certificates
and which Certificate Authorities should be accepted.

## 7. Protobuf Definition

```proto
{% include "../.gitbook/assets/rfc005/network.proto" %}
```

## 8. Security Considerations

This section describes anticipated attacks vectors and (non malicious) situations that may threat a node, and how they're mitigated.

When a peer performs an action which is identified as a threat, nodes SHOULD immediately close the connection and
inform the peer of the rule that was violated. That way the operator of the peer can identify what should be fixed
on their node. However, when offences are repeated the node SHOULD apply "Three Strikes Out"; after 3 violations the node
SHOULD deny further connections using the peer's certificate (identified by certificate issuer DN and serial number),
until the ban is lifted by an operator.

### 8.1. Denial of Service

#### Threat: Memory Exhaustion due to Large Messages

A (malicious) peer could exhaust the node's memory with (many) large network messages.

Countermeasures:
- Nodes MUST NOT accept incoming network messages larger than 5mb (5242880 bytes).

#### Threat: Resource Exhaustion through Connection Flooding

Multiple peers might share the same IP address and certificate in clustering or cloud environments. However, it can also
be used by attackers to trying to flood the node with a very large number of connections exhausting resources like 
file descriptors, connection pools or thread pools.
 
Countermeasures:
- Nodes SHOULD limit the number of active connections from/to a single IP address (e.g. 5 connections).
- Nodes SHOULD limit the number of active connections from/to a single certificate subject (e.g. 5 connections).

#### Threat: Resource Exhaustion through Message Flooding

A peer that floods the node with many messages threatens the stability of a node, and the network in general: resources
for processing incoming messages often have hard limits in the form connection pools, thread pools or backlogs.

Countermeasures:
- Nodes SHOULD limit the number of network messages received from a peer to 5 per second.

#### Threat: Uncontrolled DAG Growth

A single peer or orchestrated group of peers can quickly produce many transactions, quickly growing the DAG. Since history
must be retained to verify the DAG integrity in the future, it's desirable to limit the number of faulty transactions or
transactions without a meaningful payload.

### 8.2. Data Manipulation

#### Threat: Manipulating Transaction Payload

By altering a transaction's payload when responding to a payload query an attacker can hamper nodes or even
steal identities (e.g. DIDs).

Countermeasures:
- Nodes MUST verify the payload hash when receiving transaction payload as specified by [RFC004 section 3.6 (Signature and payload verification)](rfc004-verifiable-transactional-graph.md)

## 9. Issues

The following issues must be either be solved in this RFC or acknowledged being acceptable:

- Peers attempt to build a full mesh, which might break down with many nodes
  - Nodes broadcast their last hash to sync (every 2 secs), lots of chatter
- Fast replay (from another node) when starting a new node
- We need some kind of flooding detection and prevention
- Detect dead nodes
  - Possible solution: SWIM?
- Retrieve transaction payload from other nodes than the one who sent you the hash
  - Possible solution: kademlia-ish hash distance comparison to determine which node to query for the contents? 
- Detect nodes that refuse to sync with you (but keep sending different hashes)
- Querying a peer when the local node receives a local hash now works by just querying all transactions for that block,
  which might become too slow when there are many transactions in a block. Possible optimizations:
  - Pathfinding: let peer find a path from the block's first hash to the unknown head and return all transactions.
    Pro: sure way to find all transactions leading up to an unknown head.
    Con: Pathfinding might be even more expensive on the peer's side than just returning all tx's for that block?
    Con: When DAGs just differ a little, this protocol has lots of overhead (CPU)
  - Query previous transactions of the unknown head, until all unknown transactions are resolved
    Pro: sure way to find all transactions
    Pro: works well when DAGs differ a little
    Con: When DAGs differ a lot (peer has produced lots of transactions leading up to the unknown head) this protocol
         has lots of overhead (network traffic)
