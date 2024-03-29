# RFC015 Node identity

|  |  |
| :--- | :--- |
| Nuts foundation | W.M. Slakhorst |
| Request for Comments: 015 | Nedap |
|  | Nov 2021 |

## Node identity

### Abstract

This RFC describes how to identify a node in the network. Node identification is required for exchanging private data.
Measures to keep data private can range from encrypting data to sending data over a specific connection.
Mutual TLS authentication form the basis of the node identification.
The *Subject Alternative Name* of the certificate is matched against the *serviceEndpoint* of a specific service.
The requirement for the additional service in the DID Document also makes it possible to find other nodes and thus enable service discovery of Nuts nodes.

Concepts like private addressing can greatly benefit from targeting specific nodes instead of individual DIDs.
This RFC links individual DIDs to the service provider DID.

### Status

This document describes a Nuts standards protocol.

### Copyright Notice

![](../.gitbook/assets/license.png)

This document is released under the [Attribution-ShareAlike 4.0 International \(CC BY-SA 4.0\) license](https://creativecommons.org/licenses/by-sa/4.0/).

## 1. Introduction

When all participants of a network share the same public data, there's no need to know which node services a particular organization.
This changes when data becomes private. Private data needs to be routed to the correct node.
In order for that to work, there's a need to identify each node and link that identity to services organizations.

Two things need to be resolved:

1. Node identity for peers that are connected. Network connections are the only interaction a node has to another node, so the connecting peer needs to be identified.
2. The DID subject that needs to receive the private data needs to be linked to a DID that has a node identity.

For the first point TLS and certificates will be used as basis.

## 2. Terminology

* **DID**: [Decentralized Identifiers](https://www.w3.org/TR/did-core/).
* **SAN**: [subject alternative name](https://datatracker.ietf.org/doc/html/rfc5280#section-4.2.1.6)
* **Service provider**: organization that runs the Nuts node and manages private keys for other organizations.
* **VC**: [Verifiable Credential Data Model](https://www.w3.org/TR/vc-data-model/).

Other terminology comes from the [Nuts Start Architecture](rfc001-nuts-start-architecture.md#nuts-start-architecture).

## 3. Identification

### 3.1 Certificates

TLS certificates are linked to a domain name via a SAN extension.

```
X509v3 extensions:
  X509v3 Subject Alternative Name:
    DNS:example.com
```

This domain is checked by a client that connects to a server with that domain. The certificate the server uses must match the domain the client dialed.
Only identities of type `dNSName` are to be considered.
Multiple identities may exist in a certificate and wildcard domains are also possible.
The value of the identity, `example.com` in the example is used to link DIDs to certificates.

### 3.2 DID Document

A DID Document can define *services*, this is decribed in [RFC006](rfc006-distributed-registry.md). This RFC defines a new type of service: `NutsComm`.
The service has a `serviceEndpoint` that defines the gRPC endpoint of a node.

```
{
  "id": "did:nuts:123#IyvdTGF1Dsgwngfdg3SH6TpDv0Ta1aOEkw",
  "type": "NutsComm",
  "serviceEndpoint": "grpc://example.com:5555"
}
```

The `type` MUST be `NutsComm`. Inter-node communication uses gRPC over HTTP/2.
The `serviceEndpoint` MUST start with `grpc://` and end with the correct port number, `:5555` in this example.

### 3.3 Connecting identity

To determine the connecting identity, a `DID` metadata header MAY be sent by each node upon establishing a connection. 
See [RFC005](rfc005-distributed-network-using-grpc.md) for more information about sending headers. If no such a header is sent, the connected peer remains anonymous.
The validating node will resolve the DID document of the sent DID and will compare the `serviceEndpoint` of the `NutsComm` service with the SAN of the certificate that is related to the connection.
A single match is sufficient when multiple DNS entries are available in the SAN extension. If no match is found the connected peer remains anonymous.

The hostname of the node and certificate SAN are validated according to [RFC 6125, Appendix B](https://tools.ietf.org/html/rfc6125) (`Common Name` is not supported).

This description applies to both client and server.

## 4. Service provider

A service provider acts on behalf of its customers. It also operates the node that has access to the private keys and thus controls the DID Documents.
When addressing an organization, it's usually sufficient to address the service provider since the service provider will route any message internally to the correct organization.
The same holds for encrypting data for an organization: the service provider has access to their private key to decrypt the data.
Addressing the service provider can greatly reduce the effort of the service provider to handle the data.

A great example for this is the [RFC017 v2 network protocol](rfc017-distributed-network-grpc-v2.md) where the added `pal` header on the transaction is encrypted with the DID encryption key of the service provider.
Decrypting such a header is a trial and error proces where the number of tries is reduced by a factor of hundreds.

To find the correct service provider for a DID, the DID Document MUST link to the DID Document of the service provider.
The `NutsComm` service is reused for this purpose. It'll contain a reference instead of an actual endpoint.

```
{
  "id": "did:nuts:456#IyvdTGF1Dsgwngfdg3SH6TpDv0Ta1aOEkw",
  "type": "NutsComm",
  "serviceEndpoint": "did:nuts:123/serviceEndpoint?type=NutsComm"
}
```

## Appendix A: Design decisions

### A.1 Using TLS certificates as base for identification

In order to really know *who* is at the other side of the connection, you'll have to do something with cryptography. This has already been invented and is called TLS.
Not relying on TLS and building a system of our own would be very hard to get right. It's better to rely on a system that already checks for security vulnerabilities and fixes them fast.

