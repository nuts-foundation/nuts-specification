# RFC009 Verifiable presentations



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

The Nuts framework MUST support the use of verifiable credentials in the roles of Issuer, Holder, and Verifier as described in [Verifiable Credentials Use Cases](https://www.w3.org/TR/vc-use-cases/) and [W3C Verifiable Credentials Data Model](https://www.w3.org/TR/vc-data-model/). In the terms of decentralized Identifiers, the Nuts framework acts as a software agent of the person or organisation accountable for the role of Issuer, Holder, and Verifier.

The Nuts framework MUST support the use decentralized identifiers as described in https://www.w3.org/TR/did-core/.

The Nuts framework MAY have limited support on the following issues:

- The framework supports the JSON Web Keys (JWK) as key format (verification method type equals `JsonWebKey2020`). See also https://www.w3.org/TR/did-spec-registries/.
- The framework has limited support for key types (minimal OKP) and algorithms (minimal EdDSA).
- The framework has limited support for DID methods (only web method), which resolve to a DID document in a verifiable data registry. See https://w3c-ccg.github.io/did-method-web/.

The Nuts framework MUST implement a secure Wallet to store decentralized Identifiers (DID) and their private key, and to store verifiable credentials that the agent holds.

The Nuts framework MAY introduce 'irma' as did method to support the IRMA signing and verification.



