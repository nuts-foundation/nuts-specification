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
* **Genesis document** the first document published on a network.

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
* **crit** MUST contain the **sigt** and **version** headers.

The **jku**, **jwk**, **kid** and **x5u** header parameters SHOULD NOT be used and MUST be ignored by when processing the document.

In addition to the registered header parameters, the following header MUST be present as protected headers:
* **sigt**: (signing time) MUST contain the signing time of the document in UTC as string, formatted according to [RFC3339](https://tools.ietf.org/html/rfc3339).
* **version**: MUST contain the format version of the document as number. For this version of the format the version MUST be 1.
* **prev**: MUST contain the reference (see section 3.2) of the preceding document, or a zeroed reference if a genesis document.

The JWS payload MUST be the actual contents of the document. The format is unspecified so MAY any data type supported by the JSON including a nested JSON document.

### 3.2. Document Reference
The document reference uniquely identifies a document and is used to fill the **prev** header. It MUST be calculated by
taking the bytes of the JWS EXACTLY as received and hashing it using SHA-1.

When serializing a reference to string form it MUST be hexadecimal encoded and SHOULD be lowercase.

Example:

```148b3f9b46787220b1eeb0fc483776beef0c2b3e```

### 3.3. Ordering
Documents ordering is casual: a document MUST refer to the last known document on the network by specifying its
reference in the *prev* field. This is essential for maintaining consistent order across peers.
The *iat* field SHOULD have a timestamp equal to or after the referenced document's *iat* field.
The first document published on the network (genesis document) MUST have a zeroed *prev* field:

```0000000000000000000000000000000000000000```

There MUST only be one genesis document for a network. Subsequent genesis documents MUST be ignored.

TODO: What about branching?


### 3.4. Validation
Before interpreting a document's payload it SHOULD be validated according to the following rules:

1. Assert that if it's a genesis document, we didn't already receive one.
2. Assert that the previous document (*prev* field) is valid. Preceding documents should be validated first.
3. Verify the document signature:
   * Validate keyUsage, validity of the certificate in the *x5c* field and whether the issuer is trusted.
   * Verify the cryptographic signature with the public key from the certificate.
   * Assert that the certificate was valid at the time of signing (as specified by *iat*).
   * Assert that the certificate was not revoked on at time of signing. 

If any of the steps above fail the document SHOULD be rejected and its payload SHALL NOT be deemed valid.

# 4. Example

## 4.1. Compact Serialization
```json
{ "TODO": "..." }
```

## 4.2. JSON Serialization
```json
{ "TODO": "..." }
```