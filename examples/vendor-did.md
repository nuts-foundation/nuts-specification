The example below describes a DID Document for a vendor:

```json
{
  "@context": [
    "https://www.w3.org/ns/did/v1"
  ],
  "id": "did:nuts:6181aab9-549b-4143-b215-5d77bf9eb266",
  "service": [
    {
      "id": "did:nuts:6181aab9-549b-4143-b215-5d77bf9eb266#service-1",
      "type": "nuts:core:oauth",
      "version": "1.0",
      "serviceEndpoint": "https://api.example.org/oauth"
    },
    {
      "id": "did:nuts:6181aab9-549b-4143-b215-5d77bf9eb266#service-2",
      "type": "nuts:core:consent",
      "version": "1.0",
      "serviceEndpoint": "https://corda.example.org/"
    },
    {
      "id": "did:nuts:6181aab9-549b-4143-b215-5d77bf9eb266#service-76",
      // Specifies the type of the service, generally a Nuts Bolt. If a care organization refers to this service,
      // it must specifiy the same type (`nuts:bolt:eoverdracht` in this case).
      "type": "nuts:bolt:eoverdracht",
      "version": "1.0",
      "serviceEndpoint": {
        "data": "https://api.example.org/fhir/",
        "auth": "did:nuts:6181aab9-549b-4143-b215-5d77bf9eb266#service-1"
      }
    }
  ],
  "created": "2019-03-23T06:35:22Z",
  "updated": "2020-08-10T13:40:06Z"
}
```