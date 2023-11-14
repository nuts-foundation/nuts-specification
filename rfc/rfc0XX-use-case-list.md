# RFC0XX Use Case List

|                          |           |
|:-------------------------|:----------|
| Nuts foundation          | R.G. Krul |
| Request for Comments: 0  | Nedap     |
| Amends:                  |           |

## Use Case List

### Abstract

### Status of document

This document is currently in draft.

### Copyright Notice

![](../.gitbook/assets/license.png)

This document is released under the [Attribution-ShareAlike 4.0 International \(CC BY-SA 4.0\) license](https://creativecommons.org/licenses/by-sa/4.0/).

## 1.  Introduction

## 2. Terminology

* **Verifiable Credential**: Verifiable Credential according to the [Verifiable Credential Data Model](https://www.w3.org/TR/vc-data-model/).
* **Verifiable Presentation**: Verifiable Presentation according to the [Verifiable Credential Data Model](https://www.w3.org/TR/vc-data-model/).
* **Maintainer**: party or system hosting the list and accepting new list registrations.
* **Client**: party or system reading the list and using it to execute use cases.

## 3. List endpoint

The list endpoint is an HTTP endpoint exposed by the list maintainer.
It can be read in full, or only the Verifiable Presentations that changed after the last time it was read.
It can be updated by sending a Verifiable Presentation to it.
All presentations on the list MUST conform to the Presentation Definition associated with the list (see Use Case List Definition).

Presentations MUST be encoded in JWT format as JSON string.

### 3.2 Registering on the list

Clients can register a Verifiable Presentation on the list by sending an HTTP POST to the maintainer's list endpoint.
The HTTP request body MUST be a Verifiable Presentation (in JWT format). The content type of the request MUST be `application/json`.

The maintainer MUST validate the Verifiable Presentation as specified in section 4. If the validation fails, it MUST return a 400 Bad Request response.
If the validation succeeds, the Verifiable Presentation MUST be added to the list and the maintainer MUST return a 201 Created response.
The presenter (meaning the credential holder, identified by `credentialSubject.id`) MUST NOT appear more than once on the list,
so a new registration MUST replace the previous one from the same presenter. 

An example posting a Verifiable Presentation in JWT format to the list:

```http request
POST /list HTTP/1.1
Content-Type: application/json

"eyCAFE.etc.etc"
```

The Verifiable Presentation MUST NOT be valid longer than the Verifiable Credentials it contains. 

### 3.3 Reading the list

Clients MUST load the list by sending an HTTP GET to the maintainer's list endpoint.
The maintainer MUST return a 200 OK response with a JSON object containing:
- `entries` REQUIRED. MUST contain a JSON array with zero or more presentations.
- `tombstones` REQUIRED. MUST contain a JSON array with zero or more presentation IDs that have been removed from the list.
- `timestamp` REQUIRED. MUST be a JSON number containing the Lamport timestamp of the last entry.

The `timestamp` query parameter MAY be used by the client to request a delta next time it reads the list.
If no `timestamp` query parameter is provided, the maintainer MUST return the full list.
Regardless whether the full list or a delta is returned, credential subjects MUST be unique.
Maintainers MUST only return the latest presentation for each credential subject.

Example:

```http request
GET /list?timestamp=510 HTTP/1.1
Content-Type: application/json

{
  "entries": [
    "ey1234.etc.etc"
    "ey5678.etc.etc"
  ],
  "tombstones": [
      "did:web:example.com#1",
      "did:web:example.com#2"
  ]
  "timestamp": 515,
}
```

Clients MUST validate each presentation in the list as specified in section 4.
If a presentation is valid, the client use it in its system. If a presentation is not valid, it MUST be rejected.
If one or more presentations are not valid, it SHOULD NOT reject the other presentations.

### 3.4 List pruning

To only keep valid presentations on the list, maintainers MUST remove presentations which `exp` (expiration) has passed from the list.

Clients that wish to retain their presentation MUST re-submit their registration before that period of time has passed, if they want to keep their presentation on the list (once a day, for instance).

Maintainers SHOULD remove presentations which contains credentials that have been revoked.

### 3.5. Removing presentations

Clients can remove presentations from a specific credential subject from a list, e.g. when a care organization stops supporting the use case.
To remove presentations of a credential subject the client MUST send an HTTP DELETE request to the list endpoint,
where the body contains an empty Verifiable Presentation signed by the credential subject.
Any credentials in the presentation MUST be ignored by the maintainer.

If the signing key (identified by `kid`) and signature are valid, the maintainer MUST remove all presentations of the credential subject from the list.
The maintainer MUST add the removed presentations to the tombstone set.
If the request was valid (regardless whether there were any presentations), the maintainer MUST respond with a `201 No Content` response.

Presentations that have expired SHOULD be removed from the tombstone set by the maintainer, to keep it as small as possible.

## 4. Presentation processing

To process a presentation, the following validation steps MUST be performed:

- ``jti`` (JWT ID) of the presentation MUST be a non-empty JSON string.
- ``nbf`` (not before) of the presentation MUST have passed.
- ``exp`` (expiration) of the presentation MUST NOT have passed.
- ``exp`` MUST be after ``nbf``.
- all credential issuers MUST be trusted (see section 5). 
- all credentials MUST have the same `credentialSubject.id`.
- ``exp`` (expiration) of the presentation MUST NOT be after the expiration date of the credentials.
- the key used to sign the presentation MUST be owned by the credential subject (see 4.1):
  the JWT ``kid`` header MUST reference an `assertionMethod` key from the subject's DID document.
- the credentials MUST conform to the Presentation Definition associated with the list (see Use Case List Definition).
  The Verifiable Presentation MUST NOT contain other Verifiable Credentials than required by the Presentation Definition.

If a validation step fails, the presentation MUST be rejected.

JWT credential and presentation encoding as specified by [VC data model](https://www.w3.org/TR/vc-data-model/#jwt-decoding) MUST be applied.

### 4.1 Supported formats

The ``did:web`` DID method MUST be supported. Support for other methods is optional.
Verifiable Presentations and Verifiable Credentials MUST be in JWT format.

## 5. Use Case List Definition

Maintainers MUST share a JSON document describing the list. This document is known as the _list definition_.
The document MUST contain the following properties:
- `id` REQUIRED. JSON string containing a value identifies the list. MAY be used to version the definition.
- `endpoint` REQUIRED. JSON string containing the URL where the maintainer serves the list.
- `presentation_definition` REQUIRED. JSON object with the Presentation Definition (see [Presentation Exchange](https://identity.foundation/presentation-exchange/#presentation-definition)) describing requirements for presentations on the list.
- `presentation_max_validity` REQUIRED. JSON number containing the maximum validity period (number of seconds between `nbf` and `exp`) of a presentation in seconds.

For example:

```json
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
```

## 6. Trust

Trust of credential issuers (e.g. `did:example:education-accredetor` issuing `EducationalInstitutionCredential`) SHOULD be defined by the presentation definition.
In this case, there should be 2 constraints: one for the type and one for the issuer:

```json
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
```

## Appendix A - Design Decisions

### Supporting multiple lists

This RFC does not specify nested or multiple lists as this can be achieved by hosting a separate list on an alternate HTTP endpoint.

### Using deltas to reduce indexing overhead

When only the full list is available, clients need to validate and re-index (for fulltext search) all presentations.
By having the maintainer return only new presentations instead, clients can only process those new entries.
