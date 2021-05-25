# RFC013 Verifiable Credential IRMA Proof Type

|  |  |
| :--- | :--- |
| Nuts foundation | W.M. Slakhorst |
| Request for Comments: 013 | Nedap |
|  | May 2021 |

## RFC013 Verifiable Credential IRMA Proof Type
### Abstract

This RFC describes requirements for a [Verifiable Credential Proof](https://www.w3.org/TR/vc-data-model/#proofs-signatures) using [IRMA](https://irma.app/) signatures. It also specifies the requirements for Verifiable Credentials that support this proof type. 
An IRMA proof type acts as bridge between the IRMA ecosystem and Nuts ecosystem. Nuts uses DIDs, DID Documents and Verifiable Credentials as its base. Nuts node operators must trust specific DIDs. IRMA publishes public keys of its issuers on Github. By trusting that specific repository and by running the IRMA server, a Nuts node operator can trust all Verifiable Credentials signed by IRMA.
Verifiable Credentials with an IRMA proof are still issued to DIDs but not issued by DIDs. 
The trust in the issuer is replaced by the trust in the IRMA ecosystem.

### Status

This document is currently a draft.

### Copyright Notice
![](../.gitbook/assets/license.png)

This document is released under the [Attribution-ShareAlike 4.0 International \(CC BY-SA 4.0\) license](https://creativecommons.org/licenses/by-sa/4.0/).

## 1. Introduction

A Nuts network relies on trust. Verifiable Credentials only have value when their issuer is trusted. 
Without trusting any issuers, a Nuts node will just be a collection of identifiers and public keys. 
Not all desired issuers support DID and VC technology. Some of these trusted third parties do support the IRMA ecosystem. 
IRMA does not support DIDs and/or VCs but is capable of generating zero knowledge proof signatures.
Using IRMA could therefore support adaption of Nuts until DIDs and VCs are widely accepted as the technology of choice. The IRMA ecosystem consists of a server and a mobile app. The existence of this mobile app makes it possible for a person to be in control of setting a signature. This is also needed since VC wallets are not yet widely available.

## 2. Terminology

* **DID**: [Decentralized Identifiers](https://www.w3.org/TR/did-core/).
* **holder**: The party that receives a VC from an issuer.
* **IRMA**: [I Reveal My Attributes](https://irma.app).
* **IRMA credential holder**: The IRMA user that issues a Verifiable Credential using the app on his phone.
* **issuer**: The party that issues a VC.
* **implementing party**: Organization or person that creates software that adheres to the Nuts specs. 
* **proof**: Proof asserting that a VC is valid using cryptography.
* **VC**: Verifiable Credential according to the [Verifiable Credential Data Model](https://www.w3.org/TR/vc-data-model/).
* **VP**: Verifiable Presentation according to the [Verifiable Credential Data Model](https://www.w3.org/TR/vc-data-model/).

## 3. Verifiable Credential requirements

### 3.1 Type

This RFC introduces a new proof type. 
Any Verifiable Credential type that supports this proof type MUST state this in the corresponding RFC.
The proof type identifier is:

```text
NutsIrmaCredentialProof
```

### 3.2 CredentialSubject

The `credentialSubject` part of the VerifiableCredential MUST have an `id` and the `id` MUST be a Nuts DID. The `id` contains the holder of the Verifiable Credential.
Since the proof is unrelated to Nuts DIDs, the `issuer` MUST be the same as `credentialSubject.id`.
Trust with this type of proof is not placed in a specific DID but in an IRMA issuer from a specific IRMA scheme.
Any implementing party MUST state how different IRMA schemes are supported.

Other attributes in the `credentialSubject` MUST relate to attributes from an IRMA scheme.
For example the [irma-demo](https://github.com/privacybydesign/irma-demo-schememanager) scheme contains the credential `email` issued by `sidn-pbdf`. 
This credential contains the attributes `email` and `domain`. All entries in the `credentialSubject` MUST follow the same hierarchy. 
It MUST be a JSON object and MUST contain the scheme, the issuer, the credential and attribute name. The `email` and `domain` attributes would be included as:

```json
{
  "credentialSubject": {
    "id": "did:nuts:1",
    "irma-demo": {
      "sidn-pbdf": {
        "email": {
          "email": "alice@example.com",
          "domain": "example.com"
        }
      }
    }
  }
}
```

All attributes in the `credentialSubject` MUST also be part of the IRMA signature.

### 3.3 Proof

As stated at the beginning of this chapter, the `type` for the `proof` is `NutsIrmaCredentialProof`.
The `proof` MUST also contain a `proofValue`. The `proofValue` MUST contain a base64 encoded IRMA signature.
The proof does not contain a signature created with an `assertionMethod` key from the DID Document.

## 4. IRMA signature

### 4.1 Format

An IRMA signature is represented as:

```json
{
"@context": "https://irma.app/ld/signature/v2",
"signature": [
  {
    "c": "yIAiQyBhpME2mOEMxLzXur/Kik5+9z9w/CP8z3HWHRc=",
    "A": "RlJ2zRDAJR1GTf0Bm05sDMKw3vGON9VGvH4Y1v+gXdbI71Nh1ReUfsUd2ACurCx2TtKVc6XqylF1G70NimJdt2sA3ItfYz3R5+TxXypwTfCYbXz8AYlgG+MRowzM45xTqqfZMqS4Fs8oIrdAI9ayUeKFDpTQju8b/JpC+v8U5ww=",
    "e_response": "bTHAQd2FYOzTJL3BbWZg2DeW91o9b1WAmQPWnJTwrDYNVHkZ6NVgo1V9paGMtX7OySN5yvO3gR1u",
    "v_response": "BbFK6CKsKnBzCzjYuAAUB9mZfJZ6OA0rnd+mCviL5hlu4v7DXdNrFWyIOqBDfeUce2vESyJnGi59JAbCfLROGK9Ehp+sRTbTvBHazZk5coyalz18nlARj+yi6pnORuU+nAMBhTpQXhnewNSQATuDOTS+KB1mY02Um3KQVXT1jLRhAJlU5yM0jnt1kuuRGh/tTCWmp0TMt5pS3J3yLS1ob909nuJB8Iv6Eaco2YGCFuc6gqNXtTGEhcG4CEHVe7MqLYPh9BC9Uk4bQiTeafZ9d88FBG1Uk2PBvrZMFItFDRmzWnYWU9MiMRSOBJzVtN6t2ofET9/HJ4mP5WfIcdo/",
    "a_responses": {
      "0": "ZyTWJl0205ubywYFH2bGPWp/71sOwAjMEr0ZHW1R9D7xK1tOFO9GYVI8CQ4tD6fZuHrHmwFWBy6QQA2EP9A8kSvuNR0A28MXOiY=",
      "10": "GamYIBWLPlPJ1woU0WrCQ9SQaDW3vK0ikUQCW0097ssuWC62mXm+a9AIjdSxa/AFTr8Q5N7bMkV08N640qdA5wpYgF/uN3Dlw/c=",
      "11": "mEym6gcrFps/XQihA/9gQWYEQszD3+101Kz04cBWJqcBtGl+qjq9ybtaCZiGBqlj8EKom5u6zC8DRsusxuKYCceU4kWKZjEI0I4=",
      "12": "3VZPNJ6xkThx2VJlDzCxCGVwin+6bSY5YwhKISVgphc9oeF8AyMkmKZkvV2JiwfUvNs7KpQlNyiLXw2s8iddpV8zLDlRHcHJDkE=",
      "13": "zieEles1/UQBuP/JH7fmYAAK/7ofiL5bkcUM49dPJaQGU7FuOzVZyBH1nSgocUr/IqhQTelzh2Yy0DDlS6JUSkP4FEhTDaoV4Bc=",
      "14": "GYDhalrXE0+7Qwzk/4Lyyz3kvZceT/gSB/Nxin4f+d4F3weuUnN13FvKDlARVEcIN427ex054dJxq1FtR2mjHK6HY1mmsFzYUi8=",
      "15": "kWH1kZCoiSz/EnbZh0GK7Nohte02KDnPsjjjMPIylSOCzqTi7YlaHy04YqypOLZWKXBHYMoNC6s9lEBpa8eQGMIKgBEOmfwXgj4=",
      "16": "8kO9SIYdGFr1j2dcw81E2poBNmPVCcdeuxMRBIIxVFj6yNOG/VT61B5NNlIruK2c3iLfGB5IzkHfggQiZ/qiamSfSjYeHo3BlRU=",
      "17": "wtN7WXZO7PC2bMpP+BTBIuW3ek4zIe9CagQIv8kZRueqZ8zJjTi6gfG+CmFV7Zv0Wnz1UaR4uk1UUIZM8PNKeFlOKef8t9wbcxU=",
      "18": "ncGLDudDnGucNaPd2KgAzAycVSs8RRtQCXknl7R9LUgf9xNyRT89Y6SdHOS82t9lijF1cnoBJWnafSAAEylqXTsBVDsKkM/ebcc=",
      "19": "iccLNvHK/WcVVUtr/NU59okczziBGyBL4ZlRfQ9tTQwpxnB3Qrew74AIVd+3mnx29NUxMw89Yp0sRqvLSKeQjoFYalD3sr9QLrE=",
      "2": "1TXHps+1ipZL8sb0yk+dQUVeV5Sps0FeRoL9TksPwEQ48zaB7+d3h0ery7ORjIN9F/X5/xfDoN4pTxylSzinkd1r6NrpVkN3sAQ=",
      "7": "r3/vPSuNl5g+azUiPfXXsao4PWuX1MsJgEAR6yQUXEctsokdBNQgprBulsofKMY3+emaW1KQatmIC99AQYO8p8qf8HdlnotrpPw=",
      "8": "OB7YmQXtpbRc/PBMCKIevou8GaAzQ1yx2995hPCNdA7Bzw83XInoQlUV381Tjn3GSi2bFFSEB06EfEvvM8qwcsEWrHjNzw+f5Ks=",
      "9": "PYVsNO5RP3BYMPDHlG88WArTKScAI70Zr7OVCkzIErnAL9AiaVgxGNeDhtsgC6ikHWqlEsalUDYs2vYXy68lYHpiSKT4FrL10VM="
    },
    "a_disclosed": {
      "1": "AwAKOAAaAADzEDKAtyC9EOmkzPGKMnV8",
      "3": "rtLY2MrWyw==",
      "4": "yMs=",
      "5": "hOTq0tTd",
      "6": "rtLY2MrWykDIykCE5OrS1N0="
    }
  }
],
"indices": [
  [
    {
      "cred": 0,
      "attr": 6
    },
    {
      "cred": 0,
      "attr": 3
    },
    {
      "cred": 0,
      "attr": 4
    },
    {
      "cred": 0,
      "attr": 5
    }
  ]
],
"nonce": "VShoLPhVET+5/jq1HyG70w==",
"context": "AQ==",
"message": "did:nuts:1",
"timestamp": {
  "Time": 1582557442,
  "ServerUrl": "https://keyshare.privacybydesign.foundation/atumd/",
  "Sig": {
    "Alg": "ed25519",
    "Data": "+owJxphmD16dXlfpVfdRGfGnKp84RMU7ewk9lpCxV7DeQfvYgztSEVozqISLvnSz5ABNiQr1zyYOb3bths5TBw==",
    "PublicKey": "MKdXxJxEWPRIwNP7SuvP0J/M/NV51VZvqCyO+7eDwJ8="
  }
}
}
```

[This paper](https://dominoweb.draco.res.ibm.com/reports/rz3730_revised.pdf) contains an explanation on how the cryptographic scheme works. 
The interesting bits for getting the data from a signature are the `message`, `A`, and `a_disclosed`fields. 
The `message` is used to relate the signatures and its attributes to a DID. This is the actual text the user has signed using the IRMA app.
The `A` fields identifies which credential is used from the IRMA scheme.
The `a_disclosed` field contains the actual attribute values. These values are base64 representations of big integers. Big integers are required for the cryptography to work.
Underneath they contain strings, booleans, integers or enums.
All the other fields are required to validate the authenticity of the signature. 

The example signature from above will be base64 encoded when used in a Verifiable Credential proof:

Below is a complete example of how an Irma proof can be embedded in a Verifiable Credential:

```json
{
  "@context": [
    "https://www.w3.org/2018/credentials/v1",
    "https://nuts.nl/credentials/v1"
  ],
  "id": "did:nuts:t1DVVAs5fmNba8fdKoTSQNtiGcH49vicrkjZW2KRqpv#6f91673b-afa9-4d26-9e0f-00d989943275",
  "issuanceDate": "2021-03-15T16:34:17.687862+01:00",
  "issuer": "did:nuts:t1DVVAs5fmNba8fdKoTSQNtiGcH49vicrkjZW2KRqpv",
  "type": ["VerifiableCredential", "NutsKVKCredential"],
  "credentialSubject": {
    "id": "did:nuts:t1DVVAs5fmNba8fdKoTSQNtiGcH49vicrkjZW2KRqpv",
    "pbdf": {
      "kvk": {
        "official": {
          "legalEntity": "CareBears",
          "officeAddress": "CarePlace",
          "kvkNumber": "12345678"
        }
      }
    }
  },
  "proof": {
    "type": "NutsIrmaCredentialProof",
    "proofValue": "DgYdYMUYHURJLD7xdnWRinqWCEY5u5fK...j915Lt3hMzLHoPiPQ9sSVfRrs1D"
  }
}
```

The `proofValue` has been abbreviated for readability.

### 4.2 Verification

Verification of an IRMA signature MUST be done by an [IRMA server](https://irma.app/docs/irma-server/) instance. 
It's also possible for an implementing party to embed the [IRMA go libraries](https://github.com/privacybydesign/irmago).
Verification is done as follows:

- Verify IRMA signature using IRMA server (or embedded library), which returns the attributes contained in the IRMA signature.
- Verify each attribute in credentialSubject against its counterpart in the IRMA signature:
  - Verify it's present in the IRMA signature
  - Verify the attribute's value exactly matches its value in the IRMA signature
- Verify that `message` from the IRMA signature exactly matches the value of `credentialSubject.id`

The dot-delimited format of the attributes in the verification result can be extended as follows:

```text
A.B.C.D=X
```

becomes

```json
{
  "A": {
    "B": {
      "C": {
        "D": "X"
      }
    }
  }
}
```

The last validation is to check if the `message` from the IRMA signature matches the value of `credentialSubject.id`.

### 4.3 IRMA scheme

The IRMA ecosystem has two different schemes: `pbdf` and `irma-demo`. The latter one is not intended for production since all private keys are published with the scheme. Any implementing party MUST ignore all signatures made with that scheme in production.

The contents of the schemes can be viewed on Github: [pbdf](https://github.com/privacybydesign/pbdf-schememanager) and [irma-demo](https://github.com/privacybydesign/irma-demo-schememanager)

### 4.4 Revocation

Although IRMA supports revocation for credentials, the revocation check is only applied during issuance.
An IRMA credential revoked today will not invalidate any signatures done in the past.
This means that a Verifiable Credential with an IRMA proof MUST adhere to the generic revocation definition stated in §4.5 of [RFC011 - Verifiable Credentials](rfc011-verifiable-credentials.md#4-5-revocation).
It's the responsibility of the IRMA credential holder to revoke verifiable credentials that could have been compromised. 
The IRMA credential holder can do this with a renewed, not revoked IRMA credential.
The next paragraph contains a default revocation for a Verifiable Credential with an IRMA proof

### 4.4.1 Default revocation

VCs that are issued with an IRMA proof can be revoked by publishing the following transaction on the network:

```json
{
  "issuer": "did:nuts:t1DVVAs5fmNba8fdKoTSQNtiGcH49vicrkjZW2KRqpv",
  "subject": "did:nuts:t1DVVAs5fmNba8fdKoTSQNtiGcH49vicrkjZW2KRqpv#6f91673b-afa9-4d26-9e0f-00d989943275",
  "date": "2021-03-15T16:34:47.422436+01:00",
  "proof": {
    "type": "NutsIrmaCredentialProof",
    "proofValue": "DgYdYMUYHURJLD7xdnWRinqWCEY5u5fK...j915Lt3hMzLHoPiPQ9sSVfRrs1D"
  }
}
```

Such a revocation transaction has the following requirements:

* the **issuer** MUST match the DID of the `issuer` field of the VC.
* the **subject** MUST match the `id` field of the VC.
* a **reason** MAY be filled with a revocation reason.
* the **date** MUST provide the date in RFC3339 format. From this moment in time the VC is revoked.
* the **proof** MUST be a `NutsIrmaCredentialProof` proof.
* the IRMA signature in the `proofValue` MUST use the same attributes as the IRMA signature in the original VC. 

The transaction MUST be published on the Nuts network.
The content-type is `application/vc+json;type=revocation`

