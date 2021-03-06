# RFC004 Verifiable Transactional Graph

|  |  |
| :--- | :--- |
| Nuts foundation | R.G. Krul |
| Request for Comments: 004 | Nedap |
|  | September 2020 |

## Verifiable Transactional Graph

### Abstract

This RFC describes an interoperable, content agnostic data format for distributed networks which provides cryptographic authentication and verification of its contents and guaranteed ordering using a Directed Acyclic Graph \(DAG\).

### Status of document

This document is currently a draft.

### Copyright Notice

![](../.gitbook/assets/license.png)

This document is released under the [Attribution-ShareAlike 4.0 International \(CC BY-SA 4.0\) license](https://creativecommons.org/licenses/by-sa/4.0/).

## 1.  Introduction

Because Nuts is a decentralized network there needs to be a standardized format for structuring public information \(like where to find another care organization's data\) in a way that allows each party to independently verify the authenticity and integrity of that information. This document proposes such a format. Systems can use the format to build their application on top of it making it suitable for publishing on a distributed network.

This document does not define how to transport data to other participants in a distributed network.

## 2. Terminology

* **Application data**: data produced and consumed by the application that uses the formats described by this RFC.
* **Transaction**: node on the graph consisting of application data and metadata like signatures and references to other transactions published on a distributed network.
* **Root transaction**: the first transaction published on a network.
* **JWS payload**: The payload as defined by [RFC7515](https://tools.ietf.org/html/rfc7515#section-3).
* **Transaction contents**: The actual data in the transaction.
* **Head**: Transaction that is not referenced by another (newer) transaction.

Other terminology comes from the [Nuts Start Architecture](rfc001-nuts-start-architecture.md#nuts-start-architecture).

## 3. Transaction Format

Transactions MUST be encoded as [RFC7515 JSON Web Signature](https://tools.ietf.org/html/rfc7515). It MUST be serialized in compact \([RFC7515 section 7.1](https://tools.ietf.org/html/rfc7515#section-7.1)\) serialization form.

### 3.1. JWS implementation

In addition to required header parameters as specified in RFC7515 the following requirements apply:

* The signing key is indicated by **kid** or **jwk**. One of them MUST be present, but not both. If **kid** is present the key must be known and looked up locally. Otherwise the key must be taken from the **jwk** header.
* **alg**: MUST be one of the following algorithms: `PS256`, `PS384`, `PS512`, `ES256`, `ES384` or `ES512`.

  other algorithms SHALL NOT be used.

* **cty**: MUST contain the type of the transaction content indicating how to interpret it.
* **crit** MUST contain the **sigt**, **ver** and **prevs** headers.

The **jku**, **x5c** and **x5u** header parameters SHOULD NOT be used and MUST be ignored by when processing the transaction.

In addition to the registered header parameters, the following headers MUST be present as protected headers:

* **sigt**: \(signing time\) MUST contain the signing time of the transaction as Unix time since epoch encoded as NumericValue.
* **ver**: MUST contain the format version of the transaction as number. For this version of the format the version MUST be 1.
* **prevs**: \(previous transactions\) MUST contain the references \(see section 3.2\) of the preceding transactions \(see section 3.4\).

  When it's a root transaction the field SHALL NOT have any entries.

To aid performance of validating the DAG the JWS SHALL NOT contain the actual application data of the transaction. Instead, the JWS payload MUST contain the SHA-256 hash of the contents encoded as hexadecimal, lower case string, e.g.: `386b20eeae8120f1cd68c354f7114f43149f5a7448103372995302e3b379632a`

The contents then MAY be stored next to or apart from the transaction itself \(but that's out of scope for this RFC\).

There SHOULD be only 1 signature on the JWS. If there are multiple signatures all signatures except the first one MUST be ignored.

### 3.2. Transaction Reference

The transaction reference uniquely identifies a transaction and is used to refer to it. It MUST be calculated by taking the bytes of the JWS EXACTLY as received and hashing it using SHA-256.

When serializing a reference to string form it MUST be hexadecimal encoded and SHOULD be lowercase, e.g.: `386b20eeae8120f1cd68c354f7114f43149f5a7448103372995302e3b379632a`

### 3.3. Ordering, branching and merging

Transactions MUST form a rooted DAG \(Directed Acyclic Graph\) by referring to the previous transaction. This MAY be used to establish _casual ordering_, e.g. registration of a care organization as child object of a vendor. A new transaction MUST be appended to the end of the DAG by referring to the last transaction of the DAG \(_head_\) by including its reference in the **prevs** field.

All transactions referred to by `prev` MUST be present, since failing to do so would corrupt the DAG.

As the name implies the DAG MUST be acyclic, transactions that introduce a cycle are invalid MUST be ignored. ANY following transaction that refers to the invalid transaction \(direct or indirect\) MUST be ignored as well.

Since it takes some time for the transactions to be synced to all network peers \(eventual consistency\) there COULD be multiple transactions referring to the previous transactions in the **prevs** field, a phenomenon called _branching_. Since branches \(especially old and/or long ones\) may cause transactions to be reordered which hurts performance they MUST be merged as soon as possible. Branches are merged by specifying their heads in the **prevs** field:

![DAG structure](../.gitbook/assets/rfc004-branching.svg)

The first transaction in the DAG is the _root transaction_ and SHALL NOT have any **prevs** entries. There MUST only be one root transaction for a network and subsequent root transactions MUST be ignored.

When processing a DAG the system MUST start at the root transaction and work its way to the head\(s\) processing subsequent transactions. When encountering a branch the transactions on all branches from that point up until the merge MUST be processed before processing the merge itself. Since ordering is casual processing order over parallel branches isn't important. When looking at the diagram above, the following processing orders are valid:

* `A -> B -> C -> D -> E -> F` \(mixed parallel order\)
* `A -> B -> D -> C -> E -> F` \(branch D first, then branch C\)
* `A -> B -> C -> E -> D -> F` \(branch C first, then branch D\)

The following orders are invalid:

* `A -> B -> C -> E -> F -> D` \(merger processed before all previous were processed\)
* `A -> B -> D -> F -> C -> E` \(merger processed before all previous were processed\)
* `A -> B -> D -> E -> C -> F` \(branch C is processed out-of-order\)

### 3.4. Updates & Ordering

Since transactions are immutable, the only way to update the application data they contain is by creating a new transaction. All updates are considered as full updates, partial updates are not supported. Parallel updates of the same application data, in the form of branches, MUST be processed the same way by all nodes.

Consider the DAG from the previous chapter. If transaction `C` and `D` where to update the same application data, nodes could process the transaction in a different order. This would create an inconsistency in the network. To fix this the following rules MUST be taken into account:

* If the transaction content is equal, process as normal. (This can be determined by the JWS payload being the same)
* If the transaction content is not equal, create a representation that is a merger of the transaction content according to rules for the specific type.

When updates are required for a type of transaction content, it MUST be composed of [conflict-free replicated data types](https://en.wikipedia.org/wiki/Conflict-free_replicated_data_type)

### 3.5. Processing the DAG

Processing the DAG can be seen as planning tasks required for construction: some tasks can happen in parallel \(laying floors and installing electricity\), some tasks must happen sequentially \(foundation must be poured before building the walls\). This is the same for transactions on the DAG: transactions on a branch MUST be processed sequentially but processing order of parallel branches is unspecified, and before processing a merging transaction all **prevs** \(branches\) must have been processed.

An algorithm that COULD be used is _Breadth-First-Search_. However, with branches with more than 1 transaction this algorithm processes the merging transaction before all preceding transactions were processed. This CAN be solved by adding an extra check that doesn't process a transaction which's previous transactions haven't all been processed. When all the merge transaction's previous transactions have been processed, it will be re-added to the queue for processing. The pseudo code for this algorithm looks as follows:

```text
given FIFO queue
add root transaction to queue
until queue empty; take transaction from queue
    if current transaction already visited
        continue loop

    if any of the previous (prevs) transactions is not yet processed
        continue loop

    for all next transactions of the current transaction
        add next transaction to queue

    process transaction
```

### 3.6. Signature and transaction content verification

Before interpreting a transaction's content the JWS' signature MUST be validated. When **kid** is used to specify the signing key and the system knows additional usage restrictions \(e.g. the key is valid from X to Y, to be checked against **sigt**\), the system MUST assert the usage is compliant.

Furthermore, since the transaction content is detached from the transaction itself and referred to by hash, the transaction content MUST be hashed and compared to the hash specified in the transaction, to assert that the retrieved transaction content is actually the expected transaction content.

## 4. Example

```text
eyJhbGciOiJFUzI1NiIsImN0eSI6ImZvby9iYXIiLCJjcml0IjpbInNpZ3QiLCJ2ZXIiLCJwcmV2cyIsImp3ayJdLCJqd2siOnsia3R5IjoiRUMiLCJjcnYiOiJQLTI1NiIsIngiOiJUTXVzeXNWQTJJcHduNnZFMjhNWUQtOGtPZFN6ajZVTy1MeGE0ZWhLd0d3IiwieSI6IjdZbC1hb2ZPOC1qNHN6aVBYeGREdVVVSXdDSHlaeWtnTTJmdWlISEQxUzgifSwicHJldnMiOlsiMzk3MmRjOTc0NGY2NDk5ZjBmOWIyZGJmNzY2OTZmMmFlN2FkOGFmOWIyM2RkZTY2ZDZhZjg2YzlkZmIzNjk4NiIsImIzZjJjM2MzOTZkYTFhOTQ5ZDIxNGU0YzJmZTBmYzlmYjVmMmE2OGZmMTg2MGRmNGVmMTBjOTgzNWU2MmU3YzEiXSwic2lndCI6MTYwMzQ1Nzk5OSwidmVyIjoxfQ.NDUyZDllODlkNWJkNWQ5MjI1ZmI2ZGFlY2Q1NzllNzM4OGExNjZjNzY2MWNhMDRlNDdmZDNjZDg0NDZlNDYyMA.-jpKBZQ3sc0x34MwnbO8mSiGdUYCfQXNO91RMnvFRq0YZ5pmbKmRYg--zaie-N7wIJhIFbZyuOyJdlcPwZrELQ
```

