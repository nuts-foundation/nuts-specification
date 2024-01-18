# RFC0XX Discovery Service

|                          |           |
|:-------------------------|:----------|
| Nuts foundation          | R.G. Krul |
| Request for Comments: 0  | Nedap     |
| Amends:                  |           |

## Discovery Service

### Abstract

### Status of document

This document is currently in draft.

### Copyright Notice

![](../.gitbook/assets/license.png)

This document is released under the [Attribution-ShareAlike 4.0 International \(CC BY-SA 4.0\) license](https://creativecommons.org/licenses/by-sa/4.0/).

## 1.  Introduction

## 2. Terminology

* **Verifiable Credential**: cryptographically verifiable claim by a trusted third party over a subject, according to the [Verifiable Credential Data Model](https://www.w3.org/TR/vc-data-model/).
* **Credential subject**: entity the credential is about (e.g. organization, person, system or device). The credential subject is identified by the `id` property of the credential subject.
* **Verifiable Presentation**: cryptographically verifiable presentation of a credential signed by the subject, according to the [Verifiable Credential Data Model](https://www.w3.org/TR/vc-data-model/).
* **Presentation Definition**: format specifying what constraints a presentation must conform to, according to the [Presentation Exchange](https://identity.foundation/presentation-exchange/#presentation-definition).
* **Server**: system hosting the discovery service and accepting new registrations.
* **Client**: system registering presentations on the list and reading it to discover other systems/parties.

## 3. Service endpoint

The service endpoint is an HTTP endpoint exposed by the server which hosts a list of Verifiable Presentations.
Clients can query the list or submit a new Verifiable Presentation to it.
All presentations MUST conform to the Presentation Definition associated with the discovery service (see Service Definition).

Presentations MUST be encoded in JWT format as string.

### 3.1 Registration

Clients can publish a Verifiable Presentation to the list by sending it in an HTTP POST to the service endpoint.
The HTTP request body MUST be a Verifiable Presentation (in JWT format). The content type of the request MUST be `application/json`.

The server MUST validate the Verifiable Presentation as specified in section 4. If the validation fails, it MUST return a 400 Bad Request response.
If the validation succeeds, the Verifiable Presentation MUST be added to the list and the server MUST return a 201 Created response.

A credential subject (identified by `credentialSubject.id`) MUST NOT appear more than once on the list,
so a new registration MUST replace the previous one from the same credential subject.

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
- `entries` REQUIRED. MUST contain a JSON array with zero or more presentations.
- `tag` REQUIRED. MUST be a JSON string referring to the last entry. The tag is an opaque value; the client SHOULD NOT derive any meaning from it.

The `tag` query parameter MAY be used by the client to request a delta next time it reads the list.
If no `tag` query parameter is provided, the server MUST return the full list.
Servers MUST only return the latest (valid) presentation per credential subject.

Example:

``http request
GET /list?tag=510 HTTP/1.1
Content-Type: application/json

{
  "entries": [
    "ey1234.etc.etc",
    "ey5678.etc.etc"
  ],
  "tag": "515",
}
``

Clients MUST validate each presentation in the list as specified in section 4.
If a presentation is valid, the client uses it in its system. If a presentation is not valid, it MUST be rejected.
If one or more presentations are not valid, it SHOULD NOT reject the other presentations.

### 3.3 List pruning

To only keep valid presentations on the list, servers MUST remove presentations whose `exp` (expiration) time has passed from the list.

Clients that wish to retain their presence on the list MUST submit a new registration before the current entry's `exp` time has passed (once a day, for instance).

Servers SHOULD remove presentations which contain credentials that have been revoked.

### 3.4 Retracting presentations

Clients can remove presentations from the list, e.g. when a care organization stops supporting the use case.
To remove presentations of a credential subject the client MUST register a Verifiable Presentation as specified in section 3.1,
with the following additional requirements:

- it MUST specify `RetractedVerifiablePresentation` as type, in addition to the `VerifiablePresentation`.
- it MUST contain a `retract_jti` JWT claim, containing the `jti` of the presentation to retract.
- it MUST NOT contain any credentials.

If a server receives a retraction that references an unknown presentation it MUST respond with a 400 Bad Request response.
The response MUST be a JSON error response describing the error.

Clients processing a presentation retraction MUST remove the presentation indicated by `retract_jti`.

### 3.5 Error responses

If a server encounters an error while processing a request, it MUST respond with a JSON response containing an `error` property, describing the error.

## 4. Presentation processing

To process a presentation, the following validation steps MUST be performed:

- `jti` (JWT ID) of the presentation MUST be a non-empty string.
- the presentation ID  (`jti`) MUST NOT already exist on the list.
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

JWT credential and presentation encoding as specified by [VC data model](https://www.w3.org/TR/vc-data-model/#jwt-decoding) MUST be applied.

### 4.1 Supported formats

The `did:web` DID method MUST be supported. Support for other methods is optional.
Verifiable Presentations and Verifiable Credentials MUST be in JWT format.

## 5. Service Definition

Servers MUST share a JSON document describing the discovery service. This document is known as the _service definition_.
The document MUST contain the following properties:
- `id` REQUIRED. String containing a value that identifies the service definition. MAY be used to version the definition.
- `endpoint` REQUIRED. String containing the URL where the server hosts the service endpoint.
- `presentation_definition` REQUIRED. JSON object with the Presentation Definition (see [Presentation Exchange](https://identity.foundation/presentation-exchange/#presentation-definition)) describing requirements for presentations on the list.
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
In this case, there should be 2 constraints in the input descriptor object: one for the type and one for the issuer:

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

### Tag implementation

The tag is specified as an opaque value (JSON string). It is up to server to decide how to implement this.

One way to implement this is to use a [Lamport timestamp](https://en.wikipedia.org/wiki/Lamport_timestamp) which is incremented for every presentation received.
That way, timestamps unambiguously reference the last presentation the client received.

A Unix timestamp does not offer this property, as it is possible that multiple presentations are received at the second,
possibly serving clients duplicate presentations.