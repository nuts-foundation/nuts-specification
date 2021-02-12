# RFC011 Nuts Identity Credential

|  |  |
| :--- | :--- |
| Nuts foundation | W.M. Slakhorst |
| Request for Comments: 011 | Nedap |
|  | February 2021 |

## Nuts Identity Credential
### Abstract

A way to add identifying data to a DID.

### Status

This document is currently a draft.

### Copyright Notice
![](../.gitbook/assets/license.png)

This document is released under the [Attribution-ShareAlike 4.0 International \(CC BY-SA 4.0\) license](https://creativecommons.org/licenses/by-sa/4.0/).

## 1. Introduction

Searching on DID is clumsy

## 2. Terminology

* **DID**: [Decentralized Identifiers (DID)](https://www.w3.org/TR/did-core/).
* **VC**: [Verifiable Credential Data Model](https://www.w3.org/TR/vc-data-model/).

## 3. Issuance

```json
{
  "@context": [
    "https://www.w3.org/2018/credentials/v1",
    "https://nuts.nl/credentials/v1"
  ],
  "id": "did:nuts:1#90382475609238467",
  "type": ["VerifiableCredential", "NutsCompanyCredential"],
  "issuer": "did:nuts:1",
  "issuanceDate": "2010-01-01T19:73:24Z",
  "expirationDate": "2010-02-01T19:73:24Z",
  "credentialSubject": {
    "id": "did:nuts:2",
    "company": {
      "tradeName": "De Nootjes",
      "city": "Eibergen"
    }
  },
  "proof": {
    "type": "EcdsaSecp256r1Signature2019",
    "created": "2017-06-18T21:19:10Z",
    "proofPurpose": "assertionMethod",
    "verificationMethod": "did:nuts:1#key-1",
    "jws": "eyJhbGciOiJSUzI1NiIsImI2NCI6ZmFsc2UsImNyaXQiOlsiYjY0Il19..TCYt5X
      sITJX1CxPCT8yAV-TVkIEq_PbChOMqsLfRoPsnsgw5WEuts01mq-pQy7UJiN5mgRxD-WUc
      X16dUEMGlv50aqzpqh4Qktb3rk-BuQy72IFLOqV0G_zS245-kronKb78cPN25DGlcTwLtj
      PAYuNzVBAh4vGHSrQyHUdBBPM"
  }
}
```

content-type should be something like `application/ld+json` but fo th network it's not descriptive enough...
`application/nc+vc+json` using abbr letters as first part.
`application/nc+revocation+json` for revocation?

Signature: https://w3c-ccg.github.io/ld-proofs/ and https://tools.ietf.org/html/rfc7797

### 3.1 Acceptation & trust

First check publication on network using rfc004.
Then validate VC using https://w3c-ccg.github.io/ld-proofs/ and https://tools.ietf.org/html/rfc7797.
When all valid, add VC to store as untrusted.
Mark VC as trusted when combination of `issuer` and `type` is marked as trusted within node.

## 4 Revocation

For revocation, we can just use a system where an issuer publishes revoked credential IDs. 
All credentials used within a healthcare setting are not supposed to be anonymous, so linkability is not a problem.
Also when using an accumulator for revocation checking, it's already possible to check this for every VC, since they are all known before verification.

```json
{
  "issuer": "did:nuts:1",
  "subject": "did:nuts:1#90382475609238467",
  "type": "NutsCompanyCredential",
  "revocationDate": "2010-02-01T19:73:24Z",
  "proof": {
    "type": "EcdsaSecp256r1Signature2019",
    "created": "2017-06-18T21:19:10Z",
    "proofPurpose": "assertionMethod",
    "verificationMethod": "did:nuts:1#key-1",
    "jws": "eyJhbGciOiJSUzI1NiIsImI2NCI6ZmFsc2UsImNyaXQiOlsiYjY0Il19..TCYt5X
    sITJX1CxPCT8yAV-TVkIEq_PbChOMqsLfRoPsnsgw5WEuts01mq-pQy7UJiN5mgRxD-WUc
    X16dUEMGlv50aqzpqh4Qktb3rk-BuQy72IFLOqV0G_zS245-kronKb78cPN25DGlcTwLtj
    PAYuNzVBAh4vGHSrQyHUdBBPM"
  }
}
```

https://w3c-ccg.github.io/ld-proofs/ and https://tools.ietf.org/html/rfc7797 compliant

## 5 Usage
Dynamic usage via undefined `credentialSubject` payload. 
In API as object, in code as some interface which is serializable to json or already is json.

```
Issue(from: DID, to: DID, type: string, expirationDate: *time.Time, credentialSubject interface{}) error
```
## 6 Security Considerations

Almost all security considerations are covered by the mechanisms described in [RFC004](rfc004-distributed-document-format.md). An overview of countermeasures:

- **eavesdropping** - All communications is sent over two-way TLS. All data is public anyway.
- **replay** - VCs are issued to a subject, replaying doesn't change that.
- **message insertion** - [RFC004](rfc004-distributed-document-format.md) defines hashing and signing of published documents.
- **deletion** - Most VCs are published and copied on a mesh network. Deletion of a single document will only occur locally and will not damage other nodes.
- **modification** - VCs can only be modified if they are published with a signature from one of the `assertionMethod` keys.
- **man-in-the-middle** - All communications is sent over two-way TLS and all transactions/VCs are signed.
- **denial of service** - This is out of scope and handled by [RFC004](rfc004-distributed-document-format.md).

All considerations regarding DIDs apply from rfc006.

## 7 Privacy considerations

All data is public knowledge. All considerations from [§10 of did-core](https://www.w3.org/TR/did-core/#privacy-considerations) apply.

## 8. Services

no required services

## 9. Supported Cryptographic Algorithms and Key Types

RFC006 chapter 9 applies
