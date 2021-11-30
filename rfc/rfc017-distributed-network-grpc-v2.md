# RFC017 Distributed Network Protocol (v2) using gRPC

|  |  |
| :--- | :--- |
| Nuts foundation | R.G. Krul |
| Request for Comments: 017 |  |
| Updates: RFC017 | December 2021 |

## Distributed Network Protocol (v2) using gRPC

### Abstract

This RFC specifies a new gRPC-based distributed network protocol. It succeeds version 1 of the protocol as defined by [RFC005](rfc005-distributed-network-using-grpc.md).
At first, it will provide additions/optimizations over version 1, when feature complete it will replace it.

### Status of document

This document is currently a draft.

### Copyright Notice

![](../.gitbook/assets/license.png)

This document is released under the [Attribution-ShareAlike 4.0 International \(CC BY-SA 4.0\) license](https://creativecommons.org/licenses/by-sa/4.0/).

## 1.  Introduction


## 2. Terminology

* **peer**: remote Nuts node that communicates with the local node.
* **node identity**: (a.k.a. node DID) the DID a node uses to identify itself on the Nuts network. Multiple logical nodes (e.g. a cluster) may share the same node identity.
* **private transactions**: transactions that are meant for a specific receiver (node) on the network. The transaction payload is solely shared with that particular node.

Other terminology comes from the [Nuts Start Architecture](rfc001-nuts-start-architecture.md#nuts-start-architecture).

## X. Authenticating node identity

Particular exchanges (private transactions) might require authentication of the peer's identity. This identity (a.k.a. node identity) is specified by [RFC015 Node identity](rfc015-node-identity.md).
If a node has configured a node DID, it MUST send it as `nodeDID` gRPC header when establishing inbound or outbound connections.
The node DID SHOULD be authenticated when received, so that exchanges that require authentication can proceed without interruption. 

As specified by RFC015, the node MUST authenticate the peer's node DID as follows:

1. Resolve peer's node DID to its corresponding DID document.
2. Assert that one of the `NutsComm` endpoint's host (including the port) matches (one of) the `dNSName` SANs in the peer's TLS client certificate.

## X. Private Transactions