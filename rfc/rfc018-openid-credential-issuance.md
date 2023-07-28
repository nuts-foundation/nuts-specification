# RFC018 Verifiable Credential Issuance using OpenID

|                           |           |
|:--------------------------|:----------|
| Nuts foundation           | R.G. Krul |
| Request for Comments: 018 | Nedap     |

## Verifiable Credential Issuance using OpenID

### Abstract

This RFC describes how to use [OpenID4VCI](https://openid.net/specs/openid-4-verifiable-credential-issuance-1_0.html)
to issue credentials from a Nuts node, to another Nuts node. It adds security requirements, wallet discovery and
server-to-server responses to make it applicable to Nuts.

### Status of document

This document describes a Nuts standards protocol.

### Copyright Notice

![](../.gitbook/assets/license.png)

This document is released under the [Attribution-ShareAlike 4.0 International \(CC BY-SA 4.0\) license](https://creativecommons.org/licenses/by-sa/4.0/).

## 1. Introduction

This specification describes how to apply OpenID Connect for Verifiable Credential Issuance (OpenID4VCI) to issue Verifiable Credentials (VCs)
to resolvable DIDs in server-to-server flows, without user (human) interaction from the holder.

## 2. Terminology

This specification uses the terms "DID", "DID document" and "Service" as defined by the [DID Core Data Model](https://www.w3.org/TR/did-core/),
"Verifiable Credential", "Credential Subject", "Holder" and "Issuer" as defined by the [Verifiable Credential Data Model](https://www.w3.org/TR/vc-data-model/), and
"Wallet", "Client Metadata" and "Credential Offer" as defined by [OpenID 4 Verifiable Credential Issuance](https://openid.net/specs/openid-4-verifiable-credential-issuance-1_0.html).

## 3. Overview

While OpenID4VCI mainly describes issuing credentials through a browser or to a mobile wallet app,
this specification describes how to use OpenID4VCI to issue credentials server-to-server, without user interaction from the holder.

Nodes MUST support both processes of issuing and receiving credentials over OpenID4VCI.

## 3.1 Supported Flows and Features

Conforming implementations MUST support the following features:

- Pre-authorized code flow
- Credential offer:
  - with `credential_offer` parameter
  - with `string` values, referring to a credential's `id` in `credentials_supported`
- Credentials in JSON-LD (`ldp_vc`) format
- Issuer- and wallet metadata

## 3.2 Credential Offer Response

The OpenID4VCI specification does not specify responses for the credential offer.
But since this specification describes server-to-server interaction
a response is useful for the issuer to know whether the credential offer was accepted.

### 3.2.1 Credential Offer Success Response

In case the wallet considers the offer valid, it MUST respond with HTTP status `200 OK`.
This is regardless of whether the wallet will actually retrieve the wallet.

If the wallet immediately retrieved the credential, it SHOULD respond with `status` set to `credential_received`.
Below is a non-normative example of such a response:

```http
HTTP/1.1 200 OK
Content-Type: application/json
Cache-Control: no-store

{
  "status": "credential_received"
}
```

### 3.2.2 Credential Offer Error Response

This specification extends the specification with the following error responses a wallet MUST support when receiving a credential offer:

- `invalid_request`: the offer is missing or invalid.
- `invalid_grant`: the offer does not contain a grant the wallet could use to retrieve the credential.

It MUST respond with HTTP status `400 Bad Request` and a response body containing the error code.

Below is a non-normative example of such a response:

```http
HTTP/1.1 400 Bad Request
Content-Type: application/json
Cache-Control: no-store

{
  "error": "invalid_grant"
}
```

## 4. Client Metadata Discovery using DID documents

To offer a credential the issuer needs to discover the wallet's credential offer endpoint, which is registered in the client metadata.

To allow a (Nuts) wallet to be discovered, the wallet MUST be represented by a resolvable DID.
The DID document MUST contain a `node-http-services-baseurl` service, which contains the base URL for node-to-node HTTP services.
The issuer appends the following path to it to resolve the client metadata: `/n2n/identity/<did>/.well-known/openid-credential-wallet`,
replacing `<did>` with the DID of the wallet holder.

## 5. Security

### 5.1 Credential Issuer Trust

In server-to-server issuance of credentials, there's no user approving the credential offer before requesting it and loading it into the holder's wallet.
Therefore, the holder must make a decision whether to trust the credential issuer.
Trust SHOULD be derived from the TLS trust anchors referred to by section 5.2 (Transport Security).

### 5.2 Transport Security

To prevent against eavesdropping and man-in-the-middle attacks, all exchanges MUST be done over HTTPS.
Clients should refuse to connect to plain HTTP endpoints.
Refer to [RFC008 Certificate Structure](rfc008-certificate-structure.md) for requirements regarding these certificates and which Certificate Authorities should be accepted.

## Appendix A: Design decisions

### A.1 No PIN requirement

OpenID4VCI specifies a PIN to mitigate credential stealing,
e.g. someone looking over the shoulder of the intended holder standing at a terminal,
quickly scanning the QR code and stealing the resulting credential.
Since this specification describes server-to-server flows, there is no user involved from who the QR code can be stolen
or coerced into giving the QR code.
To prevent eavesdropping, HTTPS MUST be used to protect communication (see Appendix B.1).

### A.2 Trust issuer based on TLS certificates

Since the TLS trust anchors are curated, trusting credential issuers can be derived from trust TLS server certificates:
Credential issuer identifiers are URLs and thus link to a trusted TLS certificate.

Future, more involved trust models could involve the issuer presenting a credential from which trust is derived.
For instance, proving that the issuer is a registered legal entity.
