# RFC006 Distributed Registry with Decentralized Identifiers \(DID\)

|  |  |
| :--- | :--- |
| Nuts foundation | W.M. Slakhorst |
| Request for Comments: 006 | Nedap |
|  | S. van der Vegt |
|  | Nedap |
|  | R.G. Krul |
|  | Nedap |
|  | January 2021 |

## Distributed Registry with Decentralized Identifiers \(DID\)

### Abstract

This RFC describes a protocol to build a registry containing information required for \(care\) organizations to exchange data. The registry typically contains organizations, software vendors acting on behalf of their client \(organizations\), data exchange services offered by organizations and their technical endpoints. It describes how these are mapped to [Decentralized Identifiers \(DID\)](https://www.w3.org/TR/did-core/) and how DID Documents are encapsulated in transactions \([RFC004 Verifiable Transactional Graph](rfc004-verifiable-transactional-graph.md)\) to provide cryptographic integrity and consistent state across distributed networks.

### Status

This document is currently a draft.

### Copyright Notice

![](../.gitbook/assets/license.png)

This document is released under the [Attribution-ShareAlike 4.0 International \(CC BY-SA 4.0\) license](https://creativecommons.org/licenses/by-sa/4.0/).

## 1. Introduction

When care organizations want to exchange data using Nuts they need to know where to find that data and how to authenticate it. This knowledge is recorded in a distributed registry, writable and queryable by all network participants. This RFC describes how to create, update and resolve the data structures required to achieve that goal.

## 2. Terminology

* **Bolt**: a use case built on top of the functionality Nuts provides.
* **JWS**: JSON Web Signature as specified by [RFC004](rfc004-verifiable-transactional-graph.md).
* **Organization**: a care organization exchanging data with other care organizations over a Nuts network.

  By giving other DIDs control over its DID Document organizations can delegate data exchange to another party,

  e.g. a care software vendor or SaaS provider.

* **Endpoint**: a URI or URL exposed by a network participant which can be used by other participants to pull data.
* **Service**: a group of endpoints implementing a service required by a Bolt.
* **CRDT**: Conflict-free replicated data type.

## 3. Nuts DID Method

### 3.1 Namespace Specific Identifier \(NSI\)

The Nuts DID URI scheme is defined as follows:

```text
did = "did:nuts:" idstring
idstring = 21*22(base58char)
base58char = "1" / "2" / "3" / "4" / "5" / "6" / "7" / "8" / "9" / "A" / "B" / "C"
    / "D" / "E" / "F" / "G" / "H" / "J" / "K" / "L" / "M" / "N" / "P" / "Q"
    / "R" / "S" / "T" / "U" / "V" / "W" / "X" / "Y" / "Z" / "a" / "b" / "c"
    / "d" / "e" / "f" / "g" / "h" / "i" / "j" / "k" / "m" / "n" / "o" / "p"
    / "q" / "r" / "s" / "t" / "u" / "v" / "w" / "x" / "y" / "z"
```

The `idstring` is derived from the public part of a key pair that was used to sign the transaction that created the DID document.

`idstring = BASE-58(hash)`

The `hash` is calculated conforming [rfc7638](https://tools.ietf.org/html/rfc7638) using the `SHA256` hashing algorithm.

For example, consider the following Ed25519 key \(as JWK\):

```javascript
{
  "kty" : "EC",
  "crv" : "P-256",
  "x"   : "Qn6xbZtOYFoLO2qMEAczcau9uGGWwa1bT+7JmAVLtg4=",
  "y"   : "d20dD0qlT+d1djVpAfrfsAfKOUxKwKkn1zqFSIuJ398=",
  "kid" : "did:nuts:3gU9z3j7j4VCboc3qq3Vc5mVVGDNGjfg32xokeX8c8Zn#J9O6wvqtYOVwjc8JtZ4aodRdbPv_IKAjLkEq9uHlDdE"
}
```

Will be reduced to:

```text
{"crv":"P-256","kty":"EC",x":"Qn6xbZtOYFoLO2qMEAczcau9uGGWwa1bT+7JmAVLtg4=","y":"d20dD0qlT+d1djVpAfrfsAfKOUxKwKkn1zqFSIuJ398="}
```

The `idstring` will then be: `idstring = BASE-58(SHA-256(json))`

Outputs:

```javascript
{
  "id": "did:nuts:3gU9z3j7j4VCboc3qq3Vc5mVVGDNGjfg32xokeX8c8Zn"
}
```

Nuts DID Documents are wrapped in a JWS \(JSON Web Signature\) to ensure cryptographic authenticity and integrity through [RFC004](rfc004-verifiable-transactional-graph.md). Please refer to that RFC on how to create the JWS.

The JWS MUST be signed by the private part of the key pair. The public key MAY also be present in the `verificationMethods` and referenced by the `capabilityInvocation` field.

### 3.2 Method operations

DID documents are enclosed in a message envelope to ensure consistency in the network. The envelope is in the form of a JWS as described in [RFC004](rfc004-verifiable-transactional-graph.md). Once the network layer has confirmed the correctness of the signature of the JWS, the verifiable data registry MUST validate if the submitter was authorized to create, update or deactivate the document. If the authorization fails, the document MUST NOT be processed by the registry.

A DID document can only be created directly by its subject. A new DID is created by generating a key pair and proving control over the private key by signing the transaction containing the DID document with it. The corresponding public key MAY be embedded as a `verificationMethod` and referenced as a `capabilityInvocation`. The DID document MAY be configured as its own controller when created. It MAY also list another DID as controller upon creation.

A DID document can only be updated or deactivated by one of its controllers. The `controller` field MAY list the controllers of a document. If no controllers are listed, the DID subject itself is the only controller. If the DID document is not one of the listed controllers, the subject can't update or deactivate its own DID document. There can only be one level of controllers. A JWS signed by any other DID document than a direct controller of the DID document MUST NOT be processed by the registry.

#### 3.2.1 Create \(Register\)

A Create operation for a DID Document puts the following additional requirements on the JWS header parameters:

* `jwk` MUST be present
* `cty` MUST contain the value `application/json+did-document`
* `tiv` MUST be absent or `0`
* `tid` MUST be absent

The `kid` field of the `jwk` header parameter MUST be prefixed by the `id` of the DID document. In order for the contents to be accepted in the Verifiable Data Registry, the JWK MUST match the `capabilityInvocation` key in the DID document with the same identifier. The `kid` field from the JWK MUST match the `id` from the verification key in the DID document.

Example JWS header

```javascript
{
  "alg": "PS256",
  "cty": "application/json+did-document",
  "jwk": {
    "crv": "P-256",
    "x": "38M1FDts7Oea7urmseiugGW7tWc3mLpJh6rKe7xINZ8",
    "kty": "EC",
    "kid": "did:nuts:123#_TKzHv2jFIyvdTGF1Dsgwngfdg3SH6TpDv0Ta1aOEkw"
  },
  "crit": [
    "sigt",
    "ver",
    "prevs"
  ],
  "sigt": "",
  "ver": "1",
  "prevs": [
    "148b3f9b46787220b1eeb0fc483776beef0c2b3e"
  ]
}
```

The DID document has the following basic requirements:

* the `id` MUST be generated according to the method specified at the beginning of §3
* all key references in `capabilityInvocation`, `capabilityDelegation`, `assertionMethod`, `authentication` and `keyAgreement` MUST refer to keys listed under `verificationMethods`

Each key listed in `verificationMethod` MUST have an `id` equal to the DID followed by a `#` and the public key fingerprint according to [rfc7638](https://tools.ietf.org/html/rfc7638) Each key MUST be of type `JsonWebKey2020` according to §5.3.1 of the [did-core-spec](https://www.w3.org/TR/did-core/#key-types-and-formats)

The `controller` field MAY be present. This RFC follows the [did-core-spec](https://www.w3.org/TR/did-core/#did-controller).

The `assertionMethod` field MAY be present. Keys referenced from this field are used for signing Verifiable Credentials/Presentations and for signing JWTs in the OAuth flow.

Example DID document:

```javascript
{
  "@context": [ "https://www.w3.org/ns/did/v1" ],
  "id": "did:nuts:123",
  "controller": [
    "did:nuts:123",
    "did:nuts:7368245"
  ],
  "verificationMethod": [{
    "id": "did:nuts:123#_TKzHv2jFIyvdTGF1Dsgwngfdg3SH6TpDv0Ta1aOEkw",
    "controller": "did:nuts:123",
    "type": "JsonWebKey2020",
    "publicKeyJwk": {
      "crv": "P-256",
      "x": "38M1FDts7Oea7urmseiugGW7tWc3mLpJh6rKe7xINZ8",
      "kty": "EC"
    }
  }],
  "capabilityInvocation": ["did:nuts:123#_TKzHv2jFIyvdTGF1Dsgwngfdg3SH6TpDv0Ta1aOEkw"],
  "service": []
}
```

#### 3.2.2 Read \(Resolve\)

A Nuts DID can only be resolved locally. The concept of the Verifiable Data Registry is the state based upon all Create, Update and Delete operations received through the Nuts Network. Therefore, any DID document SHOULD already be present in local storage.

**3.2.2.1 Resolution Input Metadata**

All historic versions of a DID Document SHOULD be stored and queryable. This allows clients to resolve the document for a specific moment in time \(e.g. a previous version\) instead of the last one. This allows for resolving keys and services at a given moment.

**3.2.2.2 Document Metadata**

The resolved DID Document Metadata contains the `created` and `updated` fields, in accordance with the [did-core-spec](https://www.w3.org/TR/did-core/#did-document-metadata-properties). They are derived from the underlying Nuts Documents. `created` MUST contain the `sigt` timestamp from the first version of the document. `updated` MUST contain the `sigt` timestamp of the last version of the DID Document.

#### 3.2.3 Update \(Replace\)

The complete document gets replaced with a newer version.

Changes to DID documents can only be accepted if the update is signed with a capabilityInvocation key from one of the controllers:

* if no controllers are listed under `controllers`, use a key referenced from the `capabilityInvocation` section of the DID Document itself.
* if there are entries under `controllers`, then use a key referenced from the `capabilityInvocation` section from within the DID Document of one of those controllers.
* a `capabilityInvocation` key MUST always come from the latest version of a DID Document.

The following requirements on the JWS header parameter apply:

* `kid` MUST hold the reference to the correct key.
* `tid` and `tiv` MUST be filled according to [RFC004](rfc004-verifiable-transactional-graph.md).

Example JWS header:

```javascript
{
  "alg": "PS256",
  "cty": "application/json+did-document",
  "kid": "did:nuts:123#_TKzHv2jFIyvdTGF1Dsgwngfdg3SH6TpDv0Ta1aOEkw",
  "crit": ["sigt", "ver", "prevs"],
  "sigt": "2020-01-06T15:00:00Z",
  "ver": "1",
  "prevs": ["148b3f9b46787220b1eeb0fc483776beef0c2b3e"],
  "tid": "SflKxwRJSMeKKF2QT4fwpMeJf36POk6yJV_adQssw5c",
  "tiv": "1"
}
```

#### 3.2.4 Delete \(Deactivate\)

DID documents cannot be deleted as in being "erased", only its contents can be removed as to prevent future changes \(deactivation\). To deactivate a DID Document, remove all contents except the `id` and `context`.

Deletion can not be undone.

Example of a deactivated DID document:

```javascript
{
  "@context": [ "https://www.w3.org/ns/did/v1" ],
  "id": "did:nuts:123"
}
```

### 3.3 Security Considerations

Almost all security considerations are covered by the mechanisms described in [RFC004](rfc004-verifiable-transactional-graph.md). An overview of countermeasures:

* **eavesdropping** - All communications is sent over two-way TLS. All data is public anyway.
* **replay** - DID documents are identified and published by their hash \(SHA-256\). Replaying will result in replaying the exact same content.
* **message insertion** - [RFC004](rfc004-verifiable-transactional-graph.md) defines hashing and signing of published documents.
* **deletion** - All DID documents are published and copied on a mesh network. Deletion of a single document will only occur locally and will not damage other nodes.
* **modification** - DID documents can only be modified if they are published with a signature from one of the `capabilityInvocation` keys.
* **man-in-the-middle** - All communications is sent over two-way TLS and all documents are signed. A DID can not be hijacked since it is derived from the public key.
* **denial of service** - This is out of scope and handled by [RFC004](rfc004-verifiable-transactional-graph.md).

#### 3.3.1 Protection against DID hijacking

The Nuts network is a mesh network without central authority. This means that any party can generate a DID. This DID must be protected against forgery and hijacking since duplicates are accepted in the Nuts network. The duplicates are sorted and one will eventually be accepted \(consistency rules of [RFC004](rfc004-verifiable-transactional-graph.md)\). This would open up a DID to hijacking. Therefore, the DID MUST be a derivative of the public key used to sign the transaction as described in §3.

#### 3.3.2 Protection against loss of private key

The loss of a single private key can be countered by registering multiple keys. Keys can be kept offline, in a vault for example. Such a key can later be used to register new keys when needed. Another option is to add a second controller that acts as an emergency backup. The keys of that controller can be kept offline.

#### 3.3.3 Protection against theft of private key

A stolen key can alter the DID document in such a way that the attacker can get full control with a new key and can exclude the previous owner from making changes. Appropriate measures MUST be taken to keep authentication keys secure.

The consequences of a theft can be mitigated by chaining controllers where each DID Document can only be altered by its controller. A root document of this chain would be the only DID Document that can be altered with a key present in the root document. The key for the root document should be stored offline \(like a vault or offline machine\).

If a DID Document doesn't have any other controllers, and the control over the private key has been lost, the DID subject will have to have all Verifiable Credentials revoked. Without Verifiable Credentials linked to the DID document, the DID document no longer has any value. The DID subject will have to go through the process of reacquiring all Verifiable Credentials for a new DID document.

### 3.4 Privacy considerations

All data is public knowledge. All considerations from [§10 of did-core](https://www.w3.org/TR/did-core/#privacy-considerations) apply.

## 4. Services

It is to be expected that each DID subject will add services to the DID Document. 
Specific services will be specified in their own RFC or Bold specification. 
A service can define an absolute endpoint URI or be a compound service.
A service can also refer to other services.
This is often the case when a SaaS provider defines endpoints to be used for all clients.

For an absolute endpoint URI the `serviceEndpoint` MUST be a string containing the URI. 
For a compound service the `serviceEndpoint` MUST contain a map with absolute endpoint URIs or references to other services. 
References MUST be by query and not by fragment. Only `type` can be used as query param and it refers to the `type` field in a service. A compound service MAY refer to absolute endpoints from other DID Documents. See [§3.2 of did-core](https://www.w3.org/TR/did-core/#did-url-syntax) for DID URL syntax and [RFC3986](https://tools.ietf.org/html/rfc3986) for generic URL standards.

A DID Document MAY NOT contain more than one service with the same type.

The service identifier MUST be constructed from the DID followed by a `#` and an id string. The service identifier MUST be unique to the DID document.

The id string is calculated as: `idstring = BASE-58(SHA-256(json-bytes-without-id))`

Below is an example of a service registered by a care organization that uses the endpoints from a SaaS provider.
The SaaS provider defines the actual URL:

```javascript
{
  "@context": [ "https://www.w3.org/ns/did/v1" ],
  "id": "did:nuts:123",
  "service": [
    {
      "id": "did:nuts:123#IyvdTGF1Dsgwngfdg3SH6TpDv0Ta1aOEkw",
      "type": "oauth",
      "serviceEndpoint": "https://example.com/oauth"
    },
    {
      "id": "did:nuts:123#_TKzHv2jFIF1Dsgwngfdg3SH6TpDv0Ta1aOEkw",
      "type": "fhir",
      "serviceEndpoint": "https://example.com/fhir"
    }
  ]
}
```

The care organization refers to it:

```javascript
{
  "@context": [ "https://www.w3.org/ns/did/v1" ],
  "id": "did:nuts:abc",
  "service": [
    {
      "id": "did:nuts:abc#F1Dsgwngfdg3SH6TpDv0Ta1aOE",
      "type": "NutsCompoundService",
      "serviceEndpoint": {
        "oauth": "did:nuts:123?type=oauth",
        "fhir": "did:nuts:123?type=fhir"
      }
    }
  ]
}
```

### 4.1 Service references

Any `serviceEndpoint` with a value that starts with `did` is a reference to another service.
As specified earlier, all references use the `type` query parameter.
The origin of a reference is from the `serviceEndpoint` context, while the target is a complete `service` object.
When a reference targets a `service`, it actually targets the `serviceEndpoint` of the service.

A client doesn't care about the references. When a client queries for particular services, the references need to be resolved.
Resolving a reference might lead to resolving more references and may lead to an infinite loop.
Resolving a reference MUST NOT go deeper than 5 levels. Given 5 services, A through E, A references B, B references C, etc. Given the maximum depth of 5, E MUST contain an absolute endpoint URI.

Another limitation is that a reference that originates from a compound service MUST NOT resolve to another compound service. When resolving that would lead to a map of URIs nested within a map of URIs.

An example of a deeply nested structure:

```javascript
{
  "@context": [ "https://www.w3.org/ns/did/v1" ],
  "id": "did:nuts:abc",
  "service": [
    {
      "id": "did:nuts:abc#1",
      "type": "NutsCompoundServiceRef",
      "serviceEndpoint": "did:nuts:123?type=NutsCompoundService"
    },
    {
      "id": "did:nuts:abc#2",
      "type": "NutsCompoundService",
      "serviceEndpoint": {
        "oauth": "did:nuts:123?type=oauth"
      }
    },
    {
      "id": "did:nuts:abc#3",
      "type": "oauth",
      "serviceEndpoint": "did:nuts:123?type=oauth_prod"
    },
    {
      "id": "did:nuts:abc#4",
      "type": "oauth_prod",
      "serviceEndpoint": "https:\\auth.example.com"
    }
  ]
}
```

### 4.2 Contact information

A DID MAY contain a service of type `node-contact-info` that contains information which can be used to contact the operator of the node that controls the DID document. The information MUST NOT contain any personally identifiable information \(PII\) such as personal names, e-mail addresses or telephone numbers. It SHOULD contain a company/unit name, e-mail address and/or telephone number instead. The `serviceEndpoint` MUST be a map and MUST contain the `email` property. It addition it MAY contain the following properties: `tel` \(telephone number\), `name` \(company/unit name\), `web` \(website URL\). All properties MUST be formatted as string. For example:

```javascript
{
    "serviceEndpoint": {
        "name": "Some Vendor",
        "email": "administrator@nuts.node",
        "telephone": "00316123456",
        "website": "https://www.nuts.node"
    }
}
```

The service MAY also refer to another DID's contact information service \(in case of a SaaS provider\). In this case the `serviceEndpoint` MUST be a DID URL as string which queries the referenced service:

```javascript
{
  "serviceEndpoint": "did:nuts:some-other-did?type=node-contact-info"
}
```

The reference MUST resolve to a contact information service as described above.

Since the information is self-proclaimed and not authenticated or verified in any way, applications MUST treat it as untrusted, with great care. Failing to do so could make the operator of the node vulnerable for spoofing and other attacks.

## 5. Conflict resolution

§3.4 of [RFC004](rfc004-verifiable-transactional-graph.md) requires each payload type to describe how conflicts are resolved when parallel transactions are encountered. A DID Document is not a CRDT, but its contents are. The paragraphs below describe how each of the elements should be treated in case of a conflict. First we describe the mechanism of detecting and resolving conflicts. DID Documents do not refer to their previous version in any way. The transactions described in [RFC004](rfc004-verifiable-transactional-graph.md) do refer to previous transactions. This system can be used to detect conflicts and to resolve them. For DID Documents additional transactional requirements are added:

* Updates to DID Documents MUST refer in their transaction to the transaction of the previous version. They SHOULD also refer to the latest transactions as described by [RFC005](rfc005-distributed-network-using-grpc.md).
* In case of a conflict, the conflict can be resolved with an additional transaction referring to the all the conflicting transactions.

## 5.1 Conflict detection

The algorithm for conflict detection can be summarized as follows:

* A DID Document update transaction is received.
* The current version is resolved together with the resolution metadata \(the metadata refers to the transaction\(s\) that created the current version\).
* If the received transaction refers to all transactions present in the metadata, the update is valid, and the DID Document is replaced.
* If the received transaction does not refer to all transactions, it is a conflict, and the rules from the paragraphs below are applied. 

A transaction is modelled as:

```text
hash // sha256(data)
data // A signed JWS in compact form, here it'll represent a DID Document. It also contains the `prevs` field which refers to previous transactions.
```

The `hash` values of the transactions MUST be stored for each DID Document. How these are stored is up to the implementation. The [did-core-spec](https://www.w3.org/TR/did-core/#did-resolution-metadata) defines the concept of resolution metadata when a DID Document is resolved. This metadata could contain the `hash` value, but this is implementation specific.

If a new transaction `tx` is received where `exists(tx.data.did)`. Let `current` be the existing DID Document and `current.meta` the resolution metadata of the existing DID Document. A conflict exists when:

```text
current.meta.hash - tx.data.prevs != []
```

When this condition is met, the metadata of the conflicted DID Document SHOULD refer to both conflicting transactions.

### 5.2 Element specific conflict resolution

#### 5.2.1 @context

The `@context` is determined by the protocol version and MUST be equal if both transactions use the same protocol version.

#### 5.2.2 id

The `id` identifies the DID Document and by definition will be equal for updates.

#### 5.2.3 controller

Controller entries are references to other DID Documents. If the list of controllers differs, the result MUST be a set of the elements from both documents combined.

#### 5.2.4 verification methods

Verification methods are defined by their public key and thus their contents. This makes each method immutable. When the list of methods differs between documents, the result MUST be a set of the elements from both documents combined.

#### 5.2.5 authentication, assertion, capabilityInvocation, capabilityDelegation & keyAgreement

These verification relationships only refer to verificationMethods by their id. This makes them immutable and can therefore use the same mechanism as described by §5.2.4.

#### 5.2.6 services

§4 describes services as immutable and non-referable by `id`. This makes a list of services suitable for merging as well. When a service is referenced by query and multiple services match due to a merger, it's up to the user of the service to handle this.

### 5.3 Conflict removal

The conflict can be removed by constructing a new update of the DID Document. The transaction for this update MUST refer to all values from `current.meta.hash`, where `current` is the resolved state of the DID Document.

## 6. Supported Cryptographic Algorithms and Key Types

Since RSA algorithms are deemed to be insecure for medium to long term, only elliptic curve-type algorithms are supported. The library support for newer algorithms \(e.g. `Ed25519`\) and curves \(`X25519`\) however is limited, so for now only the `secp256r1`, `secp384r1` and `secp521r1` NIST curves MUST be supported. This curve is considered to provide enough security for the next 10 years, according to the [Dutch Cyber Security Council](https://www.ncsc.nl/).

It is expected however, that as library support improves more \(stronger\) algorithms and key types will be supported, which should be taken in account by implementors.

## 7. Deployment scenarios

The definition of the Nuts DID method enables a wide variety of usages. To better understand the usages, this chapter illustrates some example scenarios.

This section is non-normative.

### 7.1 SaaS provider

This example consists of a SaaS provider that acts as enabler, controller and node operator for all of its customers. The SaaS provider has access to all the key material, while the care organizations hasn't.

The SaaS provider registers itself with:

```javascript
{
  "@context": [ "https://www.w3.org/ns/did/v1" ],
  "id": "did:nuts:1",
  "verificationMethod": [
    {
      "id": "did:nuts:1#_TKzHv2jFIyvdTGF1Dsgwngfdg3SH6TpDv0Ta1aOEkw",
      "controller": "did:nuts:1",
      "type": "JsonWebKey2020",
      "publicKeyJwk": {
        "crv": "P-256",
        "x": "38M1FDts7Oea7urmseiugGW7tWc3mLpJh6rKe7xINZ8",
        "y": "nDQW6XZ7b_u2Sy9slofYLlG03sOEoug3I0aAPQ0exs4",
        "kty": "EC"
      }
    },
    {
      "id": "did:nuts:1#_kalsjdyurtnAa4895akljnjghl584B9lkEJHNLJKFGA",
      "controller": "did:nuts:1",
      "type": "JsonWebKey2020",
      "publicKeyJwk": {
        "crv": "P-256",
        "x": "38M1FDts7Oea7urmseiugGW7tWc3mLpJh6rKe7xINZ8",
        "y": "nDQW6XZ7b_u2Sy9slofYLlG03sOEoug3I0aAPQ0exs4",
        "kty": "EC"
      }
    }],
  "capabilityInvocation": [
    "did:nuts:1#_TKzHv2jFIyvdTGF1Dsgwngfdg3SH6TpDv0Ta1aOEkw",
    "did:nuts:1#_kalsjdyurtnAa4895akljnjghl584B9lkEJHNLJKFGA"
  ],
  "service": [
    {
      "id": "did:nuts:123#Dsgwngfdg3SH6TpDTa1",
      "type": "oauth",
      "serviceEndpoint": "https://example.com/oauth"
    },
    {
      "id": "did:nuts:123#Dsgwngf3SH6TpDv0Ta1",
      "type": "fhir",
      "serviceEndpoint": "https://example.com/fhir"
    }
  ]
}
```

It registers two keys: one is kept offline as backup, and the other is available in the software of the SaaS provider.

The SaaS provider registers a care organization as:

```javascript
{
  "@context": [ "https://www.w3.org/ns/did/v1" ],
  "id": "did:nuts:2",
  "verificationMethod": [
    {
      "id": "did:nuts:2#_TKzHv2jFIyvdTGF1Dsgwngfdg3SH6TpDv0Ta1aOEkw",
      "controller": "did:nuts:2",
      "type": "JsonWebKey2020",
      "publicKeyJwk": {
        "crv": "P-256",
        "x": "38M1FDts7Oea7urmseiugGW7tWc3mLpJh6rKe7xINZ8",
        "y": "nDQW6XZ7b_u2Sy9slofYLlG03sOEoug3I0aAPQ0exs4",
        "kty": "EC"
      }
    }],
  "capabilityInvocation": [],
  "controller": [
    "did:nuts:1"
  ],
  "service": [
    {
      "id": "did:nuts:abc#Dsgwngfdg3SH6TpDv0Ta1",
      "type": "NutsCompoundService",
      "serviceEndpoint": {
        "oauth": "did:nuts:1?type=oauth",
        "fhir": "did:nuts:1?type=fhir"
      }
    }
  ]
}
```

The care organization doesn't have a `capabilityInvocation` key. The SaaS provider will not store the key that generated the DID since that key is not in control of the DID document. The DID document of the SaaS provider is the controller of the DID document.

### 7.2 Hospital

We assume that a hospital has its own data centre and therefore, runs its own node. This means that the DID document of the hospital doesn't need an additional controller. The hospital is in control of its own private keys. It does however, need to take precautions for private key loss/theft.

The hospital would be able to register a single DID document:

```javascript
{
  "@context": [ "https://www.w3.org/ns/did/v1" ],
  "id": "did:nuts:1",
  "verificationMethod": [
    {
      "id": "did:nuts:1#_TKzHv2jFIyvdTGF1Dsgwngfdg3SH6TpDv0Ta1aOEkw",
      "controller": "did:nuts:1",
      "type": "JsonWebKey2020",
      "publicKeyJwk": {
        "crv": "P-256",
        "x": "38M1FDts7Oea7urmseiugGW7tWc3mLpJh6rKe7xINZ8",
        "y": "nDQW6XZ7b_u2Sy9slofYLlG03sOEoug3I0aAPQ0exs4",
        "kty": "EC"
      }
    },
    {
      "id": "did:nuts:1#_kalsjdyurtnAa4895akljnjghl584B9lkEJHNLJKFGA",
      "controller": "did:nuts:1",
      "type": "JsonWebKey2020",
      "publicKeyJwk": {
        "crv": "P-256",
        "x": "38M1FDts7Oea7urmseiugGW7tWc3mLpJh6rKe7xINZ8",
        "y": "nDQW6XZ7b_u2Sy9slofYLlG03sOEoug3I0aAPQ0exs4",
        "kty": "EC"
      }
    }],
  "capabilityInvocation": [
    "did:nuts:1#_TKzHv2jFIyvdTGF1Dsgwngfdg3SH6TpDv0Ta1aOEkw"
  ],
  "assertion": [
    "did:nuts:1#_kalsjdyurtnAa4895akljnjghl584B9lkEJHNLJKFGA"
  ],
  "service": [
    {
      "id": "did:nuts:1#Dsgwngfdg3SH6TpDTa1",
      "type": "oauth",
      "serviceEndpoint": "https://example.com/oauth"
    },
    {
      "id": "did:nuts:1#Dsgwngf3SH6TpDv0Ta1",
      "type": "fhir",
      "serviceEndpoint": "https://example.com/fhir"
    },
    {
      "id": "did:nuts:abc#Dsgwngfdg3SH6TpDv0Ta1",
      "type": "NutsCompoundService",
      "serviceEndpoint": {
        "oauth": "did:nuts:1?type=oauth",
        "fhir": "did:nuts:1?type=fhir"
      }
    }
  ]
}
```

The hospital registered 2 keys, one is used for assertions and one for authorization as the DID controller. The authorization key, listed under `capabilityInvocation` is kept offline, meaning that any change to the DID document will require an administrator to sign the changes manually. The assertion key is available online and used within the defined services. All services are registered directly on the DID document.

### 7.3 Single person deployment

It is possible to deploy a node as a person. You'll probably not offer any services, but you'll still be able to consume them. Also losing key material is less of a problem since you only have to restore it for yourself.

This example is the most simple, there's one key and it's used for all cases.

```javascript
{
  "@context": [ "https://www.w3.org/ns/did/v1" ],
  "id": "did:nuts:1",
  "verificationMethod": [
    {
      "id": "did:nuts:1#_TKzHv2jFIyvdTGF1Dsgwngfdg3SH6TpDv0Ta1aOEkw",
      "controller": "did:nuts:1",
      "type": "JsonWebKey2020",
      "publicKeyJwk": {
        "crv": "P-256",
        "x": "38M1FDts7Oea7urmseiugGW7tWc3mLpJh6rKe7xINZ8",
        "y": "nDQW6XZ7b_u2Sy9slofYLlG03sOEoug3I0aAPQ0exs4",
        "kty": "EC"
      }
    }],
  "capabilityInvocation": [
    "did:nuts:1#_TKzHv2jFIyvdTGF1Dsgwngfdg3SH6TpDv0Ta1aOEkw"
  ],
  "assertion": [
    "did:nuts:1#_TKzHv2jFIyvdTGF1Dsgwngfdg3SH6TpDv0Ta1aOEkw"
  ],
  "service": []
}
```

## Appendix A: Design decisions

### A.1 DID generation

The choice has been made to use [rfc7638](https://tools.ietf.org/html/rfc7638) over generating a hash from the public key bytes. rfc7638 Describes how to generate a hash for public keys. If a public key can be presented as a JWK [rfc7517](https://tools.ietf.org/html/rfc7517) then it is possible to generate a fingerprint. There's no alternative that describes how a hash can be computed over different key types. JWK libraries are present for a multitude of languages. Base58 encoding is used since a DID can not contain `-` and `_`.

