# RFC016 Transaction Lamport clock

|  |  |
| :--- | :--- |
| Nuts foundation | W.M. Slakhorst |
| Request for Comments: 016 | Nedap |
| Updates: RFC004 | November 2021 |

## Transaction Lamport clock

### Abstract

This RFC updates [RFC004](rfc004-verifiable-transactional-graph.md). It specifies a version two transaction format and adds additional requirements to version 1.
All additional requirements are backwards compatible. This RFC introduces the concept of the [Lamport clock](https://en.wikipedia.org/wiki/Lamport_timestamp) to the transaction contents.

### Status of document

This document is currently a draft.

### Copyright Notice

![](../.gitbook/assets/license.png)

This document is released under the [Attribution-ShareAlike 4.0 International \(CC BY-SA 4.0\) license](https://creativecommons.org/licenses/by-sa/4.0/).

## 1.  Introduction

Version one of the transaction format introduced an ordening based on predecessors. This creates a strict ordering but does not support the exchange of missing transactions between nodes.
More efficient set reconciliation methods can be used when transactions can be grouped into ranges. 

## 2. Terminology

* **Lamport clock**: Logical clock creating a casual ordering between transactions.

Other terminology comes from the [Nuts Start Architecture](rfc001-nuts-start-architecture.md#nuts-start-architecture).

## 3. New transaction format

Version 2 transactions remain encoded as JWS: [RFC004 ref](rfc004-verifiable-transactional-graph.md#3-transaction-format).

Changes for the version 2 transactions:

* The **ver** protected header MUST be 2.
* The **lc** protected header is added. It MUST be a positive number constructed as follows:
  * if the transaction has no entries in **prevs**, it's' lc value is **0**.
  * otherwise, the value MUST be equal to `max(prev1, ... prevN)+1`.

These additions interact with the requirement that each transaction point to one of the latest *HEADS* by means of their **prevs**. 
Because there can only be a single root transaction, that transaction will automatically have a **lc** value of `0`.
The requirement that a transaction points to previous transaction(s) also makes it possible to apply the latest point.
[RFC004 ref](rfc004-verifiable-transactional-graph.md#3-transaction-format) also defines the notion of branches. 
**LC** values will overlap between branches.

## 4. Changes to version 1

### 4.1 Virtual Lamport clock

Version 1 transactions do not have a **lc** value. Given the algorithm of the previous chapter, a value can easily be calculated.
Although version 1 transactions do not have a **lc** value in the transaction, implementing software MUST assign a value based on the before mentioned algorithm.

### 4.2 Processing the DAG

[§3.5 Processing the DAG](rfc004-verifiable-transactional-graph.md#35-processing-the-dag)is replaced by this paragraph.

Processing the DAG can be seen as planning tasks required for construction: some tasks can happen in parallel \(laying floors and installing electricity\), some tasks must happen sequentially \(foundation must be poured before building the walls\). This is the same for transactions on the DAG: transactions on a branch MUST be processed sequentially but processing order of parallel branches is unspecified, and before processing a merging transaction all **prevs** \(branches\) must have been processed.

Processing MUST be performed in order of the (virtual) **lc** values. 
If multiple transactions have the same **lc** value, they MUST be processed in sorted order.
The transactions MUST be sorted by their hexadecimal hash value.
