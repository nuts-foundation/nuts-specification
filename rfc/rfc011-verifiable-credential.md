# RFC011 Verifiable Credentials

|  |  |
| :--- | :--- |
| Nuts foundation | W.M. Slakhorst |
| Request for Comments: 011 | Nedap |
|  | February 2021 |

## RFC011 Verifiable Credentials
### Abstract

This RFC describes the generic requirements for [Verifiable Credentials (VC)](https://www.w3.org/TR/vc-data-model/) within the Nuts specification. 
It also lists all required chapters for a specific VC RFC.

### Status

This document is currently a draft.

### Copyright Notice
![](../.gitbook/assets/license.png)

This document is released under the [Attribution-ShareAlike 4.0 International \(CC BY-SA 4.0\) license](https://creativecommons.org/licenses/by-sa/4.0/).

## 1. Introduction

It's to be expected that multiple sources must be supported that can claim certain information about a subject. 
These claims have to be structured in such a way, that they are verifiable and searchable. 
Examples of claims are a company name, and it's chamber of commerce number. Users will search on the name of an organization to find out whuch services are supported.
This will directly influence the interaction the user can have with the system of that organization. 
Making sure this information is correct and that it can be trusted is therefore extremely important.

## 2. Terminology

* **DID**: [Decentralized Identifiers](https://www.w3.org/TR/did-core/).
* **holder**: The party that receives a VC from an issuer.  
* **issuer**: The party that issues and signs a VC.  
* **proof**: Proof asserting that a VC is valid using cryptography.
* **VC**: Verifiable Credential according to the [Verifiable Credential Data Model](https://www.w3.org/TR/vc-data-model/).
* **VP**: Verifiable Presentation according to the [Verifiable Credential Data Model](https://www.w3.org/TR/vc-data-model/).

## 3. W3C Verifiable Credential

To support a variety of claims, the [W3C Verifiable Credential specification](https://www.w3.org/TR/vc-data-model/) is used. 
Every Nuts specific VC and proof type must follow the W3C specification. If the W3C specification offers options, then the specific VP must specify which option is to be used.
All VCs MUST use DIDs as specified in [RFC004](rfc004-distributed-document-format.md) for the issuer. 

### 3.1 Supported proofs

The following proof types must be supported:

#### 3.1.1 JsonWebSignature2020

resources:

* https://w3c-ccg.github.io/ld-proofs/ 
* https://tools.ietf.org/html/rfc7797
* https://w3c-ccg.github.io/lds-jws2020

The *JsonWebSignature2020* refers to https://w3id.org/rdf#URDNA2015 canonicalization but it's unclear how this is to be implemented for json, therefore it's skipped for now.
It's quite hard to decipher those specifications and they seem to be very drafty as well. The algorithm used to create the signature is as follows:

- compose a `proof` json object:

```json
{
  "type": "JsonWebSignature2020",
  "proofPurpose": "assertionMethod",
  "verificationMethod": "<<kid>>",
  "created": "<<RFC3339>>"
}
```
where `<<kid>>` is replaced with an assertionMethod ID from the DID Document and `<<RFC3339>>` is replaced with a RFC3339 compliant time string.

- take the sha256 of the `proof` and concatenate with the sha256 of the input verifiable credential (without the proof)
- sign the bytes from the previous step with the private key corresponding to the `kid`
- Base64 encode (url encoding) the following headers: `{"alg":"ES256","b64":false,"crit":["b64"]}` giving `eyJhbGciOiJFUzI1NiIsImI2NCI6ZmFsc2UsImNyaXQiOlsiYjY0Il19`
- concat the base64 encoded headers with `..` and the base64 encoded signature
- place the result in the `jws` field of the `proof`:
```json
{
  "type": "JsonWebSignature2020",
  "proofPurpose": "assertionMethod",
  "verificationMethod": "<<kid>>",
  "created": "<<RFC3339>>",
  "jws": "eyJhbGciOiJFUzI1NiIsImI2NCI6ZmFsc2UsImNyaXQiOlsiYjY0Il19..ZXlKaGJHY2lPaUpGVXpJMU5pSjkuVHVqZndMVVJwcnUzbjhuZklhODB1M1M0LW9LcWY0WUs5S2hoZEktUkZPSzdlbnZJTTdLN1E5SzBSeHhRSzNIVWJPTUJyLVlZX1g0eW1YR0pXOHF4UkEuN0F4a3lZekNXTElPZ2Q5TlpnR3p2aHd2UzZZQ3FpRTRPX3FwWGVOSEN6X091S1c0TmJsWkJueTBkZVhXT0lXZ3JNczF4OTZlNmtnaGZGYTRNd0J3TlE="
}
```

### 3.2 Content-type

All VCs have a content-type equal to `application/vc+json;type=<vc>` when published on the network. `<vc>` MUST be defined more specifically. 
All VCs MUST be handled as normal JSON. All issued VCs MUST add the `https://nuts.nl/credentials/v1` context.

### 3.3 Updates

VCs are not updatable, an update can be performed by revoking the current and issuing a new vc.

### 3.4 Identifiers

VC identifiers MUST be constructed as `DID#id` where `id` is unique for the given issuer.

### 3.5 VC Example

Below is an example of a credential. The `proof` and `credentialSubject` are omitted as they are custom per VC.

```json
{
  "@context": [
    "https://www.w3.org/2018/credentials/v1",
    "https://nuts.nl/credentials/v1"
  ],
  "id": "did:nuts:B8PUHs2AUHbFF1xLLK4eZjgErEcMXHxs68FteY7NDtCY#90382475609238467",
  "type": ["VerifiableCredential", "NutsCredential"],
  "issuer": "did:nuts:B8PUHs2AUHbFF1xLLK4eZjgErEcMXHxs68FteY7NDtCY",
  "issuanceDate": "2010-01-01T19:73:24Z",
  "expirationDate": "2010-02-01T19:73:24Z",
  "credentialSubject": {...},
  "proof": {...}
}
```

## 4. Required chapters for a Verifiable Credential type

All the following chapters MUST be present in the specification of a VC.

### 4.1 CredentialSubject

It MUST specify the contents of the `credentialSubject` JSON field. It MUST specify which parts of the `credentialSubject` are mandatory or optional and the format.

### 4.2 Issuance & distribution

A VC specification MUST specify how a VC is issued and who may issue it. It MUST specify which protocols and standards are used in obtaining the VC. 
It MUST specify where it MAY be stored and if it's distributed to others in some way. It MUST specify any requirements for the holder.
It MUST specify if VCs or other credentials are required in order to obtain the VC. It MUST specify the content-type `type` selector

### 4.3 Supported proofs

It MUST specify which proof types are acceptable or if there's a limitation to certain proof types. 
This also means that specifications SHOULD to be updated when new proof types are available.

### 4.4 Trust

It MUST list the requirements for when a VC is to be trusted. This can be any combination of configuration, other VCs and/or automatic mechanism.

### 4.5 Revocation

It MUST specify how a VC can be revoked by the issuer, or it MUST specify an expiration duration. It MAY refer to the default revocation mechanism stated below.

### 4.5.1 Default revocation

VCs that are issued by a Nuts DID can be revoked by publishing the following transaction on the network:

```json
{
  "issuer": "did:nuts:B8PUHs2AUHbFF1xLLK4eZjgErEcMXHxs68FteY7NDtCY",
  "subject": "did:nuts:B8PUHs2AUHbFF1xLLK4eZjgErEcMXHxs68FteY7NDtCY#90382475609238467",
  "currentStatus": "Revoked",
  "statusReason": "Disciplinary action",
  "statusDate": "2010-02-01T19:73:24Z",
  "proof": {
    "type": "JsonWebSignature2020",
    "created": "2017-06-18T21:19:10Z",
    "proofPurpose": "assertionMethod",
    "verificationMethod": "did:nuts:B8PUHs2AUHbFF1xLLK4eZjgErEcMXHxs68FteY7NDtCY#90382475609238467#qjHYrzaJjpEstmDATng4-cGmR4t-_V3ipbDVYZrVe4A",
    "jws": "eyJhbGciOiJSUzI1NiIsImI2NCI6ZmFsc2UsImNyaXQiOlsiYjY0Il19..TCYt5X
    sITJX1CxPCT8yAV-TVkIEq_PbChOMqsLfRoPsnsgw5WEuts01mq-pQy7UJiN5mgRxD-WUc
    X16dUEMGlv50aqzpqh4Qktb3rk-BuQy72IFLOqV0G_zS245-kronKb78cPN25DGlcTwLtj
    PAYuNzVBAh4vGHSrQyHUdBBPM"
  }
}
```

Such a revocation transaction has the following requirements:

* the **issuer** MUST match the DID as the `issuer` field of the VC.
* the **subject** MUST match the `id` field of the VC.
* the **currentStatus** MUST be changed to `Revoked`.
* the **statusReason** MAY be filled with a revocation reason.
* the **statusDate** MUST provide the date in rfc3339 format. From this moment in time the VC is revoked.
* the **proof** MUST be a `JsonWebSignature2020` proof.

The transaction MUST be published on the Nuts network. It MAY use the `verificationMethod` for the transaction.
The content-type is `application/vc+json;type=revocation`

### 4.6 Use cases

It MUST specify where the VC SHOULD be used: as requirement for other VCs, in the oauth flow according to [RFC003](rfc003-oauth2-authorization.md) or any other use case.
If a VC can be used in a Verifiable Presentation (VP), it MUST specify if additional VCs are to be expected in the VP.
It SHOULD describe the use case well enough so any implementation can take appropriate measures, such as indexing certain fields.

### 4.7 Privacy considerations

It MUST specify if a VC SHOULD be published via some means. It MUST specify if it contains personal information as identified by the GDPR or local legislation. 
It MUST specify who MAY receive and/or store the VC.

### 4.8 Services

It MUST specify if other services than the Nuts network are required for the VC to be issued and/or obtained. If so, it MUST specify the protocol.
