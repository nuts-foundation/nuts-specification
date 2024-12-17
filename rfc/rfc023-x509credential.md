# RFC023: X509Credential

|                           |               |
|:--------------------------|:--------------|
| Nuts foundation           | R.G. Krul     |
| Request for Comments: 023 | Zorg bij jou  |
|                           | R. Groen      |
|                           | Zorg bij jou  |
|                           | December 2024 |

## Abstract

This RFC specifies the requirements and validation process for the `X509Credential`, a W3C Verifiable Credential (VC)
type issued by a `did:x509` DID. The `X509Credential` ensures strong alignment with the properties of the associated
X.509 certificate and defines mechanisms to validate the credential and verify its association with a `did:x509` DID.

## Status of this document

This document is a proposal for discussion and implementation. Feedback is welcome to improve the interoperability
and robustness of the specification.

## Introduction

The `X509Credential` is a W3C Verifiable Credential type designed for use cases where trust anchors are based on X.509
certificates. It leverages the `did:x509` method, as specified in
the [Trust Over IP DID:X509 Method Specification](https://trustoverip.github.io/tswg-did-x509-method-specification/).
By aligning credential subject validation with the fields of the associated `did:x509` DID and enforcing
certificate revocation checks, the `X509Credential` ensures integrity and adherence to the PKI trust model.

Its intended use is to bridge the gap in ecosystems where issuers don't support issuance of Verifiable Credentials yet,
but do issue X.509 certificates containing relevant information about the credential subject.

## Definitions

- **X509Credential**: A Verifiable Credential whose issuer is a `did:x509` DID and whose structure adheres to the
  requirements in this document.
- **did:x509**: A Decentralized Identifier (DID) method specified by the Trust Over IP Foundation, where the DID is
  derived from an X.509 certificate.
- **Issuer Certificate**: The X.509 certificate associated with the `did:x509` DID that issued the credential.
- **Credential Subject**: The entity described by the credential.
- **Revocation Check**: The process of verifying the revocation status of the issuer certificate using mechanisms like
  OCSP or CRL.

## Credential Structure

An `X509Credential` must conform to the general structure of a W3C Verifiable Credential and conform to the following
rules:

- The credential MUST be in JWT format.
- `type`: MUST include `VerifiableCredential` and `X509Credential`.
- `issuer`: MUST be a valid `did:x509` identifier.
- `credentialSubject`: MUST only contain fields explicitly present in the `did:x509` DID policies.

The credential subject can be identified by any DID method (e.g. `did:web`) accepted by the credential verifier.

### Example `X509Credential`

Below is an example of an `X509Credential` issued by a `did:x509` DID. The credential subject is identified by a
`did:web`.
The first snippet is the JWT header, and the second snippet is the credential payload.

(TODO: check this example)

```json
{
  "alg": "PS256",
  "typ": "JWT",
  "x5c": [
    "<base64 encoded leaf-certificate>",
    "<base64 encoded issuer-intermediate-certificate>",
    "<base64 encoded issuer-certificate>"
  ],
  "x5t": "<thumbprint>",
  "kid": "did:x509:0:sha256:<hash>::subject:O:Library%20The%20Bookworm::subject:L:Bookland::san:otherName:123#1"
}
```

(TODO: Add right JSON-LD context)
Payload:

```json
{
  "vc": {
    "@context": [
      "https://www.w3.org/2018/credentials/v1"
    ],
    "type": [
      "VerifiableCredential",
      "X509Credential"
    ],
    "issuer": "did:x509:0:sha256:<hash>::subject:O:Library%20The%20Bookworm::subject:L:Bookland::san:otherName:123",
    "issuanceDate": "2024-12-01T00:00:00Z",
    "credentialSubject": {
      "id": "did:web:example.com",
      "subject:O": "Library The Bookworm",
      "subject:L": "Bookland",
      "san:otherName": "123"
    }
  }
}
```

## Validation

To validate an `X509Credential`, the following steps MUST be performed:

- Verify that the credential is in JWT format.
- Verify that the issuer's DID is a `did:x509` DID.
- Resolve the `did:x509` DID document according to the did:x509 specification and check the certificate chain for
  revocation.
- Validate that the `credentialSubject` fields match the policies in the `did:x509` DID.

### 1. Verify Credential Structure

Ensure that the credential:

- Includes the `X509Credential` type.
- Contains a valid `did:x509` issuer.
- Includes a `credentialSubject` whose fields match the `did:x509` DID Document.

### 2. Validate the Issuer Certificate

The certificate associated with the `did:x509` issuer MUST be validated as follows:

- **Certificate Chain Validation**: The certificate must have a valid trust chain to a known root CA.
- **Revocation Check**: Verify the revocation status of the certificate using OCSP or CRL.

Failure to validate the issuer certificate invalidates the credential.

### 3. Validate the Credential Subject

The `credentialSubject` MUST be verified against the `did:x509` DID Document. Specifically:

- Every field in the `credentialSubject` MUST be present in the `did:x509` DID Document.
- Fields not present in the `did:x509` DID Document invalidate the credential.

### 4. Verify the Proof

The cryptographic proof of the credential MUST be verified using the public key associated with the `did:x509` DID.
This involves:

- Resolving the public key from the DID Document.
- Verifying the signature on the credential's `proof`.

### 5. Check Credential Expiry

If the `issuanceDate` or any other relevant date constraints (e.g., `expirationDate`) are present, they MUST be
validated to ensure the credential is within its valid timeframe.

## Security Considerations

TODO: Trust, which ca-fingerprint to use, ...

### Certificate Revocation

The revocation status of the issuer's certificate is a critical component of `X509Credential` validation. Implementers
MUST use reliable revocation checking mechanisms (e.g., OCSP or CRL) and handle failures (e.g., network issues)
appropriately to avoid false-positive validations.

### Field Alignment

Restricting the `credentialSubject` fields to those present in the `did:x509` DID Document ensures alignment with the
X.509 certificate, reducing the risk of unauthorized data inclusion.

### Proof Verification

The cryptographic proof verification ensures that the credential has not been tampered with and was issued by the
entity controlling the `did:x509` DID.

## References

- [W3C Verifiable Credentials Data Model](https://www.w3.org/TR/vc-data-model/)
- [DID:X509 Method Specification](https://trustoverip.github.io/tswg-did-x509-method-specification/)
- [X.509 Certificate Revocation (OCSP/CRL)](https://datatracker.ietf.org/doc/html/rfc5280)
