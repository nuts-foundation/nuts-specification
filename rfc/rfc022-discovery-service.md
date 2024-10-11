# RFC022 Discovery Service

|                           |                |
|:--------------------------|:---------------|
| Nuts foundation           | R.G. Krul      |
| Request for Comments: 022 | Nedap          |
|                           | W.M. Slakhorst |
|                           | October 2024   |

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
clients can verify the authenticity and integrity of the Verifiable Presentations and Verifiable Credentials.

## 2. Terminology

* **Discovery Service**: collection of Verifiable Presentations that allows discovery of systems/parties that offer a particular service.
* **Discovery Definition**: document describing the Discovery Service.
* **Server**: system hosting the discovery service and accepting new registrations.
* **Client**: system registering presentations on the list and reading it to discover other systems/parties.
* **List**: refers to the list of Verifiable Presentations hosted by the server for a specific service.

Other terms are as defined in the following specifications: "JSON Web Token (JWT)" [JWT], Verifiable Credentials Data Model 1.1 [VC-DATA-MODEL], Verifiable Credentials Presentation Exchange [PEX], Decentralized Identifiers 1.0 [DID].

## 3. Service endpoint

The service endpoint is an HTTP endpoint exposed by the server which hosts a list of Verifiable Presentations.
Clients can query the list or submit a new Verifiable Presentation to it.
All presentations MUST conform to the Presentation Definition associated with the discovery service (see Service Definition).

Presentations MUST be encoded in JWT format as string ([JWT]).

The protocol identifies the server and client roles. A client can be a server and vice versa. The individual operations define what to do in certain cases.

Each list has a unique `seed` value. The seed value is used by a server to identify a specific instance of a list.
This is used to prevent client _locking_ based on timestamps and enables switching between servers for the same service.

### 3.1 Registration

Clients can publish a Verifiable Presentation to the server by sending it in an HTTP POST to the service endpoint.
The HTTP request body MUST be a Verifiable Presentation (in JWT format). The content type of the request MUST be `application/json`.

The server MUST validate the Verifiable Presentation as specified in section 4. If the validation fails, it MUST return a 400 Bad Request response.
If the validation succeeds, the Verifiable Presentation MUST be added to the list and the server MUST return a 201 Created response.
If an implementation is configured as client and receives a registration, it MUST send the presentation to the server it has configured.

A credential subject (identified by `credentialSubject.id`) MUST NOT appear more than once on the list,
so a new registration MUST replace the previous one from the same credential subject.
The server MUST assign a timestamp to each new registration. It MUST not assign the same timestamp more than once.
The timestamp definition used here is a lamport clock, which is a monotonically increasing integer.

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
- `seed` REQUIRED. MUST be a JSON string containing the seed value of the list.
- `entries` REQUIRED. MUST contain a JSON object with a mapping of timestamp (as string) to presentation.
- `timestamp` REQUIRED. MUST be a JSON integer equal to the timestamp of the last presentation.

The `timestamp` query parameter MAY be used by the client to request a delta next time it reads the list.
The server SHOULD return the presentations with a timestamp greater than the provided value.
The server MUST return the latest `timestamp` value based on the value of the last presentation on the list.
If the query parameter is not provided, the server MUST return the full list.
If based on the timestamp the server has no new presentations it returns an empty list for the `entries` object.
Servers MUST only return the latest (valid) presentation per credential subject.
If the `seed` value has changed, the client MUST discard all entries and query the entire list again.

Example:

``http request
GET /list?timestamp=6 HTTP/1.1
Content-Type: application/json

{
  "seed": "1234-5678-90ab-cdef",
  "entries": {
    "7": "ey1234..."
  }
  "timestamp": 7
}
``

Clients MUST validate each presentation in the list as specified in section 4.
If a presentation is not valid, it MUST be rejected.
If one or more presentations are not valid, it SHOULD NOT reject the other presentations.
If new presentations contain an already known credential subject (identified by `credentialSubject.id`), 
the client MUST replace the old presentation with the new one if the timestamp of the new presentation is greater than the timestamp of the old presentation.
Clients SHOULD use the `timestamp` value in the return object for their next call.

### 3.3 List pruning

To only keep valid presentations on the list, servers and clients MUST remove presentations whose `exp` (expiration) time has passed from the list.

Clients that wish to retain their presence on the list MUST submit a new registration before the current entry's `exp` time has passed.

### 3.4 Retracting presentations

Clients can remove presentations from the list, e.g. when a organization stops supporting the use case.
To remove presentations of a credential subject the client MUST register a Verifiable Presentation as specified in section 3.1,
with the following additional requirements:

- it MUST specify `RetractedVerifiablePresentation` as type, in addition to the `VerifiablePresentation`.
- it MUST contain a `retract_jti` JWT claim, containing the `jti` of the presentation to retract.
- it MUST NOT contain any credentials.

If a server receives a retraction that references an unknown presentation it MUST respond with a 400 Bad Request response.
The response MUST be a JSON error response (§3.5) describing the error.

