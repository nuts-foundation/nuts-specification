# RFC018 Verifiable Credential Issuance using OpenID

|                           |           |
|:--------------------------|:----------|
| Nuts foundation           | R.G. Krul |
| Request for Comments: 018 | Nedap     |

## Verifiable Credential Issuance using OpenID

### Abstract

### Status of document

This document describes a Nuts standards protocol.

### Copyright Notice

![](../.gitbook/assets/license.png)

This document is released under the [Attribution-ShareAlike 4.0 International \(CC BY-SA 4.0\) license](https://creativecommons.org/licenses/by-sa/4.0/).

## 1. Introduction

This specification describes how to apply OpenID for Verifiable Credential Issuance (OIDC4VCI) to issue Verifiable Credentials (VCs)
to resolvable DIDs in server-to-server flows, without user (human) interaction from the holder.

## 2. Terminology

This specification uses the terms "DID", "DID document" and "Service" as defined by the [DID Core Data Model](https://www.w3.org/TR/did-core/),
"Verifiable Credential", "Credential Subject" and "Issuer" as defined by the [Verifiable Credential Data Model](https://www.w3.org/TR/vc-data-model/), and
"Wallet", "Client Metadata" and "Credential Offer" as defined by [OpenID 4 Verifiable Credential Issuance](https://openid.net/specs/openid-4-verifiable-credential-issuance-1_0.html).

## 3. Overview

While OIDC4VCI is typically designed to be used to issue credentials through a browser or to a mobile wallet app,
this specification describes how to use OIDC4VCI to issue credentials server-to-server, without user interaction from the holder.

## 3.1 Supported Flows

Issuers MUST support the pre-authorized code flow to issue credentials.
While OIDC4VCI specifies a PIN to mitigate credential stealing,
implementations MUST NOT require a PIN in server-to-server flows (see Appendix A.1).

## 4. Client Metadata Discovery using DID documents

To offer a credential the issuer needs to discover the wallet's credential offer endpoint, which is registered in the client metadata.

When the wallet is represented by a resolvable DID the client should register a `oidc4vci-wallet-metadata` service on the DID document,
which contains a URL pointing to the client metadata. This indicates the wallet supports receiving credentials according to the OIDC4VCI specification.

The issuer can resolve this URL and retrieve the client metadata to find the credential offer endpoint.

## 5. Credential Issuer Trust

In server-to-server issuance of credentials, there's no user approving the credential offer before requesting it and loading it into the holder's wallet.
Therefore, the holder must make a decision whether to trust credential issuer.
Trust SHOULD be derived from the TLS trust anchors referred to by Appendix B.1.

## Appendix A: Design decisions

### A.1 No PIN requirement

OIDC4VCI specifies a PIN to mitigate credential stealing,
e.g. someone looking over the shoulder of the intended holder standing at a terminal,
quickly scanning the QR code and stealing the resulting credential.
Since this specification describes server-to-server flows, there is no user involved from who the QR code can be stolen
or coerced into giving the QR code.
To prevent eavesdropping, HTTPS MUST be used to protect communication (see Appendix B.1).

### A.2 Trust issuer based on TLS certificates

Since the TLS trust anchors are curated, trusting credential issuers can be derived from trust TLS server certificates:
Credential issuer identifiers are URLs and thus link to a trusted TLS certificate.

Future, more involved trust models could involve the issuer presenting a credential from which trust is derived.
For instance, proving that the issuer is a registered legal entity, which can be held accountable.

## Appendix B: Security Considerations

### B.1 Use HTTPS for all exchanges

To prevent against eavesdropping and man-in-the-middle attacks, all exchanges must be done over HTTPS.
Clients should refuse to connect to plain HTTP endpoints.
Refer to [RFC008 Certificate Structure](rfc008-certificate-structure.md) for requirements regarding these certificates and which Certificate Authorities should be accepted.
