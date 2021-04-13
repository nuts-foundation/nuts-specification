# RFC012 Nuts Organization Credential

|  |  |
| :--- | :--- |
| Nuts foundation | W.M. Slakhorst |
| Request for Comments: 012 | Nedap |
|  | February 2021 |

## Nuts Organization Credential
### Abstract

Creating a network with trusted Verified Credentials is victim of the chicken-and-egg-problem. 
Issuers are not yet convinced they should support VCs until the network is mature enough and nobody is willing to use the network without official credentials/identities.
The **Nuts organization credential** offers a temporary solution. It allows for any DID subject to issue a VC. 
Trust is established manually by adding the DID to a trusted list of issuers. 
For development use-cases and as bootstrap for a production network, the `Nuts Organization Credential` will add names to DIDs.

### Status

This document is currently a draft.

### Copyright Notice
![](../.gitbook/assets/license.png)

This document is released under the [Attribution-ShareAlike 4.0 International \(CC BY-SA 4.0\) license](https://creativecommons.org/licenses/by-sa/4.0/).

## 1. Introduction

Users that want to interact with another system want to use real-world names when they search for organizations. DIDs do not solve this problem.
VCs can add this information (claims) to a DID. This RFC builds upon [RFC011](rfc011-verifiable-credential.md). 

## 2. Terminology

* **DID**: [Decentralized Identifiers](https://www.w3.org/TR/did-core/).
* **VC**: [Verifiable Credential Data Model](https://www.w3.org/TR/vc-data-model/).

## 3 CredentialSubject

The `credentialSubject` field contains the following:

```json
{
    "id": "did:nuts:8jx6GU9JipE6TXY2nak8RgFXMk3zaoPWsCb53N1Zjw9R",
    "organization": {
        "name": "De Nootjes",
        "city": "Eibergen"
    }
}
```

and has the following requirements:

* all fields are required.
* all fields are encoded as strings.  
* **id** MUST refer to a known Nuts DID as specified by [RFC006](rfc006-distributed-registry.md).

## 4 Issuance & distribution

A Nuts Organization Credential is public and MUST be published over the Nuts network. Every DID MAY issue the credential.
The VC does not have any other requirements nor does it add requirements to other VCs.

## 5 Supported proofs

Only proofs from [RFC011](rfc011-verifiable-credential.md) are supported.

## 6 Trust

The Nuts Organization Credential MUST be trusted manually. 
An implementation can choose to trust each VC individually or trust a certain **Issuer**. 
The mechanism for this is up to the implementation.

## 7 Revocation

The Nuts Organization Credential uses the revocation mechanism as stated by [RFC011](rfc011-verifiable-credential.md).

## 8 Use cases

The Nuts Organization Credential MAY be used as credential in the Nuts OAuth flow as stated in [RFC003](rfc003-oauth2-authorization.md).
The credential MAY also be used as a way to find the correct DID and its services.

## 9 Privacy considerations

All information in the credential SHOULD be public knowledge. The VC MAY NOT contain private information.

## 10 Services

No additional services other than the Nuts network are required.

## 11. Example

```json
{
  "@context": [
    "https://www.w3.org/2018/credentials/v1",
    "https://nuts.nl/credentials/v1"
  ],
  "id": "did:nuts:B8PUHs2AUHbFF1xLLK4eZjgErEcMXHxs68FteY7NDtCY#90382475609238467",
  "type": ["VerifiableCredential", "NutsOrganizationCredential"],
  "issuer": "did:nuts:B8PUHs2AUHbFF1xLLK4eZjgErEcMXHxs68FteY7NDtCY",
  "issuanceDate": "2010-01-01T19:73:24Z",
  "expirationDate": "2010-02-01T19:73:24Z",
  "credentialSubject": {
    "id": "did:nuts:8jx6GU9JipE6TXY2nak8RgFXMk3zaoPWsCb53N1Zjw9R",
    "organization": {
      "name": "De Nootjes",
      "city": "Eibergen"
    }
  },
  "proof": {
    "type": "EcdsaSecp256r1Signature2019",
    "created": "2017-06-18T21:19:10Z",
    "proofPurpose": "assertionMethod",
    "verificationMethod": "did:nuts:B8PUHs2AUHbFF1xLLK4eZjgErEcMXHxs68FteY7NDtCY#90382475609238467#qjHYrzaJjpEstmDATng4-cGmR4t-_V3ipbDVYZrVe4A",
    "jws": "eyJhbGciOiJSUzI1NiIsImI2NCI6ZmFsc2UsImNyaXQiOlsiYjY0Il19..TCYt5XsITJX1CxPCT8yAV-TVkIEq_PbChOMqsLfRoPsnsgw5WEuts01mq-pQy7UJiN5mgRxD-WUcX16dUEMGlv50aqzpqh4Qktb3rk-BuQy72IFLOqV0G_zS245-kronKb78cPN25DGlcTwLtjPAYuNzVBAh4vGHSrQyHUdBBPM"
  }
}
```
