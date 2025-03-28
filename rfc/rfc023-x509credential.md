# RFC023: X509Credential

|                           |               |
|:--------------------------|:--------------|
| Nuts foundation           | R.G. Krul     |
| Request for Comments: 023 | Zorg bij jou  |
|                           | R. Groen      |
|                           | Zorg bij jou  |
|                           | December 2024 |

## Abstract

Trust is a key element in the Nuts network. This RFC describes how X.509 certificates can be used to make use of X.509 sourced trust in the NUTS framework. The X.509 certification process has been around for a long time and is widely used in the internet. This RFC describes how X.509 certificates can be used in the Nuts network to establish trust between parties. It does so by linking the X.509 certificate to a Nuts identity using a Verifiable Credential that is issued by the holder of the x509 identity.

This RFC specifies the requirements and validation process for the `X509Credential`, a W3C Verifiable Credential (VC) type issued by the subject of a X.509 certificate, represented by a `did:x509` DID. The `X509Credential` ensures strong alignment with the properties of the associated X.509 certificate and defines mechanisms to validate the credential and verify its association with a `did:x509` DID.

## Status of this document

This document is currently in draft status. Feedback is welcome to improve the interoperability and robustness of the specification.

### Copyright Notice

![](../.gitbook/assets/license.png)

## 1. Introduction

