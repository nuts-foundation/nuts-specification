## X. Example

But does this require to send the specific VC needed for a request? Or just send all VCs regarding actor, custodian and subject

Zorginzage
```json
{
  "@context": [
    "https://www.w3.org/2018/credentials/v1",
    "https://nuts.nl/credentials/v1"
  ],
  "id": "did:nuts:custodian#90382475609238467",
  "type": ["VerifiableCredential", "NutsLegalBaseCredential"],
  "issuer": "did:nuts:custodian",
  "issuanceDate": "2010-01-01T19:73:24Z",
  "expirationDate": "2010-02-01T19:73:24Z",
  "credentialSubject": {
    "id": "did:nuts:actor",
    "subject": "bsn or DID",
    "scope": ["zorginzage"]
  },
  "evidence": {
    "id": "pdf/f2aeec97-fc0d-42bf-8ca7-0548192d4231",
    "type": ["Document"],
    "endpoint": "did:nuts:custodian?type=fhir-document"
  },
  "proof": {...}
}
```
eOverdracht
```json
{
  "@context": [
    "https://www.w3.org/2018/credentials/v1",
    "https://nuts.nl/credentials/v1"
  ],
  "id": "did:nuts:custodian#90382475609238467",
  "type": ["VerifiableCredential", "NutsLegalBaseCredential"],
  "issuer": "did:nuts:custodian",
  "issuanceDate": "2010-01-01T19:73:24Z",
  "expirationDate": "2010-02-01T19:73:24Z",
  "credentialSubject": {
    "id": "did:nuts:actor",
    "subject": "bsn or DID",
    "scope": ["eOverdracht"]
  },
  "evidence": {
    "id": "consent/f2aeec97-fc0d-42bf-8ca7-0548192d4231",
    "type": ["FHIR-consent"],
    "endpoint": "did:nuts:custodian?type=fhir"
  },
  "proof": {...}
}
```
