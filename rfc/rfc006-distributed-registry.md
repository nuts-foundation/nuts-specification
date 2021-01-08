# RFC006 Distributed Registry with Decentralized Identifiers (DID)

|  |  |
| :--- | :--- |
| Nuts foundation | W.M. Slakhorst |
| Request for Comments: 006 | Nedap |
|  | S. van der Vegt |
|  | Nedap |
|  | R.G. Krul |
|  | Nedap |
|  | January 2021 |

## Distributed Registry with Decentralized Identifiers (DID)
### Abstract

This RFC describes a protocol to build a registry containing information required for (care) organizations to exchange data.
The registry typically contains organizations, software vendors acting on behalf of their client (organizations),
data exchange services offered by organizations and their technical endpoints.
It describes how these are mapped to [Decentralized Identifiers (DID)](https://www.w3.org/TR/did-core/) and how DID Documents
are encapsulated in [RFC004 Distributed Documents](rfc004-distributed-document-format.md) to provide cryptographic
integrity and consistent state across distributed networks.

### Status

This document is currently a draft.

### Copyright Notice
![](../.gitbook/assets/license.png)

This document is released under the [Attribution-ShareAlike 4.0 International \(CC BY-SA 4.0\) license](https://creativecommons.org/licenses/by-sa/4.0/).

## 1. Introduction

When care organizations want to exchange data using Nuts they need to know where to find that data and how to authenticate it.
This knowledge is recorded in a distributed registry, writable and queryable by all network participants.
This RFC describes how to create, update and resolve the data structures required to achieve that goal.

## 2. Terminology

* **Bolt**: a use case built on top of the functionality Nuts provides.
* **JWS**: JSON Web Signature as specified by [RFC004](rfc004-distributed-document-format.md).
* **Organization**: a care organization exchanging data with other care organizations over a Nuts network.
  By giving other DIDs control over its DID Document organizations can delegate data exchange to another party,
  e.g. a care software vendor or SaaS provider.
* **Endpoint**: a URI or URL exposed by a network participant which can be used by other participants to pull data.
* **Service**: a group of endpoints implementing a service required by a Bolt.

## 3. Nuts DID Method

The Nuts DID URI scheme is defined as follows:
```
did = "did:nuts:" idstring
idstring = 21*22(base58char)
base58char = "1" / "2" / "3" / "4" / "5" / "6" / "7" / "8" / "9" / "A" / "B" / "C"
    / "D" / "E" / "F" / "G" / "H" / "J" / "K" / "L" / "M" / "N" / "P" / "Q"
    / "R" / "S" / "T" / "U" / "V" / "W" / "X" / "Y" / "Z" / "a" / "b" / "c"
    / "d" / "e" / "f" / "g" / "h" / "i" / "j" / "k" / "m" / "n" / "o" / "p"
    / "q" / "r" / "s" / "t" / "u" / "v" / "w" / "x" / "y" / "z"
    
```

Where the `idstring` is derived from the public key:

`idstring = BASE-58(SHA-256(raw-public-key-bytes))`

For example, consider the following Ed25519 key (as JWK):

```json
{
  "kty" : "OKP",
  "crv" : "Ed25519",
  "x"   : "11qYAYKxCrfVS_7TyWQHOg7hcvPapiMlrwIaaPcHURo",
  "use" : "sig",
  "kid" : "FdFYFzERwC2uCBB46pZQi4GG85LujR8obt-KWRBICVQ"
}
```

For this key type the `x` parameter is used to derive `idstring`:
`idstring = BASE-59(SHA-256(BASE64URL-DECODE(key.x)))`

Outputs:
```json
{
  "id": "did:nuts:e3cacd5c2d931295a64f6c3bb3f6ea58c3a9b253b990e32c5abce43c2f94c564"
}
```

Nuts DID Documents are wrapped in a JWS (JSON Web Signature) to ensure cryptographic authenticity and integrity through
([RFC004](rfc004-distributed-document-format.md)). Please refer to that RFC on how to create the JWS.

### 3.1 Namespace Specific Identifier (NSI)

Identifiers are derived from public keys that are valid at the moment of creating the DID document. 
It MUST be the public key that corresponds to the private key that was used to sign the JWS.
The public key MUST also be present in the `verificationMethods` and referenced by the `authentication` field. 
When multiple keys are present, one MUST verify in this matter.

### 3.2 Method operations

DID documents are enclosed in a message envelope to ensure consistency in the network.
The envelope is in the form of a JWS as described in [RFC004](rfc004-distributed-document-format.md).
Once the network layer has confirmed the signature of the JWS, the registry MUST validate if the submitter is authorized to create, update or delete the document.
If the authorization fails, the document should be ignored.

The `controller` field MAY be present. If no `controller` field is present, the DID subject itself is the controller.
If the `controller` field is present, only the DID subjects from the `controller` field can change the DID document.

#### 3.2.1 Create (Register)

A Create operation for a DID Document puts the following additional requirements on the JWS header parameters:

- `jwk` MUST be present
- `cty` MUST contain the value `application/json+did-document`
- `tiv` MUST be absent or `0`
- `tid` MUST be absent

The `kid` field of the `jwk` header parameter MUST be prefixed by the `id` of the DID document.
In order for the contents to be accepted in the Nuts registry, the JWK MUST match the `authentication` key in the DID document with the same identifier. 
The `kid` field from the JWK MUST match the `id` from the verification key in the DID document.

Example JWS header

```json
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

- the `id` MUST be generated according to the method specified at the beginning of §3
- at least 1 key MUST be present in the `verificationMethod` and `authentication`
- all key references in `authentication` MUST refer to keys listed under `verificationMethods`

Each key listed in `verificationMethod` MUST have an `id` equal to the DID followed by a `#` and the public key fingerprint according to [rfc7638](https://tools.ietf.org/html/rfc7638)
Each key MUST be of type `JsonWebKey2020` according to §5.3.1 of the [did-core-spec](https://www.w3.org/TR/did-core/#key-types-and-formats)

The `controller` field MAY be present. This RFC follows the [did-core-spec](https://www.w3.org/TR/did-core/#did-controller).

The `assertion` field MAY be present. 
Keys referenced from this field are used for signing Verifiable Credentials/Presentations and for signing JWTs in the OAuth flow.

Example DID document:

```json
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
  "authentication": ["did:nuts:123#_TKzHv2jFIyvdTGF1Dsgwngfdg3SH6TpDv0Ta1aOEkw"],
  "service": []
}
```

#### 3.2.2 Read (Resolve)
A Nuts DID can only be resolved locally. The concept of the Nuts registry is the state based upon all Create, Update and Delete operations received through the Nuts Network.
Therefore, any DID document SHOULD already be present in local storage.

##### 3.2.2.1 Resolution Input Metadata
All historic versions of a DID Document SHOULD be stored and queryable. This allows clients to resolve the document for a specific moment in time (e.g. a previous version) instead of the last one. This allows for resolving keys and services at a given moment

##### 3.2.2.2 Document Metadata
The resolved DID Document Metadata contains the `created` and `updated` fields, in accordance with the [did-core-spec](https://www.w3.org/TR/did-core/#did-document-metadata-properties). They are
derived from the underlying Nuts Documents. `created` MUST contain the `sigt` timestamp from the first version of the
document. `updated` MUST contain the `sigt` timestamp of the last version of the DID Document.

#### 3.2.3 Update (Replace)
The complete document gets replaced with a newer version. 

Changes to DID documents can only be accepted if the update is signed with a current controller authentication key:

- a key referenced from the `authentication` section of the latest DID document version.
- a key referenced from the `authentication` section of a controller. One of the entries in the `controller` field MAY refer to a different DID Document.
  The `authentication` keys of the latest version of that DID Document can be used as authorized key.
- a key referenced from the `authentication` section of the given DID document if it is a `Create` action.

The following requirements on the JWS header parameter apply:

- `kid` MUST hold the reference to the correct key.
- `tid` and `tiv` MUST be filled according to [RFC004](rfc004-distributed-document-format.md).

Example JWS header:
```json
{
  "alg": "PS256",
  "cty": "application/json+did-document",
  "kid": "did:nuts:123#_TKzHv2jFIyvdTGF1Dsgwngfdg3SH6TpDv0Ta1aOEkw",
  "crit": ["sigt", "ver","prevs"],
  "sigt": "2020-01-06T15:00:00Z",
  "ver": "1",
  "prevs": ["148b3f9b46787220b1eeb0fc483776beef0c2b3e"],
  "tid": "SflKxwRJSMeKKF2QT4fwpMeJf36POk6yJV_adQssw5c",
  "tiv": "1"
}
```

#### 3.2.4 Delete (Revoke)

DID Documents cannot be deleted as in being "erased", only its keys can be removed as to prevent future changes (revocation).
To revoke the keys to prevent future updates;

1. Remove all keys (specified by `verificationMethod`) and references (specified by e.g. `authentication`) to
   these keys from the document. 
2. Remove all controllers from the document.

Deletion can not be undone.

Example DID document:

```json
{
  "@context": [ "https://www.w3.org/ns/did/v1" ],
  "id": "did:nuts:123",
  "controller": [],
  "verificationMethod": [],
  "authentication": []
}
```

### 3.3 Security Considerations

Almost all security considerations are covered by the mechanisms described in [RFC004](rfc004-distributed-document-format.md). An overview of countermeasures:

- **eavesdropping** - All communications is sent over two-way TLS. All data is public anyway.
- **replay** - DID documents are identified and published by their hash (SHA-256). Replaying will result in replaying the exact same content.
- **message insertion** - [RFC004](rfc004-distributed-document-format.md) defines hashing and signing of published documents.
- **deletion** - All DID documents are published and copied on a mesh network. Deletion of a single document will only occur locally and will not damage other nodes.
- **modification** - DID documents can only be modified if they are published with a signature from one of the `authentication` keys.
- **man-in-the-middle** - All communications is sent over two-way TLS and all documents are signed. A DID can not be hijacked since it is derived from the public key.
- **denial of service** - This is out of scope and handled by [RFC004](rfc004-distributed-document-format.md).

#### 3.3.1 Protection against DID hijacking

The Nuts network is a mesh network without central authority. This means that any party can generate a DID. 
This DID must be protected against forgery and hijacking since duplicates are accepted in the Nuts network. 
The duplicates are sorted and one will eventually be accepted (consistency rules of [RFC004](rfc004-distributed-document-format.md)). This would open up a DID to hijacking. 
Therefore, the DID MUST be a derivative of the public key used to sign the document as described in §3. 

#### 3.3.2 Protection against loss of private key

The loss of a single private key can be countered by registering multiple keys. Keys can be kept offline, in a vault for example.
Such a key can later be used to register new keys when needed. Another option is to add a second controller that acts as an emergency backup.
The keys of that controller can be kept offline.

#### 3.3.3 Protection against theft of private key

A stolen key can alter the DID document in such a way that the attacker can get full control with a new key and can exclude the previous owner from making changes.
Appropriate measures MUST be taken to keep authentication keys secure. 

When control over a DID document has been lost, the DID subject will have to have all Verifiable Credentials revoked. 
Without Verifiable Credentials linked to the DID document, the DID document no longer has any value.
The DID subject will have to go through the process of reacquiring all Verifiable Credentials for a new DID document.

### 3.4 Privacy considerations

All data is public knowledge. All considerations from [§10 of did-core](https://www.w3.org/TR/did-core/#privacy-considerations) apply.

## 4. Services
It is to be expected that each DID subject will add services to the DID Document. Specific services will be specified in their own RFC or Bold specification.
A service can define an absolute endpoint URI or be a compound service, referring to a set of services. This is often the case
when a SaaS provider defines endpoints to be used for all clients.

For an absolute endpoint URI the `serviceEndpoint` MUST be a string containing the URI. For a compound service the
`serviceEndpoint` MUST contain a map containing references to absolute endpoint URI services.

The service identifier MUST be constructed from the DID followed by a `#`, the service type, a `-` and an identifier unique to the DID document.

Below is an example of a service registered by a care organization that uses the endpoints from a SaaS provider:

The SaaS provider defines the actual URL:
```json
{
  "@context": [ "https://www.w3.org/ns/did/v1" ],
  "id": "did:nuts:123",
  "service": [
    {
      "id": "did:nuts:123#NutsOAuth-1",
      "type": "NutsOAuth",
      "serviceEndpoint": "https://example.com/oauth"
    },
    {
      "id": "did:nuts:123#NutsFHIR-1",
      "type": "NutsFHIR",
      "serviceEndpoint": "https://example.com/fhir"
    }
  ]
}
```

The care organisation refers to it:
```json
{
  "@context": [ "https://www.w3.org/ns/did/v1" ],
  "id": "did:nuts:abc",
  "service": [
    {
      "id": "did:nuts:abc#NutsCompoundService-1",
      "type": "NutsCompoundService",
      "serviceEndpoint": {
        "oauthEndpoint": "did:nuts:123#NutsOAuth-1",
        "fhirEndpoint": "did:nuts:123#NutsFHIR-1"
      }
    }
  ]
}
```

## 5. Supported Cryptographic Algorithms and Key Types

Since RSA algorithms are deemed to be insecure for medium to long term, only elliptic curve-type algorithms are supported.
The library support for newer algorithms (e.g. `Ed25519`) and curves (`X25519`) however is limited, so for now only
the `secp256r1`, `secp384r1` and `secp521r1` NIST curves MUST be supported. This curve is considered to provide enough security for the next 10 years,
according to the [Dutch Cyber Security Council](https://www.ncsc.nl/).

It is expected however, that as library support improves more (stronger) algorithms and key types will be supported,
which should be taken in account by implementors.

## 6. Deployment scenarios

The definition of the Nuts DID method enables a wide variety of usages.
To better understand the usages, this chapter illustrates some example scenarios.

This section is non-normative.

### 6.1 SaaS provider

This example consists of a SaaS provider that acts as enabler, controller and node operator for all of its customers. The SaaS provider has access to all the key material, while the care organizations hasn't.

The SaaS provider registers itself with:

```json
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
  "authentication": [
    "did:nuts:1#_TKzHv2jFIyvdTGF1Dsgwngfdg3SH6TpDv0Ta1aOEkw",
    "did:nuts:1#_kalsjdyurtnAa4895akljnjghl584B9lkEJHNLJKFGA"
  ],
  "service": [
    {
      "id": "did:nuts:123#NutsOAuth-1",
      "type": "NutsOAuth",
      "serviceEndpoint": "https://example.com/oauth"
    },
    {
      "id": "did:nuts:123#NutsFHIR-1",
      "type": "NutsFHIR",
      "serviceEndpoint": "https://example.com/fhir"
    }
  ]
}
```

It registers two keys: one if kept offline as backup, and the other is available in the software of the SaaS provider.

The SaaS provider registers a care organization as:

```json
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
  "authentication": [
    "did:nuts:2#_TKzHv2jFIyvdTGF1Dsgwngfdg3SH6TpDv0Ta1aOEkw"
  ],
  "controller": [
    "did:nuts:1"
  ],
  "service": [
    {
      "id": "did:nuts:abc#NutsCompoundService-1",
      "type": "NutsCompoundService",
      "serviceEndpoint": {
        "oauthEndpoint": "did:nuts:1#NutsOAuth-1",
        "fhirEndpoint": "did:nuts:1#NutsFHIR-1"
      }
    }
  ]
}
```

The care organization does have an `authentication` key, this is required for the generation of the `id`. The SaaS provider will most likely not store the key since that key is not in control of the DID document.
Since this key isn't after initial creation of the document the SaaS provider SHOULD remove it from the authentication document afterwards to improve security and clarity.
The DID document of the SaaS provider is the controller of the DID document.

### 6.2 Hospital

We assume that a hospital has its own data centre and therefore, runs its own node. 
This means that the DID document of the hospital doesn't need an additional controller. The hospital is in control of its own private keys. It does however, need to take precautions for private key loss/theft.

The hospital would be able to register a single DID document:

```json
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
  "authentication": [
    "did:nuts:1#_TKzHv2jFIyvdTGF1Dsgwngfdg3SH6TpDv0Ta1aOEkw"
  ],
  "assertion": [
    "did:nuts:1#_kalsjdyurtnAa4895akljnjghl584B9lkEJHNLJKFGA"
  ],
  "service": [
    {
      "id": "did:nuts:1#NutsOAuth-1",
      "type": "NutsOAuth",
      "serviceEndpoint": "https://example.com/oauth"
    },
    {
      "id": "did:nuts:1#NutsFHIR-1",
      "type": "NutsFHIR",
      "serviceEndpoint": "https://example.com/fhir"
    },
    {
      "id": "did:nuts:abc#NutsCompoundService-1",
      "type": "NutsCompoundService",
      "serviceEndpoint": {
        "oauthEndpoint": "did:nuts:1#NutsOAuth-1",
        "fhirEndpoint": "did:nuts:1#NutsFHIR-1"
      }
    }
  ]
}
```

The hospital registered 2 keys, one if used for assertions and one for authentication. 
The authentication key is kept offline, meaning that any change to the DID document will require an administrator to sign the changes manually.
The assertion key is available online and used within the defined services.
All services are registered directly on the DID document.

### 6.3 Single person deployment

It is possible to deploy a node as a person. You'll probably not offer any services, but you'll still be able to consume them.
Also losing key material is less of a problem since you only have to restore it for yourself.

This example is the most simple, there's one key and it's used for all cases.

```json
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
  "authentication": [
    "did:nuts:1#_TKzHv2jFIyvdTGF1Dsgwngfdg3SH6TpDv0Ta1aOEkw"
  ],
  "assertion": [
    "did:nuts:1#_TKzHv2jFIyvdTGF1Dsgwngfdg3SH6TpDv0Ta1aOEkw"
  ],
  "service": []
}
```

## Current issues

### Management of VCs by a trusted service provider
Considering the case a care provider has outsourced it key management to a service provider.
When obtaining a VC from a trusted party, how does the care provider prove that it is the party represented by the DID without being able to provide private key material?

### Resolvability of the DID document
The fundamental idea of a DID document is that it should be resolvable by other parties. Section [7.2.2 of the did-core spec](https://w3c.github.io/did-core/#read-verify) requires a specification how a DID resolver could resolve and verify a DID document from the registry. Since the Nuts registry will be a local registry this is not yet a consideration but when federation with other registries will become relevant, a proper specification should be written.