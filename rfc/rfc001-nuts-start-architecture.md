# RFC001 Nuts Start Architecture

|  |  |
| :--- | :--- |
| Nuts foundation | W.M. Slakhorst |
| Request for Comments: 001 | Nedap |
|  | September 2020 |

## Nuts Start Architecture

### Abstract

RFC's within the Nuts architecture are very explicit and do not always state why certain decisions have been made. This RFC describes basic requirements on which all architectural decisions have been made. Most of those decisions find their roots in the Nuts manifest.

### Status of document

This document describes a Nuts standards protocol.

### Copyright Notice

![](../.gitbook/assets/license.png)

This document is released under the [Attribution-ShareAlike 4.0 International \(CC BY-SA 4.0\) license](https://creativecommons.org/licenses/by-sa/4.0/).

## 1.  Introduction

This document describes the general principles of all related Nuts architecture. It translates the Nuts manifest into architectural principles. More detailed architectural and design documents are available addressing different aspects of the network. Those documents will refer back to this document regarding high level choices. The idea is that this document will remain relatively the same whereas the individual documents change over time.

## 2. Terminology

* **Actor / Requester**: The care organization that, in the context of a specific subject, wants to access medical data of a subject available at a custodian.
* **Custodian / Authorizer**: The care organization that, in the context of a specific subject, is responsible for storing the medical data.
* **Digital signature**: A cryptographic signature e.g. specified in [RFC5280](https://tools.ietf.org/html/rfc5280) or [RFC7515](https://tools.ietf.org/html/rfc7515). Specifically not the digital version of the wet signature.
* **Legal Basis**: Grounds for medical data exchange conforming to national and international legislation.
* **Node**: A piece of software implementing the Nuts specification.
* **Service provider**: An organization providing software services which support care organizations in their daily work.
* **Subject**: The subject of the medical data, usually the patient, client or civilian.

## 3. Manifest

The start architecture is a translation of the Nuts manifest to a high level architecture. Each paragraph will refer to the different manifest entries when applicable. References will be denoted between square brackets, they’ll start with a capital M followed by the number corresponding to the numbers below. The text is translated from Dutch, check [https://nuts.nl/manifest](https://nuts.nl/manifest) for the original.

#### 1 - Base

We believe that the starting point of service providers is to support care professionals and patients in digital form so they can perform their work optimally. Software influences the quality of care, maintainability, client/employee happiness and sustainability. We think it’s the responsibility of the service providers to acknowledge this and handle it diligently.

#### 2 - Patient first

We believe the patient deserves an active role in the network, for example through their Personal Health Environment \(PHE\). The patient, mandated caregiver or financial representative must be given the choice to communicate about the patients health, have control over who is allowed to view sensitive data and must be able to see who has access to their data.

#### 3 - Ownership

We believe medical data belongs to the patient and care professional, not the service provider. We believe service providers should charge for services that provide added value to their users and that data should be freely available to whomever is involved.

#### 4 - Distributed network

We believe in a distributed model for communication and data exchange, similar to the internet. We do not want to invest in network technology with a specific supplier or party that directly or indirectly controls all care data for a set of patients. We want a robust network for everyone from everyone without any central points of failure or monopolies.

#### 5 - Open standards

We believe that joining the network must be without friction. The common communication infrastructure must be open and standardised. No \(license\) costs should be charged for using it. An open-source reference implementation must be available.

#### 6 - Privacy by design

We believe privacy by design within the network is of the utmost importance. Therefore we want a system where information is consciously shared with each other. We think that it’s important that a human makes the choice for sharing data, for example by recording consent or accepting a request for data sharing.

#### 7 - Security by design

We believe in a system that is designed with the belief that parties in the network can’t be trusted and that malicious parties get smarter every day. We want to prevent that the infrastructure allows any unauthorized person to access, change or remove medical data.

#### 8 - Cryptographic base

We believe that the technology is ready and the time is now for a system based upon cryptographic principles. We want to give guarantees on identities and go beyond “access by configuration”. We believe that we can win the trust of the entire healthcare sector and eventually that of the common citizen.

## 4. Requirements

### 4.1 Identity

A strong identity is the base for authentication, authorization and auditing. An identity in itself is not that important, it's one of the three use-cases that puts requirements on the identity. An identity can be seen as the link between the user and the virtual domain \[M6\], an identity consists of one or more attributes which are relevant for the different use-cases. The following sections will describe those requirements in detail.

#### 4.1.1 Authentication

Medical data is regarded as one of the most sensitive types of data. It therefore requires the highest possible level of security. In the case of an identity this is translated as a _strong identity_. This means that any attributes of this identity will have to be verified by a trusted 3rd party \[M7\]. If the identity is to be exchanged, this also means that a digital signature has to be present alongside the attributes \[M8\]. When an identity has a digital signature from a trusted party, it can be used for authentication in the context of medical information exchange. For authentication purposes it’s not required that the identity has any specific attributes. A curated list of trusted 3rd parties should be published for each participant to trust.

#### 4.1.2. Authorization

When accessing sensitive resources, an authenticated identity also must be authorized for the requested resources \[M6, M7\]. It depends on the particular use-case which attribute is required to link the identity to an authorized resource. The authorization might be given to the identity directly but in most cases the authorization lies with the organization the user \(holding the identity\) is representing. In the case of an individual authorization, the identity must have an attribute that uniquely identifies the user. In the case where the authorization lies with the organization, a link between the identity and the organization must be given. This link must be represented by a chain of digital signatures where each certificate \(or public key\) used in the chain can be trusted \[M8\]. See also the chapter on Trust. When the authorization lies with the organization, the organization requires an attribute that identifies that organization.

#### 4.1.3. Auditing

Auditing is required to give the subject insight in who has seen their information \[M2\]. To assist the subject this must consist of human readable information about the user and the organization that user is working for. Therefore it’s required for an identity to at least have human readable attributes, eg: name and initials. For the organization identity this makes the organization name required.

### 4.2. Privacy & Legislation

#### 4.2.1. GDPR and local legislation

We all know what this means: right to be forgotten but 15 years of medical data storage. Breaking doctor patient confidentiality is only allowed with specific authorization from the patient.

Therefore: keep data at the source and retrieve on demand. Do not copy without reviewing it. A proof for data exchange must be known before others can be authorized.

#### 4.4.2. Data exchange parties

The GDPR speaks of data processors which play a role in digital information exchange. In the case of Nuts these are the service providers. Service providers have a direct relation with the care organization it’s providing services for. Medical \(and other personal\) data must be exchanged directly between the service providers of care organizations. No third party may play a role in this data exchange since it would be able to deduce information even when all data is encrypted. This also means that there’s no room for a central party that would be involved in data exchange or the exchange/management of any document establishing the legal basis for a data exchange \[M6\].

The result of reducing the involved parties is that the network must be a distributed network \[M4\]. Required information for care organizations, their service providers and available service descriptions must be distributed amongst all network participants. Since this registry data is security related, it must be signed with a digital signature.

#### 4.2.3. Legal basis for data exchange

When an actor wants to retrieve data from a custodian regarding a specific subject, it must act on a known legal basis \[M2\]. This legal basis must be stored at the custodian side since that is also the responsible party for the subject’s data \[M3\]. This legal basis will be the base for most of the authorizations of an identity for a particular resource.

The actor must be informed about the existence of the legal basis. This notification is a confidential action between the custodian and the actor. An actor must not query custodians for data without having been notified about the legal basis. Otherwise the actor might reveal it has a relationship with a certain subject to other parties. That would be a breach of privacy \[M4, M6\].

In a perfect world, the legal basis should also have a digital signature that can be traced back to the subject. This, with the current available technology, digital skills and level of adoption, might prove a bridge too far. Therefore any such signature will probably be labeled as optional in any architectural sub-document.

### 4.3. Trust & Security

Trust is, in the case of Nuts, the collective of authorizations, digital signatures and curated list of trusted 3rd parties. This relates closely to the general security of the network as a whole. An exchange of data between a custodian and actor is only allowed if all involved parties can be trusted \[M7\]. An action in the virtual domain must be linkable to real world parties \[M6\].

The Nuts trust model is multi-layered. All layers combined provide the level of trust that is required for exchanging medical data. Each layer has its own set of allowed digital signatures. The separation makes sure an organization can't act as a care professional and that a service provider can't act as a care organization. The only exceptions are a one-man practise where the care professional is also the owner of the organization or when a care organization is hosting its own services \(on-premise\).

| layer | responsible party | establishes trust in |
| :--- | :--- | :--- |
| user | Care professional | a user session exists. This session is linked to the care organization. [\[RFC002\]](rfc002-authentication-token.md). |
| data | Care organization | registry data. The connection between care organizations, services and endpoints. \[RFC006\]. |
| network | Service provider | members of the private network. \[RFC005\]. |

A fundamental choice has been made to place trust in a curated list of 3rd parties \[M4\]. Trust between individual service providers or care organizations does not play a role \[M7\]. Technically this is enforced with the use of digital signatures \[M8\]. Attributes that are exchanged between parties do not hold any value unless it can be verified by a digital signature. Besides the improved security, it also levels the playing field for new parties and eliminates the value of inter-party political relationships \[M1, M3\]. Placing trust in certain 3rd parties does create a dependency on those parties and their security and trust model.

The following sections describe the different relations and how this eventually relates to a root-of-trust \(or actually multiple roots\).

#### 4.3.1. Request & Service Provider

Service providers are the gatekeepers for care organizations for the virtual domain. They provide access to the virtual domain by means of software. So in order to authorize a specific technical request, the requesting service provider must be identified and authenticated first. This must be done via digital signature where the secure key must be verifiable by a trusted 3rd party.

#### 4.3.2. Request & Care organization

In order to do authorization, a request for data must contain information about the custodian, actor and subject. The identity of the requesting party must be verifiable by means of a digital signature. The requesting party usually is the actor and may consist of a combination of organization and/or user identity. The relationship between the requesting service provider and the requesting party must also be verifiable.

#### 4.3.3. Service provider & Care Organization

When a custodian and actor interact, they do this through their service providers, therefore the relationship between care organization and service provider must be verifiable by others \[M4, M7\]. A relationship can only be verified if both sides provide proof of the relationship. As with other proofs in Nuts, this requires digital signatures of specific attributes representing the relationship \[M8\]. The secure key used for the signatures must be verified by a trusted 3rd party. The proof of the relationship must be distributed along the network since it is required before any interaction can take place \[M4\]. The most important identifiable part of a care organization is it’s commonly known name. A proof for this name must be available from a trusted 3rd party if it’s to be distributed amongst all network participants.

#### 4.3.4. Care organization & User identity

In the case a user acts as the actor and the authorization is given to the care organization which the actor represents, a digital signature proving the relation between user and organization must be present in the request \[M7\]. This proof can’t be distributed beforehand since it’ll contain information about the user identity which also has its own privacy rights. This digital signature also prevents service providers from acting as a representative of the care organization.

#### 4.3.5. User identity & User

In the end it’s the user that is viewing sensitive data. So the final proof of relationship is that the user, currently behind the controls, proves it has the user identity \[M6\]. This must be done by signing a contract stating the extent of the required privileges for the current session. This is also known as the authentication process. The signature must validate the required user attributes and the secure key used for the signature must be verified by a trusted 3rd party. This signature must also validate that the user is working in the context of a care organization.

### 4.4. Configuration & automation

One of the points of the manifests speaks of a distributed network \[M4\], this means that there’s no owner of the network and thus no central body that makes decisions. Every aspect of a fully functioning node thus requires configuration that determines what can be trusted. A node operator should be in full control of what a node is doing. Doing everything by hand can become very costly when the network grows. Therefore to be successful, some layer of automation must be available for most of the functionality.

Every Nuts RFC/specification must explicitly mention what part is based on mutual trust. This could be data or some sort of configuration. If a RFC describes a mechanism where multiple nodes must trust each other based on some sort of configuration or shared state it must also specify \(or delegate it to a different RFC\) how this can be exchanged in an automated manner. The described automated process does not replace the manual version but rather builds upon it.

An important part of the automated process is how nodes can make sure they can trust the offered data. Non-repudiation is the magic keyword. This means that every piece of data must either be obtained from the source or it must have been signed with a digital signature from the authoring party.

### 4.5. Data Consistency

#### 4.5.1. Eventual Consistency for Distributed Data

Since the Nuts network is distributed \[M4\] there's a trade-off to be made between consistency, availability and network partition tolerance \(CAP-theorem, [https://en.wikipedia.org/wiki/CAP\_theorem](https://en.wikipedia.org/wiki/CAP_theorem)\). Since strong consistency is very expensive for distributed networks the Nuts network aims for maximum availability \(of data\) and network partition tolerance, thus sacrificing some consistency. This means distributed data is _eventual consistent._

