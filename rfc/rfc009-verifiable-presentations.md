# RFC009 Verifiable presentations

| :memo: NOTE                                                                              |
|:-----------------------------------------------------------------------------------------|
| This RFC is currently under review to determine its place within the Nuts specification. | 
| The content is not be implemented yet!                                                   |

## Abstract

We use credentials every day. Your driving license, certificates of courses, university degrees. All are examples of credentials. There are also many examples in healthcare such as a prescription, diagnosis, or allergies. Credentials can be issued to a person or an organization. Verifiable credentials are a generic mechanism to issue, hold, and verify claims on a subject. This use case describes how a holder presents verifiable credentials to a verifier. The system in its role of verifier receives and verifies the origin, integrity, and validity of the credentials. 

Verifiable credentials are an implementation of the [W3C Verifiable Credentials Data Model](https://www.w3.org/TR/vc-data-model/). Other roles are issuer and holder. These roles are described in other use cases (RFCs).

## Status of document

This document is currently a draft. 

## Copyright Notice

<img style="float: left;" src="https://mirrors.creativecommons.org/presskit/buttons/88x31/png/by-sa.png">

This document is released under the [Attribution-ShareAlike 4.0 International (CC BY-SA 4.0) license](https://creativecommons.org/licenses/by-sa/4.0/).



## 1. Introduction

The use cases for verifying credentials data model holds the four roles as described in [Verifiable Credentials Use Cases](https://www.w3.org/TR/vc-use-cases/). The data model (https://www.w3.org/TR/vc-data-model/) adds an extra role, the verifiable data registry. Verifiable credentials are containers for different types of claims.

This RFC describes the implementation of decentralized identifiers (https://www.w3.org/TR/did-core/) and the verifiable credentials data model in the Nuts network. The goal is to enable the issuing, keeping, and verification of credentials. 



## 2. Terminology

The terminology used is defined in https://www.w3.org/TR/did-core/#terminology and https://www.w3.org/TR/vc-data-model/#basic-concepts.



## 3. Requirements for verifiable credentials

The implementation MUST support the use of verifiable credentials in the roles of Issuer, Holder, and Verifier as described in [Verifiable Credentials Use Cases](https://www.w3.org/TR/vc-use-cases/) and [W3C Verifiable Credentials Data Model](https://www.w3.org/TR/vc-data-model/). In the terms of decentralized Identifiers, the implementation acts as a software agent of the person or organization accountable for the role of Issuer, Holder, and Verifier.

The implementation MUST support the DID web method. See https://w3c-ccg.github.io/did-method-web/. To be able to support the web method, the implementation MUST support the use core specification of decentralized identifiers as described in https://www.w3.org/TR/did-core/.

The implementation MAY have limited support on the following issues:

- The implementation supports the JSON Web Keys (JWK) as key format (verification method type equals `JsonWebKey2020`). See also https://www.w3.org/TR/did-spec-registries/.
- The implementation has limited support for key types (minimal OKP) and algorithms (minimal EdDSA).

The implementation MUST implement a secure Wallet to store decentralized Identifiers (DID) and their private key, and to store verifiable credentials that the agent holds.



