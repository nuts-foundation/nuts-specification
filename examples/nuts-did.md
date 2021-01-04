The example below describes a generic DID as can be used in the Nuts network:

```json
{
  "@context": [
    "https://www.w3.org/ns/did/v1"
  ],
  // The ID of this care organization DID document. Notice the ID doesn't indicate the entity type (vendor/care organization),
  // that should be derived from the entity's credentials.
  // MUST start with `did`
  // MUST be a `nuts` method
  // Should be a UUID (type 4) without meaning (in contrary to e.g. a public key like in DIDKey) to decouple resolving from presentation.
  // How to make sure someone does not try to recreate a DID with same id? -> TODO: fix in network layer?
  "id": "did:nuts:04cf1e20-378a-4e38-ab1b-401a5018c9ff",
  // Who may create/update/delete this document?
  // OPTIONAL: 
  //   If not present DID subject is the only controller of the DID. 
  //   If present and lists another DID, that subject can also control the DID
  //   If present but the DID subject itself is not listed, the DID subject DOES NOT have control over this document.
  "controller": [
    // This organisation controls its own document
    "did:nuts:04cf1e20-378a-4e38-ab1b-401a5018c9ff",
    // This entity is allowed to make changes to this document as well
    // example usages are:
    // - a trusted third party (e.g. nuts foundation or a vendor).
    // - another document for the same legal organisation (when care organizations merge).
    // - parent document of this organisation document which holds all key material (allows nesting for larger or complex organizations).
    "did:nuts:f03a00f1-9615-4060-bd00-bd282e150c46"
  ],
  // All cryptographic methods associated with this DID
  // SHOULD these keys always be referenced in one of the verification relationships?
  "verificationMethod": [
    // Key used by the care organization to control this document (e.g. alter services)
    {
      "id": "did:nuts:04cf1e20-378a-4e38-ab1b-401a5018c9ff#key-1",
      "type": "RsaVerificationKey2018",
      "controller": "did:nuts:04cf1e20-378a-4e38-ab1b-401a5018c9ff",
      "publicKeyJwk": {
        "crv": "Ed25519",
        "x": "VCpo2LMLhn6iWku8MKvSLg2ZAoC-nlOyPVQaO3FxVeQ",
        "kty": "OKP",
        "kid": "_Qq0UL2Fq651Q0Fjd6TvnYE-faHiOpRlPVQcY_-tA4A"
      }
    }
  ],
  // Keys specified in the section below are used to authenticate as DID controller. In other words;
  // keys specified below can alter this DID document.
  "authentication": [
    // Refers to a key specified in `verificationMethod`
    "did:nuts:04cf1e20-378a-4e38-ab1b-401a5018c9ff#key-1",
    // This could be a recovery key for when the organization loses its private key,
    // residing on a hardtoken (e.g. a Yubikey in a safe).
    // This key COULD also reside in `verificationMethod` and be referenced by this section.
    {
      "id": "did:nuts:04cf1e20-378a-4e38-ab1b-401a5018c9ff#key-2",
      "type": "JsonWebKey2020",
      "controller": "did:nuts:04cf1e20-378a-4e38-ab1b-401a5018c9ff",
      "publicKeyJwk": {
        "kty": "OKP",
        "crv": "Ed25519",
        "x": "VCpo2LMLhn6iWku8MKvSLg2ZAoC-nlOyPVQaO3FxVeQ"
      }
    }
  ],
  "assertionMethod": [
    "did:nuts:vendor:123#key-2"
    // key used to sign claims
  ],
  "service": [
    {
      "id": "did:nuts:04cf1e20-378a-4e38-ab1b-401a5018c9ff#service-1",
      // must be the same type as the type in the vendor service
      "type": "nuts:bolt:eoverdracht",
      "serviceEndpoint": "did:nuts:<vendor>#service-76"
    },
    {
      "id": "did:nuts:04cf1e20-378a-4e38-ab1b-401a5018c9ff#service-2",
      // must be the same type as the type in the vendor service
      "type": "nuts:core:consent",
      "serviceEndpoint": "did:nuts:<vendor>#service-2"
    }
  ],
  "created": "2019-03-23T06:35:22Z",
  "updated": "2020-08-10T13:40:06Z"
}
```