# RFC023: NutsX509Credential

|                           |               |
|:--------------------------|:--------------|
| Nuts foundation           | R.G. Krul     |
| Request for Comments: 023 | Zorg bij jou  |
|                           | R. Groen      |
|                           | Zorg bij jou  |
|                           | December 2024 |

## Abstract

Trust is a key element in the Nuts network. This RFC describes how x509 certificates can be used to source trust from
outside the Nuts network. The x509 certification process has been around for a long time and is widely used in the
internet. This RFC describes how x509 certificates can be used in the Nuts network to establish trust between parties by
being able to link the x509 certificate to a Nuts identity by as a Verifiable Credential that is issued by the holder of
the x509 identity.

This RFC specifies the requirements and validation process for the `NutsX509Credential`, a W3C Verifiable Credential (
VC) type issued by the subject of a x509 certificate, represented by a `did:x509` DID. The `NutsX509Credential` ensures
strong alignment with the properties of the associated X.509 certificate and defines mechanisms to validate the
credential and verify its association with a `did:x509` DID.

## Status of this document

This document is currently in draft. Feedback is welcome to improve the interoperability and robustness of the
specification.

### Copyright Notice

![](../.gitbook/assets/license.png)

## Introduction

TThe [did:x509](https://trustoverip.github.io/tswg-did-x509-method-specification/) method aims to achieve interoperability between existing X.509 solutions and Decentralized Identifiers (DIDs) to support operational models in which a full transition to DIDs is not achievable or desired yet. It supports X.509-only verifiers as well as DID-based verifiers supporting this DID method.

The `NutsX509Credential` is a W3C Verifiable Credential type designed for use cases where trust anchors are based on X.509
certificates. It leverages the `did:x509` method, as specified in
the [Trust Over IP DID:X509 Method Specification](https://trustoverip.github.io/tswg-did-x509-method-specification/).

By aligning credential subject validation with the fields of the associated `did:x509` DID and enforcing
certificate revocation checks, the `NutsX509Credential` ensures integrity and adherence to the PKI trust model.


## Definitions

- **NutsX509Credential**: A Verifiable Credential whose issuer is a `did:x509` DID and whose structure adheres to the
  requirements in this document.
- **did:x509**: A Decentralized Identifier (DID) method specified by the Trust Over IP Foundation, where the DID is
  derived from an X.509 certificate.
- **Issuer Certificate**: The X.509 certificate associated with the `did:x509` DID that issued the credential.
- **Credential Subject**: The entity described by the credential.
- **Revocation Check**: The process of verifying the revocation status of the issuer certificate using mechanisms like
  OCSP or CRL.

## Used standards and technologies

This RFC builds on the following standards and technologies:

* [X.509 Certificate Standard](https://datatracker.ietf.org/doc/html/rfc5280)
* [JSON Web Signature (JWS)](https://datatracker.ietf.org/doc/html/rfc7515#section-4.1.6)
* [did:x509 method specification](https://trustoverip.github.io/tswg-did-x509-method-specification/), with modifications
* [Verifiable Credentials Data Model v1.1](https://www.w3.org/TR/vc-data-model/)

### x509 certificates, a brief introduction

An **X.509 certificate** is a digital certificate that follows the X.509 Public Key Infrastructure (PKI) standard. It is widely used for secure communication over the internet, such as HTTPS, email encryption, and digital signatures.
#### Key Features of X.509 Certificates:
- **Structure**: It contains information about the certificate owner (e.g., organization, common name, public key) and the issuing Certificate Authority (CA). The certificate is signed by the CA's private key to ensure authenticity.
- **Trust Hierarchy**:
    - Trust is anchored in a **Certificate Authority (CA)**, which is a trusted third party.
    - Certificates can form a chain of trust, starting from a **root certificate** (trusted CA) to **intermediate certificates**, down to the **end-user/client certificates** (specific use-case certificates).

Example of the trust chain hierarchy:


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
### Nuts and X.509:
The Nuts framework extends X.509 certificates into its decentralized identity (DID) ecosystem using the `did:x509` DID method. This method facilitates linking an X.509 certificate to a verifiable credential (`NutsX509Credential`), providing trust while bridging traditional PKI with decentralized trust models. This is especially relevant in systems reliant on existing X.509 implementations, such as healthcare or government frameworks.

### Using x509 for signing JWTs

The JWT is a standard that is used to sign and encrypt JSON objects. Thus standard allows for the signing and encryption
of JSON objects with certificates part of a certificate chain. This allows for the signing of JSON objects with the
private key of the certificate and the verification of the signature with the public key of the certificate, and the
verification of the certificate chain with the public key of the CA. This is done by using the following headers fields:

* x5c, the ordered certificate chain as a list of base64 encoded certificates in the DER format, with the signing certificate
  first and the root certificate last.
* x5t#S256, the thumbprint of the signing certificate as a SHA256 hash.

### The `did:x509` DID Method

The `did:x509` DID method is a method that can be used to create a Decentralized Identifier (DID) based on an x509
certificate chain. Trust in the DID is anchored by specifying the (hash of) one of the chain's intermediate, or the root CA's certificate. The did:x509 method
is used to specify specific attributes of the signing certificate to specify the holder of the signing
certificate. By doing this, a did:x509 DID can be used to identify the holder of the signing certificate by specificying
attributes that are assigned to the signing certificate. So, for example following did:x509:

```
did:x509:0:sha256:WE4P5dd8DnLHSkyHaIjhp4udlkF9LqoKwCvu9gl38jk::subject:C:US:ST:California:O:My%20Organisation
```

ties down the holder of the signing certificate by, first having a digitally signed certificate by the CA with the
thumbprint `WE4P5dd8DnLHSkyHaIjhp4udlkF9LqoKwCvu9gl38jk` and then having the following attributes in the certificate:

* Subject:
  * C: US
  * ST: California
  * O: My Organisation

The did:x509 defines various attribute types that can be used as attributes, such as:

* Subject
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

### Extending the `did:x509` specification

This RFC extends the Subject Other Name (san) policy with the following attribute:

* Subject Other Name (san)
  * otherName: A free-form attribute that can be used to specify any attribute that is not covered by the other
    attributes.

The otherName attribute can be used to specify extra attributes in a x509 certificate. This attribute is added to the specification of this RFC to cater for the use case where the san.otherName attribute is used in the x509 certificate and plays a role in the identification of the holder of the certificate.

## The NutsX509Credential Structure

An `NutsX509Credential` must conform to the general structure of a W3C Verifiable Credential and conform to the following
rules:

- The credential MUST be in JWT format.
- `type`: MUST include `VerifiableCredential` and `NutsX509Credential`.
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

### Example `NutsX509Credential`

Below is an example of an `NutsX509Credential` issued by a `did:x509` DID. The credential subject is identified by a
`did:web`.
The first snippet is the JWT header, and the second snippet is the credential payload.

(TODO: check this example)

```json
{
  "alg": "PS256",
  "typ": "JWT",
  "x5c": [
    "<base64 encoded leaf-certificate in the DER format>",
    "<base64 encoded issuer-intermediate-certificate in the DER format>",
    "<base64 encoded issuer-certificate in the DER format>"
  ],
  "x5t": "<thumbprint>",
  "kid": "did:x509:0:sha256:WE4P5dd8DnLHSkyHaIjhp4udlkF9LqoKwCvu9gl38jk::subject:O:OLVG%20Oost::subject:L:Amsterdam::san:otherName:23419943234#1"
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
      "NutsX509Credential"
    ],
    "issuer": "did:x509:0:sha256:WE4P5dd8DnLHSkyHaIjhp4udlkF9LqoKwCvu9gl38jk::subject:O:OLVG%20Oost::subject:L:Amsterdam::san:otherName:23419943234",
    "issuanceDate": "2024-12-01T00:00:00Z",
    "credentialSubject": {
      "id": "did:web:example.com",
      "subject:O": "OLVG Oost",
      "subject:L": "Amsterdam",
      "san:otherName": "23419943234"
    }
  }
}
```

## Validation

### 1. Verify Credential Structure

To validate an `NutsX509Credential`, the following steps MUST be performed:

- Verify that the credential is in JWT format.
- Verify that the issuer's DID is a `did:x509` DID.
- Verify that the DID specifies an accepted trust anchor (CA certificate) for `ca-fingerprint`.
- Resolve the `did:x509` DID document according to
  the [did:x509 specification](https://trustoverip.github.io/tswg-did-x509-method-specification/) and check the
  certificate chain for revocation.
- Validate that the `credentialSubject` fields match the policies in the `did:x509` DID.

### 2. Validate the Issuer Certificate

The certificate associated with the `did:x509` issuer MUST be validated as follows:

- **Certificate Chain Validation**: The certificate must have a valid trust chain. The chain MUST be complete and all
  signatures need to be checked.
- **Revocation Check**: Verify the revocation status of the certificate using CRL.
- The DID MUST specify an accepted trust anchor through its `ca-fingerprint` property and the trust anchor MUST be
  present in the certificate chain as either the Root CA or on of the Intermediate CAs. What trust anchors are accepted
  depends on the use case.

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

The following security considerations need to be addressed:

### **1. Broken Trust Chain**

- If the certificate chain is not validated, attackers could present fake certificates signed by an untrusted or rogue
  Certificate Authority (CA).
- **Consequences**:
  - Users or systems may accept malicious certificates, allowing attackers to impersonate legitimate entities (e.g.,
    phishing attacks or man-in-the-middle (MitM) attacks).
  - Sensitive data (e.g., credentials, financial data) exchanged with fraudulent sites could be intercepted.

### **2. Revocation Checks Not Performed**

- If you fail to check the status of certificates for revocation using CRL (Certificate Revocation List), certificates
  that have been compromised or expired could still be considered valid.
- **Consequences**:
  - Attackers could use stolen or revoked certificates to bypass authentication or encryption.
  - Systems may continue to trust certificates issued to malicious actors.

### **3. Expired Certificates**

- If credential expiry is ignored, certificates whose validity period has elapsed could still be used and trusted.
- **Consequences**:
  - Attackers may exploit outdated certificates to perform replay attacks where previously valid credentials are reused.
  - Trust in infrastructure degrades because expired certificates no longer reflect proper certificate holder
    responsibility/accountability.

### **4. Weak Keys or Algorithms**

- If weak cryptographic algorithms (e.g., MD5, SHA-1) or small key sizes (e.g., <2048-bit RSA) are used, the
  certificates or their signatures could be cracked by modern computational power.
- **Consequences**:
  - An attacker could forge or spoof certificates.
  - Sensitive data could be decrypted easily, exposing confidential information such as passwords, personal data, etc.

### **5. Improper Credential Subject Validation**

- If the `credentialSubject` field in frameworks like `NutsX509Credential` is not properly validated, it may allow
  fields not aligned with the X.509 certificate to be added or accepted.
- **Consequences**:
  - Attackers could inject unauthorized or false data (e.g., incorrect organization name or purpose), tricking verifiers
    by impersonating trusted entities.
  - Loss of trust in the system due to inconsistencies between certificates and credentials.

### **6. Forged Proofs or Tampered Credentials**

- Failure to verify cryptographic proofs tied to certificates could allow credentials or data to be tampered with.
- **Consequences**:
  - Credentials could be modified to grant unauthorized access.
  - The integrity of systems relying on these credentials could be compromised.

### **7. Missing Root CA Verification**

- If the source of trust (Root CA) is not explicitly verified and trusted, attackers could use certificates issued by
  unapproved CAs.For instance, in case of UZI certificates the `ca-fingerprint` must match the hash of either the Root
  CA or one of the Intermediate CAs as published by the [UZI register](https://www.zorgcsp.nl/ca-certificaten).
- **Consequences**:
  - Fake certificates signed by rogue or unvalidated CAs could be accepted as valid.
  - Attackers gain the ability to impersonate legitimate entities in scenarios such as encrypted communication or
    identity verification.

### **8. Certificate Misuse**

- Without proper validation of certificate attributes (e.g., URA number in UZI certificates), certificates may be
  misused in unintended contexts.
- **Consequences**:
  - Fraud or impersonation using certificates outside their intended scope.
  - Misrepresentation of organizations or individuals.

### **9. Lack of Reliable Revocation Handling**

- If revocation checks poorly handle network issues or failures (e.g., OCSP response unavailability), it could result in
  the acceptance of revoked or invalid certificates.
- **Consequences**:
  - Increased risk of improper trust, allowing revoked credentials to function within the system.
  - Security-critical applications become susceptible to breaches.

## References

- [Verifiable Credentials Data Model v1.1](https://www.w3.org/TR/vc-data-model/)
- [DID:X509 Method Specification](https://trustoverip.github.io/tswg-did-x509-method-specification/)
- [X.509 Certificate Revocation (OCSP/CRL)](https://datatracker.ietf.org/doc/html/rfc5280)

# An application of the RFC023: UZI server certificates

## PKI overheid & UZI server certificates

The Dutch government has a Public Key Infrastructure (PKI) that is used to establish trust between parties. The PKI
framework is currently in place and makes use of PKI Overheid Certificates issued by the root CAs of the Dutch
government. In healthcare a specific instance of PKI overheid certificates are issued: the UZI certificates. These
certificates are used to establish trust between parties in the healthcare sector. The UZI certificates are issued by
the UZI register, which is a trusted party that is capable of verifying the identity of the holder of the certificate.
The UZI register signs the certificate with its own private key. The holder of the certificate can then use the public
key of the UZI register to verify the signature of the certificate. This way the holder of the certificate can prove
that the certificate is valid and that the information in the certificate is correct. The UZI certificates are issued
to:

* Individuals that work in healthcare, such as doctors, nurses, etc. They hold this certificate on a UZI card.
* Organisations that work in healthcare, such as hospitals, pharmacies, etc. They hold this certificate as server
  certificates.

#### UZI certificate structure for organisations

The UZI certificate is used to identify the holder of the certificate. The UZI certificate contains information about
the holder of the certificate. This information is used to identify the holder of the certificate. The UZI certificate
contains the following information (of intrest):

* The `subject.CN` The full FQN.
* The `subject.O` the name of the holder of the certificate.
* The `subject.serialNumber ` The URI number
* The `subject.C` The subject country
* The `subject.ST` The subject state
* The `subject.L` The subject locality (city)
* The `subject.CN` the full FQN.
* The `san.otherName` a string containing `<OID CA>-<versie-nr>-<UZI-nr>-<pastype>-<Abonnee-nr>-<rol>-<AGB-code>`,
  where:
  * `<OID CA>` is the OID of the CA that issued the certificate, `2.16.528.1.1007.99.2110` for CIBG.
  * `<versie-nr>` is the version number of the certificate.
  * `<UZI-nr>` is the UZI number of the holder of the certificate, same as `subject.serialNumber`.
  * `<pastype>` is the type of the holder of the certificate, always `S`.
  * `<Abonnee-nr>` is the subscriber URA of the holder of the certificate.
  * `<rol>` is the role of the holder of the certificate, always "0.00"
  * `<AGB-code>` is the AGB code of the holder of the certificate.

## Mapping UZI certificate to NutsX509Credential

The mapping of certificates to x509 is depending

### The ROOT G3

The `did:x509` specification dictates that the fingerprint of the Root CA is part of the did:x509. For mapping an UZI
certificate to an NutsX509Credential the ROOT CA MUST match one of the certificates in the UZI ROOT CA register hierarchy.
For G3 this is:

```asciidoc
        ┌────────────────────────────────────┐        
        │ Staat der Nederlanden Root CA - G3 │        
        └────────────────┬───────────────────┘        
                         │                            
┌────────────────────────▼───────────────────────────┐
│ Staat der Nederlanden Organisatie Services CA - G3 │
└────────────────────────┬───────────────────────────┘
                         │                            
    ┌────────────────────▼───────────────────────┐    
    │ UZI-register Medewerker niet op naam CA G3 │    
    └────────────────────────────────────────────┘    
```

### Field mapping of the UZI credential

The following fields are commonly used for mapping UZI certificates to NutsX509Credentials
* The `subject.O` the name of the holder of the certificate. Maps to `subject.O` in the NutsX509Credential.
* The `subject.L` The subject locality (city)
* The `san.otherName` a string containing `<OID CA>-<versie-nr>-<UZI-nr>-<pastype>-<Abonnee-nr>-<rol>-<AGB-code>`,
  where:
  * `<OID CA>` is the OID of the CA that issued the certificate, `2.16.528.1.1007.99.2110` for CIBG.
  * `<versie-nr>` is the version number of the certificate.
  * `<UZI-nr>` is the UZI number of the holder of the certificate, same as `subject.serialNumber`.
  * `<pastype>` is the type of the holder of the certificate, always `S`.
  * `<Abonnee-nr>` is the subscriber URA of the holder of the certificate.
  * `<rol>` is the role of the holder of the certificate, always "0.00"
  * `<AGB-code>` is the AGB code of the holder of the certificate.

## The use of UZI server certificate in the Nuts network or identifying organizations

The focus on trust in the NUTS network for organizations lies primarily on the URA number identified as the
`<Abonnee-nr>` on the UZI certificate. This number is used to identify the subject of the certificate within the Dutch
healthcare ecosystem . The subject of the certificate can use the UZI certificate in combination with the private
key to proof the ownership of the URA number. The diagram below shows how the UZI certificate can be used to transfer
the trust from the UZI register acting as "authentieke bron" into the NUTS ecosystem using the `did:x509` method and the `NutsX509Credential` Verifiable
Credential.

```asciidoc
                            ┌─────────┐       ┌──────────┐                                        
                            │ Keypair ┼───────┤ did:x509 │                                        
                            └────┬────┘       └────┬─────┘                                        
                                 │                 │                                              
                                 │                 │                                              
┌───────────┐            ┌───────┴───────┐         │                                              
│  ROOT CA  │            │     UZI       │ ┌───────────────────┐            ┌────┐                
└─────┬─────┘            │  Certificate  │ │ NutsX509Credential┼────────────► VP │                
      │                  └───────────────┘ └───────────────────┘            └─┬──┘                
      │                          │                 │  │    ┌────────────┐     │     ┌────────────┐
┌─────┴─────┐ 1.Request  ┌───────┴───────┐         │  │    │            │     │     │            │
│ Authentic ◄────────────┼   Holder of   ┼─────────┴──┼────►   Wallet   ┼─────┴─────►  Verifier  │
│ Source of ┌────────────►     Trust     │   3.Issue  │    │            │ 4.Present │            │
│   Trust   │  2.Issue   └───────────────┘            │    └──────┬─────┘           └────────────┘
└───────────┘                                         │           │                               
                                                      │           │                               
                                              ┌───────┴─┐    ┌────┴────┐                          
                                              │ did:web ├────┤ Keypair │                          
                                              └─────────┘    └─────────┘                          
```

This diagram represents the process of establishing trust, based on the use of X.509 certificates, the
`NutsX509Credential` and `did:x509` within a trust network. Below is a step-by-step explanation of the diagram:

### **Key Components**

1. **Root CA**:
  - The starting point for trust. The Root Certificate Authority (CA) is a trusted source that issues and signs
    certificates to intermediate or end-user entities.

2. **UZI Certificate**:
  - A specific X.509 certificate issued by the Root CA (or its intermediaries) to establish trust for the holder (e.g.,
    an organization or an individual).

3. **Keypair**:
  - Generated by the certificate holder, this is the private-public key pair required for signing and authentication
    processes.

4. **did:x509**:
  - A Decentralized Identifier (DID) based on an X.509 certificate. It links decentralized systems with the trust of
    traditional X.509 certificates.

5. **NutsX509Credential**:
  - A Verifiable Credential (VC), such as a "NutsX509Credential," which is issued by the certificate holder using its
    `did:x509` identifier and signed with the corresponding keypair. This credential is stored in the holder's wallet.

6. **Wallet**:
  - A secure digital storage system for holding the NutsX509Credential. It manages credentials and is used for
    presenting them to verifiers.

7. **Verifier**:
  - An entity that validates the presented credential and establishes the holder's identity based on its associated
    trust components (e.g., did:x509, certificate chain, etc.).

8. **did:web**:
  - Another Decentralized Identifier (DID) the holder may use to represent their identity and interact within the
    decentralized trust ecosystem.

### **Process Steps**

#### **Step 1: Keypair Generation and Request**

The  **holder** generates a keypair (private and public key) to represent their identity. They submit the
public key as part of a **Certificate Signing Request (CSR)** to the Root CA (or intermediate CA). Within the CSR
terminology, the **holder** is the **subject** of the CSR.

#### **Step 2: Certificate Issuance**

The Root CA (or its intermediate CA) verifies the request and issues an **X.509 certificate** (e.g., a UZI certificate)
to the subject. This certificate includes information about the subject (e.g., subject name and organization) and is
signed by the CA. This guarantees the authenticity of the certificate. Note that the **holder** and **subject** are the
same concepts but are named differently between the different terminologies.

#### **Step 3: NutsX509Credential Issuance**

The **holder** uses their X.509 certificate to create a **NutsX509Credential or Verifiable Credential**. The process
includes:

1. Using the certificate's `did:x509` identifier as the credential's **issuer**.
2. Signing this credential with the holder's private key (from the keypair).
3. Storing the credential securely in the **Wallet** for future use.

#### **Step 4: Credential Presentation**

When the holder needs to prove their identity to a verifier (e.g., during authentication), they present the *
*NutsX509Credential** from their wallet to the **Verifier**. This process includes:

1. The presentation of the digital credential as a Verifiable Presentation (VP).
2. Signing the presentation with the holder's private key to ensure it hasn't been tampered with.

#### **Step 5: Verification**

The **Verifier** validates the credential and presentation. This includes:

1. Checking the integrity of the credential and presentation signature.
2. Confirming the certificate chain back to the **Root CA** to ensure the issuer of the X.509 certificate is authentic
   and trusted.
3. Validating the use case-specific attributes in the credential (e.g., fields like organization, UZI number, or other
   subject information).
4. Ensuring the credential has not been revoked using methods like CRL.

#### Trust is Established:

If all checks pass, the Verifier trusts the credential presented by the holder. The credential's trustworthiness is
derived from:

1. The Root CA that anchors trust.
2. The validity of the X.509 certificate and the associated DID (`did:x509`).
3. Alignment of attributes between the X509Credential and the certificate.
