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

* **DID**: [Decentralized Identifiers](https://www.w3.org/TR/did-core/).
* **VC**: [Verifiable Credential Data Model](https://www.w3.org/TR/vc-data-model/).

## 3. Issuance

```json
{
  "@context": [
    "https://www.w3.org/2018/credentials/v1",
    "https://nuts.nl/credentials/v1"
  ],
  "id": "did:nuts:B8PUHs2AUHbFF1xLLK4eZjgErEcMXHxs68FteY7NDtCY#90382475609238467",
  "type": ["VerifiableCredential", "NutsCompanyCredential"],
  "issuer": "did:nuts:B8PUHs2AUHbFF1xLLK4eZjgErEcMXHxs68FteY7NDtCY",
  "issuanceDate": "2010-01-01T19:73:24Z",
  "expirationDate": "2010-02-01T19:73:24Z",
  "credentialSubject": {
    "id": "did:nuts:8jx6GU9JipE6TXY2nak8RgFXMk3zaoPWsCb53N1Zjw9R",
    "company": {
      "tradeName": "De Nootjes",
      "city": "Eibergen"
    }
  },
  "proof": {
    "type": "EcdsaSecp256r1Signature2019",
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

content-type is constructed as `application/vc+json;type=<vc>`. So in this case it would be `application/vc+json;type=NutsCompanyCredential`
Although the format suggests we use json-ld. Everything is just plain json. So validation on types should be on the string values, eg `NutsCompanyCredential`
Signature: https://w3c-ccg.github.io/ld-proofs/ and https://tools.ietf.org/html/rfc7797

### 3.1 Acceptation & trust

First check publication on network using rfc004.
Then validate VC using https://w3c-ccg.github.io/ld-proofs/ and https://tools.ietf.org/html/rfc7797.
When all valid, add VC to store as untrusted.
Mark VC as trusted when combination of `issuer` and `type` is marked as trusted within node. For each type, a node must configure a list of trusted DIDs.

## 4 Revocation

VCs are not updatable, an update can be performed by revoking the current and issuing a new vc.
For revocation, we can just use a system where an issuer publishes revoked credential IDs. 
All credentials used within a healthcare setting are not supposed to be anonymous, so linkability is not a problem.
Also when using an accumulator for revocation checking, it's already possible to check this for every VC, since they are all known before verification.
Nevertheless, forward compatibility can come in handy. So VC identifiers must be constructed as `DID#number` where `number` is a number usable in an accumulator.

More info on an accumulator can be found here: https://hyperledger-indy.readthedocs.io/projects/hipe/en/latest/text/0011-cred-revocation/README.html

```json
{
  "issuer": "did:nuts:B8PUHs2AUHbFF1xLLK4eZjgErEcMXHxs68FteY7NDtCY",
  "subject": "did:nuts:B8PUHs2AUHbFF1xLLK4eZjgErEcMXHxs68FteY7NDtCY#90382475609238467",
  "currentStatus": "Revoked",
  "statusReason": "Disciplinary action",
  "statusDate": "2010-02-01T19:73:24Z",
  "proof": {
    "type": "EcdsaSecp256r1Signature2019",
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

`currentStatus` can not be anything other than `revoked` (for the time being).
https://w3c-ccg.github.io/ld-proofs/ and https://tools.ietf.org/html/rfc7797 compliant

## 5 Usage
Generic usage, as only ID is required for revocation, this ID is present as the `subject` field. `statusReason` and `statusDate` give additional information.

## 6 Security Considerations

Almost all security considerations are covered by the mechanisms described in [RFC004](rfc004-distributed-document-format.md). An overview of countermeasures:

- **eavesdropping** - All communications is sent over two-way TLS. All data is public anyway.
- **replay** - VCs are issued to a subject, replaying doesn't change that.
- **message insertion** - [RFC004](rfc004-distributed-document-format.md) defines hashing and signing of published documents.
- **deletion** - Most VCs are published and copied on a mesh network. Deletion of a single document will only occur locally and will not damage other nodes.
- **modification** - VCs are immutable.
- **man-in-the-middle** - All communications is sent over two-way TLS and all transactions/VCs are signed.
- **denial of service** - This is out of scope and handled by [RFC004](rfc004-distributed-document-format.md).

All considerations regarding DIDs apply from rfc006.

## 7 Privacy considerations

All data is public knowledge. All considerations from [§10 of did-core](https://www.w3.org/TR/did-core/#privacy-considerations) apply.

## 8. Services

no required services

## 9. Supported Cryptographic Algorithms and Key Types

RFC006 chapter 9 applies
