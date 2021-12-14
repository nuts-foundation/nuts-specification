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
 
* **DAG** \(Directed Acyclic Graph\): graph formed of all transactions that. It provides casual ordering for transactions and means to efficiently compare the local DAG with those of peers.
* **Invertible Bloom Lookup Table (IBLT)**: A bloom filter where instead of just a 0 or 1, a count, key_sum and hash_sum is stored per bucket. Original paper: https://arxiv.org/pdf/1101.2245 or https://www.ics.uci.edu/~eppstein/pubs/EppGooUye-SIGCOMM-11.pdf.
* **Lamport Clock (LC)**: Monotonically increasing clock as defined in [RFC016](rfc016-lamport-clock.md).
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
If a node has a node DID and wishes create an authenticated connection, it MUST send the DID as `nodeDID` gRPC header when establishing inbound or outbound connections.
If authentication fails, the authenticating side MUST close the connection with the following error: `nodeDID authentication failed`.
The received node DID MUST be authenticated upon receiving, to directly inform the peer should its configuration be incorrect. 

As specified by RFC015, the node MUST authenticate the peer's node DID as follows:

1. Resolve peer's node DID to its corresponding DID document.
2. Assert that one of the `NutsComm` endpoint's host matches (one of) the `dNSName` SANs in the peer's TLS client certificate.

## 4. Set reconciliation protocol

### 4.1 Data requirements

The protocol requires an XOR value and IBLT structure to be sent in a message. 
These are to be calculated over different transaction ranges based on LC value.
Both data structures have associative properties allowing for optimized data storage.

#### 4.1.1 XOR

The exclusive-or (XOR) value is calculated over all transactions reference hashes within a range.
The length of the XOR value is therefore the same as the transaction reference length (SHA256: 32 bytes).
The XOR value is used to quickly determine if two nodes have processed the exact same set of transactions.

#### 4.1.2 Invertible Bloom Lookup Table

The IBLT is a data structure capable of finding differences between sets.
The performance of this data structure is not impacted by the number of entries, but only by the size of the set difference.
This allows two nodes to compare the entire set of transactions with a relative small data structure.

An IBLT has several parameters that define its characteristics. When comparing two IBLTs, these parameters MUST be the same.
An overview of the parameters:

* **buckets**: defines the size of the IBLT, much like the size of a bloom filter. The larger the size, the bigger the set difference can be. 
* **Hk**: each key is hashed to select the buckets the key is inserted to. The hash function is applied on the previous output to create extra hash values. This continues until k different values are found. The hash function, its seed and the value for k MUST all be the same for the entire network.
* **Hc**: each key is hashed to a checksum which is used to validate the inserted key during the deconstruction phase. 
* **data types**: the data types for *count*, *val_sum* and *hash_sum* MUST match ((un)signed, byte-size).

The following parameters are used:

* **buckets**: 1024
* **Hc**: murmur3, 4 bytes, seed: 0x00
* **Hk**: murmur3, 4 bytes, seed: 0x01, k = 5 
* **key sum size**: 32 bytes
* **bucket count field**: 4 bytes

### 4.1.3 IBLT page size

Calculating an IBLT is relatively cheap, but it grows linearly with the number of transactions. This would harm performance of larger networks.
The immutable nature of transactions make it possible to store intermediate IBLTs and use those as an optimization.
The LC range parameters of the protocol should be adjusted accordingly to optimize use of these intermediate IBLTs.
Therefore, they should not be chosen freely.

Any LC value used in the protocol MUST be a multiple of 512 (minus 1, because the LC is 0 based). 
The use of inclusive or exclusive range parameters must make sure that `end-start` is a multiple of 512.
In the context of an IBLT this size will be referred to as the `IBLT page size`.

### 4.2 Operation

All messages and values mentioned in this chapter are scoped to a single connection between two nodes.

The protocol generally operates as follows:

1. The local node broadcasts at a set interval:
    * XOR of all known transaction references.
    * Highest Lamport Clock value over all transactions (LC).
    
2. When receiving a peer's broadcast, compare the XOR value:
    * When the XOR is the same: no action required.
    * When different, send a message containing
      * If the LC is incompatible with its own registration: no action required.
      * the IBLT that includes the Lamport Clock range of 0-LC
      * the peer LC value
      * the own highest Lamport Clock value

3. When receiving the message, the local node subtracts the given IBLT from its own IBLT for the given range:
   * If not deconstructable, go to 1 and send values based on a smaller Lamport Clock value.
   * If deconstructable, send a request for missing transactions.
   * If the peers highest Lamport Clock value is higher than the LC value sent in 1, then request transactions by Lamport Clock value:
     * Lamport Clock start value.
     * Lamport Clock end value.

4. When receiving a request for transactions, respond with a message including the requested transactions.

5. After adding a peer's transaction to the DAG \(making sure its cryptographic signature is valid\), query the payload if it's missing.

The node MUST make sure to only add transactions of which all previous transactions are present. The node MUST also add the transaction atomically.
If one transaction is invalid, ignore the entire message.

