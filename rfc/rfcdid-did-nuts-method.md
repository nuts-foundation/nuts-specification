# Nuts DID Method Specification

## About

## Abstract

## Status
Draft

#### Copyright Notice
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

## 2. Namespace Specific Identifier (NSI)

Identifiers are derived from public keys that are valid at the moment of creating the DID document. 
It MUST be the public key that corresponds to the private key that was used to sign the Nuts registry document (RFC004).
The public key MUST also be present in the `verificationMethods`. When multiple keys are present, one MUST verify in this matter.

## 3. Method operations

Changes to DID documents can only be accepted if the update is signed with one of the following keys:

- a key referenced from the `authentication` section of the latest DID document version.
- a key referenced from the `authentication` section of a controller. One of the entries in the `controller` field MAY refer to a different DID Document. 
  The `authentication` keys of the latest version of that DID Document can be used as authorized key.
- a key referenced from the `authentication` section of the given DID document if it is a `Create` action.

The `controller` field MAY be present. If no `controller` field is present, the DID subject itself is the controller.
If the `controller` field is present, only the DID subjects from the `controller` field can change the DID document.

### Create (Register)

Jws payload:
```json
{
  "id": "<nuts did id>",
  "controller": [],
  "verificationMethod": [],
  "authentication": [],
  "service":[]
}
```

### Read (Resolve)
A Nuts DID can only be resolved locally. The concept of the Nuts registry is the state based upon all Create, Update and Delete operations received through the Nuts Network.
Therefore any DID SHOULD already be present in the local storage.

### Update (Replace or patch?)
Only the following properties COULD be updated:


The following example shows a replace:
```json
{
  "did": "<nuts did id>",
  "document": {
    "controller": [],
    "verificationMethod": [],
    "authentication": [],
    "service":[]
  }
}
```

### Delete (Revoke)

DID Documents cannot be deleted as in being "erased", only its keys can be removed as to prevent future changes (revocation).
To revoke the keys to prevent future updates;

1. Remove all keys (specified by `verificationMethod`) and references (specified by e.g. `authentication`) to
   these keys from the document. 
2. Remove all controllers from the document.

## 4. Security Considerations
