The example below describes a generic Nuts DID to be used in the Nuts network:

```json
{
  "@context": [
    "https://www.w3.org/ns/did/v1"
  ],
  // The ID of the DID. Notice the ID doesn't indicate the entity type (vendor/care organization),
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
    // First controller equals this document's DID, meaning it controls its own document
    "did:nuts:04cf1e20-378a-4e38-ab1b-401a5018c9ff",
    // This entity is allowed to make changes to this document as well
    // example usages are:
    // - a trusted third party (e.g. nuts foundation or a vendor).
    // - another document for the same legal care organization (when organizations merge).
    // - parent document of this document which holds all key material (allows nesting for larger or complex organizations).
    "did:nuts:f03a00f1-9615-4060-bd00-bd282e150c46"
  ],
  // `verificationMethod` contains all cryptographic keys associated with this DID. They are referenced by the 
  // verification relationships (e.g. `authentication` or `assertionMethod`)
  "verificationMethod": [
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
    },
    // This could be a recovery key for when the party loses its private key,
    // residing on a hardtoken (e.g. a Yubikey in a safe).
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
  // Keys specified in the section below are used to authenticate as DID controller. In other words;
  // keys specified below can alter this DID document.
  // Entries MUST NOT contain embedded key material and all MUST reference keys containing in the verificationMethod
  "authentication": [
    // Key used by the party to control this document (e.g. alter services)
    "did:nuts:04cf1e20-378a-4e38-ab1b-401a5018c9ff#key-1",
    // Recovery key (e.g. hardtoken)
    "did:nuts:04cf1e20-378a-4e38-ab1b-401a5018c9ff#key-2"
  ],
  // `assertionMethod` indicates the keys used when signing Verifiable Credentials (VC). Examples use cases:
  // - Vendor issuing a VC to care organization stating "this is my client"
  // - Care organization issuing a VC stating "this is my software vendor"
  // TODO: discuss these use cases and specify
  "assertionMethod": [
    "did:nuts:04cf1e20-378a-4e38-ab1b-401a5018c9ff#key-1"
  ],
  // `service` indicate high-level 
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
  ]
}
```