The [did:x509](https://trustoverip.github.io/tswg-did-x509-method-specification/) method aims to achieve interoperability between existing X.509 solutions and Decentralized Identifiers (DIDs). This to support operational models in which a full transition to DIDs is not yet achievable or desired.

The `X509Credential` is a W3C Verifiable Credential type designed for use cases where trust anchors are based on X.509 certificates. It leverages the `did:x509` method, as specified in the [Trust Over IP DID:X509 Method Specification](https://trustoverip.github.io/tswg-did-x509-method-specification/).

By aligning credential subject validation with the fields of the associated `did:x509` DID and enforcing certificate revocation checks, the `X509Credential` ensures integrity and adherence to the PKI trust model.


## 2. Definitions

- **X509Credential**: A Verifiable Credential whose issuer is a `did:x509` DID and whose structure adheres to the
 requirements in this document.
- **did:x509**: A Decentralized Identifier (DID) method specified by the Trust Over IP Foundation, where the DID is
 derived from an X.509 certificate.
- **Issuer Certificate**: The X.509 certificate associated with the `did:x509` DID that issued the credential.
- **Credential Subject**: The entity described by the credential as the subject of the credential.
- **Revocation Check**: The process of verifying the revocation status of the issuer certificate using mechanisms like CRL.

## 3. Used standards and technologies

This RFC builds on the following standards and technologies:

* [X.509 Certificate Standard](https://datatracker.ietf.org/doc/html/rfc5280)
* [JSON Web Signature (JWS)](https://datatracker.ietf.org/doc/html/rfc7515#section-4.1.6)
* [did:x509 method](https://trustoverip.github.io/tswg-did-x509-method-specification/), with modifications
* [Verifiable Credentials Data Model v1.1](https://www.w3.org/TR/vc-data-model/)

### 3.1 X.509 certificates, a brief introduction

An **X.509 certificate** is a digital certificate that follows the X.509 Public Key Infrastructure (PKI) standard. It is widely used for secure communication over the internet, such as HTTPS, email encryption, and digital signatures.

#### Key Features of X.509 Certificates:
- **Structure**: It contains information about the certificate owner (e.g., organization, common name, public key) and the issuing Certificate Authority (CA). The certificate is signed by the CA's private key to ensure authenticity.
- **Trust Hierarchy**:
    - Trust is anchored in a **Certificate Authority (CA)**, which is a trusted third party.
    - Certificates can form a chain of trust, starting from a **root certificate** (trusted CA) to **intermediate certificates**, down to the **end-user/client certificates**.

An example of the trust chain hierarchy:


```asciidoc
┌────────────────────┐
│      Root CA       │
└─────────┬──────────┘
          │           
┌─────────▼──────────┐
│  Intermediate CA   │
└─────────┬──────────┘
          │           
┌─────────▼──────────┐
│  Intermediate CA   │
└─────────┬──────────┘
          │           
┌─────────▼──────────┐
│Signing Certificate │
└────────────────────┘
```

### 3.2 Nuts and X.509:
The Nuts framework extends X.509 certificates into its decentralized identity (DID) ecosystem using the `did:x509` DID method. This method facilitates linking an X.509 certificate to a verifiable credential (`X509Credential`), providing trust while bridging traditional PKI with decentralized trust models. This is especially relevant in systems reliant on existing X.509 implementations, such as healthcare or government frameworks.

### 3.3 Using X.509 Certificates for signing JWTs

JWT is a standard that is used to sign JSON objects. By signing JSON objects with the
private key of the certificate, authenticity of the JWT signer can be established. The signature of the JWT can be verified using the public key of the certificate.
The certificate chain is included in the JWT header as `x5c` field, as specified by [JSON Web Signature (JWS)](https://datatracker.ietf.org/doc/html/rfc7515).

### 3.4 The `did:x509` DID Method

The `did:x509` DID method is a method that can be used to create a Decentralized Identifier (DID) based on an X.509 certificate chain.

Trust in the DID is anchored by specifying the one of the chain's intermediate, or the root CA's certificate. This trust anchor is encoded in the `did:x509` as `ca-fingerprint` property, which is the hash of the certificate. Choosing the right trust anchors (or accepted `ca-fingerprint`s) is very important, since it limits which certificates give access, and which doesn't. E.g. PKI trees often have subtrees for different user groups.

The did:x509 method is used to bind attributes of the signing certificate in the did:x509 string. The DID method defines different types of attributes via `DID policies`. 

For example following did:x509:

```
did:x509:0:sha256:WE4P5dd8DnLHSkyHaIjhp4udlkF9LqoKwCvu9gl38jk::subject:C:US:ST:California:O:My%20Organisation
```

Specifies a certificate subject to an organization based in California, issued by a CA certified identified by thumbprint `WE4P5dd8DnLHSkyHaIjhp4udlkF9LqoKwCvu9gl38jk`. It expresses the following attributes from the certificate through the `subject` DID policy:

* Subject:
  * C: US
  * ST: California
  * O: My Organisation

`did:x509` specifies the following set of DID policies (between parenthesis) and their attributes:

* Subject (subject)
  * C: Country
  * CN: Common Name
  * L: Locality
  * ST: State
  * O: Organisation
  * OU: Organisation Unit
  * STREET: Street Address
* Subject Other Name (san)
  * email
  * dns
  * uri
* Extended Key Usage (eku)
  * Any OID
* A Free-to-Use CA For Code Signing (fulcio-issuer)
  * Any issuer hostname

### 3.5 Extending the `did:x509` specification

This RFC extends the Subject Other Name (san) policy with the following attribute:

* Subject Other Name (san)
  * otherName: A free-form attribute that can be used to specify any attribute that is not covered by the other
    attributes.

The otherName attribute can be used to specify extra attributes in a X.509 certificate. This attribute is added to the specification of this RFC to cater for the use case where the san.otherName attribute is used in the X.509 certificate and plays a role in the identification of the holder of the certificate.

## 4. The X509Credential Structure

An `X509Credential` must conform to the general structure of a W3C Verifiable Credential and conform to the following rules:

- The credential MUST be in JWT proof format.
- `type`: MUST include `VerifiableCredential` and `X509Credential`.
- `issuer`: MUST be a valid `did:x509` identifier.
- `credentialSubject`: MUST only contain fields explicitly present in the `did:x509` DID policies with the fields mapped by each type as a separate map. An example: 
```
{
  "subject": {
    "O" : "My Organisation"
  },
  "san" : {
    "email" : "info@example.com"
  }
}
```

The credential subject can be identified by any DID method (e.g. `did:web`) accepted by the credential verifier.


### 4.1 Example `X509Credential`

Below is an example of an `X509Credential` issued by a `did:x509` DID. The credential subject is identified by a `did:web`.  The first snippet is the JWT header, and the second snippet is the credential payload.

```json
{
  "alg": "PS256",
  "typ": "JWT",
  "x5c": [
    "<base64 encoded leaf-certificate in the DER format>",
    "<base64 encoded intermediate CA-certificate in the DER format>",
    "<base64 encoded root CA-certificate in the DER format>"
  ],
  "kid": "did:x509:0:sha256:WE4P5dd8DnLHSkyHaIjhp4udlkF9LqoKwCvu9gl38jk::subject:O:OLVG%20Oost::subject:L:Amsterdam::san:otherName:23419943234#1"
}
```

Payload:
```json
{
  "vc": {
    "@context": [
      "https://www.w3.org/2018/credentials/v1",
      "https://nuts.nl/credentials/v1"
    ],
    "type": [
      "VerifiableCredential",
      "X509Credential"
    ],
    "issuer": "did:x509:0:sha256:WE4P5dd8DnLHSkyHaIjhp4udlkF9LqoKwCvu9gl38jk::subject:O:OLVG%20Oost::subject:L:Amsterdam::san:otherName:23419943234",
    "issuanceDate": "2024-12-01T00:00:00Z",
    "credentialSubject": {
      "id": "did:web:example.com",
      "subject:O": "Ziekenhuis Oost",
      "subject:L": "Amsterdam",
      "san:otherName": "23419943234"
    }
  }
}
```

## 5. Validation

### 5.1 Verify Credential Structure

To validate an `X509Credential`, the following steps MUST be performed:

- Verify that the credential is in JWT format.
- Verify the validity of the credential: its signature (see 5.4), revocation status if applicable.
- Verify that the issuer's DID is a `did:x509` DID.
- Verify that the DID specifies an accepted trust anchor (CA certificate) for `ca-fingerprint`.
- Resolve the `did:x509` DID document according to
  the [did:x509 specification](https://trustoverip.github.io/tswg-did-x509-method-specification/) and check the
  certificate chain for revocation.
- Validate that the `credentialSubject` fields match the policies in the `did:x509` DID.
- Validate that the credential contains the attributes required by the use case (see 5.3).

### 5.2 Validate the Issuer Certificate

The certificate associated with the `did:x509` issuer MUST be validated as follows:

- **Certificate Chain Validation**: The certificate must have a valid trust chain. The chain MUST be complete (from root to leaf) and all
  signatures need to be checked. The certificate's time-validity period must be checked (`notBefore` and `notAfter`).
- **Revocation Check**: Verify the revocation status of the certificate using CRL.
- The DID MUST specify an accepted trust anchor through its `ca-fingerprint` property and the trust anchor MUST be
  present in the certificate chain as either the Root CA or one of the Intermediate CAs. What trust anchors are accepted
  depends on the use case.

Failure to validate the issuer certificate invalidates the credential.

### 5.3 Validate the Credential Subject

The `credentialSubject` MUST be verified against the `did:x509` DID Document. Specifically:

- Every field in the `credentialSubject` MUST be present in the `did:x509` policies. The fields in the `credentialSubject` are nested with their DID policies as keys, so `subject:L:Amsterdam`, becomes: 
```json
{
  "sub" :  "did:x509:subject:O:Amsterdam",
  ...
  "credentialSubject" : {
    "id" : "did:x509:subject:O:Amsterdam",
    "subject": {
      "L" : "Amsterdam"
    }
  }
}
```
- Fields not present in the `did:x509` DID Document invalidate the credential.
- The JWT's `sub` claim MUST match the `credentialSubject.id` field.

### 5.4 Verify the Proof and signing algorithm

The cryptographic proof of the credential MUST be verified using the public key associated with the `did:x509` DID.
This involves:

- Resolving the public key from the DID Document.
- Verify that the header value of `alg` is set allowed/secure.
- Verifying the signature on the credential against the signing algorithm in the `alg` header.

### 5.5 Check Credential Expiry

The following time-related fields in the credential MUST be validated:

- `issuanceDate` MUST be equal, or later than the certificate's `notBefore` date.
- `expirationDate` MUST be equal, or earlier than the certificate's `notAfter` date.

## 6. Security Considerations

The following security considerations need to be addressed:

### 6.1 Broken Trust Chain

- If the certificate chain is not validated, attackers could present fake certificates signed by an untrusted or rogue
  Certificate Authority (CA).
- **Consequences**:
  - Users or systems may accept malicious certificates, allowing attackers to impersonate legitimate entities (e.g.,
    phishing attacks or man-in-the-middle (MitM) attacks).
  - Sensitive data (e.g., credentials, financial data) exchanged with fraudulent sites could be intercepted.

### 6.2 Revocation Checks not Performed

- If you fail to check the status of certificates for revocation using CRL (Certificate Revocation List), certificates
  that have been compromised or expired could still be considered valid.
- **Consequences**:
  - Attackers could use stolen or revoked certificates to bypass authentication or encryption.
  - Systems may continue to trust certificates issued to malicious actors.

### 6.3 Expired Certificates

- If credential expiry is ignored, certificates whose validity period has elapsed could still be used and trusted.
- **Consequences**:
  - Attackers may exploit outdated certificates to perform replay attacks where previously valid credentials are reused.
  - Trust in infrastructure degrades because expired certificates no longer reflect proper certificate holder
    responsibility/accountability.

### 6.4 Weak Keys or Algorithms

- If weak cryptographic algorithms (e.g., MD5, SHA-1) or small key sizes (e.g., <2048-bit RSA) are used, the
  certificates or their signatures could be cracked by modern computational power.
- **Consequences**:
  - An attacker could forge or spoof certificates.
  - Sensitive data could be decrypted easily, exposing confidential information such as passwords, personal data, etc.

### 6.5 Improper Credential Subject Validation

- If the `credentialSubject` field in frameworks like `X509Credential` is not properly validated, it may allow
  fields not aligned with the X.509 certificate to be added or accepted.
- **Consequences**:
  - Attackers could inject unauthorized or false data (e.g., incorrect organization name or purpose), tricking verifiers
    by impersonating trusted entities.
  - Loss of trust in the system due to inconsistencies between certificates and credentials.

### 6.6 Forged Proofs or Tampered Credentials

- Failure to verify cryptographic proofs tied to certificates could allow credentials or data to be tampered with.
- **Consequences**:
  - Credentials could be modified to grant unauthorized access.
  - The integrity of systems relying on these credentials could be compromised.

### 6.7 Missing Trust Anchor Verification

- If the trust anchor (root or any intermediate CA) is not explicitly verified and trusted, attackers could use certificates issued by
  unapproved CAs. For instance, in case of UZI server certificates, the `ca-fingerprint` must match the hash of either the Root
  CA or one of the Intermediate CAs as published by the [UZI register](https://www.zorgcsp.nl/ca-certificaten).
- **Consequences**:
  - Fake certificates signed by rogue or unvalidated CAs could be accepted as valid.
  - Attackers gain the ability to impersonate legitimate entities in scenarios such as encrypted communication or
    identity verification.

### 6.8 Certificate Misuse

- Without proper validation of certificate attributes (e.g., URA number in UZI certificates), certificates may be
  misused in unintended contexts.
- **Consequences**:
  - Fraud or impersonation using certificates outside their intended scope.
  - Misrepresentation of organizations or individuals.

### 6.9 Lack of Reliable Revocation Handling

- If revocation checks poorly handle network issues or failures, it could result in
  the acceptance of revoked or invalid certificates.
- **Consequences**:
  - Increased risk of improper trust, allowing revoked credentials to function within the system.
  - Security-critical applications become susceptible to breaches.

## 7. References

- [Verifiable Credentials Data Model v1.1](https://www.w3.org/TR/vc-data-model/)
- [DID:X509 Method Specification](https://trustoverip.github.io/tswg-did-x509-method-specification/)
- [X.509 Certificate Revocation (OCSP/CRL)](https://datatracker.ietf.org/doc/html/rfc5280)
- [JSON Web Signature (JWS)](https://datatracker.ietf.org/doc/html/rfc7515)
