# RFC005 Distributed Network using gRPC

|  |  |
| :--- | :--- |
| Nuts foundation | R.G. Krul |
| Request for Comments: 005 | Nedap |
|  | February 2021 |

## Distributed Network using gRPC
### Abstract

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
* **Transaction**: ...
* **DAG** (Directed Acyclic Graph): ... 
* **Heads**: Latest transactions of the DAG with no succeeding transactions that refer to it as previous transaction,

## 3. Goals

The protocol aims to synchronize the local DAG with that of peers by;

1. making sure transactions produced by the local node are retrieved by its peers, and
2. retrieving transactions produced by peers.

## 4. Network Topology

The network is a full mesh peer-to-peer network: all participants in the network try to connect to any peer they discover.

The full mesh topology is expected to be performant up to ca. 20 nodes (source?). When a production network is expected
surpass that number of nodes, the protocol should be adjusted to form a partial mesh where nodes only connect to a
maximum number of peer. For instance, a IPFS node tries to connect to 10 other nodes which are randomly distributed.

## 5. Operation

The protocol operates as follows;

1. The local node broadcasts the node's DAG heads at a set interval.
2. Compare the node's DAG heads with those broadcast by our peers.
3. If a peer has a head which is not present on the node's DAG, the node queries the peer's transactions.
4. The peer responds to the query with all transactions on its DAG.
5. The node checks each peer transaction on its local DAG.
   - Verify its signature is correct.
   - Add it to the local DAG if not present yet.
   - If the transaction's payload is missing, query it from the peer. The peer responds with the payload. 

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

In this protocol each day considered a block and a new block is started at midnight (UTC). The signing time of a transaction
determines which block it belongs to. As such the signing time of the DAG's root transaction determines the first block.

### 5.1.1. 
It can be problematic when peers add new transactions from old branches, because 


Parameters:
 - Maximum age of new transactions, in blocks: 3 days
 - Block size: 1 day
 - A new block is started at 00:00 UTC

### 5.2. Broadcasting

The local node's DAG heads MUST be broadcast at an interval using the `AdvertHashes` message, by default every 2 seconds. The interval MAY be adjusted
by the node operator but SHALL NOT be so short that it clutters the network. It is advised to keep it relatively short
because it directly influences the speed by which new transactions are propagated to peers.  

### 5.3. Querying Peer's DAG

When the local node decides to query a peer's DAG because it differs from its own, it uses the `TransactionListQuery` message.
The receiving peer MUST respond with the `TransactionList` message containing all transactions (without payload) from its DAG.

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

### 6.3. Peer Discovery

The protocol doesn't provide a way to discover new peers automatically and is relying on the system to provide it with
new peers to connect to.

### 6.4. Unresponsive Peers

When (re)connecting to a peer that's unresponsive the node MUST take measures to avoid flooding it, since that only
adds more load to a system possibly under stress. A back-off strategy SHOULD be used which only reconnects after an
every increasing waiting period.

In future versions we can add the concept of "dead nodes", but due to the anticipated size of the network it's not
a feature at this moment.

### 6.5. Protocol Version

When connecting, the node and peer MUST exchange their protocol version as `version` metadata header. When either side
received a version that's not equal to `1` it MUST close the connection.

### 6.6. Security

Connections MUST be secured using TLS v1.2 (or higher) with both client- and server X.509 certificates.
Refer to [RFC008 Certificate Structure](rfc008-certificate-structure.md) for requirements regarding these certificates
and which Certificate Authorities should be accepted.

## 7. Protobuf Definition

```proto
{% include "../.gitbook/assets/rfc005/network.proto" %}
```

## 9. Attack Vectors

This section describes anticipated attacks vectors and (non malicious) situations that may threat a node, and how they're mitigated.

When a peer performs an action which is identified as a threat, nodes SHOULD immediately close the connection and
inform the peer of the rule that was violated. That way the operator of the peer can identify what should be fixed
on their node. However, when offences are repeated the node SHOULD apply "Three Strikes Out"; after 3 violations the node
SHOULD deny further connections with that peer, until the ban is lifted by an operator.

TODO: Close and deny connections from a certain certificate?
TODO: Limit number of connections from a certain certificate?

### 9.1. Denial of Service

#### Threat: Memory Exhaustion due to Large Messages

A (malicious) peer could exhaust the node's memory with (many) large messages.

Countermeasures:
- Nodes SHOULD NOT accept incoming messages larger than 5mb (5242880 bytes).

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
- Nodes SHOULD limit the number of messages received from a peer.

TODO: What would be a good number? 5/second?

#### Threat: Uncontrolled DAG Growth

A single peer or orchestrated group of peers can quickly produce many transactions, quickly growing the DAG. Since history
must be retained to verify the DAG integrity in the future, it's desirable to limit the number of faulty transactions or
transactions without a meaningful payload.

TODO: How to mitigate this?

### 9.2. Data Manipulation

#### Threat: Manipulating Transaction Payload

By altering a transaction's payload when responding to a payload query an attacker can hamper nodes or even
steal identities (e.g. DIDs).

Countermeasures:
- Nodes MUST verify the payload hash when receiving transaction payload.

## 8. Issues

The following issues must be either be solved in this RFC or acknowledged being acceptable:

- Peers attempt to build a full mesh, which might break down with many nodes
  - Nodes broadcast their last hash to sync (every 2 secs), lots of chatter
- Most semi-public DLTs are block-based and have 'proof' (of work, of stake, voting, etc). We don't, which allows parties to easily add data (the good) but that also goes for malicious parties (the bad).
  - Possible solution: introduce time-based blocks (compare hash of all transactions up until last midnight UTC)
  - Additionally, we could require transactions to have a timestamp that's relatively actual (using Bitcoin's network-adjusted-time, https://en.bitcoin.it/wiki/Block_timestamp)
- Fast replay (from another node) when starting a new node
- We need some kind of flooding detection and prevention
- Fast comparison of remote node's graph vs own graph (current implementation just requests whole DAG and compares it).
  This might will become slower and slower when the DAG grows.
  - Possible solution: use the time-based blocks, which can be used as Merkle tree.
    - Problem: what if a party produces 100.000s of transactions on a single day?
      If a SHA-256 hash takes 32 bytes it would be 3mb worth of hashes. Not a problem to transfer
      once or twice, but becomes problematic when parties keep appending and it must be exchanged
      over and over again to compare DAGs   
- Maybe; more authorisation than just "you need to have a PKIOverheid certificate"?
	- What if a malicious node produces lots (e.g. gigabytes) of crap that clutters the DAG?
- Detect dead nodes
  - Possible solution: SWIM?
- Retrieve transaction payload from other nodes than the one who sent you the hash
  - Possible solution: kademlia-ish hash distance comparison to determine which node to query for the contents? 
- Detect nodes that refuse to sync with you (but keep sending different hashes)
- Peer ID spoofing: peer ID can be spoofed by copying a peer's ID and providing it when connecting to other nodes.
  Depending on the other peers' implementation, spoofing could be used to impersonate other nodes, which may cause interruptions the other nodes' operation, 
  - Possible solution: peer ID MUST be created by generating a P-256 elliptic curve key pair, taking the public key and signing it with the private key.
    - Option 1: signing challenge (downside: requires a handshake, which is complicated).
    - Option 2: signing a nonce (downside: peer needs to track the nonce to make sure it's not used more than once).

