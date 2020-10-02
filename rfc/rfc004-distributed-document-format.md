# RFC004 Distributed Document Format 

|  |  |
| :--- | :--- |
| Nuts foundation | R.G. Krul |
| Request for Comments: 004 | Nedap |
|  | September 2020 |

## Distributed Document Format

### Abstract

### Status of document

This document is currently a draft.

### Copyright Notice

![](../.gitbook/assets/license.png)

This document is released under the [Attribution-ShareAlike 4.0 International \(CC BY-SA 4.0\) license](https://creativecommons.org/licenses/by-sa/4.0/).

## 1.  Introduction

This RFC describes an interoperable, content agnostic data format for distributed networks which provides
cryptographic authentication and verification of its contents, guaranteed ordering and access control.      

## 2. Terminology

* **Document**: piece of application data enveloped with metadata like signatures and references to other documents
  published on a distributed network.

Other terminology comes from the [Nuts Start Architecture](rfc001-nuts-start-architecture.md#nuts-start-architecture).

## 3. Requirements
The following requirements apply for the document format:

* **Flexible content**: the format must support any kind of payload, textual or binary, large or small.
* **Consistent ordering**: since documents might revise or require a previous document, there must be ordering that's consistent over the network.
* **Integrity protected**: since second- or third parties MAY relay documents, document integrity MUST be cryptographically verifiable.
* **Authenticity protected**: for the same reason as above, document authenticity (who authored the document) MUST be cryptographically verifiable.
* **Anonymous Confidentiality Control**: since some documents MAY be confidential (just to be read by specific parties)
  there MUST be a way to indicate that the document is confidential. Which party (or parties) is allowed to read the document
  sensitive information in itself, and should only be disclosed to the involved parties.
  In other words if party A publishes a document only to be read by party B, only they should see that fact. Uninvolved
  party C MAY only see a confidential document from party A but not the involvement of party B. 
  (TBD: should this actually be here yet?)

## 4. Document Format
The [RFC7159 JavaScript Object Notation](https://tools.ietf.org/html/rfc7159) (JSON) MUST be used as base format.

### 4.1. Document Structure

A document MUST contain the following fields:
* **ref**: (reference) MUST contain the reference (see section 4.2) of the document as hexadecimal string.
* **issuedAt**: MUST contain the creation time of the event in UTC as string, formatted according to [RFC3339](https://tools.ietf.org/html/rfc3339).
* **type**: MUST contain the type of the payload indicating how to interpret the payload encoded as string.
* **jws**: MUST contain a base-64 encoded [RFC7515 JSON Web Signature](https://tools.ietf.org/html/rfc7515) (JWS, see section 4.3).
* **payload**: MUST contain the actual content of the document. The format is unspecified so MAY any data type supported by the JSON including a nested JSON document.  
* **version**: MUST contain the format version of the document as number. For this version of the format the version MUST be 1.

A document MAY contain the following fields:
* **prev**: (previous) MUST contain (if present) the reference (see section 4.2) of the preceding document.

Field MAY appear in any order. If one of the fields is in an unexpected format the document MUST be rejected.
Other fields SHOULD NOT be present and MUST be ignored.

### 4.2. Document Reference
The document reference uniquely identifies a document. All fields defined in section 4.1 MUST be included in the calculation
EXCEPT the reference itself. Other fields SHALL NOT be included in the calculation.

It MUST be calculated as follows:
* Remove the *ref* field from the document (if present).
* Canonicalize the document using the [Rundgren JSON Canonicalization Scheme (draft v17)](https://www.ietf.org/id/draft-rundgren-json-canonicalization-scheme-17.html).
* Hash the canonicalized document using SHA-1.

When serializing a reference to string form it MUST be hexadecimal encoded and SHOULD be lower case.

Example:

```148b3f9b46787220b1eeb0fc483776beef0c2b3e```

### 4.3. Document Signature
The document signature asserts the authenticity (who ais the author) and integrity (is it original or did someone alter it) of the document.
The signature MUST be created according to the JWS standard and represented in compact form.
It MUST be created using one of the following algorithms: `RS256`, `RS384`, `RS512`, `ES256`, `ES384` or `ES512`.
Other algorithms SHALL NOT be used.

The payload to be signed MUST be calculated as follows:
1. Take the input document.
2. Remove *ref* and *sig* properties (if present).
3. Canonicalize the document using the [Rundgren JSON Canonicalization Scheme (draft v17)](https://www.ietf.org/id/draft-rundgren-json-canonicalization-scheme-17.html).
4. Sign the payload using public key of the document author.

The *x5c* field of the JWS MUST contain exactly one entry with the X.509 signing certificate associated with the private key.
The certificate MUST have the digitalSignature and contentCommitment key usages and must be valid at time of signing.

### 4.4. Ordering
All documents except the very first document MUST specify the *prev* field referencing the last known document on the network.
This is essential for maintaining consistent ordering across peers on the network. The *issuedAt* field SHOULD have a timestamp
equal to or after the referenced document's *issuedAt* field. The *prev* field's reference MUST be calculated as
documented in section 4.2.

### 4.5. Example
TBD