# RFC017 Distributed Network using gRPC v2

|  |  |
| :--- | :--- |
| Nuts foundation | R.G. Krul |
| Request for Comments: 017 |  |
| Updates: RFC0-5 | December 2021 |

## Transaction Lamport clock

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

* **node identity**: (a.k.a. node DID) the DID a node uses to identify itself on the Nuts network. Multiple logical nodes (e.g. a cluster) may share the same node identity.

Other terminology comes from the [Nuts Start Architecture](rfc001-nuts-start-architecture.md#nuts-start-architecture).

## X. Authenticating node identity

## X. Private Transactions