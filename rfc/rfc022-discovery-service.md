# RFC022 Discovery Service

|                           |           |
|:--------------------------|:----------|
| Nuts foundation           | R.G. Krul |
| Request for Comments: 022 | Nedap     |
|                           |           |

## Discovery Service

### Abstract

This specification defines the use of a Verifiable Presentation for Service Discovery using a client-server model.

### Status of document

This document is currently in draft.

### Copyright Notice

![](../.gitbook/assets/license.png)

This document is released under the [Attribution-ShareAlike 4.0 International \(CC BY-SA 4.0\) license](https://creativecommons.org/licenses/by-sa/4.0/).

## 1.  Introduction

Service Discovery is the starting point of most data exchange protocols.
Users need to know which (remote) systems offer a particular service and systems need to know how to connect to those systems.

This RFC defines a protocol for Service Discovery using Verifiable Presentations using a client-server model.
The advantage of using Verifiable Presentations is that the client needs little trust in the server:
clients can verify the authenticity and integrity of the Verifiable Presentation and Verifiable Credential.

## 2. Terminology

* **Discovery Service**: collection of Verifiable Presentations that allows discovery of systems/parties that offer a particular service.
* **Discovery Definition**: document describing the Discovery Service.
* **Server**: system hosting the discovery service and accepting new registrations.
* **Client**: system registering presentations on the list and reading it to discover other systems/parties.

Other terms are as defined in the following specifications: "JSON Web Token (JWT)" [JWT], Verifiable Credentials Data Model 1.1 [VC-DATA-MODEL], Verifiable Credentials Presentation Exchange [PE], Decentralized Identifiers 1.0 [DID].

## 3. Service endpoint

The service endpoint is an HTTP endpoint exposed by the server which hosts a list of Verifiable Presentations.
Clients can query the list or submit a new Verifiable Presentation to it.
All presentations MUST conform to the Presentation Definition associated with the discovery service (see Service Definition).

Presentations MUST be encoded in JWT format as string ([JWT]).

The protocol identifies the server and client roles. A client can be a server and vice versa. The individual operations define what to do in certain cases.

### 3.1 Registration

Clients can publish a Verifiable Presentation to the server by sending it in an HTTP POST to the service endpoint.
The HTTP request body MUST be a Verifiable Presentation (in JWT format). The content type of the request MUST be `application/json`.

The server MUST validate the Verifiable Presentation as specified in section 4. If the validation fails, it MUST return a 400 Bad Request response.
If the validation succeeds, the Verifiable Presentation MUST be added to the list and the server MUST return a 201 Created response.
If an implementation is configured as client and receives a registration, it MUST send the presentation to the server it has configured.

A credential subject (identified by `credentialSubject.id`) MUST NOT appear more than once on the list,
so a new registration MUST replace the previous one from the same credential subject.
If two presentations have the same issuance date, the server MUST keep the one with the highest byte value of the sha256 of the presentation. 

An example posting a Verifiable Presentation in JWT format to the list:

``http request
POST /list HTTP/1.1
Content-Type: application/json

"eyCAFE.etc.etc"
``

The Verifiable Presentation MUST NOT be valid longer than the Verifiable Credentials it contains. 

### 3.2 Reading

Clients MUST read the list by sending an HTTP GET to the server's service endpoint.
The server MUST return a 200 OK response with a JSON object containing:
- `entries` REQUIRED. MUST contain a JSON object with a mapping of timestamp (as string) to presentation.
- `timestamp` REQUIRED. MUST be a JSON integer equal to the timestamp of the last presentation.

The timestamp definition used here is a lamport clock, which is a monotonically increasing integer.

The `timestamp` query parameter MAY be used by the client to request a delta next time it reads the list.
The server SHOULD return the presentations with a timestamp greater than the provided value.
The server MUST return the latest `timestamp` value based on the value of the last presentation on the list.
If the query parameter is not provided, the server MUST return the full list.
If based on the timestamp the server has no new presentations it returns an empty list for the `entries` object.
Servers MUST only return the latest (valid) presentation per credential subject.

Example:

``http request
GET /list?timestamp=6 HTTP/1.1
Content-Type: application/json

{
  "entries": {
    "7": "ey1234..."
  }
  "timestamp": 7
}
``

Clients MUST validate each presentation in the list as specified in section 4.
If a presentation is valid, the client uses it in its system. If a presentation is not valid, it MUST be rejected.
If one or more presentations are not valid, it SHOULD NOT reject the other presentations.
Clients SHOULD use the `timestamp` value in the return object for their next call.

### 3.3 List pruning

To only keep valid presentations on the list, servers and clients MUST remove presentations whose `exp` (expiration) time has passed from the list.

Clients that wish to retain their presence on the list MUST submit a new registration before the current entry's `exp` time has passed (once a day, for instance).

### 3.4 Retracting presentations

Clients can remove presentations from the list, e.g. when a organization stops supporting the use case.
To remove presentations of a credential subject the client MUST register a Verifiable Presentation as specified in section 3.1,
with the following additional requirements:

- it MUST specify `RetractedVerifiablePresentation` as type, in addition to the `VerifiablePresentation`.
- it MUST contain a `retract_jti` JWT claim, containing the `jti` of the presentation to retract.
- it MUST NOT contain any credentials.

If a server receives a retraction that references an unknown presentation it MUST respond with a 400 Bad Request response.
The response MUST be a JSON error response describing the error.

Clients processing a retraction entry MUST remove the presentation indicated by `retract_jti`.
The expiration of the retraction presentation MUST be equal to the expiration of the presentation to retract.

### 3.5 Error responses

If a server encounters an error while processing a request, it MUST respond with a JSON response containing an `error` property, describing the error.

## 4. Presentation processing

To process a presentation, the following validation steps MUST be performed:

- `jti` (JWT ID) of the presentation MUST be a non-empty string.
- `nbf` (not before) of the presentation MUST have passed.
- `exp` (expiration) of the presentation MUST NOT have passed.
- `exp` MUST be after `nbf`.
- `aud` MUST contain the ID of the service.
- the number of seconds between `nbf` and `exp` MUST NOT exceed `presentation_max_validity` (see Service Definition).
- all credential issuers MUST be trusted (see section 5). 
- all credentials MUST have the same `credentialSubject.id`.
- `exp` (expiration) of the presentation MUST NOT be after the expiration date of the credentials.
- the key used to sign the presentation MUST be owned by the credential subject (see 4.1):
  the JWT `kid` header MUST reference an `assertionMethod` key from the subject's DID document.
- the credentials MUST conform to the Presentation Definition associated with the list (see Service Definition).
  The Verifiable Presentation MUST NOT contain other Verifiable Credentials than required by the Presentation Definition.

If a validation step fails, the presentation MUST be rejected.

JWT credential and presentation encoding as specified by [VC-DATA-MODEL] MUST be applied.
A clock skew of 5 seconds MAY be applied to `nbf` and `exp` claims.

### 4.1 Supported formats

The `did:web` DID method ([DID]) MUST be supported. Support for other methods is optional.
Verifiable Presentations and Verifiable Credentials MUST be in JWT format.

## 5. Service Definition

Servers MUST share a JSON document describing the discovery service. This document is known as the _service definition_.
The document MUST contain the following properties:
- `id` REQUIRED. String containing a value that identifies the service definition. MAY be used to version the definition.
- `endpoint` REQUIRED. String containing the URL where the server hosts the service endpoint.
- `presentation_definition` REQUIRED. JSON object with the Presentation Definition (see [PE]) describing requirements for presentations on the list.
- `presentation_max_validity` REQUIRED. JSON number containing the maximum validity period (number of seconds between `nbf` and `exp`) of a presentation in seconds.

For example:

``json
{
  "id": "uc_university_v1",
  "endpoint": "https://example.com/usecase/university/v1",
  "presentation_max_validity": 259200,
  "presentation_definition": {
    "id": "pd_university",
    "input_descriptors": [
      {
        "id": "pd_university_type",
        "constraints": {
          "fields": [
            {
              "path": ["$.type"],
              "filter": {
                "type": "string",
                "const": "UniversityCredential"
              }
            },
            {
              "path": "$.credentialSubject.name",
              "filter": {
                "type": "string"
              }
            }
          ]
        }
      }
    ]
  }
}
``

## 6. Trust

Trust of credential issuers (e.g. `did:example:education-accredetor` issuing `EducationalInstitutionCredential`) SHOULD be defined by the presentation definition.
In this case, there should be 2 constraints: one for the type and one for the issuer:

``json
[
  {
    "path": ["$.type"],
    "filter": {
      "type": "string",
      "const": "EducationalInstitutionCredential"
    }
  },
  {
    "path": "$.issuer",
    "filter": {
      "type": "string",
      "const": "did:example:education-accredetor"
    }
  }
]
``

## Appendix A - Design Decisions

### Supporting multiple lists

This RFC does not specify nested or multiple lists as this can be achieved by hosting a separate list on an alternate HTTP endpoint.

### Using deltas to reduce indexing overhead

When only the full list is available, clients need to validate and re-index (for fulltext search) all presentations.
By having the server return only new presentations instead, clients can only process those new entries.

### Timestamp implementation

The timestamp is specified as an integer. It should be implemented as a [Lamport timestamp](https://en.wikipedia.org/wiki/Lamport_timestamp) which is incremented for every presentation received.
That way, timestamps unambiguously reference the last presentation the client received.

A Unix timestamp does not offer this property, as it is possible that multiple presentations are received at the second,
possibly serving clients duplicate presentations.

A strict implementation requirement for a Lamport timestamp is that the client will have the same timestamp as the server.
This makes it possible for the client to become the server and vice versa. Changing the server still requires a well orchestrated migration.

## References

* [JWT] M. Jones, J. Bradley, N. Sakimura, "JSON Web Token (JWT)", RFC 7519, May 2015, <https://www.rfc-editor.org/info/rfc7519>.
* [VC-DATA-MODEL] M. Sporny, D. Longley, D. Chadwick, "Verifiable Credentials Data Model 1.1", W3C Recommendation, 3 March 2022, <https://www.w3.org/TR/vc-data-model/>.
* [PE] D. Buchner, B. Zundel, M. Riedel, K.H. Duffy, "Presentation Exchange 2.0.0", 12 September 2023, <https://identity.foundation/presentation-exchange/spec/v2.0.0/>.
* [DID] M. Sporny, D. Longley, M. Sabadello, D. Reed, O. Steele, C. Allen, "Decentralized Identifiers (DIDs) v1.0", W3C Recommendation, 19 July 2022, <https://www.w3.org/TR/did-core/>.