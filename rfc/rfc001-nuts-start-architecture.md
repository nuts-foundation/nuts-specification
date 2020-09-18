# RFC001 Nuts Start Architecture

|  |  |
| :--- | :--- |
| Nuts foundation | W.M. Slakhorst |
| Request for Comments: 001 | Nedap |
|   | September 2020 |

## Nuts Start Architecture

### Abstract

short description on the RFC

### Status of document

This document is currently in draft.

### Copyright Notice

![](https://lh3.googleusercontent.com/OOulGS7n3iwBmxopI-5TrBWVzAgShugACJOq3D61UVu8-Z3VJbcO-lkvnnpOlTrxTJQ1p5FasQVujHPNna64P8p5EPQkhdCnQGJl39LVokDNWNRTCtOKOM3MtWwHd2_BR6adVOIn)

This document is released under the [Attribution-ShareAlike 4.0 International \(CC BY-SA 4.0\) license](https://creativecommons.org/licenses/by-sa/4.0/).

## 1.  Introduction

This document describes the general principles of all related Nuts architecture. It translates the Nuts manifest into architectural principles. More detailed architectural and design documents are available addressing different aspects of the network. Those documents will refer back to this document regarding high level choices. The idea is that this document will remain relatively the same whereas the individual documents change over time.

## 2. Terminology

* Actor: The care organization that, in the context of a specific subject, wants to access medical data of a subject available at a custodian.
* Custodian: The care organization that, in the context of a specific subject, is responsible for storing the medical data.
* Digital signature: a cryptographic signature like RFC5280 or RFC7515. Specifically not the digital version of the wet signature.
* Legal Base: grounds for medical data exchange given by the subject conforming to national and international legislation. By default no medical data may be exchanged.
* Node: a piece of software implementing the Nuts specification.
* Service provider: an organisation providing software services which support care organisations in their daily work.
* Subject: The subject of the medical data, usually the patient.

## 2. Manifest

The start architecture is a translation of the Nuts manifest to a high level architecture. Each paragraph will refer to the different manifest entries when applicable. References will be denoted between square brackets, they’ll start with a capital M followed by the number corresponding to the numbers below. The text is translated from Dutch, check https://nuts.nl for the original.

#### 1 - Base
We believe that the starting point of service providers is to support care professionals and  patients in digital form so they can perform their work optimally. Software influences the quality of care, maintainability, client/employee happiness and sustainability. We think it’s the responsibility of the service providers to acknowledge this and handle it diligently.

#### 2 - Patient first
We believe the patient deserves an active role in the network, for example through their Personal Health Environment (PHE). The patient, mandated caregiver or financial representative must be given the choice to communicate about the patients health, have control over who is allowed to view sensitive data and must be able to see who has access to their data.

#### 3 - Ownership
We believe medical data belongs to the patient and care professional, not the service provider. We believe service providers should charge for services that provide added value to their users and that data should be freely available to whomever is involved.

#### 4 - Distributed network
We believe in a distributed model for communication and data exchange, similar to the internet. We do not want to invest in network technology with a specific supplier or party that directly or indirectly controls all care data for a set of patients. We want a robust network for everyone from everyone without any central points of failure or monopolies.

#### 5 - Open standards
We believe that joining the network must be without friction. The common communication infrastructure must be open and standardised. No (license) costs should be charged for using it. An open-source reference implementation must be available.

#### 6 - Privacy by design
We believe privacy by design within the network is of the utmost importance. Therefore we want a system where information is consciously shared with each other. We think that it’s important that a human makes the choice for sharing data, for example by recording consent or accepting a request for data sharing.

#### 7 - Security by design
We believe in a system that is designed with the belief that parties in the network can’t be trusted and that malicious parties get smarter every day. We want to prevent that the infrastructure allows any unauthorized person to access, change or remove medical data.

#### 8 - Cryptographic base
We believe that the technology is ready and the time is now for a system based upon cryptographic principles. We want to give guarantees on identities and go beyond “access by configuration”. We believe that we can win the trust of the entire healthcare sector and eventually that of the common citizen.

## 4. Identity
A strong identity is the base for authentication, authorization and auditing. An identity in itself is not that important, it's one of the three use-cases that puts requirements on the identity. An identity can be seen as the link between the user and the virtual domain [M6], an identity consists of one or more attributes which are relevant for the different use-cases. The following sections will describe those requirements in detail.

### 4.1. Authentication
Medical data is regarded as one of the most sensitive types of data. It therefore requires the highest possible level of security. In the case of an identity this is translated as a “strong identity”. This means that any attributes of this identity will have to be verified by a trusted 3rd party [M7]. If the identity is to be exchanged, this also means that a digital signature has to be present alongside the attributes [M8]. When an identity has a digital signature from a trusted party, it can be used for authentication in the context of medical information exchange. For authentication purposes it’s not required that the identity has any specific attributes. A curated list of trusted 3rd parties should be published for each participant to trust.

### 4.2. Authorization
When accessing sensitive resources, an authenticated identity also must be authorized for the requested resources [M6, M7]. It depends on the particular use-case which attribute is required to link the identity to an authorized resource. The authorization might be given to the identity directly but in most cases the authorization lies with the organisation the user (holding the identity) is representing. In the case of an individual authorization, the identity must have an attribute that uniquely identifies the user. In the case where the authorization lies with the organisation, a link between the identity and the organisation must be given. This link must be represented by a chain of digital signatures where each certificate (or public key) used in the chain can be trusted [M8]. See also the chapter on Trust. When the authorization lies with the organisation, the organisation requires an attribute that identifies that organisation. 

### 4.3. Auditing
Auditing is required to give the subject insight in who has seen their information [M2]. To assist the subject this must consist of human readable information about the user and the organisation that user is working for. Therefore it’s required for an identity to at least have human readable attributes, eg: name and initials. For the organisation identity this makes the organisation name required.

## 5. Privacy & Legislation
### 5.1. GDPR and local legislation
We all know what this means: right to be forgotten but 15 years of medical data storage. Breaking doctor patient confidentiality is only allowed with specific authorization from the patient.

Therefore: keep data at the source and retrieve on demand. Do not copy without reviewing it. A proof for data exchange must be known before others can be authorized.

### 5.2. Data exchange parties
The GDPR speaks of data processors which play a role in digital information exchange. In the case of Nuts these are the service providers. Service providers have a direct relation with the care organisation it’s providing services for. Medical (and other personal) data must be exchanged directly between the service providers of care organisations. No third party may play a role in this data exchange since it would be able to deduce information even when all data is encrypted. This also means that there’s no room for a central party that would be involved in data exchange or the exchange/management of any document establishing the legal base for a data exchange [M6].

The result of reducing the involved parties is that the network must be a distributed network [M4]. Required information for care organisations, their service providers and available service descriptions must be distributed amongst all network participants. Since this registry data is security related, it must be signed with a digital signature.

### 5.3. Legal base for data exchange
When an actor wants to retrieve data from a custodian regarding a specific subject, it must act on a known legal base [M2]. This legal base must be stored at the custodian side since that is also the responsible party for the subject’s data [M3]. This legal base will be the base for most of the authorizations of an identity for a particular resource.

The actor must be informed about the existence of the legal base. This notification is a confidential action between the custodian and the actor. An actor must not query custodians for data without having been notified about the legal base. Otherwise the actor might reveal it has a relationship with a certain subject to other parties. That would be a breach of privacy [M4, M6].

In a perfect world, the legal base should also have a digital signature that can be traced back to the subject. This, with the current available technology, digital skills and level of adoption, might prove a bridge too far. Therefore any such signature will probably be labeled as optional in any architectural sub-document.

## 6. Trust & Security
Trust is, in the case of Nuts, the collective of authorizations, digital signatures and curated list of trusted 3rd parties. This relates closely to the general security of the network as a whole. An exchange of data between a custodian and actor is only allowed if all involved parties can be trusted [M7]. An action in the virtual domain must be linkable to real world parties [M6]. The following sections describe the different relations and how this eventually relates to a root-of-trust (or actually multiple roots).

A fundamental choice has been made to place trust in a curated list of 3rd parties [M4]. Trust between individual service providers or care organisations does not play a role [M7]. Technically this is enforced with the use of digital signatures [M8]. Attributes that are exchanged between parties do not hold any value unless it can be verified by a digital signature. Besides the improved security, it also levels the playing field for new parties and eliminates the value of inter-party political relationships [M1, M3]. Placing trust in certain 3rd parties does create a dependency on those parties and their security and trust model. 

### 6.1. Request & Service Provider
Service providers are the gatekeepers for care organisations for the virtual domain. They provide access to the virtual domain by means of software. So in order to authorize a specific technical request, the requesting service provider must be identified and authenticated first. This must be done via digital signature where the secure key must be verifiable by a trusted 3rd party.

### 6.2. Request & Care organisation
In order to do authorization, the request must contain information about the custodian, actor and subject. The identity of the requesting party must be verifiable by means of a digital signature. The requesting party usually is the actor and may consist of a combination of organisation and/or user identity. The relationship between the requesting service provider and the requesting party must also be verifiable.

### 6.3. Service provider & Care Organisation
When a custodian and actor interact, they do this through their service providers, therefore the relationship between care organisation and service provider must be verifiable by others [M4, M7]. A relationship can only be verified if both sides provide proof of the relationship. As with other proofs in Nuts, this requires digital signatures of specific attributes representing the relationship [M8]. The secure key used for the signatures must be verified by a trusted 3rd party. The proof of the relationship must be distributed along the network since it is required before any interaction can take place [M4]. The most important identifiable part of a care organisation is it’s commonly known name. A proof for this name must be available from a trusted 3rd party if it’s to be distributed amongst all network participants.

### 6.4. Care organisation & User identity
In the case a user acts as the actor and the authorization is given to the care organisation which the actor represents, a digital signature proving the relation between user and organisation must be present in the request [M7]. This proof can’t be distributed beforehand since it’ll contain information about the user identity which also has its own privacy rights. This digital signature also prevents service providers from acting as a representative of the care organisation.

### 6.5. User identity & User
In the end it’s the user that is viewing sensitive data so the final proof of relationship is that the user, currently behind the controls, proves it has the user identity [M6]. This must be done by signing a contract stating the extent of the required privileges for the current session. This is also known as the authentication process. The signature must validate the required user attributes and the secure key used for the signature must be verified by a trusted 3rd party. This signature must also validate that the user has chosen to use the service provider’s software.

## 7. Configuration & automation
One of the points of the manifests speaks of a distributed network [M4], this means that there’s no owner of the network and thus no central body that makes decisions. Every aspect of a fully functioning node thus requires configuration that determines what can be trusted. A node operator should be in full control of what a node is doing. Doing everything by hand can become very costly when the network grows. Therefore to be successful, some layer of automation must be available for most of the functionality.

Every Nuts RFC/specification must explicitly mention what part is based on mutual trust. This could be data or some sort of configuration. If a RFC describes a mechanism where multiple nodes must trust each other based on some sort of configuration or shared state it must also specify (or delegate it to a different RFC) how this can be exchanged in an automated manner. The described automated process does not replace the manual version but rather builds upon it.

An important part of the automated process is how nodes can make sure they can trust the offered data. Non-repudiation is the magic keyword. This means that every piece of data must either be obtained from the source or it must have been signed with a digital signature from the authoring party.

