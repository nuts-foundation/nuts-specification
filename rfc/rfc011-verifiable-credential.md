# RFC011 Verifiable Credential

|  |  |
| :--- | :--- |
| Nuts foundation | W.M. Slakhorst |
| Request for Comments: 011 | Nedap |
|  | February 2021 |

## RFC011 Verifiable Credentials

### Abstract

This RFC describes the generic requirements for [Verifiable Credentials \(VC\)](https://www.w3.org/TR/vc-data-model/) within the Nuts specification. It also lists all required chapters for a specific VC RFC. This RFC uses the [Verifiable Credential Data Model](https://www.w3.org/TR/vc-data-model/) specification.

### Status

This document is currently a draft.

### Copyright Notice

![](../.gitbook/assets/license.png)

This document is released under the [Attribution-ShareAlike 4.0 International \(CC BY-SA 4.0\) license](https://creativecommons.org/licenses/by-sa/4.0/).

## 1. Introduction

It's to be expected that multiple sources must be supported that can claim certain information about a subject. These claims have to be structured in such a way, that they are verifiable and searchable. Examples of claims are a company name, and it's chamber of commerce number. Users will search on the name of an organization to find out which services are supported. This will directly influence the interaction the user can have with the system of that organization. Making sure this information is correct and that it can be trusted is therefore extremely important.

## 2. Terminology

* **DID**: [Decentralized Identifiers](https://www.w3.org/TR/did-core/).
* **holder**: The party that receives a VC from an issuer.  
* **issuer**: The party that issues a VC.  
* **proof**: Proof asserting that a VC is valid using cryptography.
* **VC**: Verifiable Credential according to the [Verifiable Credential Data Model](https://www.w3.org/TR/vc-data-model/).
* **VP**: Verifiable Presentation according to the [Verifiable Credential Data Model](https://www.w3.org/TR/vc-data-model/).

## 3. W3C Verifiable Credential

To support a variety of claims, the [W3C Verifiable Credential specification](https://www.w3.org/TR/vc-data-model/) is used. Every Nuts specific VC and proof type must follow the W3C specification. If the W3C specification offers options, then the specific VP must specify which option is to be used. All VCs MUST use DIDs as specified in [RFC004](https://github.com/nuts-foundation/nuts-specification/tree/cf30c150a86ad3c717840873af1c9c1a547a4076/rfc/rfc004-distributed-document-format.md) for the issuer.

### 3.1 Supported proofs

To ensure the authenticity and integrity of the VC, a VC document MUST contain a signature in a `proof` section as described in section 2.2.1 of [Verifiable Credential Data Integrity Specification](https://w3c.github.i.o/vc-data-integrity/#signatures). The following proof types must be supported by a Nuts node implementation:

#### 3.1.1 JsonWebSignature2020

This signature suite is specified in [https://w3c-ccg.github.io/lds-jws2020](https://w3c-ccg.github.io/lds-jws2020). It uses a detached JWS for presenting the signature in the `jws` part of the proof. [RFC7797](https://tools.ietf.org/html/rfc7797) describes how a detached JWS works. The hashing algorithm should be of type `ES256`.

The proof MUST have a `verificationMethod` which contains an assertMethod ID from a resolvable Nuts DID Document of a public key of the `JSONWebKey2020` type. 

```javascript
{
  "type": "JsonWebSignature2020",
  "proofPurpose": "assertionMethod",
  "verificationMethod": "did:nuts:EgFjg8zqN6eN3oiKtSvmUucao4VF18m2Q9fftAeANTBd#twlH6rB8ArZrknmBRWLXhao3FutZtvOm0hnNhcruenI",
  "created": "2021-03-09T11:44:56.382202+01:00"
  "jws": "eyJhbGciOiJFUzI1NiIsImI2NCI6ZmFsc2UsImNyaXQiOlsiYjY0Il19..ZXlKaGJHY2lPaUpGVXpJMU5pSjkuVHVqZndMVVJwcnUzbjhuZklhODB1M1M0LW9LcWY0WUs5S2hoZEktUkZPSzdlbnZJTTdLN1E5SzBSeHhRSzNIVWJPTUJyLVlZX1g0eW1YR0pXOHF4UkEuN0F4a3lZekNXTElPZ2Q5TlpnR3p2aHd2UzZZQ3FpRTRPX3FwWGVOSEN6X091S1c0TmJsWkJueTBkZVhXT0lXZ3JNczF4OTZlNmtnaGZGYTRNd0J3TlE="
}
```

### 3.2 Content-type

All VCs MUST have a content-type equal to `application/vc+json` when published on the network. All issued VCs MUST contain the `https://nuts.nl/credentials/v1` context. VCs MAY NOT specify more than one additional type next to `VerifiableCredential`.

### 3.3 Updates

VCs are not updatable, an update can be performed by revoking the current and issuing a new VC.

### 3.4 Identifiers

VC identifiers MUST be constructed as `DID#id` where `id` is unique for the given issuer.

### 3.5 Active issuer

A VC MUST have an active issuer at the time of usage. Usage includes using the VC to generate a VP or sending it along in an OAuth flow. If the issuer is deactivated at the time of usage, the VC MUST be regarded as invalid.

### 3.6 Transaction requirements

If a VC issuer is a Nuts DID Subject and the VC is published via a transaction according to [RFC004](rfc004-verifiable-transactional-graph.md) and [RFC005](rfc005-distributed-network-using-grpc.md) then the transaction MUST refer to the correct transaction as specified by [RFC004](rfc004-verifiable-transactional-graph.md#36-signature-verification).
This ensures the correct ordering of transactions.

### 3.7 VC Example

Below is an example of a credential. **Issuer** and **Subject** are the same in the example. This specification neither requires nor prevents this.

```javascript
{
  "@context": [
    "https://www.w3.org/2018/credentials/v1",
    "https://nuts.nl/credentials/v1"
  ],
  "type": [
    "NutsOrganizationCredential",
    "VerifiableCredential"
  ],
  "id": "did:nuts:t1DVVAs5fmNba8fdKoTSQNtiGcH49vicrkjZW2KRqpv#6f91673b-afa9-4d26-9e0f-00d989943275",
  "issuanceDate": "2021-03-15T16:34:17.687862+01:00",
  "issuer": "did:nuts:t1DVVAs5fmNba8fdKoTSQNtiGcH49vicrkjZW2KRqpv",
  "credentialSubject": {
    "id": "did:nuts:t1DVVAs5fmNba8fdKoTSQNtiGcH49vicrkjZW2KRqpv",
    "organization": {
      "city": "IJbergen",
      "name": "Because we care B.V."
    }
  },
  "proof": {
    "type": "JsonWebSignature2020",
    "proofPurpose": "assertionMethod",
    "verificationMethod": "did:nuts:t1DVVAs5fmNba8fdKoTSQNtiGcH49vicrkjZW2KRqpv#h22vbXHX7-lRd1qAJnU63liaehb9sAoBS7RavhvfgR8",
    "created": "2021-03-15T16:34:17.687862+01:00",
    "jws": "eyJhbGciOiJFUzI1NiIsImI2NCI6ZmFsc2UsImNyaXQiOlsiYjY0Il19..hKcboC8m6YnZPi6ReJAYs0J0Ztn5nxcx2EavoXdtrkWxmE1JZmImW89_8IIgjvfI8XtGeDlEnGywAuY2u7y9Bw"
  }
}
```

## 4. Required chapters for a Verifiable Credential type

All the following chapters MUST be present in the specification of a VC.

### 4.1 CredentialSubject

It MUST specify the contents of the `credentialSubject` JSON field. It MUST specify which parts of the `credentialSubject` are mandatory or optional, and the format.

### 4.2 Issuance & distribution

A VC specification MUST specify how a VC is issued and if there are any requirements on the issuer. It MUST specify any requirements for the holder. It MUST specify if VCs or other credentials are required in order to obtain the VC. It MUST specify the content-type `type` selector.

### 4.3 Supported proofs

It MUST specify which proof types are acceptable or if there's a limitation to certain proof types. This also means that specifications SHOULD to be updated when new proof types are available.

### 4.4 Trust

It MUST list the requirements for when a VC is to be trusted. It could, for example, require every node to trust a certain issuer by default.

### 4.5 Revocation

It MUST specify how a VC can be revoked by the issuer, or it MUST specify an expiration duration. It MAY refer to the default revocation mechanism stated below.

### 4.5.1 Default revocation

VCs that are issued by a Nuts DID can be revoked by publishing the following transaction on the network:

```javascript
{
  "issuer": "did:nuts:t1DVVAs5fmNba8fdKoTSQNtiGcH49vicrkjZW2KRqpv",
  "subject": "did:nuts:t1DVVAs5fmNba8fdKoTSQNtiGcH49vicrkjZW2KRqpv#6f91673b-afa9-4d26-9e0f-00d989943275",
  "date": "2021-03-15T16:34:47.422436+01:00",
  "proof": {
    "type": "JsonWebSignature2020",
    "proofPurpose": "assertionMethod",
    "verificationMethod": "did:nuts:t1DVVAs5fmNba8fdKoTSQNtiGcH49vicrkjZW2KRqpv#h22vbXHX7-lRd1qAJnU63liaehb9sAoBS7RavhvfgR8",
    "created": "2021-03-15T16:34:47.422436+01:00",
    "jws": "eyJhbGciOiJFUzI1NiIsImI2NCI6ZmFsc2UsImNyaXQiOlsiYjY0Il19..Rc7iK7wXabMx24ZNAFJIwxqpYGCdye0EmdOYwu5CO54pwEPNIQt-9qIvqEZ7ZBFcFUdhnNCvYkR8IDtFAM18Rw"
  }
}
```

Such a revocation transaction has the following requirements:

* the **issuer** MUST match the DID as the `issuer` field of the VC.
* the **subject** MUST match the `id` field of the VC.
* a **reason** MAY be filled with a revocation reason.
* the **date** MUST provide the date in RFC3339 format. From this moment in time the VC is revoked.
* the **proof** MUST be a `JsonWebSignature2020` proof.

The transaction MUST be published on the Nuts network. The content-type is `application/vc+json;type=revocation`

The signature is calculated as stated in §3.1.1.

### 4.6 Use cases

It MUST specify where the VC SHOULD be used: as requirement for other VCs, in the OAuth flow according to [RFC003](rfc003-oauth2-authorization.md) or any other use case. If a VC can be used in a Verifiable Presentation \(VP\), it MUST specify if additional VCs are to be expected in the VP. It SHOULD describe the use case well enough so any implementation can take appropriate measures for optimizing querying/checking, such as indexing certain fields.

### 4.7 Privacy considerations

It MUST specify if a VC SHOULD be published via some means. It MUST specify if it contains personal information as identified by the GDPR or local legislation. It MUST specify who MAY receive and/or store the VC.

### 4.8 Services

It MUST specify if other services than the Nuts network are required for the VC to be issued and/or obtained. If so, it MUST specify the protocol.