Clients processing a retraction entry MUST remove the presentation indicated by `retract_jti`.
The expiration of the retraction presentation MUST be equal to the expiration of the presentation to retract.

### 3.5 Error responses

If a server encounters an error while processing a request, it MUST respond with a JSON response according to [RFC7807].

## 4. Presentation processing

The abbreviations below are conform definitions by [IANA](https://www.iana.org/assignments/jwt/jwt.xhtml#claims). 
To process a presentation, the following validation steps MUST be performed:

- `jti` of the presentation MUST be a non-empty string.
- `nbf` of the presentation MUST have passed.
- `exp` of the presentation MUST NOT have passed.
- `exp` MUST be after `nbf`.
- `aud` MUST contain the ID of the service.
- the number of seconds between `nbf` and `exp` MUST NOT exceed `presentation_max_validity` (see Service Definition).
- all credentials MUST have the same `credentialSubject.id`.
- `exp` of the presentation MUST NOT be after the expiration date of the credentials.
- the key used to sign the presentation MUST be owned by the credential subject (see 4.1):
  the JWT `kid` header MUST reference an `assertionMethod` key from the subject's DID document.
- The DID of the credential subject MUST be of a DID method defined in the service definition.
- the presentation MAY contain a `DiscoveryRegistrationCredential` credential defined by the Nuts JSON-LD context (https://nuts.nl/credentials/v1).
  If present, it MUST NOT be included in the next validation steps unless specifically referenced in the Presentation Definition.
- the credentials MUST conform to the Presentation Definition associated with the list (see Service Definition).
  The Verifiable Presentation MUST NOT contain any Verifiable Credentials besides the ones that conform to the Presentation Definition and the `DiscoveryRegistrationCredential`.

If a validation step fails, the presentation MUST be rejected.

JWT presentation encoding as specified by [VC-DATA-MODEL] MUST be applied.
A clock skew of 5 seconds MAY be applied to `nbf` and `exp` claims.

### 4.1 DiscoveryRegistrationCredential

The optional `DiscoveryRegistrationCredential` is used to add additional parameters to the registration.
It's a _self-asserted_ credential, meaning it has been issued by the holder itself and doesn't contain a proof. 
Fields under `credentialSubject` (except `id`) can be used by clients depending on the use case.
The credential has the following requirements:
- `type` MUST contain `DiscoveryRegistrationCredential`.
- `@context` MUST contain the Nuts JSON-LD context (https://nuts.nl/credentials/v1).
- `credentialSubject` MUST contain a JSON object with additional parameters.
- `credentialSubject.id` MUST be the same as the `issuer` of the credential.
- `id` MUST be a unique identifier for the credential.
- `issuanceDate` MUST be a valid date.

## 5. Service Definition

Servers MUST share a JSON document describing the discovery service. This document is known as the _service definition_.
The document MUST contain the following properties:
- `id` REQUIRED. String containing a value that identifies the service definition. MAY be used to version the definition.
- `did_methods` OPTIONAL. Array of strings containing the DID methods enabled for the service. If not present, all DID methods are allowed.
- `endpoint` REQUIRED. String containing the URL where the server hosts the service endpoint.
- `presentation_definition` REQUIRED. JSON object with the Presentation Definition (see [PEX]) describing requirements for presentations on the list.
  Contents of the `format` field are ignored for the Verifiable Presentation. The Verifiable Presentation is always encoded in JWT format.
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

Trust of credential issuers (e.g. `did:example:education-accreditor` issuing `EducationalInstitutionCredential`) SHOULD be defined by the presentation definition.
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
      "const": "did:example:education-accreditor"
    }
  }
]
``

## 7. Security Considerations

## 7.1 Denial of Service

Since the server is hosting a public endpoint, it should take the following measures to prevent denial-of-service attacks:
- rate limiting on the number of requests per client.
- rate limiting on the number of requests per second.
- limit the size of the list.
- limit the size of the presentations that can be uploaded.

Clients should also take measures when the server has been compromised and is hosting malicious content:
- limit the number of presentations to process.
- limit the size of the presentations to process.
- use streaming processing to prevent excessive memory usage.

## 7.2 Trust

Designers of services should write trust into the Presentation Definition of the Service Definition. This can be done by specifying specific issuers of credentials.

## 8. Privacy Considerations

All Verifiable Presentations are public and can be read by anyone. Clients should be aware of this when submitting presentations.

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
* [PEX] D. Buchner, B. Zundel, M. Riedel, K.H. Duffy, "Presentation Exchange 2.0.0", 12 September 2023, <https://identity.foundation/presentation-exchange/spec/v2.0.0/>.
* [DID] M. Sporny, D. Longley, M. Sabadello, D. Reed, O. Steele, C. Allen, "Decentralized Identifiers (DIDs) v1.0", W3C Recommendation, 19 July 2022, <https://www.w3.org/TR/did-core/>.
* [RFC7807] M. Nottingham, E. Wilde, "Problem Details for HTTP APIs", RFC 7807, March 2016, <https://www.rfc-editor.org/info/rfc7807>.