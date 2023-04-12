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

## Appendix A: Design decisions

### A.1 No PIN requirement

OIDC4VCI specifies a PIN to mitigate credential stealing,
e.g. someone looking over the shoulder of the intended holder standing at a terminal,
quickly scanning the QR code and stealing the resulting credential.
Since this specification describes server-to-server flows, there is no user involved from who the QR code can be stolen
or coerced into giving the QR code.
To prevent eavesdropping, HTTPS MUST be used to protect communication (see Appendix B.1).

## Appendix B: Security Considerations

### B.1 Use HTTPS for all exchanges

To prevent against eavesdropping and man-in-the-middle attacks, all exchanges must be done over HTTPS.
Clients should refuse to connect to plain HTTP endpoints.
Refer to [RFC008 Certificate Structure](rfc008-certificate-structure.md) for requirements regarding these certificates and which Certificate Authorities should be accepted.
