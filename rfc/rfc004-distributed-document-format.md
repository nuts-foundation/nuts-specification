# RFC004 Distributed Document Format 

|  |  |
| :--- | :--- |
| Nuts foundation | R.G. Krul |
| Request for Comments: 004 | Nedap |
|  | September 2020 |

## Distributed Document Format

### Abstract

This RFC describes an interoperable, content agnostic data format for distributed networks which provides
cryptographic authentication and verification of its contents, guaranteed ordering and access control.

### Status of document

This document is currently a draft.

### Copyright Notice

![](../.gitbook/assets/license.png)

This document is released under the [Attribution-ShareAlike 4.0 International \(CC BY-SA 4.0\) license](https://creativecommons.org/licenses/by-sa/4.0/).

## 1.  Introduction

Because Nuts is a decentralized network there needs to be a standardized format for structuring public information
(like where to find another care organization's data) in a way that allows each party to independently verify
the authenticity and integrity of that information. This document proposes such a format. Systems can use the format
to build their application on top of it making it suitable for publishing on a distributed network.

This document does not define how to transport data to other participants in a distributed network.

## 2. Terminology

* **Document**: piece of application data enveloped with metadata like signatures and references to other documents
  published on a distributed network.
* **Root document** the first document published on a network.

Other terminology comes from the [Nuts Start Architecture](rfc001-nuts-start-architecture.md#nuts-start-architecture).

## 3. Document Format
Documents MUST be encoded as [RFC7515 JSON Web Signature](https://tools.ietf.org/html/rfc7515). It can be serialized in
either compact ([RFC7515 section 7.1](https://tools.ietf.org/html/rfc7515#section-7.1)) or JSON
([RFC7515 section 7.2](https://tools.ietf.org/html/rfc7515#section-7.2)) form. When the document data starts with an
accolade (`{`) it MUST be assumed the document is serialized in JSON format, otherwise compact format MUST be assumed.

### 3.1. JWS implementation
In addition to required header parameters as specified in RFC7515 the following requirements apply:
* **x5c**: MUST be present and contain exactly one entry with the X.509 signing certificate associated with the private key.
  The certificate MUST have the digitalSignature and contentCommitment key usages and must be valid at time of signing.
  The certificate SHOULD conform to [RFC008 Certificate Structure](rfc008-certificate-structure.md) so other parties can validate it.
* **alg**: MUST be one of the following algorithms: `PS256`, `PS384`, `PS512`, `ES256`, `ES384` or `ES512`.
  other algorithms SHALL NOT be used.
* **cty**: MUST contain the type of the payload indicating how to interpret the payload encoded as string.
* **crit** MUST contain the **sigt**, **ver** and **prevs**" headers.

The **jku**, **jwk**, **kid** and **x5u** header parameters SHOULD NOT be used and MUST be ignored by when processing the document.

In addition to the registered header parameters, the following headers MUST be present as protected headers:
* **sigt**: (signing time) MUST contain the signing time of the document as Unix time since epoch encoded as NumericValue.
* **ver**: MUST contain the format version of the document as number. For this version of the format the version MUST be 1.
* **prevs**: (previous documents) MUST contain the references (see section 3.2) of the preceding documents (see section 3.4).
  When it's a root document the field SHALL NOT have any entries.

The following headers MAY be present as protected headers (see section 3.4 for details):
* **tid**: (timeline ID) MUST contain the reference to the document that started the timeline.
* **tiv**: (timeline version) MUST contain a numeric version indicating the version of the document on the timeline.

To aid performance of validating the DAG the JWS SHALL NOT contain the actual contents of the document. Instead, the
JWS payload MUST contain the SHA-1 hash of the contents encoded as hexadecimal, lower case string.

Example:

```148b3f9b46787220b1eeb0fc483776beef0c2b3e```

The contents then MAY be stored next to or apart from the document itself (but that's out of scope for this RFC).

There SHOULD be only 1 signature on the JWS. If there are multiple signatures all signatures except the first one MUST be ignored.

### 3.2. Document Reference
The document reference uniquely identifies a document and is used to refer to it. It MUST be calculated by
taking the bytes of the JWS EXACTLY as received and hashing it using SHA-1.

When serializing a reference to string form it MUST be hexadecimal encoded and SHOULD be lowercase.

Example:

```148b3f9b46787220b1eeb0fc483776beef0c2b3e```

### 3.3. Ordering, branching and merging
Documents MUST form a rooted DAG (Directed Acyclic Graph) by referring to the previous document.
This MAY be used to establish *casual ordering*, e.g. registration of a care organization as child object of a vendor. A new document MUST be appended to the end of the DAG
by referring to the last document of the DAG (*leaf*) by including its reference in the **prevs** field.

As the name implies the DAG MUST be acyclic, documents that introduce a cycle are invalid MUST be ignored. ANY following
document that refers to the invalid document (direct or indirect) MUST be ignored as well.

Since it takes some time for the documents to be synced to all network peers (eventual consistency) there COULD be
multiple documents referring to the previous documents in the **prevs** field, a phenomenon called *branching*. Since
branches (especially old and/or long ones) may cause documents to be reordered which hurts performance they MUST be
merged as soon as possible. Branches are merged by specifying their leafs in the **prevs** field:

![RFC structure](../.gitbook/assets/rfc004-branching.svg)

The first document in the DAG is the *root document* and SHALL NOT have any **prevs** entries. There MUST only be
one root document for a network and subsequent root documents MUST be ignored.

When processing a DAG the system MUST start at the root document and work its way to the leaf(s) processing subsequent
documents. When encountering a branch the documents on all branches from that point up until the merge MUST be processed
before processing the merge itself. Since ordering is casual processing order over parallel branches isn't important.
When looking at the diagram above, the following processing orders are valid:

* `A -> B -> C -> D -> E -> F` (mixed parallel order)
* `A -> B -> D -> C -> E -> F` (branch D first, then branch C)
* `A -> B -> C -> E -> D -> F` (branch C first, then branch D)

The following orders are invalid:

* `A -> B -> C -> E -> F -> D` (merger processed before all previous were processed)
* `A -> B -> D -> F -> C -> E` (merger processed before all previous were processed)
* `A -> B -> D -> E -> C -> F` (branch C is processed out-of-order)

### 3.4. Timelines
Since documents are immutable, the only way to update them it by creating a new document. Subsequent versions of a
document SHOULD be tracked by creating a *timeline* using the optional **tid** and **tiv** fields. These fields SHALL NOT be used
on the first document in the timeline, only on updates. The **tid** field identifies the timeline and MUST contain the
reference of the first document. The **tid** field MUST be present when **tiv** is specified.

For signalling updates based on out-of-date state **tiv** (timeline version) CAN be used to indicate the version of the
document. It MUST be an incrementing integer value starting at `1`. Version increments SHOULD be complete since gaps MAY
indicate the consumer is missing a (branching) document. A duplicate version MAY indicate a race condition where the
producer updated the document based on out-of-date state. 

Before processing payload applications MUST assert that updates in a timeline have been issued by the owner (initial signer)
of the original document.

### 3.5. Processing the DAG
Processing the DAG can be seen as planning tasks required for construction: some tasks can happen in parallel
(laying floors and installing electricity), some tasks must happen sequentially (foundation must be poured before
building the walls). This is the same for documents on the DAG: documents on a branch MUST be processed sequentially
but processing order of parallel branches is unspecified, and before processing a merging document all **prevs** (branches)
must have been processed.

An algorithm that COULD be used is *Breadth-First-Search*. However with branches with more than 1 document this algorithm
processes the merging document before all preceding documents were processed. This CAN be solved by adding an extra check
that skips the document when not all previous documents have been processed. When the missing previous document is processed
the merging document will be re-added to the queue for processing. The pseudo code for this algorithms looks as follows: 

```
given FIFO queue
add root document to queue
until queue empty; take document from queue
    if current document already visited
        continue loop

    if any of the previous (prevs) documents is not yet processed
        continue loop

    for all next documents of the current document
        add next document to queue

    process document
```

### 3.6. Signature verification

Before interpreting a document's payload it SHOULD be validated according to the following rules:

- Assert cryptographic signature; can it be validated with the public key in the signing certificate?
- Assert the certificate is trusted.
- Assert the certificate and chain was valid at signing time.
- Assert the certificate is meant for signing; key usage digital MUST be `digitalSignature`.

Note there's no need for certificate revocation status checking; certificates are generally short-lived
 (as specified by [RFC008 Certificate Structure](rfc008-certificate-structure.md)).

# 4. Example

## 4.1. Compact Serialization
```json
{ "TODO": "..." }
```

## 4.2. JSON Serialization
```json
{ "TODO": "..." }
```