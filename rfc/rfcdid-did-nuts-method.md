# Nuts DID Method Specification

## About

## Abstract

## Status
Draft

#### Copyright Notice
![](../.gitbook/assets/license.png)

This document is released under the [Attribution-ShareAlike 4.0 International \(CC BY-SA 4.0\) license](https://creativecommons.org/licenses/by-sa/4.0/).

## 1. Nuts DID Method
The Nuts DID scheme is defined as follows:
```
did = "did:nuts:" idstring
idstring = 21*22(base58char)
base58char = "1" / "2" / "3" / "4" / "5" / "6" / "7" / "8" / "9" / "A" / "B" / "C"
    / "D" / "E" / "F" / "G" / "H" / "J" / "K" / "L" / "M" / "N" / "P" / "Q"
    / "R" / "S" / "T" / "U" / "V" / "W" / "X" / "Y" / "Z" / "a" / "b" / "c"
    / "d" / "e" / "f" / "g" / "h" / "i" / "j" / "k" / "m" / "n" / "o" / "p"
    / "q" / "r" / "s" / "t" / "u" / "v" / "w" / "x" / "y" / "z"
    
```

Where the `idstring` is generated from taking the SHA-256 hash of the public key from a Ed25519:

`idstring = BASE-58(SHA-256(raw-public-key-bytes))`

Example:
Consider the following JWK encoded Ed25519.

```javascript
var key = {
  "kty" : "OKP",
  "crv" : "Ed25519",
  "x"   : "11qYAYKxCrfVS_7TyWQHOg7hcvPapiMlrwIaaPcHURo",
  "d"   : "nWGxne_9WmC6hEr0kuwsxERJxWl7MmkZcDusAxyuf2A",
  "use" : "sig",
  "kid" : "FdFYFzERwC2uCBB46pZQi4GG85LujR8obt-KWRBICVQ"
}
```
Where the `x` parameter is the base64 encoded public key.
`idString = Base58Encode(Sha256(Base64urlDecode(key.x)))`

example:
```json
{
  "id": "did:nuts:e3cacd5c2d931295a64f6c3bb3f6ea58c3a9b253b990e32c5abce43c2f94c564"
}
```

## 2. Namespace Specific Identifier (NSI)

Identifiers are derived from public keys that are valid at the moment of creating the DID document. 
It MUST be the public key that corresponds to the private key that was used to sign the Nuts registry document (RFC004).
The public key MUST also be present in the `verificationMethods`. When multiple keys are present, one MUST verify in this matter.

## 3. Method operations

DID documents are enclosed in a message envelope to ensure consistency in the network.
The envelope is in the form of a JWS as described in [RFC004].
Once the network layer has confirmed the signature of the JWS, the registry MUST validate if the submitter is authorized to create, update or delete the document.
If the authorization fails, the document should be ignored.

Changes to DID documents can only be accepted if the update is signed with one of the following keys:

- a key referenced from the `authentication` section of the latest DID document version.
- a key referenced from the `authentication` section of a controller. One of the entries in the `controller` field MAY refer to a different DID Document. 
  The `authentication` keys of the latest version of that DID Document can be used as authorized key.
- a key referenced from the `authentication` section of the given DID document if it is a `Create` action.

The `controller` field MAY be present. If no `controller` field is present, the DID subject itself is the controller.
If the `controller` field is present, only the DID subjects from the `controller` field can change the DID document.

jws specification

`cty` MUST contain the value `application/json+did-document`


### Create (Register)

Example JOSE header

```json
{
  "alg": "PS256",
  "cty": "application/json+did-document",
  "kid": "did:nuts:123#_TKzHv2jFIyvdTGF1Dsgwngfdg3SH6TpDv0Ta1aOEkw",
  "jwk": {
    "crv": "P-256",
    "x": "38M1FDts7Oea7urmseiugGW7tWc3mLpJh6rKe7xINZ8",
    "y": "nDQW6XZ7b_u2Sy9slofYLlG03sOEoug3I0aAPQ0exs4",
    "kty": "EC",
    "kid": "_TKzHv2jFIyvdTGF1Dsgwngfdg3SH6TpDv0Ta1aOEkw"
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

Jws payload:

```json
{
  "id": "<nuts did id>",
  "controller": [],
  "verificationMethod": [{
    "id": "did:nuts:123#_TKzHv2jFIyvdTGF1Dsgwngfdg3SH6TpDv0Ta1aOEkw",
    "type": "JsonWebKey2020",
    "controller": "did:nuts:123",
    "publicKeyJwk": {
      "crv": "P-256",
      "x": "38M1FDts7Oea7urmseiugGW7tWc3mLpJh6rKe7xINZ8",
      "y": "nDQW6XZ7b_u2Sy9slofYLlG03sOEoug3I0aAPQ0exs4",
      "kty": "EC",
      "kid": "_TKzHv2jFIyvdTGF1Dsgwngfdg3SH6TpDv0Ta1aOEkw"
    }
  }],
  "authentication": ["did:example:123#_TKzHv2jFIyvdTGF1Dsgwngfdg3SH6TpDv0Ta1aOEkw"],
  "service": []
}
```

Example signature:
```
SflKxwRJSMeKKF2QT4fwpMeJf36POk6yJV_adQssw5c
```

### Read (Resolve)
A Nuts DID can only be resolved locally. The concept of the Nuts registry is the state based upon all Create, Update and Delete operations received through the Nuts Network.
Therefore, any DID SHOULD already be present in the local storage.

```json
{
  "document": {
    "@context": [ "https://www.w3.org/ns/did/v1" ],
    "id": "did:nuts:a665a45920422f9d417e4867efdc4fb8a04a1f3fff1fa07e998e86f7f7a27ae3",
    "controller": [],
    "verificationMethod": [],
    "authentication": [],
    "service":[]
  },
  "metadata": {
    "kid": "did:nuts:a665a45920422f9d417e4867efdc4fb8a04a1f3fff1fa07e998e86f7f7a27ae3#key1",
    "created": "2021-01-04T12:15:06Z",
    "updated": "2021-01-05T10:27:22Z"
  }
}
```

### Update (Replace)
The complete document gets replaced.

The following example shows a replace (not a patch):
Example JOSE header
```json
{
  "alg": "PS256",
  "cty": "application/json+did-document",
  "kid": "did:nuts:123#_TKzHv2jFIyvdTGF1Dsgwngfdg3SH6TpDv0Ta1aOEkw",
  "crit": ["sigt", "ver","prevs"],
  "sigt": "",
  "ver": "1",
  "prevs": ["148b3f9b46787220b1eeb0fc483776beef0c2b3e"],
  "tid": "SflKxwRJSMeKKF2QT4fwpMeJf36POk6yJV_adQssw5c",
  "tiv": "1"
}
```

Example payload:
```json
{
  "id": "did:nuts:123",
  "controller": [],
  "verificationMethod": [{
    "id": "did:nuts:123#_TKzHv2jFIyvdTGF1Dsgwngfdg3SH6TpDv0Ta1aOEkw",
    "type": "JsonWebKey2020",
    "controller": "did:nuts:123",
    "publicKeyJwk": {
      "crv": "P-256",
      "x": "38M1FDts7Oea7urmseiugGW7tWc3mLpJh6rKe7xINZ8",
      "y": "nDQW6XZ7b_u2Sy9slofYLlG03sOEoug3I0aAPQ0exs4",
      "kty": "EC",
      "kid": "_TKzHv2jFIyvdTGF1Dsgwngfdg3SH6TpDv0Ta1aOEkw"
    }
  }],
  "authentication": ["did:example:123#_TKzHv2jFIyvdTGF1Dsgwngfdg3SH6TpDv0Ta1aOEkw"],
  "service":[]
}
```

### Delete (Revoke)

DID Documents cannot be deleted as in being "erased", only its keys can be removed as to prevent future changes (revocation).
To revoke the keys to prevent future updates;

1. Remove all keys (specified by `verificationMethod`) and references (specified by e.g. `authentication`) to
   these keys from the document. 
2. Remove all controllers from the document.

Deletion can not be undone.

## 4. Security Considerations

Almost all security considerations are covered by the mechanisms described in RFC004. An overview of countermeasures:

- **eavesdropping** - All communications is sent over two-way TLS. All data is public anyway.
- **replay** - DID documents are identified and published by their hash (SHA256). Replaying will result in replaying the exact same content.
- **message insertion** - RFC004 defines hashing and signing of published documents.
- **deletion** - All DID documents are published and copied on a mesh network. Deletion of a single document will only occur locally and will not damage other nodes.
- **modification** - DID documents can only be modified if they are published with a signature from one of the `authentication` keys.
- **man-in-the-middle** - All communications is sent over two-way TLS and all documents are signed. A DID can not be hijacked since it is derived from the public key.
- **denial of service** - This is out of scope and handled by RFC004.

### 4.1 Protection against DID hijacking

The Nuts network is a mesh network without central authority. This means that any party can generate a DID. 
This DID must be protected against forgery and hijacking since duplicates are accepted in the Nuts network. 
The duplicates are sorted and one will eventually be accepted (consistency rules of RFC004). This would open up a DID to hijacking. 
Therefore, the DID MUST be a derivative of the public key used to sign the document as described in chapter 2. 

### 4.2 Protection against loss of private key

The loss of a single private key can be countered by registering multiple keys. Keys can be kept offline, in a vault for example.
Such a key can later be used to register new keys when needed. Another option is to add a second controller that acts as an emergency backup.
The keys of that controller can be kept offline.

### 4.3 Protection against theft of private key

A stolen key can alter the DID document in such a way that the attacker can get full control with a new key and can exclude the previous owner from making changes.
Appropriate measures MUST be taken to keep authentication keys secure. 

When control over a DID document has been lost, the DID subject will have to have all Verifiable Credentials revoked. 
Without Verifiable Credentials linked to the DID document, the DID document no longer has any value.
The DID subject will have to go through the process of reacquiring all Verifiable Credentials for a new DID document.

## 5. Privacy considerations

All data is public knowledge. All considerations from [§10 of did-core](https://www.w3.org/TR/did-core/#privacy-considerations) apply.