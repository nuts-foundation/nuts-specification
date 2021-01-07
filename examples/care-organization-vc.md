```json
{
  "@context": ["https://www.w3.org/2018/credentials/v1"],
  // specify the identifier for the credential
  "id": "did:nuts:123/credentials/1",
  "type": ["VerifiableCredential", "NutsNameCredential"],
  "issuer": "did:nuts:123",
  "issuanceDate": "2010-01-01T19:73:24Z",
  "credentialSubject": {
    // identifier for the only subject of the credential
    "id": "did:nuts:123",
    "name": "Verpleegtehuis de nootjes",
    "locality": "Groenlo"
  },
  "proof": {
    "type": "RsaSignature2018",
    "created": "2017-06-18T21:19:10Z",
    "proofPurpose": "assertionMethod",
    "verificationMethod": "did:nuts:1#key-1",
    // the digital signature value containing all of the above
    "jws": "eyJhbGciOiJSUzI1NiIsImI2NCI6ZmFsc2UsImNyaXQiOlsiYjY0Il19..TCYt5X
      sITJX1CxPCT8yAV-TVkIEq_PbChOMqsLfRoPsnsgw5WEuts01mq-pQy7UJiN5mgRxD-WUc
      X16dUEMGlv50aqzpqh4Qktb3rk-BuQy72IFLOqV0G_zS245-kronKb78cPN25DGlcTwLtj
      PAYuNzVBAh4vGHSrQyHUdBBPM"
  }
}
```