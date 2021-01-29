# RFC010 Multiple operator service access

|  |  |
| :--- | :--- |
| Nuts foundation | W.M. Slakhorst |
| Request for Comments: 010 | Nedap |
|  | January 2021 |

## Multiple operator service access
### Abstract

This RFC describes an addition to [RFC003](rfc003-oauth2-auithorization.md) and [RFC006](rfc006-distributed-registry.md) on how to handle resource access when a legal party has multiple SaaS providers. A SaaS provider can alter the care organization DID document so the DID document of the other provider is added as synonym. This is done by using the *also-known-as* property.

### Status

This document is currently a draft.

### Copyright Notice
![](../.gitbook/assets/license.png)

This document is released under the [Attribution-ShareAlike 4.0 International \(CC BY-SA 4.0\) license](https://creativecommons.org/licenses/by-sa/4.0/).

## 1. Introduction

Care organizations rarely rely on the services of one software vendor. Every software vendor that has signed a **Data Processing Agreement** with the care organization is allowed to transfer data to any other software vendor within the limits of the **Data Processing Agreement**. 

To support this functionality, some registration and extended validation is required throughout the process of data transfer.

## 2. Terminology

* **Bolt**: a use case built on top of the functionality Nuts provides.
* **DID**: a decentralized identifier according to the [core-did-spec](https://www.w3.org/TR/did-core). 
* **DPA**: Data Processing Agreement as defined by the GDPR.
* **GDPR**: General Data Protection Regulation (https://eur-lex.europa.eu/eli/reg/2016/679/oj).  
* **SaaS provider**: Party operating a Nuts node as a service on behalf of their clients (care organizations).

The term *subject* in the context of a SaaS provider can be substituted for *Care organization*.

Other terminology is taken from the [Nuts Start Architecture](rfc001-nuts-start-architecture.md#nuts-start-architecture).

## 3. Registration

DID Documents define public keys which the SaaS provider controls. SaaS providers must not share private keys, 
thus each subject that is registered at a node by a SaaS provider will have its own DID document.
If a subject uses 2 SaaS providers, it'll have two different DID documents. 
Since identification of the subject within the OAuth flow is done via DID, the subjects need to be aware of each other.
This can be done by having the DID documents of the SaaS providers refer to each other.
This is done by using the `alsoKnownAs` field defined in the [core-did-spec](https://www.w3.org/TR/did-core/#also-known-as).

Example:

```json
{
  "@context": [ "https://www.w3.org/ns/did/v1" ],
  "id": "did:nuts:1",
  "alsoKnownAs": [
    "did:nuts:2"
  ]
}
```

This is a unary relation. In the above example, `did:nuts:2` does not know anything about `did:nuts:1`.
The controller of the DID document MUST verify that it added the correct DID in the `alsoKnownAs` field.

## 4. Authentication

As defined in point 3 in [§6.2 of RFC003](rfc003-oauth2-auithorization.md#6-2-authorization), if the actor and custodian are the same, access is granted. If a custodian has added an actor in their DID document under the `alsoKnownAs` field, it MUST grant access as defined by point 3 in §6.2 of RFC003. Bolts MUST describe what resources may be accessed in this way and if user context is required. If nothing is described, resources SHOULD NOT be accessible.

## 5. Hijacked DID documents 

A DID subject that is added under the `alsoKnownAs` field is granted access to resources from the custodian.
In the case when the private key of the actor is compromised and the attacker is changing keys, the custodian SHOULD take appropriate action.
The node MUST remove a DID from the `alsoKnownAs` field when the `authentication` or `controller` field changes in the referenced DID document.