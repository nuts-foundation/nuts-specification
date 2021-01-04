The example below describes a DID Document for a care organization:

```json
{
  "@context": [
    "https://www.w3.org/ns/did/v1"
  ],
  "id": "did:nuts:04cf1e20-378a-4e38-ab1b-401a5018c9ff",
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