# RFC017 Distributed Network Protocol (v2) using gRPC

|                           |                |
|:--------------------------|:---------------|
| Nuts foundation           | R.G. Krul      |
| Request for Comments: 017 | Nedap          |
| Replaces: RFC005          | W.M. Slakhorst |
|                           | Nedap          |
|                           | December 2021  |

## Distributed Network Protocol (v2) using gRPC

### Abstract

This RFC specifies a new gRPC-based distributed network protocol. It replaces version 1 of the protocol as defined by [RFC005](rfc005-distributed-network-using-grpc.md).

### Status of document

This document describes a Nuts standards protocol.

### Copyright Notice

![](../.gitbook/assets/license.png)

This document is released under the [Attribution-ShareAlike 4.0 International \(CC BY-SA 4.0\) license](https://creativecommons.org/licenses/by-sa/4.0/).

## 1. Introduction

Please see the [protobuf definition](https://raw.githubusercontent.com/nuts-foundation/nuts-node/master/network/transport/v2/protocol.proto)
for the messages that are referred to in this RFC.
Connections are subject to the requirements specified by [RFC008 Certificate Structure](rfc/rfc008-certificate-structure.md).

## 2. Terminology
 
* **DAG** \(Directed Acyclic Graph\): graph formed of all transactions. It provides casual ordering for transactions and means to efficiently compare the local DAG with those of peers. See [RFC004](rfc004-verifiable-transactional-graph.md) for details.
* **Invertible Bloom Lookup Table (IBLT)**: A bloom filter where instead of just a 0 or 1, a count, key_sum and hash_sum is stored per bucket. [Original paper](https://www.ics.uci.edu/~eppstein/pubs/EppGooUye-SIGCOMM-11.pdf).
* **Lamport Clock (LC)**: Logical clock as defined in [RFC016](rfc016-lamport-clock.md).
* **Node**: local Nuts software system acting \(a.k.a. _Nuts Node_\).
* **Node identity**: (a.k.a. node DID) the DID a node uses to identify itself on the Nuts network. Multiple logical nodes (e.g. a cluster) may share the same node identity.
* **Peer**: remote Nuts node that communicates with the local node.
* **Private transactions**: transactions that are meant for a specific receiver (node) on the network. The transaction payload is solely shared with that particular node.
* **Set reconciliation**: process of finding the differences in two sets of data entries and using the least amount of possible transfers to synchronize the sets.
* **Transaction**: self-contained unit of application data on the DAG.
* **Transaction reference**: SHA256 of the transaction data.

Other terminology comes from the [Nuts Start Architecture](rfc001-nuts-start-architecture.md#nuts-start-architecture).

## 3. Authenticating node identity

Particular exchanges (private transactions) might require authentication of the peer's identity. This identity (a.k.a. node identity) is specified by [RFC015 Node identity](rfc015-node-identity.md).
If a node has a node DID and wishes to create an authenticated connection, it MUST send the DID as `nodeDID` gRPC header when establishing inbound or outbound connections.
If authentication fails, the authenticating side MUST close the connection and SHOULD return the following error: `nodeDID authentication failed` (status code: `16 (Unauthenticated)`).
The received node DID MUST be authenticated upon receiving, to directly inform the peer should its configuration be incorrect. 

As specified by RFC015, the node MUST authenticate the peer's node DID as follows:

1. Resolve peer's node DID to its corresponding DID document.
2. Assert that one of the `NutsComm` endpoint's host matches (one of) the `dNSName` SANs in the peer's TLS client certificate.

## 4. Conversations

Certain messages in the protocol are a response to a certain request. To make sure a response corresponds to a certain request, a `conversationID` is added to the request. Response type messages MUST add the same `conversationID` in the response.
The `conversationID` is scoped to a connection.
A node is free to choose the form of a `conversationID`, it MUST be unique during the lifetime of the connection.
It MUST be valid for at least 10 seconds.
It MUST be invalidated after 30 seconds of the last processed message.
If a lot of messages have to be processed, it could take a while for a conversation to finish.
To prevent a timeout during a long conversation, the timeout should be reset after handling of each valid message.

If a node receives a response with a `conversationID`, it MUST match its contents with the original request.
If a `conversationID` is unknown or if the response doesn't match the requirements, the message MUST be ignored.
The individual messages describe the requirements.

A node SHOULD limit the number of conversations to a single peer when these conversations contain overlapping data.
The protocol requires only 1 active conversation per peer at a time.
It's up to the implementation on how to mark a conversation as finished and remove it from its administration.

## 5. Gossip protocol

The most efficient protocol for synchronizing transactions over nodes is to simply send new transactions to all nodes.
This only works for the case where all the nodes are connected and when the nodes are always online.
This might be the case for 99% of the time. For the last 1%, the set reconciliation protocol of chapter 6 is used.
Although the set reconciliation protocol can also synchronize any difference in transactions sets between nodes, it's not very efficient for small differences.

### 5.1 Data requirements

#### 5.1.1 XOR

The exclusive-or (XOR) value is calculated over all transaction references within a range.
The size of the XOR value is therefore the same as the transaction reference size (SHA256: 32 bytes).
The XOR value is used to quickly determine if two nodes have processed the exact same set of transactions.

### 5.2 Operation

All messages and values mentioned in this chapter are scoped to a single connection between two nodes.
*Alice* and *Bob* are used to represent two nodes connected to each other.

The protocol generally operates as follows:

1. Alice sends at a set interval (§5.2.1):
   * XOR of all known transaction references.
   * The highest LC value.
   * A list of transaction references since the previous message. If no previous message has been sent, the list is empty.

2. When receiving Alice's message, Bob compares the XOR value from Alice with its own (§5.2.2):
   * When the XOR is the same: no action required.
   * When different:
      * Remove all known transaction references from the list.
      * Bob calculates a new XOR value combining its own XOR value and the remaining new/unknown transactions hashes. 
      * If this new XOR value matches that of Alice, Bob requests the list of unknown transactions.
      * If this new XOR value does not match that of Alice, Bob has two options to try and resolve their differences:
        * If Alice's LC is lower and the list of new transactions is *not* empty, Bob still requests the list of unknown transactions.
        * In any other case Bob sends a `State` message as defined in Chapter 6.

3. Alice responds with the requested transactions (§5.2.3).

4. After adding Alice's transactions to the DAG (making sure its cryptographic signature is valid), Bob SHOULD query the payload if it's missing. If Bob is missing transactions referenced by the received transactions, it sends a `State` message.

A node MUST make sure to only add transactions of which all previous transactions are present.

#### 5.2.1 Broadcasting new transactions

A node's new transaction references SHOULD be broadcast at an interval using the `Gossip` message, by default every 2 seconds. The interval MAY be adjusted by the node operator but MUST conform to the limits (min/max interval) defined by the network. It is advised to keep it relatively short because it directly influences the speed by which new transactions are retrieved.

The `Gossip` message MUST contain an `XOR` value, an `LC` value and a list of transaction references (`transactions`).
The list MUST contain transaction references added to the DAG since the last `Gossip` message. 
This includes transactions received from other nodes.
The list of transaction references MUST be tracked per connection.
If no new transactions have been added/received or for the first message for a connection, an empty list is sent.
The list MUST not contain more than 100 transactions.
This could create a backlog of messages at the sending node's side. 
A node SHOULD take precautions to keep this backlog to a minimum.
A node SHOULD filter out transactions received from a peer to prevent sending duplicates. If Alice sends transaction A, B and C to Bob then Bob SHOULD not send A, B and C to Alice.

The `LC` value MUST equal the highest Lamport Clock value of all transaction references included in the `XOR` calculation.
If no transactions are present, an all-zero `XOR` and `LC` of 0 is sent.

This message does not require a `conversationID`

#### 5.2.2 Transaction List Query

Upon receiving a `Gossip` message from a peer, a node MUST first compare the `XOR` value from the message with its own `XOR` value from the DAG.
If the values are equal, no further action is required.

If they are not equal, the transaction list MUST be filtered. All known transaction references MUST be removed.
The resulting list contains all Gossiped transactions the node is missing.
The node then calculates a temporary XOR value by applying TX hashes from the filtered list to its own XOR value.

If the temporary XOR value matches the XOR value of the peer, or the peer's `LC` is lower and there are new transactions references, the node SHOULD send a `TransactionListQuery` message containing the list of missing transaction references.
This message MUST also add a new `conversationID`.
In any other case the node SHOULD send a `State` message, see §6.2.1. 

#### 5.2.3 Transaction List

When a node receives a `TransactionListQuery` or `TransactionRangeQuery` message, it SHOULD respond with a `TransactionList` message.
This is a response type message, so it MUST include the request's `conversationID`.
All transactions in the `TransactionList` message MUST be sorted by LC value (lowest first).
All transactions without a `pal` header MUST be added with their payload.
All transactions with a `pal` header MUST be added without their payload. Transactions with a `pal` header are discussed in [§7](rfc017-distributed-network-grpc-v2.md#7-private-transactions).

A `TransactionList` message MAY be broken up into smaller messages to not exceed the maximum message size, each message should still conform to these rules. 
Before sending the first message, the node MUST calculate the `totalMessages` it will send and MUST include this in all messages. 
Each part MUST also include a `messageNumber` starting from 1, incrementing by 1 for every new message until all messages are sent. 

#### 5.2.4 Receiving new transactions

For every part of a `TransactionList` a node receives, it MUST confirm that the `conversationID` matches that of the request.
Transactions received in response to a `TransactionListQuery` MUST have been present in the request.
Transactions received in response to a `TransactionRangeQuery` MUST have an LC value that is within the requested range.
If any of these requirements are not met, the entire message MUST be ignored.

If a transaction can not be processed due to missing previous transaction, the node SHOULD send a `State` message. It SHOULD also stop processing any further transactions from the list.
If a transaction is received without a payload, and it does not contain a `pal` header, it MUST stop processing the message.
Any transaction until that point MAY still be added.

When `messageNumber` and`totalMessages` are the same, all parts of the `TransactionList` are received and the conversation MAY be closed.

## 6. Set reconciliation protocol

One of the problems in a distributed network is how to make sure every node has processed all transactions.
When two nodes haven't processed the same set of transactions, the second problem is how to efficiently synchronize the missing transactions.
The higher the efficiency, the more transactions the entire network can process.
This part of the protocol is used to synchronize transactions that are missed by the gossip protocol (e.g. due to nodes being offline/network partitions). 

### 6.1 Data requirements

The protocol requires an XOR value and IBLT structure to be sent in a message.
The XOR value has already been explained in §5.1.1.
These are calculated over different transaction ranges based on LC value.

#### 6.1.1 Invertible Bloom Lookup Table

The IBLT is a data structure capable of finding differences between sets.
The performance of this data structure is not impacted by the number of entries, but only by the size of the set difference.
This allows two nodes to compare the entire set of transactions with a relative small data structure.

An IBLT has several parameters that define its characteristics.
The transaction reference is used as *key*.
When comparing two IBLTs, these parameters MUST be the same.
An overview of the parameters:

* **buckets**: defines the size of the IBLT, much like the size of a bloom filter. The larger the size, the bigger the set difference can be. 
* **Hk**: each key is hashed to select the buckets the key is inserted to. The hash function is applied recursively to create extra hash values. This continues until *k* different values are found. The hash function, its seed and the value for *k* MUST all be the same for the entire network.
* **Hc**: each key is hashed to a checksum which is used to validate the inserted key during the decode phase. The hash function and its seed MUST all be the same for the entire network.
* **data types**: the data types for *count*, *val_sum* and *hash_sum* MUST match ((un)signed, byte-size).

The following parameters are used for the IBLT:

* **buckets**: 1024
* **Hc**: [murmur3](https://en.wikipedia.org/wiki/MurmurHash) 64 bit hash, seed: 0x00
* **Hk**: murmur3 32 bit hash, seed: 0x01
* **k** = 6

Computing the murmur3 hash requires conversion of integers to bytes, this MUST be done in little-endian fashion.

Per bucket the following field sizes are used:

* **val_sum**: 32 bytes
* **hash_sum**: 8 bytes
* **count**: 4 bytes

See Appendix A.1 for an explanation.

#### 6.1.2 IBLT page size

Calculating an IBLT is relatively cheap, but it grows linearly with the number of transactions. This would harm performance in larger networks.
The immutable nature of transactions make it possible to store intermediate IBLTs and use those as an optimization.
The LC range parameters of the protocol should be set to optimize use of these intermediate IBLTs.
Therefore, they should not be chosen freely.

To simplify the description of the protocol we introduce a new term: **page**.
A page contains transaction references for a range of LC values. Each page is a multiple of 512.
The first page includes transaction references with LC values between 0 (inclusive) and 512 (exclusive).
When a *next* or *previous* page is mentioned, this means 512 has to be added/removed from the `start` and `end` values of an LC range.

#### 6.1.3 IBLT decoding

Subtracting the IBLTs generated from sets A and B and decoding the resulting IBLT, as described in the original papers [1](https://arxiv.org/pdf/1101.2245), [2](https://www.ics.uci.edu/~eppstein/pubs/EppGooUye-SIGCOMM-11.pdf), produces two sets of values: those that are missing in set A and those that are missing in set B.
When sets A and B contain all the transactions on the DAG of two different nodes, IBLTs can easily identify their set difference. 

#### 6.1.4 IBLT serialization

IBLTs from different nodes can only be compared when they are generated in the same way. 
The IBLT parameters specified in §6.1.1 are constant and do not need to be communicated.
The serialized IBLT consists of all buckets appended in order, and a bucket is serialized by concatenating the bytes of *count*, *hash_sum*, and *val_sum* and MUST be in this order.
Conversion of integers to bytes MUST be in little-endian format.

### 6.2 Operation

All messages and values mentioned in this chapter are scoped to a single connection between two nodes.
*Alice* and *Bob* are used to represent two nodes connected to each other.

The protocol generally operates as follows:

1. Alice sends a State message in response to a gossip message when conditions require so (§6.2.1):
    * XOR of all known transaction references.
    * Highest Lamport Clock value over all transactions (LC).
    
2. When receiving Alice's message, Bob compares the XOR value from Alice with its own (§6.2.2):
    * When the XOR is the same and the LC value equals BOB's highest Lamport Clock value: no action required.
    * When different, send a message containing:
      * LC value sent by Alice
      * the IBLT that includes the Lamport Clock range of 0-LC(Alice)
      * Bob's highest Lamport Clock value

3. When receiving the message, Alice subtracts Bob's IBLT from her own IBLT for the given range (§6.2.3) and does one of the following:
   * If not decodable, go to 1 and send values based on the previous page, or request all transactions on lowest page if already comparing the lowest page.
   * If decodable, send a request for missing transactions if there are any.
   * If Bob's highest Lamport Clock value is higher than the LC value sent in 1, then request transactions over a range of Lamport Clock values (§6.2.4)

4. When receiving a request for transactions, Bob responds with a message including the requested transactions.

5. After adding Bob's transactions to the DAG (making sure its cryptographic signature is valid), query the payload if it's missing.

A node MUST make sure to only add transactions of which all previous transactions are present.

#### 6.2.1 Requesting a peer's State

The `State` message is sent as response to various conditions of the gossip protocol (see §5).
The `State` message MUST contain a new `conversationID`.
The `State` message contains the `XOR` value and an `LC` of the local DAG.
The `LC` value MUST equal the highest Lamport Clock value of all transaction references included in the `XOR` calculation.
If no transactions are present, an all-zero `XOR` and `LC` of 0 is sent.

#### 6.2.2 IBLT response

When a peer receives a `State` message and the `XOR` differs, it SHOULD respond with a `TransactionSet` message.
The requested `LC` value falls within the bounds of a page.
The response IBLT MUST be calculated over the transactions leading up to and including that page.
If the peer's highest Lamport Clock value is lower, it MUST use the IBLT covering the entire DAG.
Next to the IBLT data, the `TransactionSet` message MUST contain the original `LC` value as `LC_req`, a new `LC` value indicating the highest LC value from the peer.
This is a response type message, so it MUST contain the `conversationID`.

#### 6.2.3 Transaction List Query

For every `TransactionSet` message a node receives, it MUST check if the `LC_req` value matches the `LC` value from the original request. 
The `TransactionSet` message MUST also contain the `conversationID` from the original `State` message.

The IBLT in the `TransactionSet` message contains every transaction of the peer in the LC range of `0-min(LC, LC_req)`. 
The node MUST lookup/compute the IBLT for this range and subtract it from the IBLT of the peer. 
Decoding the resulting IBLT will list the transaction refs the peer has and the local node misses. 
These transactions SHOULD be queried by using the `TransactionListQuery` message (see §5.2.2). 

If the decoding fails, the local node sends a new `State` message. 
The IBLT subtraction used the range `0-min(LC, LC_req)`.
The new request MUST be for one page lower.
The `LC` value MUST be the `end` of that previous page. 
The `XOR` value MUST still be calculated over all transaction references on the local node.

#### 6.2.4 Transaction Range Query

If the IBLT from a `TransactionSet` message can be decoded but contains no missing transactions, the local node SHOULD send a `TransactionRangeQuery` message.
The peer SHOULD respond with `TransactionList` message, containing the requested transactions. The peer SHOULD make sure to send all transactions that were requested.

What range should be requested depends on the local node's LC, and `LC_req` and `LC` from the `TransactionSet` message.
If the page containing `LC` comes after the page containing `LC_req`, the peer has additional transactions outside the LC range covered by the IBLT.
If the `LC_req` value is in the latest page of the local node, it SHOULD query all pages after `LC_req` leading up to and including the page containing the `LC` value.
If the `LC_req` value is NOT in the latest page of the local node, then the query MUST only cover the next page.
This last requirement prevents a node from querying the entire DAG while only some historic transactions are missing or when a page contains collisions.

The `TransactionRangeQuery` message contains a `start` (inclusive) and `end` (exclusive) parameter corresponding to the Lamport Clock values of the requested page(s).
The message MUST contain a (new) `conversationID`.

If decoding fails and the IBLT covered the first page, the local node SHOULD use a `TransactionRangeQuery` message to query the first page.


## 7. Private Transactions

When the node receives a transaction that contains a `pal` header, the transaction is considered private.
This means the transaction payload can only be retrieved by the participants listed in the `pal` header.
Transaction payload SHALL NOT be shared with other parties than listed in the `pal` header.
Since the `pal` header is encrypted (see [RFC004](rfc004-verifiable-transactional-graph.md)) to preserve anonymity of the participants,
it must be decrypted first using the node's `keyAgreement` keys from its DID document.
If the local node can decrypt the `pal` header, it means the transaction is (also) intended for the local node.
See [RFC004](rfc004-verifiable-transactional-graph.md) for more information on how to encrypt/decrypt the `pal` header.

To retrieve the payload of the private transaction, the node MUST send a `TransactionPayloadQuery` to one (or more) of the nodes listed in the `pal` header.
It COULD decide to broadcast the message to all nodes in the `pal` header (except the local node), because not all nodes (in the `pal` header) might have the payload yet (or never will).

When a `TransactionPayloadQuery` for a private transaction is received, the node MUST decrypt its `pal` header and verify that the requesting peer is listed as participant.

Both `TransactionPayloadQuery` and its success response (`TransactionPayload` with the transaction payload) MUST only be sent over an authenticated connection.

The local node SHOULD respond with an empty `TransactionPayload` message (for a private transaction) in the following situations:

- When the connection is unauthenticated.
- When the requesting party is not listed in the `pal` header.
- When the node doesn't have the transaction payload.

In an empty `TransactionPayload` response message the transaction reference MUST be set. The payload field MUST be left unset.

See Appendix A.3 for the reasoning behind the empty `TransactionPayload` response.

## 8. Diagnostics

To provide insight into the state of the network, and the DAG for informational purposes and to aid analysis of anomalies,
nodes SHOULD broadcast diagnostic information to its peers using the `Diagnostics` message.
If broadcasting, the node MUST do this at least every minute, but it MUST NOT broadcast more often than every 5 seconds \(to avoid producing too much chatter\).
A node MAY choose not to include any of the specified fields.

See the [protobuf definition](https://raw.githubusercontent.com/nuts-foundation/nuts-node/master/network/transport/v2/protocol.proto) for the fields included in the `Diagnostics` message.

## 9. Errors

Internal errors that occur in the node during message handling MUST NOT be returned to the peers. 
Only the following custom errors may be returned by the local node:

* `internal error`: the node encountered an internal error during message handling.
* `message not supported`: the node does not support the message type contained in the envelope. Indicates a protocol implementation incompatibility between the node and the peer.

See Appendix A.4 for the reasoning behind not disclosing internal errors.

## 10. Security Considerations

This section describes anticipated attacks vectors and \(non-malicious\) situations that may threaten a node, and how they're mitigated.

When a peer performs an action which is identified as a threat, nodes SHOULD immediately close the connection and inform the peer of the rule that was violated. That way the operator of the peer can identify what should be fixed on their node. However, when offences are repeated the node SHOULD apply "Three Strikes Out"; after 3 violations the node SHOULD deny further connections using the peer's certificate \(identified by certificate issuer DN and serial number\), until the ban is lifted by an operator.

### 10.1 Denial of Service

#### Threat: Memory Exhaustion due to Large Messages

A \(malicious\) peer could exhaust the node's memory with \(many\) large network messages.

Countermeasures:

* Nodes MUST NOT accept incoming network messages larger than 512 kilobytes.
* Nodes MUST NOT send network messages larger than 512 kilobytes.

#### Threat: Resource Exhaustion through Connection Flooding

Multiple peers might share the same IP address and certificate in clustering or cloud environments. However, it can also be used by attackers to trying to flood the node with a very large number of connections exhausting resources like file descriptors, connection pools or thread pools.

#### Threat: Resource Exhaustion through Message Flooding

A peer that floods the node with many messages threatens the stability of a node, and the network in general: resources for processing incoming messages often have hard limits in the form connection pools, thread pools or backlogs.

#### Threat: Uncontrolled DAG Growth

A single peer or orchestrated group of peers can quickly produce many transactions, quickly growing the DAG. Since history must be retained to verify the DAG integrity in the future, it's desirable to limit the number of faulty transactions or transactions without a meaningful content.

### 10.2 Data Manipulation

#### Threat: Manipulating Transaction Content

By altering a transaction's content when responding to a payload query an attacker can hamper nodes or even steal identities \(e.g. DIDs\).

Countermeasures:

* Nodes MUST verify the payload hash when receiving transaction content as specified by [RFC004 section 3.6 \(Signature and transaction content verification\)](rfc004-verifiable-transactional-graph.md)

## Appendix A: Design decisions

### A.1 IBLT parameters

For a small network, 1024 buckets and 6 hashes is quite heavy. 
For a nation-wide network this would be reasonable.
These parameters are chosen to reduce the chance of a collision: two different transactions that are hashed into the exact same buckets.
If there is a collision in the _set difference_, it is not be possible to deconstruct an IBLT.
In the protocol described here, this would mean that an entire range of transactions would have to be sent.
When this happens randomly, it wouldn't be such a problem, but it's also an attack vector.
An attacker could craft pairs of transactions that would trigger other nodes into requesting large amounts of transactions. 
This type of attack is called the [birthday attack](https://en.wikipedia.org/wiki/Birthday_attack).
The only way to protect against this type of attack is to make the set large enough that the cost to the attacker would become to high.
For an IBLT with 64 buckets and 4 hashes, the chance for a collision is 50% after adding ~5000 keys.
This would be trivial for current computers.
Expanding this to 1024 buckets and 6 hashes, reaching a 50% chance for a collision is only achieved after adding more than a billion keys.
This is still not much, but the keys used for the collision have to be created by real signed transactions.
Even on modern hardware, this could take an hour.
Even if an attacker manages to insert colliding transactions into the DAG of a node, that node will then use the gossip protocol to synchronize those transactions with other nodes. The impact of this type of attack is scoped to the communication of the attacker and a single node. 

### A.2 The right message for the right situation

The protocol uses a variety of message to synchronize all transactions over nodes.
Together they cover all situations.

If all nodes are operating correctly, the gossip protocol is the most efficient in synchronizing all the transactions.
It will not be able to deal with situations where a node has been offline or when nodes can no longer reach each other.
It can be used frequently because it uses small messages.

If a node has been offline for some time, the combination of an IBLT and a range query will synchronize the node efficiently.
The IBLT will resolve all missing transactions for the node's latest page and the range query will synchronize all the missing pages. Those pages will include all transactions that have been created during the period the node was offline.

If a node has not been offline, but its communication has been interrupted, it might have created new transactions that aren't synchronized with the rest of the network.
The Lamport Clock values will overlap with the transactions of the rest of the network.
The IBLT will be able to determine missing transactions. 
If the difference in transactions is too big, the protocol will reduce the range the IBLT covers until an IBLT has been found that can resolve the transaction set difference.

### A.3 Unfulfillable `TransactionPayloadRequest` responses

When a node receives a payload query (for a private transaction) it can't fulfill (e.g. it doesn't have the payload), or mustn't fulfill (e.g. requesting party is not a participant),
the response must be the same regardless the reason. This can be either an empty `TransactionPayloadResponse` message (with only the reference set) or simply no response at all.
However, the node must take care to use the same type of response at all times (so either empty responses, or no response).
This way, attackers can't derive information about the kind of response they receive.
E.g., they can't determine whether the node does not have the transaction payload, or that the attacker isn't allowed to request the transaction payload.

### A.4 Preventing internal errors disclosing internal state

Internal errors (e.g. disk is full, file does not exist, private key not found, out of memory) can provide attackers with information about the internal state of the node,
which can then be used to execute an attack. As such, the node must take care to not disclose internal errors to the attacker.
If such an error occurs, the node must log the error for analysis by an operator and send a generic error message back to the client.

### A.5 Best effort message sending

A lot of the response type messages only state a node SHOULD send a response or a followup and not a MUST. This is because a node might have a reason to stop/pause processing.
A node might be to busy doing something else, or it might be doing a migration or backup. Because nodes communicate with multiple nodes, a single busy node should not matter that much.
If all nodes send at best-effort, that will be enough due to the number of nodes available.

## X. Service Discovery

Although a (or multiple) trusted bootstrap node is required for initially connecting to the network, nodes SHOULD discover new peers by searching for `NutsComm` endpoints in the received DID documents.
This SHOULD be done after the initial DAG sync is completed, to avoid connecting to non-existing endpoints resolved from older versions of DID documents.

TODO: What if a node were to publish a competitor's NutsComm endpoint in its DID document, causing implementations to ignore the competitor's endpoint (because the malicious node's DID document is processed first)
