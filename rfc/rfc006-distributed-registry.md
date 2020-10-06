# RFC006 Distributed Registry

|  |  |
| :--- | :--- |
| Nuts foundation | R.G. Krul |
| Request for Comments: 006 | Nedap |
|  | September 2020 |

## Distributed Registry

### Abstract

This RFC describes a protocol to build a registry containing information required for care organizations to exchange data
through their software vendors. The registry typically contains vendors, organizations, services and endpoints.
It uses [RFC004](rfc004-distributed-document-format.md)'s documents as underlying data format.

### Status of document

This document is currently a draft.

### Copyright Notice

![](../.gitbook/assets/license.png)

This document is released under the [Attribution-ShareAlike 4.0 International \(CC BY-SA 4.0\) license](https://creativecommons.org/licenses/by-sa/4.0/).

## 1.  Introduction

When care organizations want to exchange data using Nuts they need to know where to find that data and how to authenticate it.
This RFC describes the document types containing the information required to form this registry.
It uses [RFC004](rfc004-distributed-document-format.md) as underlying data format. The origin and requirements of
certificates are defined by RFC008.

## 2. Terminology

* **Bolt**: a use case built on top of the functionality Nuts provides.
* **Document**: a distributed document as specified by [RFC004](rfc004-distributed-document-format.md).
* **Organization**: a care organization exchanging data with other care organizations over a Nuts network.
* **Vendor**: an object developing software for care organizations participating in a Nuts network.
  Organizations access the Nuts network through their vendor's software.
* **Endpoint**: a URI or URL exposed by a vendor which can be used by other vendors to pull data.
* **Service**: a group of endpoints implementing a service required by a Bolt.
* **Object**: an instance of a vendor, organization or endpoint.

Other terminology comes from the [RFC001 Nuts Start Architecture](rfc001-nuts-start-architecture.md#nuts-start-architecture).

## 3. Domain Model
The following UML entity diagram displays how vendors, organizations, endpoints and services relate:

![RFC006 Domain Model](../.gitbook/assets/rfc006-registry-domain-model.svg)

## 4. Format
Entities MUST be represented as JSON document as the JWS payload defined by [RFC004](rfc004-distributed-document-format.md).
The **cty** header parameter from [RFC004](rfc004-distributed-document-format.md)) MUST contain the type defined for
that particular object type. The type parameter `o` MUST be used to indicate creation or an update, e.g:

`registry/vendor;o=update`

Other values than `update` or `create` for the `o` parameter SHALL NOT be used.

## 4.1. Creating and updating
When creating entities its required fields MUST be specified as JSON document. The `o` parameter MUST be specified as `create`.

To update an object's fields [JSON Patch (RFC6902)](https://tools.ietf.org/html/rfc6902) is used. The object to update
MUST be specified by specifying the **tid** (*timeline ID*, see [RFC004](rfc004-distributed-document-format.md))
The `o` parameter MUST be specified as `update`. An update document MUST contain the following fields:
* **patch** MUST contain an array with patch operations according to [RFC6902](https://tools.ietf.org/html/rfc6902).

If one of the operations fails the document MUST be rejected.

Example for document type `registry/organization;o=update`:
```json
{
    "patch": [
       { "op": "replace", "path": "/name", "value": "NoPlaceLikeHome Thuiszorg" }
    ]
}
```

When using a destructive operation (especially when using `remove` or `replace` on arrays) vendors COULD use the
`test` operation to make sure the intended values are removed.

## 4.2. Signing
All documents (registration or update) MUST be signed by the vendor who owns the object (be it a vendor, organization or endpoint). The only
exception is when registering a new vendor; in that case it MUST be another, already registered vendor. If a new CA certificate
is registered for a vendor, signing certificate issued by that CA certificate can also be used for updating previously
published documents from that point on the DAG (see [RFC004](rfc004-distributed-document-format.md)).

In any case the signing certificate MUST conform to [RFC008](rfc008-certificate-structure.md).

## 5. Object types
This section describes each of the supported object types.

### 5.1. Vendor
This document registers a vendor on the registry. It's identified by the type `registry/vendor`.
Since new vendors can't connect to other nodes in the network since their CA certificate isn't trusted yet,
another vendor (**Alice**) SHOULD register the new vendor (**Bob**) through the following process:

1. **Bob** generates his CA certificate (see [RFC008](rfc008-certificate-structure.md)) and creates the vendor proof (see section 6, "Vendor proof").
2. **Bob** sends his CA certificate and vendor proof to **Alice** via an out-of-band mechanism (see below).
3. **Alice** creates the vendor registration document with **Bob**'s CA certificate and vendor proof, then signs and publishes it.
4. **Bob** adds **Alice**'s CA certificate to its truststore.
5. **Bob** connects to **Alice** and starts receiving documents.

Before **Alice** connects to **Bob** they communicate out-of-band. A (relatively) trusted means of communication SHOULD
be used, like signed e-mail or authenticated chat.

#### Fields
The following fields MUST be present:
* **certs** (certificates): MUST contain an array containing base64 encoded vendor CA certificates.
* **prfs**: MUST contain an array of cryptographic proofs asserting the vendor being a real software company.

Example:
```json
{
    "certs": ["CA certificate as base64 ASN.1"],
    "prfs": ["vendor proof"]
}
```

#### Duplicates
Vendors COULD re-register their vendor e.g. in case of loss of CA and signing key. As long as the new proof is valid
the new CA certificate MUST be deemed valid to authenticate that vendor. The original registration MUST be kept as well,
essentially merging the two. 

#### Validation
When processing a vendor registration or update it MUST be validated as follows:

1. Assert all required fields are present.
2. Assert all fields value formats are valid.
3. For updates: assert only modifiable fields are updated (**certs**, **prfs** are modifiable).
4. Validate **prfs**:
   * Assert there is at least one proof.
   * Assert proofs are valid at the time they're introduced.
   * Assert all proofs authenticate the same vendor.
5. Validate **certs**:
   * Assert there is at least one certificate.
   * Assert all certificates conform to the vendor CA certificate specification (see [RFC008](rfc008-certificate-structure.md)).
   * Assert all certificates have the same MUST contain the same vendor ID.

If any of these steps fail the registration or update SHALL NOT be processed.

#### Revoking CA certificates
When a vendor loses or leaks its private key, the vendor SHOULD remove the certificate from its registration to avoid
malicious parties exploiting it. However, other parties MUST regard signatures created before removal of the certificate
 as valid. How the vendor SHOULD change its CA certificate depends on the situation:

##### CA certificate private key loss
In case the vendor loses its CA private key it MUST register a new vendor CA certificate. This can be done in two ways,
depending on what was lost:

1. If the vendor still has a valid signing certificate it COULD sign the update using that private key.
2. Otherwise, when the signing certificate has expired or the private key is lost as well, the vendor MUST also provide
new proof to authenticate ownership of the vendor. The new proof MUST be the first operation in the **patch** field.

Example update:
```json
{
  "patch": [
    { "op": "add", "path": "/prfs", "value": "some new proof" },
    { "op": "replace", "path": "/crts", "value": ["CA certificate as base64 ASN.1"] }
  ]
}
``` 

##### CA certificate private key leakage
In case the vendor leaks its CA private key (or it gets stolen), it MUST remove the associated certificate as soon as possible
and register a new one.
In the same update it MUST register new proof and CA certificate as in the key loss process.
Example update:
```json
{
  "patch": [
    { "op": "add", "path": "/prfs", "value": "some new proof" },
    { "op": "replace", "path": "/crts", "value": ["CA certificate as base64 ASN.1"] }
  ]
}
``` 

To avoid attackers re-adding the previously removed CA certificate, the application SHOULD allow administrators to
 register compromised X.509 certificate thumbprints with the date it was compromised. The application SHALL NOT accept
 certificates or signatures on or after this date. Communication of network participants regarding this compromised
 certificate list SHOULD happen out-of-band over an authenticated channel.

### 5.2. Organization
This document registers a care organization as a client of registered vendor (section 5.1). It's identified by the type `registry/organization`.

#### Fields
The following fields MUST be present:
* **id**: MUST contain the organization's ID encoded as URN. This can be any valid URN, for instance an AGB-code or
  KVK-number e.g.: `urn:oid:2.16.840.1.113883.2.4.6.1:06123456` (AGB-code). It is used to refer to the organization by the
  vendor itself or other vendors. The organisation ID SHALL NOT be changed.
* **vid** (vendor ID): MUST be the organization's software vendor's ID as URN (e.g. `urn:oid:1.3.6.1.4.1.54851.4:1234`).
  The vendor ID SHALL NOT be changed; relation to a care organization SHALL NOT be transfered to another vendor. 
* **name**: MUST contain the organization's commonly known name as string. Only alphanumeric, dashes (`-`) and space characters are allowed,
  it SHALL NOT be empty or only contain spaces. It MUST conform to the following regex: `[a-zA-Z0-9 ]+`.
* **prfs** (proofs): MUST contain an array with cryptographic proofs asserting the organization as being an actual care organization.
  at the same it serves as commitment from the care organization that it uses the vendor's (identified by **vid**) software.
  See section 7 for supported proof types.

Example:
```json
{
    "id": "urn:oid:2.16.840.1.113883.2.4.6.1:06123456",
    "vid": "urn:oid:1.3.6.1.4.1.54851.4:1234",
    "name": "HomeSweetHome Thuiszorg",
    "prfs": ["organisation proof"]
}
```

#### Validation
When processing an organization registration or update it MUST be validated as follows:

1. Assert all required fields are present.
2. Assert all fields value formats are valid.
3. For updates: assert only modifiable fields are updated (**name**, **prfs** are modifiable).
4. Assert the document is signed by the vendor identified by **vid**.
5. Validate **prfs**:
   * Assert there is at least one proof.
   * Assert proofs are valid at the time they're introduced.
   * Assert all proofs authenticate the same organisation.

### 5.3. Endpoint
This document registers an endpoint exposed by a vendor which CAN be used to exchange data with the vendor or its organizations.
It's identified by the type `registry/endpoint`.

#### Fields
The following fields MUST be present:
* **id**: MUST be a string identifying this endpoint. It must be unique for the vendor.
* **vid** (vendor ID): MUST contain the ID of the vendor this endpoint belongs to.
* **type**: MUST be a URN describing the type of this endpoint in the form of `urn:oid:1.3.6.1.4.1.54851.2:type`.
  The actual type MUST be an ASCII alphanumeric (a-z, 0-9) lowercase string.
* **loc** (location): MUST be a URL pointing to where the endpoint is exposed (e.g. `https://nuts.nl/some/path`)
* **nbf** (not before): MUST contain an RFC 3339 timestamp indicating from what date/time the endpoint SHOULD be considered valid.

The following fields MAY be present:
* **exp** (expiration): MUST contain an RFC 3339 timestamp indicating when the endpoint expires and SHOULD NOT be considered valid anymore.

Example:
```json
{
    "id": "de59b062-d783-4727-a05e-57ed6035f00d",
    "vid": "urn:oid:2.16.840.1.113883.2.4.6.1:1234",
    "type": "urn:oid:1.3.6.1.4.1.54851.2:fhir",
    "loc": "https://nuts.nl/some/path",
    "nbf": "2020-04-23T18:25:43.511Z"
}
```

### 5.4. Service
A service groups a set of endpoints to be used for a specific use case (Bolt) for a specific organization. It's identified by the type `registry/service`.
It MUST contain the following fields:
* **oid** (organization ID): MUST contain the ID of the organization this service belongs to.
* **vid** (vendor ID): MUST contain the ID of the vendor this endpoint belongs to.
* **name**: MUST contain the service's case sensitive name. It MUST be unique within the organization and vendor combination.
  It is used by other parties who want to execute a specific Bolt (with this organization) to look up the correct endpoints.
* **eps** (endpoints): MUST contain an array of endpoint IDs to be used for this service. The endpoints MUST have the
  same **vid** field as the service.
* **nbf** (not before): MUST contain an RFC 3339 timestamp indicating from what date/time the service could be used.

The following fields MAY be present:
* **exp** (expiration): MUST contain an RFC 3339 timestamp indicating when the service expires and SHOULD NOT be used anymore.

Example:
```json
{
  "oid": "urn:oid:2.16.840.1.113883.2.4.6.1:06123456",
  "vid": "urn:oid:1.3.6.1.4.1.54851.4:1234",
  "name": "some-bolt",
  "eps": ["1B5E2A91-5C19-41B6-9709-BC9050D193E5", "some-other-endpoint-ID"],
  "nbf": "2020-10-20T20:30:50.52Z"
}
```

#### Validation
When processing a service registration or update it MUST be validated as follows:

1. Assert all required fields are present.
2. Assert all fields value formats are valid.
3. For updates: assert only modifiable fields are updated (**eps**, **nbf** and **exp** are modifiable).
4. Assert the vendor identified by **vid** is registered.
5. Assert the organization identified by **oid** is registered.
6. Assert the endpoints referenced in **eps** are registered. 
7. Assert the document is signed by the vendor identified by **vid**.
8. Assert the combination of **oid**, **vid** and **name** is not already registered.

## 6. Vendor proof
Vendor proof authenticates the registration of a vendor, assuring that it:
1. represents a real company, registered at the (Dutch) Chamber of Commerce and,
2. commits to an agreement if required by the network (optional).

TODO: Include public key of vendor CA certificate

Vendor proof MUST be a JSON Web Token (JWT) encoded as string. The JWT MUST be signed using the algorithm specified by
the specific proof type.
 
The JWT SHALL NOT be encrypted.

### 6.1. Fields
The following claims MUST be present:
* **sub** (subject): MUST contain the fully qualified vendor ID.
* **iat** (issued at): MUST contain the time at which the proof was issued.
* **x5c** (X.509 certificate chain): MUST contain an array of base64 encoded signing certificates.

The following claims MAY be present:
* **agrs** (agreements): MUST contain an array with network usage agreements (see Agreements section below) that the
  signer commits to when signing the proof.
  
Other claims SHOULD NOT be present and MUST be ignored.
  
### 6.2. Agreements
Networks COULD decide they want every vendor to sign an agreement when registering. In that case the vendor SHOULD specify
the SHA-1 hash(es) of the agreement(s) that it commits to in the proof. The agreements SHOULD be published out of band
on a well-known location. Other vendors SHOULD verify that the newly registered vendor specified an agreement they accept
(accepted agreements COULD be configured in the vendor software).

### 6.3. Accepted certificates
This describes which certificates are accepted as vendor proof.

#### 6.3.1. PKIoverheid Persoonlijk Organisatiegebonden Certificaat

When using this proof type the certificate MUST be a trusted PKIoverheid TSP CA (Trust Service Provider Certificate Authority).
The signature algorithm MUST be one of the following: PS256, PS384, PS512, ES256, ES384 or ES512. Other algorithms SHALL NOT be used.

The **x5c** field MUST contain the end object certificate and all intermediate CA certificates except the *Staat der Nederlanden* Root CA certificate.
The root CA certificate SHOULD NOT be included.

TODO: KVK-nummer

## 7. Organization proof
TODO: Consider using DIF Verifyable Credentials instead of JWT/JWS

Organization proof authenticates the registration of a care organization, assuring that it:
1. represents a real care organization and,
2. wants to exchange data using Nuts through the vendor's software.

Organization proof MUST be a JSON Web Token (JWT) encoded as string. The JWT MUST be signed using the algorithm specified by
the specific proof type.
 
The JWT SHALL NOT be encrypted.

### 7.1. Accepted certificates
This describes which certificates are accepted as vendor proof.

### 7.1.1 UZI certificate signature
TODO

## 8. Trust

There's no central authority controlling access to the network (see section 4.4 of [Nuts Start Architecture](rfc001-nuts-start-architecture.md));
a new vendor gets access by asking another vendor, who already has access, to register it (the new vendor) to get access.
This means there's a chain of trust: if vendor A and B form a network and vendor C joins through vendor B, new vendor C
is trusted by vendor A because it (A) trusts vendor B. This is like vouching; you should only register other vendors you
trust because if that vendor turns out to be malicious, it could backfire to you as well. In the worst case other vendors
could blacklist your vendor as well, denying you from accessing the network.

### 8.1. Network bootstrapping

Having no central authority causes a bootstrapping problem: how to establish trust between the first two nodes which
form a new network. This SHOULD be solved by exchanging the vendor CA certificate and proofs and registering the other
vendor on the local node. After that one vendor can connect to the other vendor (or both, bi-directional) causing the
registration documents to be exchanged, forming the network.

### 8.2. Accepting invalid proof

When implementing this RFC there might be no viable means to create vendor and/or organization proof. In that case
implementations SHOULD allow node administrators to explicitly accept a document with missing or invalid proof.
That way vendors can still start exchanging data when viable means of authenticating vendors and/or organizations
are not available yet. However, when the proof means become available and common the vendors and organizations MUST
update their registration so authenticated proof can be made mandatory.

## 9. Security & Privacy

All information published in the registry is public. Vendors MUST take care NOT to publish sensitive information.
The distributed nature makes it very hard or even impossible to have information removed. The authenticity and integrity
of the published information is protected by the [RFC004 Distributed Document Format](rfc004-distributed-document-format.md).


