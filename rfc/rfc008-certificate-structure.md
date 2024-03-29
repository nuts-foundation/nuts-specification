# RFC008 Certificate Structure

|  |  |
| :--- | :--- |
| Nuts foundation | W.M. Slakhorst |
| Request for Comments: 008 | Nedap |
|  | October 2020 |

## Certificate Structure

### Abstract

This RFC specifies which certificates are used in which situation. The basic concept is that certificates are only used to create a secured connection. Certificates and keys related to the certificates are not used in any signing or validation operation. Instead, the DID and Verifiable Credentials standard are used.

### Status of document

This document describes a Nuts standards protocol.

### Copyright Notice

![](../.gitbook/assets/license.png)

This document is released under the [Attribution-ShareAlike 4.0 International \(CC BY-SA 4.0\) license](https://creativecommons.org/licenses/by-sa/4.0/).

## 1.  Introduction

In a secure network certificates play an important role. They are required to set up a TLS connection, and they provide a mechanism for establishing authenticity and integrity of published data.

When it comes to certificates, we’d like to be at the edge of usability and security. All administrative actions regarding keys should be simple and logical to the administrator and yet it should follow the latest security standards. A very elaborate certificate tree structure covers all requirements but is too complicated to understand. Complication can lead to human error, which in the case of private key management leads to security holes. Apart from supporting the production use-case, we also want to make things easier for development and demonstrations. This means that the chosen solution supports a wide variety of tools and/or scripting.

## 2. Terminology

* **CA**: Certificate authority as defined by [RFC2459](https://tools.ietf.org/html/rfc2459).
* **CPS**: Certification Practise Statement.
* **CRL**: Certificate Revocation List. 
* **node**: a running instance of a Nuts implementation \(a.k.a. Nuts node\).
* **node operator**: an organization running a node. Typically a vendor that creates healthcare software for care organizations.
* **PKIo**: Public Key Infrastructure overheid. The by the Dutch government controlled PKI structure.
* **TSP**: Trusted Service Provider. Party that can issue PKIo certificates. 

Other terminology is taken from the [Nuts Start Architecture](rfc001-nuts-start-architecture.md#nuts-start-architecture).

## 3. PKIoverheid

The PKIoverheid CA tree is the choice of CA to trust for production use. It's widely trusted and supported. There are multiple vendors that can issue a certificate. A PKIo TLS certificate MUST be used to secure inter-node connections as well as the peer-to-peer connections used to exchange data. Production nodes SHOULD not accept any other certificate than a PKIoverheid certificate. The certificate SHOULD be used as both server as client certificate in a mutual TLS connection.

### 3.1 Certificate Authority trust chain

A node will need to configure the correct CA-tree so other nodes can connect. 
All certificates that are part of the `Domein Private Services` trust chain as identified by intermediate certificate `Staat der Nederlanden Private Services CA – G1` should be configured.
The certificates in this chain may change over time, and can be viewed and downloaded from the [PKIoverheid](https://cert.pkioverheid.nl/cert-pkioverheid-nl.htm) website.

At the time of writing the chain consists of the following certificates:
- **Staat der Nederlanden Private Root CA G1** (root certificate)
  - **Staat der Nederlanden Private Services CA – G1**
    - **KPN PKIoverheid Private Services CA - G1**
    - **QuoVadis PKIoverheid Private Services CA - G1** 
    - **DigiCert QuoVadis PKIoverheid Private Services CA - 2023**
    - **UZI-register Private Server CA G1**
    - **ZOVAR Private Server CA G1**
    - **Digidentity BV PKIoverheid Private Services CA - G1**

Do not include other PKIoverheid CA certificates.

The PKIoverheid private services CA is not by default accepted by browsers and operating systems.

### 3.2 Certification Practise Statement

The different Certification Practise Statements for the Dutch government can be found on [https://cps.pkioverheid.nl/](https://cps.pkioverheid.nl/) The relevant CPS is the **CPS PA PKIoverheid Private Root v1.5**.

### 3.3 CRL

The Certificate Revocation Lists for PKIoverheid certificates are maintained by the different TSPs. The time between signing of a CRL differs per TSP. A node MUST at least update all CRLs every hour.

## 4. Test and development

The use of PKIoverheid might be less suitable for test and development. The PKIoverheid certificates might be relatively expensive for small organizations and private key should not be shared amongst developers or test servers. Therefore, a node MUST be able to support any set of CAs. This will allow anyone to setup their own test/development network.

## 5. Private key management

A PKIo certificate is costly and it's associated with a single private key. Appropriate measures SHOULD be taken to reduce the number of stored instances of the private key. Reverse/forwarding proxies and centralized keystores with strong encryption can help.

In case of theft or loss of a private key and/or certificate, please consult the TSP documentation.

## 6. Governance

All connections mentioned in §3 MUST be accepted. A node operator MAY NOT deny a connection request other than those using a banned or revoked certificate. A governance framework SHOULD be in place to determine how a certificate can be banned. Banning a certificate does not replace the revoking mechanism of a CA.

## 7. TLS Protocols

Connections must be secured according to the guidelines provided by the Dutch National Cyber Security Center (NCSC).
The current version of this guideline is [2.1](https://www.ncsc.nl/documenten/publicaties/2021/januari/19/ict-beveiligingsrichtlijnen-voor-transport-layer-security-2.1).
According to the guideline, we require at least TLS v1.2 and advice strongly to support TLS v1.3 as well.