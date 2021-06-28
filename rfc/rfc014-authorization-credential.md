# RFC014 Nuts Authorization Credential

|  |  |
| :--- | :--- |
| Nuts foundation | W.M. Slakhorst |
| Request for Comments: 014 | Nedap |
|  | June 2021 |

## Nuts Authorization Credential
### Abstract

A Nuts Authorization Credential describes which data an actor may access. 
It is scoped to a specific combination of custodian, actor and service (Bolt). 
The credential also contains the legal base on which data access may occur. 
This allows this credential to be used for explicit consent from a patient but also in cases where consent is implied.
A patient can consent to a referral to a specialist. This implies that relevant data is accessible to that specialist.
This RFC adds the requirement for Bolts to define an access policy. 
A resource server, hosted by a custodian, will use the policy and the credential to grant access to a specific resource. 
The credential is currently only usable for FHIR based services.

### Status

This document is currently a draft.

### Copyright Notice
![](../.gitbook/assets/license.png)

This document is released under the [Attribution-ShareAlike 4.0 International \(CC BY-SA 4.0\) license](https://creativecommons.org/licenses/by-sa/4.0/).

## 1. Introduction

Personal data (this includes medical data) is protected under the GDPR and various local legislation. 
It's common in healthcare to receive care from multiple care organizations. 
These organizations may not share data amongst them without a valid legal reason. 
The GDPR sums up several legal reasons to share data. Most common are local legislation and explicit consent. 
When data is shared between two organizations, this only matters to those organizations (and the patient). 
Others are not allowed to see this relation. 
Simply knowing where someone receives care may give away the illness of the patient.
The type of data that is shared should also be limited to what is needed by the actor.

There are various reasons why data can be shared. 
The process of determining a valid reason happens at different places and at different times:

- A patient at the GP's office wants the GP to view data from the hospital.
- A patient consents to a specific treatment or test and this treatment/test requires specific data.
- An elderly receiving home care wants his/her GP to be able to review data from the home care organization.
- A patient is discharged from the hospital and needs to undergo physical therapy. 
- Etc.

This RFC builds upon [RFC011](rfc011-verifiable-credential.md).

## 2. Terminology

* **DID**: [Decentralized Identifiers](https://www.w3.org/TR/did-core/).
* **Policy**: A security policy defined by a Bolt. It describes the access to and operations on resources that are allowed. 
* **VC**: [Verifiable Credential Data Model](https://www.w3.org/TR/vc-data-model/).

Other terminology comes from the [Nuts Start Architecture](rfc001-nuts-start-architecture.md#nuts-start-architecture).

## 3. Credential data model

### 3.1 Credential fields

The `issuer` field of the credential MUST contain the DID of the custodian.
The `type` MUST include both the values `VerifiableCredential` and `NutsAuthorizationCredential`.
A Bolt specification is required to specify the maximum validity of the credential.

### 3.2 CredentialSubject

The `credentialSubject` field contains the following:

```json
{
  "id": "did:nuts:SjkuVHVqZndMVVJwcnUzbjhuZklhODB1M1M0LW9LcWY0WUs5S2",
  "legalBase": {
    "consentType": "explicit",
    "evidence": {
      "path": "pdf/f2aeec97-fc0d-42bf-8ca7-0548192d4231",
      "type": "application/pdf"
    }
  },
  "restrictions": [
    {
      "resource": "/DocumentReference/f2aeec97-fc0d-42bf-8ca7-0548192d4231",
      "operations": ["read"]
    }
  ],
  "service": "test-service",
  "subject": "urn:oid:2.16.840.1.113883.2.4.6.3:123456780"
}
```

### 3.2.1 Required fields

The `id`, `legalBase` and `service` fields MUST be filled. 
Within the `legalBase` object, `consentType` MUST either be `implied` or `explicit`. 
When `explicit`, the `evidence` and `subject` fields MUST be filled.
When `implied`, values MUST be added to the `restrictions` array.

### 3.2.2 Scoping

The `id` field MUST contain the DID of the actor.
The `subject` field MAY contain the patient identifier. In the example above the *oid* for the Dutch citizenship number is used.
The `service` field refers to a Bolt. A Bolt MUST describe the set of FHIR resources that can be accessed with the credential.
The `restrictions` array MAY further limit access.
When no `subject` is given, the credential MUST contain `restrictions` that refer to individual resources.
The contents of those individual resources MUST NOT contain any personal information.

### 3.2.3 Legal base

The patient consent can be either implied or explicit. 
When it's implied, this should be reflected by the type of service that is entered.
A Bolt MUST therefore also describe if it is covered by explicit or implied consent.
When the credential is given in the context of explicit consent, the `legalBase.evidence` MUST be filled.
It MUST contain a value for the `path` and `type` fields. The `path` is a relative path to the service data endpoint and the `type` contains the media type as specified by [RFC6838](https://datatracker.ietf.org/doc/html/rfc6838).
The evidence resource MUST be accessible with an access token that was created with the corresponding credential.

### 3.2.4 Restrictions

The `restrictions` object MAY be used to restrict access. The base access is provided by the policy as defined by the Bolt.
If any restrictions are added, they override the Bolt policy. 
Restrictions MAY NOT grant more access rights than the policy.
All entries in the `restrictions` list contain a `resource`. The `resource` fields contains the relative path to the service data endpoint. The `operations` field contains a list of operations allowed on this resource. The valid options are a subset of the [FHIR specification](http://hl7.org/fhir/stu3/http.html): `read`, `vread`,`update`,`patch`,`delete`,`history (instance)`, `create` and `search`.

## 4. Access Control

A resource server uses the Nuts Authorization Credential to check the access rights of an actor.
The rules for determining access are a combination of a Bolt specific policy and any additional information from the credential.
Identification and authentication are covered by [RFC003](rfc003-oauth2-authorization.md).
If no `restrictions` are added to the credential, the resource server MUST follow the policy rules of the Bolt.
The Bolt policy will list the operations and resource types that can be accessed. 
A policy MAY also require certain parameters. For example, when a `search` operation is done on a FHIR `observation` resource, the policy may have a rule that requires a query parameter using the `subject` field of the credential.
All restrictions and policy rules MUST use paths relative to the endpoint for the given service.
The registration of services is covered by [§4 of RFC006](rfc006-distributed-registry.md#4-services).
If `restrictions` are present in the credential, the resource server can compare the operation and relative path of the request to the `restrictions` present in the credential.

Although Nuts Authorization Credentials are part of the OAuth flow of [RFC003](rfc003-oauth2-authorization.md), the actual checking is done at request time. This means that the resource server will have to check the policy and make a request for the restrictions from the Nuts registry. This model can be compared with an [Attribute Based Access Control (ABAC) model](https://en.wikipedia.org/wiki/Attribute-based_access_control). The Bolt policies are added to the Policy Administration Point, the Nuts node acts as Policy Information Point. The resource server is the Policy Enforcement Point. It's up to the vendor to implement the Policy Decision Point. 

## 5. Issuance & distribution

A Nuts Authorization Credential is private and the transaction MAY be published over the Nuts network.
The contents of the Credential MUST NOT be attached to the network transaction.
Only the custodian and actor MAY retrieve the transaction payload.
Every DID MAY issue an authorization credential.
The VC does not have any other requirements nor does it add requirements to other VCs.

## 6. Supported proofs

Only proofs from [RFC011](rfc011-verifiable-credential.md) are supported.

## 7. Trust

The Nuts Authorization Credential only impacts the actor and custodian.
It MUST be trusted automatically.

## 8. Revocation

The Nuts Authorization Credential follows revocation rules as stated by [RFC011](rfc011-verifiable-credential.md).

## 9. Use cases

The Nuts Authorization Credential MUST be used in the Nuts OAuth flow as stated in [RFC003](rfc003-oauth2-authorization.md).

## 10. Privacy considerations

The credential will most likely contain a citizen service number and MUST therefore be private to the actor and custodian.

## 11. Services

A Nuts Authorization Credential is always scoped to a specific Bolt.

## 12. Examples

Example of a Nuts Authorization Credential with explicit consent:

```json
{
  "@context": [
    "https://www.w3.org/2018/credentials/v1",
    "https://nuts.nl/credentials/v1"
  ],
  "id": "did:nuts:custodian#90382475609238467",
  "type": ["VerifiableCredential", "NutsAuthorizationCredential"],
  "issuer": "did:nuts:EgFjg8zqN6eN3oiKtSvmUucao4VF18m2Q9fftAeANTBd",
  "issuanceDate": "2010-01-01T19:73:24Z",
  "expirationDate": "2010-02-01T19:73:24Z",
  "credentialSubject": {
    "id": "did:nuts:SjkuVHVqZndMVVJwcnUzbjhuZklhODB1M1M0LW9LcWY0WUs5S2",
    "legalBase": {
      "consentType": "explicit",
      "evidence": {
        "path": "pdf/f2aeec97-fc0d-42bf-8ca7-0548192d4231",
        "type": "application/pdf"
      } 
    },
    "service": "zorginzage",
    "subject": "urn:oid:2.16.840.1.113883.2.4.6.3:123456780"
  },
  "proof": {...}
}
```
Example of a Nuts Authorization Credential with implied consent:

```json
{
  "@context": [
    "https://www.w3.org/2018/credentials/v1",
    "https://nuts.nl/credentials/v1"
  ],
  "id": "did:nuts:custodian#90382475609238467",
  "type": ["VerifiableCredential", "NutsAuthorizationCredential"],
  "issuer": "did:nuts:EgFjg8zqN6eN3oiKtSvmUucao4VF18m2Q9fftAeANTBd",
  "issuanceDate": "2010-01-01T19:73:24Z",
  "expirationDate": "2010-02-01T19:73:24Z",
  "credentialSubject": {
    "id": "did:nuts:SjkuVHVqZndMVVJwcnUzbjhuZklhODB1M1M0LW9LcWY0WUs5S2",
    "legalBase": {
      "consentType": "implied"
    },
    "restrictions": ["/composition/f2aeec97-fc0d-42bf-8ca7-0548192d4231"],
    "service": "eOverdracht",
    "subject": "urn:oid:2.16.840.1.113883.2.4.6.3:123456780"
  },
  "proof": {...}
}
```
