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

This document describes a Nuts standards protocol.

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
* **Head**: Transaction that is not referenced by another \(newer\) transaction.
* **Lamport clock**: Logical clock creating a casual ordering between transactions (https://en.wikipedia.org/wiki/Lamport_timestamp).

Other terminology comes from the [Nuts Start Architecture](rfc001-nuts-start-architecture.md#nuts-start-architecture).

## 3. Transaction Format

Transactions MUST be encoded as [RFC7515 JSON Web Signature](https://tools.ietf.org/html/rfc7515). It MUST be serialized in compact \([RFC7515 section 7.1](https://tools.ietf.org/html/rfc7515#section-7.1)\) serialization form.

### 3.1 JWS implementation

In addition to required header parameters as specified in RFC7515 the following requirements apply:

* The signing key is indicated by **kid** or **jwk**. One of them MUST be present, but not both. If **kid** is present the key must be known and looked up locally. Otherwise the key must be taken from the **jwk** header.
* **alg**: MUST be one of the following algorithms: `PS256`, `PS384`, `PS512`, `ES256`, `ES384` or `ES512`.

  other algorithms SHALL NOT be used.

* **cty**: MUST contain the type of the transaction content indicating how to interpret it.
* **crit** MUST contain the **sigt**, **ver** and **prevs** headers.
* **crit** MUST also contain the **lc** header if **ver** == 2.

The **jku**, **x5c** and **x5u** header parameters SHOULD NOT be used and MUST be ignored by when processing the transaction.

In addition to the registered header parameters, the following headers MUST be present as protected headers:

* **sigt**: \(signing time\) MUST contain the signing time of the transaction as Unix time since epoch encoded as NumericValue.
* **ver**: MUST contain the format version of the transaction as number. For this version of the format the version MUST be 1 or 2.
* **prevs**: \(previous transactions\) MUST be a string array with *transaction references* as described in subsection 3.2.
  * The root transaction SHALL NOT have any **prevs** entries.
  * Non-root transactions MUST include one reference as described in subsection 3.2.1.
  * If a **kid** parameter is present, then **prevs** MUST also contain a reference to a transaction which its content includes the key entry with the respective **kid**.
* **lc**: (Lamport's Logical Clock) MUST be a natural number.
  * If the transaction has no **prevs** entries, then **lc** is 0.
  * Otherwise, **lc** MUST be the highest **prevs** entry plus 1.

The following protected headers MAY be present:

* **pal**: MUST contain the encrypted addresses of the participants \(used for private transactions, see section 3.8\).

To aid performance of validating the DAG the JWS SHALL NOT contain the actual application data of the transaction. Instead, the JWS payload MUST contain the SHA-256 hash of the contents encoded as hexadecimal, lower case string, e.g.: `386b20eeae8120f1cd68c354f7114f43149f5a7448103372995302e3b379632a`

The contents then MAY be stored next to or apart from the transaction itself \(but that's out of scope for this RFC\).

There SHOULD be only 1 signature on the JWS. If there are multiple signatures all signatures except the first one MUST be ignored.

### 3.2 Transaction Reference

Transactions are defined by one, and only one JWS (source). Transactions are refered to by the SHA-256 digests of their respective JWS (source).

#### 3.2.1 Previous Transactions

Entries in `prevs` MUST each be a hexadecimal encoding of one transaction refererce (hash digest). Entries SHOULD use lower-case, e.g. `386b20eeae8120f1cd68c354f7114f43149f5a7448103372995302e3b379632a`. Note that due to the mixed-casing possibility, string matching can **not** be used for equality comparison.

New transaction additions MUST refer [`prevs`] to a transaction with the highest `lc` value present within the applicable graph. When multiple transactions match the highest `lc` value present, then only a single one of them [arbitrary] SHOULD be refered to.

Header `prevs` entries SHOULD only include successfully-processed transactions, this to avoid publishing of unprocessable transactions. Note that “successfully-processed” is relative. Other nodes may have a different outcome.

### 3.3 Ordering, branching and merging

Transactions MUST form a rooted DAG \(Directed Acyclic Graph\) by referring to previous transactions. This MAY be used to establish _casual ordering_, e.g. registration of a care organization as child object of a vendor.

All transactions referred to by `prevs` MUST already be present in the respective graph, since failing to do so would corrupt the DAG.

As the name implies the DAG MUST be acyclic, transactions that introduce a cycle are invalid MUST be ignored. ANY following transaction that refers to the invalid transaction \(direct or indirect\) MUST be ignored as well.

Since it takes some time for the transactions to be synced to all network peers \(eventual consistency\) there COULD be multiple transactions referring to the previous transactions in the **prevs** field, a phenomenon called _branching_. Branching is common and relatively harmless. Branches are merged by specifying their heads in the **prevs** field:

![DAG structure](../.gitbook/assets/rfc004-branching.svg)

Branches may remain unmerged. As mentioned at the beginning of this paragraph, a transaction MUST refer to a previous transaction. It doesn't matter on which branch the latest transaction is published.
**LC** values will overlap between branches.

The first transaction in the DAG is the _root transaction_. There MUST only be one root transaction for a network. Subsequent root transactions MUST be ignored.

When processing a DAG the system MUST start at the root transaction and work its way to the head\(s\) processing subsequent transactions. When encountering a branch the transactions on all branches from that point up until a merge MUST be processed before processing the merge itself. Since ordering is casual processing order over parallel branches isn't important. When looking at the diagram above, the following processing orders are valid:

* `A -> B -> C -> D -> E -> F` \(mixed parallel order\)
* `A -> B -> D -> C -> E -> F` \(branch D first, then branch C\)
* `A -> B -> C -> E -> D -> F` \(branch C first, then branch D\)

The following orders are invalid:

* `A -> B -> C -> E -> F -> D` \(merger processed before all previous were processed\)
* `A -> B -> D -> F -> C -> E` \(merger processed before all previous were processed\)
* `A -> B -> D -> E -> C -> F` \(branch C is processed out-of-order\)

### 3.4 Updates & Ordering

Since transactions are immutable, the only way to update the application data they contain is by creating a new transaction. All updates are considered as full updates, partial updates are not supported. Parallel updates of the same application data, in the form of branches, MUST be processed the same way by all nodes.

Consider the DAG from the previous chapter. If transaction `C` and `D` where to update the same application data, nodes could process the transaction in a different order. This would create an inconsistency in the network. To fix this the following rules MUST be taken into account:

* If the transaction content is equal, process as normal. \(This can be determined by the JWS payload being the same\)
* If the transaction content is not equal, create a representation that is a merger of the transaction content according to rules for the specific type.

When updates are required for a type of transaction content, it MUST be composed of [conflict-free replicated data types](https://en.wikipedia.org/wiki/Conflict-free_replicated_data_type)

### 3.5 Processing the DAG

Processing the DAG can be seen as planning tasks required for construction: some tasks can happen in parallel \(laying floors and installing electricity\), some tasks must happen sequentially \(foundation must be poured before building the walls\). This is the same for transactions on the DAG: transactions on a branch MUST be processed sequentially but processing order of parallel branches is unspecified, and before processing a merging transaction all **prevs** \(branches\) must have been processed.

Processing MUST be performed in order of the **lc** values.
If multiple transactions have the same **lc** value, they MUST be processed in sorted order.
The transactions MUST be sorted by their hexadecimal hash value.

### 3.6 Signature verification

Before interpreting a transaction's content the JWS' signature MUST be validated. Almost all transactions will use the **kid** to identify the key used to sign the transaction.
Resolving the **kid** based on the signature timestamp (**sigt**) can cause problems and even adds attack vectors.
The main reason behind this, is that the signature time doesn't determine the order of transactions.
Transaction B can be signed with a key introduced in transaction A. Transaction B depends on A, this dependency can't be solved with the signature time.
This dependency can be solved by referring to the transaction that introduced the **kid**.
If a ``kid`` is present in the JWS, the `prevs` MUST contain a transaction reference of which the transaction content includes the key entry with the **kid** as identifier.
If the resolved transaction contents is not the latest version, this may indicate a broken installation or a stolen private key. These transactions can't be blocked since this would just add other attack vectors. A node operator SHOULD determine what the best course of action is.

> **_NOTE:_**  It would be solvable with a central notary or voting scheme that determines the transaction order. This would make the system more complex.  

### 3.7 Transaction content verification

Since the transaction content is detached from the transaction itself and referred to by hash, the transaction content MUST be hashed and compared to the hash specified in the transaction, to assert that the retrieved transaction content is actually the expected transaction content.

### 3.8 Private Transactions

Private transactions are transactions that contain sensitive content, intended for specific entities that participate in the transaction.
Private transactions MUST be added to the DAG like non-private transactions.
The participants MUST be specified as array that contains the node DIDs of the participants, encrypted separately for each participant.
To construct the data to be encrypted, the plaintext participants MUST be joined by a newline (`\n`)

The participant list MUST be encrypted for each participant and the ciphertext specified as array as `pal` header in the JWS.

Given 2 participants `did:nuts:participant-A` and `did:nuts:participant-B`:

```
encoded_participants = join("did:nuts:participant-A", "did:nuts:participant-B", "\\n")

pal = []

for each participant
  encrypted_participants = ecies_encrypt(encoded_participants, participant.encryption_public_key)
  pal = append(pal, encrypted_participants)

transaction.pal = pal
```

To decrypt the header, the process above is applied in reverse. If one of the decrypted values match the local node's DID,
it indicates the local node is (one of) the intended participant, and it SHOULD try to retrieve the contents.

The encryption key to be used MUST be an elliptic curve of the participant.
In the context of DIDs, it MUST be the first `keyAgreement` key in the participant's DID document.  
For encryption of the `pal` header the ECIES encryption algorithm MUST be used.

See appendix A on the reasoning behind encrypting the `pal` header.

## 4. Example

```text
eyJhbGciOiJFUzI1NiIsImNyaXQiOlsic2lndCIsInZlciIsInByZXZzIiwibGMiXSwiY3R5IjoiZm9vL2JhciIsImp3ayI6eyJjcnYiOiJQLTI1NiIsImtpZCI6IjEiLCJrdHkiOiJFQyIsIngiOiJCcjlGT3pkdUtyRXF4ZzI0emx0ZmVKbFZyZ09sbEFOekFicGNMVXU5YkYwIiwieSI6IkZMTDNBaTF3eEFRczY5ZXVxTTlFQkZjNXhkMUM5bGFzWnVxSXBKNHJnUFUifSwibGMiOjAsInByZXZzIjpbXSwic2lndCI6MTY2MjAyMzQzNSwidmVyIjoyfQ.YjQwNzExYTg4YzcwMzk3NTZmYjhhNzM4MjdlYWJlMmMwZmU1YTAzNDZjYTdlMGExMDRhZGMwZmM3NjRmNTI4ZA.OdkPIboQPbAmPCcFda8RdgQdFVM4lzAbJrOboCV742bI3dhgVGzWvyjnGvtrRIRhm1wW0PzdMi3aSqKUDqwOJA
```

## Appendix A: Design decisions

### A.1 Correlation Attacks on Private Transactions

When multiple encrypted documents, intended to be anonymous, contain (encrypted or not) data that is the same for every encrypted document,
they could be related to each other, deriving information from their context. E.g., sender, time of sending, etc.
In the context of Nuts: a care organization might issue multiple `NutsAuthorizationCredential`s (meant for giving access to EHR records) at once, to a set of care organizations.
If the participants are publicly readable, this gives knowledge of the name, physical location and care type of the organizations that have access.
Given enough public knowledge and/or information from other contexts, one could ultimately pinpoint the set of issues VCs to a person (e.g. a relative or a celebrity).

To mitigate this attack, given a plaintext and encryption key, every subsequent encryption operation (with the same plaintext and key) must yield a new, random ciphertext.

Still, this does not make correlation attacks impossible, just harder.
E.g.: in small networks, the number of care organizations might be so small, that one could guess the receiving organization with relatively large accuracy.
Especially when they are issued (and re-issued after they expire) by an automated process.
This problem gets smaller when the network grows.

### A.2 Heads, `prevs` and branches

In previous versions of the specification, the `prevs` was specified to contain all current heads of the DAG.
This requirement has changed due to:

- when creating a lot of transactions, lots of branches are created. Adding these heads to the next transaction greatly increases the size of the transaction.
- the state of the DAG is no longer indicated by its heads but rather by an XOR value and its IBLT structure. See [RFC017](./rfc017-distributed-network-grpc-v2.md)

Referring to a previous transaction is still required to make sure the Lamport Clock still increases with each transaction. 
