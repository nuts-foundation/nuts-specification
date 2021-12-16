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

This document is currently a draft.

### Copyright Notice

![](../.gitbook/assets/license.png)

This document is released under the [Attribution-ShareAlike 4.0 International \(CC BY-SA 4.0\) license](https://creativecommons.org/licenses/by-sa/4.0/).

## 1.  Introduction


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
If authentication fails, the authenticating side MUST close the connection with the following error: `nodeDID authentication failed`.
The received node DID MUST be authenticated upon receiving, to directly inform the peer should its configuration be incorrect. 

As specified by RFC015, the node MUST authenticate the peer's node DID as follows:

1. Resolve peer's node DID to its corresponding DID document.
2. Assert that one of the `NutsComm` endpoint's host matches (one of) the `dNSName` SANs in the peer's TLS client certificate.

## 4. Set reconciliation protocol

One of the problems in a distributed network is how to make sure every node has processed all transactions.
When two nodes haven't processed the same set of transactions, the second problem is how to efficiently synchronize the missing transactions.
The higher the efficiency, the more transactions the entire network can process.
The protocol described here is a compromise on efficiency and fairness. It's able to quickly determine a set of missing transactions, but also doesn't favour any node.


### 4.1 Data requirements

The protocol requires an XOR value and IBLT structure to be sent in a message. 
These are to be calculated over different transaction ranges based on LC value.

#### 4.1.1 XOR

The exclusive-or (XOR) value is calculated over all transaction references within a range.
The size of the XOR value is therefore the same as the transaction reference size (SHA256: 32 bytes).
The XOR value is used to quickly determine if two nodes have processed the exact same set of transactions.

#### 4.1.2 Invertible Bloom Lookup Table

The IBLT is a data structure capable of finding differences between sets.
The performance of this data structure is not impacted by the number of entries, but only by the size of the set difference.
This allows two nodes to compare the entire set of transactions with a relative small data structure.

An IBLT has several parameters that define its characteristics.
The transaction reference is used as *key*.
When comparing two IBLTs, these parameters MUST be the same.
An overview of the parameters:

* **buckets**: defines the size of the IBLT, much like the size of a bloom filter. The larger the size, the bigger the set difference can be. 
* **Hk**: each key is hashed to select the buckets the key is inserted to. The hash function is applied recursively to create extra hash values. This continues until k different values are found. The hash function, its seed and the value for `k` MUST all be the same for the entire network.
* **Hc**: each key is hashed to a checksum which is used to validate the inserted key during the decode phase. The hash function and its seed MUST all be the same for the entire network.
* **data types**: the data types for *count*, *val_sum* and *hash_sum* MUST match ((un)signed, byte-size).

The following parameters are used:

* **buckets**: 1024
* **Hc**: [murmur3](https://en.wikipedia.org/wiki/MurmurHash), 8 bytes, seed: 0x00
* **Hk**: murmur3, 4 bytes, seed: 0x01, k = 6 
* **val_sum field size**: 32 bytes
* **count field size**: 4 bytes

See Appendix A.1 for an explanation.

#### 4.1.3 IBLT page size

Calculating an IBLT is relatively cheap, but it grows linearly with the number of transactions. This would harm performance of larger networks.
The immutable nature of transactions make it possible to store intermediate IBLTs and use those as an optimization.
The LC range parameters of the protocol should be adjusted accordingly to optimize use of these intermediate IBLTs.
Therefore, they should not be chosen freely.

To simplify the description of the protocol we introduce a new term: **page**.
A page contains transaction references for a range of LC values. Each page is a multiple of 512.
The first page includes transactions with LC values between 0 (inclusive) and 512 (exclusive).
When a *next* or *previous* page is mentioned, this means 512 has to be added/reduced from the `begin` and `end` values of an LC range.

#### 4.1.4 IBLT decoding

When decoding an IBLT as described in [the original paper](https://arxiv.org/pdf/1101.2245 or https://www.ics.uci.edu/~eppstein/pubs/EppGooUye-SIGCOMM-11.pdf), the result is two sets of values: those that are missing in set A and those that are missing in set B. For this protocol only one of those sets is of interest, the other can be ignored.
This also makes it more likely the IBLT can be decoded.

### 4.2 Operation

All messages and values mentioned in this chapter are scoped to a single connection between two nodes.
*Alice* and *Bob* are used to represent two nodes connected to each other.

The protocol generally operates as follows:

1. Alice broadcasts at a set interval (§4.2.1):
    * XOR of all known transaction references.
    * Highest Lamport Clock value over all transactions (LC).
    
2. When receiving Alice's broadcast, Bob compares the XOR value from Alice with its own (§4.2.2):
    * When the XOR is the same and the LC value equals BOB's highest Lamport Clock value: no action required.
    * When different, send a message containing:
      * LC value sent by Alice
      * the IBLT that includes the Lamport Clock range of 0-LC(Alice)
      * Bob's highest Lamport Clock value

3. When receiving the message, Alice subtracts Bob's IBLT from her own IBLT for the given range (§4.2.3):
   * If not decodable, go to 1 and send values based on the previous page.
   * If decodable, send a request for missing transactions
   * If Bob's highest Lamport Clock value is higher than the LC value sent in 1, then request transactions by Lamport Clock value:
     * Lamport Clock start value.
     * Lamport Clock end value.

4. When receiving a request for transactions, Bob responds with a message including the requested transactions (§4.2.4 and §4.2.5).

5. After adding Bob's transactions to the DAG \(making sure its cryptographic signature is valid\), query the payload if it's missing.

A node MUST make sure to only add transactions of which all previous transactions are present.

The sections below specify the details of the protocol operation and maps it to the gRPC messages. See the section "Protobuf Definition" for a full specification of the protobuf/gRPC contract.

#### 4.2.1 Broadcasting the state

todo:
The local node's DAG state MUST be broadcast at an interval using the `State` message, by default every 2 seconds. The interval MAY be adjusted by the node operator but MUST conform to the limits \(min/max interval\) defined by the network. It is advised to keep it relatively short because it directly influences the speed by which new transactions are retrieved.

The `State` message contains a `XOR` value, an `LC` value and a `token`. The `token` is scoped to a connection and is required in the response from the peer.
The `LC` value MUST equals the highest Lamport Clock value of all transaction references included in the `XOR` calculation.
If no transactions are present, an all-zero `XOR` and `LC` of 0 is sent.
A node is free to choose how the `token` is created as long as it's able to satisfy the requirements of §4.2.3.
It MUST be valid for at least 10 seconds.

#### 4.2.2 IBLT response

When a peer receives an `State` message and the `XOR` differs, it SHOULD respond with a `TransactionSet` message.
The sent `LC` value falls within the bounds of a page.
The response IBLT MUST be calculated over the transactions leading up to and including that page.
Next to the IBLT data, the `TransactionSet` message MUST contain the original `LC` value as `LC_req`, a new `LC` value indicating the highest LC value from the peer and the `token` value from the `State` message.

#### 4.2.3 Transaction List Query

For every `TransactionSet` message a node receives, it MUST check the `token` value in the message and compare it to its own administration.
If the `token` doesn't exist or if the `LC_req` value doesn't match, the node MUST ignore the message.

The IBLT in the `TransactionSet` message contains every transaction of the peer in the LC range of `0-LC_req`. The node MUST lookup/compute the IBLT for this range and subtract it from the IBLT of the peer. Decoding the resulting IBLT will list the transaction refs the peer has and the local node misses. 

These transactions MUST be queried by using the `TransactionListQuery` message. It contains all missing transaction references.
The message MUST contain a `token` that is to be used in the response.

If the decoding fails, the local node sends a new `State` message. The `LC` value MUST be the `end` of the page before the page that included `LC_req`. 
The `XOR` value MUST be calculated over the transaction references in the range starting with 0 and ending with the new `LC` value.

#### 4.2.4 Transaction Range Query

If the IBLT from a `TransactionSet` message could be decoded, the local node MAY also send a `TransactionRangeQuery` message.
This is only needed if the page containing the `LC` value of the `TransactionSet` message comes after the page containing `LC_req`. 
This means the peer has additional transactions outside the IBLT range.
If the `LC_req` value is in the latest page of the local node, it SHOULD query all pages leading up to the `LC` value.
If the `LC_req` value is NOT in the latest page of the local node, then it MUST only query the next page.

The `TransactionRangeQuery` message contains a `start` (inclusive) and `end` (exclusive) parameter corresponding to that requested page(s).
The message MUST contain a `token` that is to be used in the response.

If decoding fails and the IBLT covered the first page, the local node MUST query the first page.

#### 4.2.5 Transaction List

The `TransactionList` message contains a list of transactions and the `token` from either the `TransactionListQuery` or `TransactionRangeQuery`.
Each transaction in the message MUST relate to the original request. Transactions that resulted from a `TransactionListQuery` MUST have been present in that message.
Transactions that resulted from a `TransactionRangeQuery` MUST have an LC value that is within the requested range.
If this is not true, the entire message MUST be ignored.
A `TransactionList` message MAY be broken up into smaller messages, each message should confirm to these rules. Each part MUST also use the same `token`.
All transactions in the `TransactionList` message MUST be sorted by LC value (lowest first).

## 5 Gossip

coming

## X. Private Transactions

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

### A.2 The right message for the right situation

The protocol uses a variety of message to synchronize all transactions over nodes.
Together they cover all known situations.

If all nodes are operating correctly, the gossip protocol is the most efficient in synchronizing all the transactions.
It will not be able to deal with situations where a node is offline or when nodes can no longer reach each other.
It can be used frequent and uses small messages.

If a node has been offline for some time, the combination of an IBLT and a range query will synchronize the node efficiently.
The IBLT will resolve all missing transactions for the node's latest page and the range query will sycnhronize all the missing pages. Those pages will include all transactions that have been created during the period the nodes was offline.

If a node has not been offline, but its communication has been interrupted, it will have created new transactions.
The Lamport Clock values will overlap with the transactions of the rest of the network.
The IBLT will be able to determine missing transactions. 
If the difference in transactions is to big, the protocol will reduce the range the IBLT covers until an IBLT has been found that can resolve the transaction set difference.