The sections below specify the details of the protocol operation and maps it to the gRPC messages. See the section "Protobuf Definition" for a full specification of the protobuf/gRPC contract.

### 4.2.1 Broadcasting

The local node's DAG state MUST be broadcast at an interval using the `AdvertState` message, by default every 2 seconds. The interval MAY be adjusted by the node operator but MUST conform to the limits \(min/max interval\) defined by the network. It is advised to keep it relatively short because it directly influences the speed by which new transactions are retrieved.

The `AdvertState` message contains a `XOR` value, a `LC` value and a `token`. The `token` is scoped to a connection and is required in the response from the peer.
A node is free to choose how the `token` is created as long as it's able to satisfy the requirements of the next paragraph.
It MUST be valid for at least 10 seconds.

### 4.2.2 IBLT response

When a peer receives an `AdvertState` message and the `XOR` differs, it SHOULD respond with a `TransactionSet` message.
The upper bound of the response IBLT MUST be the equal to `512*ceil(LC/512)` where `LC` is the value from the `AdvertState` message and `ceil` is a function that rounds a double up to the nearest integer.
This upper bound is exclusive, the lower bound is inclusive and always 0.
Next to the IBLT data, the `TransactionSet` message contains the original `LC` value as `LC_req`, a new `LC` value indicating the highest LC value the peer has and the `token` value from the `AdvertState` message.

### 4.2.3 Transaction List Query

For every `TransactionSet` message a node receives, it MUST check the `token` value in the message and compare it to its own administration.
If the `token` doesn't exist or if the `LC_req` value doesn't match, the node MUST ignore the message.

The IBLT in the `TransactionSet` message contains every transaction of the peer in the LC range of `0-LC_req`. The node MUST lookup/compute the IBLT for this range and subtract it from the IBLT of the peer. Deconstruction of the resulting IBLT will list the transaction refs the peer has and the local node misses. 

These transactions MUST be queried by using the `TransactionListQuery` message. It contains all missing transaction references. An IBLT deconstruction with the current parameters will not be able to yield anything larger than ~750 values. This will easily fit into a single message given the current maximum message size.
The message MUST contain a `token` that is to be used in the response.

If the deconstruction fails, the local node sends a new `AdvertState` message. The `LC` value MUST be `512*floor((LC_req-1)/512)` where `floor` is a function that rounds a double down to the nearest integer. 
The `XOR` value MUST be calculated over the transaction references in the range starting with 0 and ending with the new `LC` value.

### 4.2.4 Transaction Range Query

If the IBLT from a `TransactionSet` message could be deconstructed, the local node MAY also send a `TransactionRangeQuery` message.
This is only needed if the `LC` value of the `TransactionSet` message is equal or larger than `512*ceil(LC_req/512)`. 
This means the peer has additional transactions outside the IBLT range.
The `TransactionRangeQuery` message contains a `begin` and `end` parameter indicating a request for transactions which LC value is between those parameters.
The `begin` MUST be set to `512*ceil(LC_req/512)` and the end MUST be set to `LC`. Both are inclusive.
The message MUST contain a `token` that is to be used in the response.

### 4.2.5 Transaction List

The `TransactionList` message contains a list of transactions and the `token` from either the `TransactionListQuery` or `TransactionRangeQuery`.
Each transaction in the message MUST relate to the original request. Transactions that resulted from a `TransactionListQuery` MUST have been present in that message.
Transactions that resulted from a `TransactionRangeQuery` MUST have an LC value that is within the requested range.
If this is not true, the entire message MUST be ignored.
A `TransactionList` message MAY be broken up into smaller messages, each message should confirm to these rules. Each part MUST also use the same `token`.
All transactions in the `TransactionList` message MUST be sorted by LC value (lowest first).

## X. Private Transactions

## Appendix A: Design decisions

### A.1 IBLT parameters

For a small network, 1024 buckets and 5 hashes is quite heavy. 
For a nation-wide network this would be reasonable.
The reason for these parameters is the chance of a collision: a different transaction that results in the exact same buckets as another transaction.
If there's a collision, it would not be possible to deconstruct an IBLT since there'll be too many buckets that still have a count of 2.
In the protocol this would mean an entire range of transactions would have to be sent.
When this happens randomly, it wouldn't be such a problem, but it's also an attack vector.
An attacker could craft pairs of transactions that would trigger other nodes into requesting large amounts of transactions. 
This type of attack is called the [birthday attack](https://en.wikipedia.org/wiki/Birthday_attack).
The only way to protect against this type of attack is to make the set large enough that the cost to the attacker would become to big.
For an IBLT with 64 buckets and 4 hashes, the change for a collision is 50% after adding ~5000 keys.
This would be trivial for current computers.
Expanding this to 1024 buckets and 5 hashes, reaching a 50% chance for a collision is only achieved after adding more than 40 million keys.
This is still not much, but the keys used for the collision have to be created by real signed transactions.
Even on modern hardware, 40 million signing operations would take minutes. A 6th hash can be added later to require more than 1 billion keys for a 50% chance of collision.
