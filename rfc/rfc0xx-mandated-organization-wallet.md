# RFC0XX Personal Wallet Mandating Organization Wallet

|                          |           |
|:-------------------------|:----------|
| Nuts foundation          | R.G. Krul |
| Request for Comments: 0  | Nedap     |
| Amends:                  |           |

## Personal Wallet Mandating Organization Wallet

### Abstract

### Status of document

This document is currently in draft.

### Copyright Notice

![](../.gitbook/assets/license.png)

This document is released under the [Attribution-ShareAlike 4.0 International \(CC BY-SA 4.0\) license](https://creativecommons.org/licenses/by-sa/4.0/).

## 1.  Introduction

Nuts data exchanges (a care organization accessing data another care organization) use Verifiable Presentations
to authorize the request. For instance, authorization could require the requester to present a care organization credential
and the identity of the end-user (e.g. name, role) for logging purposes (as specified by NEN7513).
Given the user identity is a verifiable credential in the end-user's wallet,
and the care organization registration is a verifiable credential in the organization's server wallet,
it's impossible to create a verifiable presentation containing both credentials, since they're held by 2 different subjects:
the end-user and the care organization (and a verifiable presentation is signed by a single holder).
Existing OpenID protocols allow up to 2 presentations to be exchanged (SIOPv2's `id_token` and OpenID4VP's `vp_token`),
but how to relate those when they're not presented by the same subject is unspecified (source?).

Another constraint of SIOPv2 and OpenID4VP is that it requires the end-user to create a new presentation for each new interaction,
which is unpractical if the end-user (e.g. care professional) initiates data exchanges many times each day,
or when a data exchange involves multiple resource owners (e.g. aggregating data from multiple sources).

This RFC proposes a Verifiable Credential type that allows someone (typically an employee) holding a personal wallet,
to mandate an organizational wallet to exchange data using their identity.
The verifiable credential can then be used in SIOPv2 and/or OpenID4VP exchanges. 

## 2. Terminology

TODO

## X. Issuing Mandate Credential

(or: provisioning?)

The end-user mandates the organization by issuing a `UserIdentityMandateCredential` which contains the identity of the end-user,
which can be used (presented) by the organization in Presentation Exchanges.
The credential issuer is the personal wallet of the end-user. E.g., a `did:jwk` DID held by the end-user's wallet.
The credential subject is the mandated care organization, and contains an `identity` property that specifies the Verifiable Presentation containing the user identity.

JSON example (fields omitted for brevity):
```json
{
  "type": [
    "UserIdentityMandateCredential",
    "VerifiableCredential"
  ],
  "issuer": "did:jwk:ebfeb1f712ebc6f1c276e12ec21",
  /* expirationDate should be limited by the assurance level of the organization wallet */
  "expirationDate": "...",
  "credentialSubject": {
    "id": "did:example:organization-wallet",
    /* the identity that can be used with the mandate */
    "identity": {
      "type": [
        "VerifiablePresentation"
      ],
      "verifiableCredential": [
        {
          "type": [
            "CitizenCredential",
            "VerifiableCredential"
          ],
          "issuer": "did:example:country",
          "credentialSubject": {
            "id": "did:example:citizen-pseudo-id",
            "firstName": "Jan",
            "lastName": "Visser"
          }
          /* proof, and other fields omitted for brevity */
        }
      ],
      /* domain binds the identity VP to the end-user's authentication device */
      "domain": "did:jwk:ebfeb1f712ebc6f1c276e12ec21",
    }
  },
}
```

## X. Provisioning a User Profile for Mandating

(a.k.a. using WebAuthn)

The UserIdentityMandateCredential MUST have a restricted `expirationDate` to reflect user session expiration requirements.
E.g., NIST SP800-63B states (https://pages.nist.gov/800-63-3/sp800-63b.html#63bSec4-Table1) reauthentication should happen after 30 or 15 minutes of inactivity.
End-users interacting through a mandated organization many times per day might find it cumbersome to re-issue the UserIdentityMandateCredential,
since using a personal wallet typically involves unlocking their mobile phone, scanning a QR-code, assessing and accepting the request.
Some environments might strictly personal (mobile) devices (e.g. mental health care organizations), making re-issuance of the credential multiple times per day practically impossible.

To accommodate those use cases the end-user could use a readily accessible hardware cryptographic authentication device.
For instance, a FIDO2 USB-dongle or workstation fingerprint reader accessed through WebAuthn.
The Verifiable Presentation containing the user identity can then be bound (using the `domain` property) to a DID (e.g. `did:jwk`) on the authentication device.
To issue the UserIdentityMandateCredential the user can then simply use the device (e.g. scan their fingerprint) to authenticate request.

This way, the instances of the end-user having to use its personal wallet are limited (e.g. every 30 days),
as long as authentication device used throughout the day is sufficiently secure.