# RFC018 Nuts Consent Credential

|                           |                |
|:--------------------------|:---------------|
| Nuts foundation           | W.M. Slakhorst |
| Request for Comments: 018 | Nedap          |
|                           | March 2022     |

## Nuts Consent Credential

### Abstract

A Nuts Consent Credential describes the consent of a person to share data between two organizations. It does not allow access to any data but can be used by an organization to create a Nuts Authorization Credential. The consent can be given in various forms: oral, in writing or with a cryptographic proof. This RFC only contains the non-cryptographic proof version. The credential is supposed to be used by the holder and is therefore not published on the network.

### Status

This document is currently a draft.

### Copyright Notice

![](../.gitbook/assets/license.png)

This document is released under the [Attribution-ShareAlike 4.0 International \(CC BY-SA 4.0\) license](https://creativecommons.org/licenses/by-sa/4.0/).

## 1. Introduction

[RFC014](rfc014-authorization-credential.md) makes it possible for a custodian to create an authorization for an actor to access personal data. The problem with this is that it's not always known to the custodian when a authorization needs to be created. It also requires an interaction with the person the data belongs to, to authorize the data sharing. RFC014 also describes some of the cases in which the authorization credential alone does not support the case completely. One case in particular is of interest:

* A patient at the GP's office wants the GP to view data from the hospital.

The Nuts Consent Credential allows for the actor to be a witness to the patient's consent. The resulting credential can then be send to the custodian (via a Verifiable Presentation for example). The custodian can then decide to grant access to the actor automatically or use a custom workflow to decide on the result. The fact the actor acts as a witness also means it puts its reputation on the line.

This RFC builds upon [RFC011](rfc011-verifiable-credential.md).

## 2. Terminology

* **DID**: [Decentralized Identifiers](https://www.w3.org/TR/did-core/).
* **Policy**: An access policy defined by a Bolt. It describes the access to and operations on resources that are allowed. 
* **VC**: [Verifiable Credential Data Model](https://www.w3.org/TR/vc-data-model/).
* **VP**: Verifiable Presentation as described in the [Verifiable Credential Data Model](https://www.w3.org/TR/vc-data-model/).

Other terminology comes from the [Nuts Start Architecture](rfc001-nuts-start-architecture.md#nuts-start-architecture).

## 3. Credential data model

### 3.1 Credential fields

The `issuer` field of the credential MUST contain the DID of the actor. The `type` MUST include both the values `VerifiableCredential` and `NutsConsentCredential`.
Bolts that use this credential MUST specify the maximum validity of the credential.

### 3.2 CredentialSubject

The `credentialSubject` field contains the following:

```javascript
{
  "id": "did:nuts:SjkuVHVqZndMVVJwcnUzbjhuZklhODB1M1M0LW9LcWY0WUs5S2",
  "custodian": "did:nuts:EgFjg8zqN6eN3oiKtSvmUucao4VF18m2Q9fftAeANTBd",
  "purposeOfUse": "test-service",
  "subject": "123456780"
}
```

`id` contains the DID of the actor. `custodian` contains the DID of the custodian for whom the request is ment. `purposeOfUse` contains the purpose of use as specified by the Bolt using this RFC. `subject` contains the citizen service number.

### 3.2.1 Required fields

All fields in `credentialSubject` are required.

### 3.2.2 Scoping

The combination of the fields within `credentialSubject` scope the consent for a single person between two organizations and a specific Bolt.  The `purposeOfUse` field refers to an access policy described by a Bolt.

## 4. Issuance & distribution

A Nuts Consent Credential is private and is intended to be used by the holder when needed. The VC MUST not be published on the network. Any DID MAY issue an consent credential. The VC does not have any other requirements nor does it add requirements to other VCs.

## 5. Supported proofs

The VC MUST contain two proofs. One proof will be a proof as described by [RFC011](rfc011-verifiable-credential.md). This is the proof the issuer has verified the contents and signs it with its assertion method key. The other proof is the proof of the person giving consent. The following paragraphs list the different usable proofs.

### 5.1 NutsPaperProof2022

This proof represents a paper document recorded by the issuer. It does not contain any cryptographic proofs. It is used to bridge the gap before personal signing methods are adopted.

```javascript
{
  "type": "NutsPaperProof2022",
  "created": "2010-01-01T19:73:24Z",
  "expires": "2015-01-01T19:73:24Z",
  "description": "client signed document, filed under #803465."
}
```

The `type` MUST be `NutsPaperProof2022`. `created` MUST be a valid datetime. The `expires` field is optionally. If filled, the `expirationDate` of the credential MAY NOT exceed this date. The `description` field is required and should contain a short and human readable description on how the consent has been recorded by the actor. Bolts may put constraints on its length when used by the holder.

## 6. Trust

The Nuts Consent Credential only impacts the actor, subject and custodian. It MUST be trusted automatically.

## 7. Revocation

The Nuts Consent Credential follows revocation rules as stated by [RFC011](rfc011-verifiable-credential.md). This means revocations are published although the credential itself is not published.

## 8. Use cases

The Nuts Consent Credential MAY be used in any Bolt that requires Nuts Authorization Credentials based on explicit consent.

## 9. Privacy considerations

The credential contains a citizen service number and MUST therefore be private to the actor and custodian.

## 10. Services

A Nuts Consent Credential is always scoped to a specific Bolt.

## 11. Examples

Example of a Nuts Consent Credential:

```javascript
{
  "@context": [
    "https://www.w3.org/2018/credentials/v1",
    "https://nuts.nl/credentials/v1"
  ],
  "id": "did:nuts:custodian#90382475609238467",
  "type": ["VerifiableCredential", "NutsConsentCredential"],
  "issuer": "did:nuts:EgFjg8zqN6eN3oiKtSvmUucao4VF18m2Q9fftAeANTBd",
  "issuanceDate": "2010-01-01T19:73:24Z",
  "expirationDate": "2010-02-01T19:73:24Z",
  "credentialSubject": {
    "id": "did:nuts:EgFjg8zqN6eN3oiKtSvmUucao4VF18m2Q9fftAeANTBd",
    "custodian": "did:nuts:SjkuVHVqZndMVVJwcnUzbjhuZklhODB1M1M0LW9LcWY0WUs5S2",
    "purposeOfUse": "zorginzage",
    "subject": "123456780"
  },
  "proof": [
    {
      "proof": {
      "type": "EcdsaSecp256k1Signature2019",
      "created": "2010-01-01T19:73:24Z",
      "verificationMethod": "did:nuts:EgFjg8zqN6eN3oiKtSvmUucao4VF18m2Q9fftAeANTBd#234906587jfglout",
      "proofPurpose": "assertionMethod",
      "proofValue": "z58DAdFfa9SkqZMVPxAQpic7ndSayn1PzZs6ZjWp1CktyGesjuTSwRdoWhAfGFCF5bppETSTojQCrfFPP2oumHKtz"
    },
    {
      "type": "NutsPaperProof2022",
      "created": "2010-01-01T19:73:24Z",
      "expires": "2010-02-01T19:73:24Z",
      "description": "client signed document, filed under #803465."
    }
  ]
}
```
