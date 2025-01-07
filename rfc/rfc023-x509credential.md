# RFC023: X509Credential

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

This RFC specifies the requirements and validation process for the `X509Credential`, a W3C Verifiable Credential (VC)
type issued by the holder of a x509 certificate, represented by a `did:x509` DID. The `X509Credential` ensures strong
alignment with the properties of the associated X.509 certificate and defines mechanisms to validate the credential and
verify its association with a `did:x509` DID.

## Status of this document

This document is currently in draft. Feedback is welcome to improve the interoperability and robustness of the
specification.

### Copyright Notice

![](../.gitbook/assets/license.png)

## Introduction

The Nuts network is a network of trust. The trust is established by the use of Verifiable Credentials. These credentials
are issued by a trusted party and can be used to establish trust between parties. The Nuts network is a decentralised
network and the trust is established between parties that are not necessarily known to each other. The trust is
established by the use of Verifiable Credentials that are issued by trusted sources. Members of the Nuts network can
then trust their peers by verifying the Verifiable Credentials that are presented to them.

At this time of writing, there are not many sources of trust available that act as trusted source of identity AND that
are capable of providing such trust in the form of Verifiable Credentials. Even tough work is being done in this area.
Most trusted sources in The Netherlands make use of systems like the x509 certificates to establish trust. This RFC
describes how x509 certificates can be used to establish trust in the Nuts network by bridging the gap between the x509
certificates and the Nuts network. This is done by issuing a Verifiable Credential that is based on the x509 certificate
that makes use of the did:x509 method.

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

## Used standards and technologies

This RFC builds on the following standards and technologies:

* [X.509 Certificate Standard](https://datatracker.ietf.org/doc/html/rfc5280)
* [JSON Web Signature (JWS)](https://datatracker.ietf.org/doc/html/rfc7515#section-4.1.6)
* [did:x509 method specification](https://trustoverip.github.io/tswg-did-x509-method-specification/), with modifications
* W3C Verifiable Credentials Data Model

### x509 certificates, a brief introduction

The structure of an x509 certificate is defined by the X.509 standard. An x509 certificate contains information about
the holder and is signed by a Certificate Authority (CA). The CA is a trusted party that is capable of verifying the
identity of the holder of the certificate. The CA signs the certificate with its own private key. The holder of the
certificate can then use the public key of the CA to verify the signature of the certificate. This way the holder of the
certificate can prove that the certificate is valid and that the information in the certificate is correct.

The verifier of a x509 certificate can then trust the information in the certificate by verifying the signature of the
certificate chain of the certificate. The verifier can then trust the information in the certificate by trusting the CA
that signed the certificate.

The chain of certificates can be viewed as a hierarchy, where the root certificate is the certificate is trusted, and
signing is delegated to intermediate certificates. The root certificate is the certificate that is trusted by the
holder. The holder maintains a list of trusted CAs that the holder trusts. The holder can then
verify the signature of the certificate chain by verifying the signature of the CA that signed the intermediate
certificate and the intermediate certificates that lead to the signing certificate.

```asciidoc
┌────────────────────┐
│        CA          │
└─────────┬──────────┘
          │           
┌─────────▼──────────┐
│    Intermediate    │
└─────────┬──────────┘
          │           
┌─────────▼──────────┐
│    Intermediate    │
└─────────┬──────────┘
          │           
┌─────────▼──────────┐
│Signing Certificate │
└────────────────────┘
```

### Using x509 for signing JWEs

The JWE is a standard that is used to sign and encrypt JSON objects. Thus standard allows for the signing and encryption
of JSON objects with certificates part of the a certificate chain. This allows for the signing of JSON objects with the
private key of the certificate and the verification of the signature with the public key of the certificate, and the
verification of the certificate chain with the public key of the CA. This is done by using the following headers fields:

* x5c, the certificate chain as a list of base64 encoded certificates in the DER format, with the signing certificate
  first and the root certificate last.
* x5t, the thumbprint of the signing certificate.
* x5t#S256, the thumbprint of the signing certificate as a SHA256 hash.

### The `did:x509` DID Method

The `did:x509` DID method is a method that can be used to create a Decentralized Identifier (DID) based on an x509
certificate chain. This is done by creating a DID that is based on the root CA of the certificate chain. The did:x509
method is used to specify specific attributes of the signing certificate to specify the holder of the signing
certificate. By doing this, a did:x509 DID can be used to identify the holder of the signing certificate by specifica
attributes that are assigned to the signing certificate. So, for example following did:x509:

```
did:x509:0:sha256:WE4P5dd8DnLHSkyHaIjhp4udlkF9LqoKwCvu9gl38jk::subject:C:US:ST:California:O:My%20Organisation
```

ties down the holder of the signing certificate by, first having a digitally signed certificate by the root CA with the
thumbprint `WE4P5dd8DnLHSkyHaIjhp4udlkF9LqoKwCvu9gl38jk` and then having the following attributes in the certificate:

* Subjects:
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

### Extending the x509 specification

This RFC extends the Subject Other Name (san) attribute with the following attribute:

* Subject Other Name (san)
  * otherName: A free-form attribute that can be used to specify any attribute that is not covered by the other
    attributes.

The otherName attribute can be used to specify extra attributes in a x509 certificate. This attribute is added to the specification of this RFC to cater for the use case where the san:otherName attribute is used in the x509 certificate and plays a role in the identification of the holder of the certificate.

## The X509Credential Structure

An `X509Credential` must conform to the general structure of a W3C Verifiable Credential and conform to the following
rules:

- The credential MUST be in JWT format.
- `type`: MUST include `VerifiableCredential` and `X509Credential`.
- `issuer`: MUST be a valid `did:x509` identifier.
- `credentialSubject`: MUST only contain fields explicitly present in the `did:x509` DID policies with the format <
  policy_type>:<policy_attribute>, for example `subject:O` or `san:otherName`.

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
    "<base64 encoded leaf-certificate in the DER format>",
    "<base64 encoded issuer-intermediate-certificate in the DER format>",
    "<base64 encoded issuer-certificate in the DER format>"
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
- Resolve the `did:x509` DID document according to
  the [did:x509 specification](https://trustoverip.github.io/tswg-did-x509-method-specification/) and check the
  certificate chain for revocation.
- Validate that the `credentialSubject` fields match the policies in the `did:x509` DID.

### 1. Verify Credential Structure

Ensure that the credential:

- Includes the `X509Credential` type.
- Contains a valid `did:x509` issuer.
- Includes a `credentialSubject` whose fields match the `did:x509` DID Document.

### 2. Validate the Issuer Certificate

The certificate associated with the `did:x509` issuer MUST be validated as follows:

- **Certificate Chain Validation**: The certificate must have a valid trust chain. The use case determines if the CA is
  trusted.
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

The following security considerations are to be considered:

- The Root CA of the did:x509 needs to be checked against the root CA structure of the use case. For instance, in case
  of UZI certificates the ROOT CA must match the associated root CA chain.

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

## PKI overheid & UZI certificates

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
* The `subject:O` the name of the holder of the certificate.
* The `subject.serialNumber ` The URI number
* The `subject.C` The subject country
* The `subject.ST` The subject state
* The `subject.L` The subject locality (city)
* The `subject.commonName` the full FQN.
* The `san:dNSName` the DNS name of the holder of the certificate.
* The `san:otherName` a string containing `<OID CA>-<versie-nr>-<UZI-nr>-<pastype>-<Abonnee-nr>-<rol>-<AGB-code>`,
  where:
  * `<OID CA>` is the OID of the CA that issued the certificate, `2.16.528.1.1007.99.2110` for CIBG.
  * `<versie-nr>` is the version number of the certificate.
  * `<UZI-nr>` is the UZI number of the holder of the certificate, same as `subject.serialNumber`.
  * `<pastype>` is the type of the holder of the certificate, always `S`.
  * `<Abonnee-nr>` is the subscriber URA of the holder of the certificate.
  * `<rol>` is the role of the holder of the certificate, always "0.00"
  * `<AGB-code>` is the AGB code of the holder of the certificate.

## Mapping UZI certificate to X509Credential

### The ROOT Ca

The `did:x509` specification dictates that the fingerprint of the Root CA is part of the did:x509. For mapping an UZI
certificate to an X509Credential the ROOT CA MUST match one of the certificates in the UZI register hierarchy. 

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

### Field mapping
The following fields are commonly used for mapping UZI cetificates to X509Credentials
* The `subject:O` the name of the holder of the certificate.
* The `subject.L` The subject locality (city)
* The `san:otherName` a string containing `<OID CA>-<versie-nr>-<UZI-nr>-<pastype>-<Abonnee-nr>-<rol>-<AGB-code>`,
  where:
  * `<OID CA>` is the OID of the CA that issued the certificate, `2.16.528.1.1007.99.2110` for CIBG.
  * `<versie-nr>` is the version number of the certificate.
  * `<UZI-nr>` is the UZI number of the holder of the certificate, same as `subject.serialNumber`.
  * `<pastype>` is the type of the holder of the certificate, always `S`.
  * `<Abonnee-nr>` is the subscriber URA of the holder of the certificate.
  * `<rol>` is the role of the holder of the certificate, always "0.00"
  * `<AGB-code>` is the AGB code of the holder of the certificate.

## The use of UZI server certificate in the Nuts network

The focus on trust in the NUTS network for organizations lies primarily on the URA number identified as the
`<Abonnee-nr>` on the UZI certificate. This number is used to identify the holder of the certificate within the Dutch
healthcare ecosystem . The holder of the certificate can use the UZI certificate in combination with the private
key to proof the ownership of the URA number. The diagram below shows how the UZI certificate can be used to transfer
the trust from the CIBG register into the NUTS ecosystem using the `did:x509` method and the `X509Credential` Verifiable
Credential.

```asciidoc
                            ┌─────────┐       ┌──────────┐                                        
                            │ Keypair ┼───────┤ did:x509 │                                        
                            └────┬────┘       └────┬─────┘                                        
                                 │                 │                                              
                                 │                 │                                              
┌───────────┐            ┌───────┴───────┐         │                                              
│  ROOT CA  │            │     UZI       │ ┌───────┴────────┐               ┌────┐                
└─────┬─────┘            │  Certificate  │ │ X509Credential ┼───────────────► VP │                
      │                  └───────────────┘ └───────┬──┬─────┘               └─┬──┘                
      │                          │                 │  │    ┌────────────┐     │     ┌────────────┐
┌─────┴─────┐  Request   ┌───────┴───────┐         │  │    │            │     │     │            │
│ Source of ◄────────────┼   Holder of   ┼─────────┴──┼────►   Wallet   ┼─────┴─────►  Verifier  │
│   Trust   ┌────────────►     Trust     │       Issue│    │            │   Present │            │
└───────────┘   Issue    └───────────────┘            │    └──────┬─────┘           └────────────┘
                                                      │           │                               
                                                      │           │                               
                                              ┌───────┴─┐    ┌────┴────┐                          
                                              │ did:web ├────┤ Keypair │                          
                                              └─────────┘    └─────────┘                          
```

The main steps in the diagram are:

* The holder generates a keypair and requests a UZI certificate from the UZI register with a Certificate Signing
  Request (CSR).
* The UZI register issues the certificate to the holder of the UZI certificate and signs the request with the
  intermediate CA, linked to the root CA.
* The holder of the UZI creates a `X509Credential` Verifiable Credential:
  * The holder set the `did:x509` of the UZI certificate as issuer to the `X509Credential` Verifiable Credential.
  * The holder includes the complete chain in the `X509Credential` Verifiable Credential.
  * The holder issues the `X509Credential` Verifiable Credential to its own NUTS identity as `did:web` .
  * The holder signs the `X509Credential` Verifiable Credential with the keypair associated with the UZI certificate.
* The holder places the `X509Credential` Verifiable Credential in the wallet.
* The holder presents the `X509Credential` Verifiable Credential to the verifier, and signs the presentation with the
  keypair associated with the `did:web` of the holder.
* The verifier now can verify that:
  * The `X509Credential` Verifiable Credential is issued by a `did:x509` issued by the the UZI register.
  * The `X509Credential` Verifiable Credential is signed by the holder of the UZI certificate.
  * The attributes of the `X509Credential` Verifiable Credential match the attributes of the UZI certificate.
  * The URA number of the holder of the UZI certificate is present in the `X509Credential` Verifiable Credential.

