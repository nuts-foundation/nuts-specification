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

* **VerifiableCredential**: Verifiable Credential according to the [Verifiable Credential Data Model](https://www.w3.org/TR/vc-data-model/).
* **VerifiablePresentation**: Verifiable Presentation according to the [Verifiable Credential Data Model](https://www.w3.org/TR/vc-data-model/).
* **Maintainer**: party or system hosting the list and accepting new list registrations.
* **Client**: party or system reading the list and using it to execute use cases.

## 3. List endpoint

The list endpoint is an HTTP endpoint exposed by the list maintainer.
It acts like a REST resource, which can be read like a list and updated to by sending a Verifiable Presentation to it.

### 3.1 Format

The list is a JSON array containing zero or more Verifiable Presentations.
List entries can be a:

- presentation in JWT format as JSON string.
- presentation in JSON-LD format as JSON object.

### 3.2 Registering on the list

Clients can register a Verifiable Presentation on the list by sending an HTTP POST to the maintainer's list endpoint.
The HTTP request body MUST be a Verifiable Presentation encoded as JSON. The content type of the request MUST be `application/json`.

The maintainer MUST validate the Verifiable Presentation as specified in section 4. If the validation fails, it MUST return a 400 Bad Request response.
If the validation succeeds, the Verifiable Presentation MUST be added to the list and the maintainer MUST return a 201 Created response.
Credential subjects on the list MUST be unique, so if the subject already exists, the new presentation replaces the old one. 

An example posting a Verifiable Presentation in JWT format to the list:

```http request
POST /list HTTP/1.1
Content-Type: application/json

"eyCAFE.etc.etc"
```

An example posting a Verifiable Presentation in JSON-LD format to the list:

```http request
POST /list HTTP/1.1
Content-Type: application/json

{
    (JSON-LD object)
}
```

### 3.3 Reading the list

Clients MUST load the list by sending an HTTP GET to the maintainer's list endpoint.
The maintainer MUST return a 200 OK response with the list as the HTTP response body.

Clients MUST validate each presentation in the list as specified in section 4.
If a presentation is valid, the client use it in its system. If a presentation is not valid, it MUST be rejected.
If one or more presentations are not valid, it SHOULD NOT reject the other presentations.

### 3.4 List pruning

To only keep relevant presentations on the list, maintainers MUST remove entries after a specified period of time.
For instance, a presentations could be removed a week after they were added.

Clients MUST re-submit their registration before that period of time has passed, if they want to keep their presentation on the list (once a day, for instance).

Maintainers MUST remove presentations which ``expirationDate`` has passed from the list.

### 3.5 Caching

Since processing the list becomes more costly as it grows, HTTP cache headers MUST be used to avoid reprocessing unchanged lists.
Clients SHOULD add the ``If-Modified-Since`` header to the list request, specifying the value of the ``Modified-Since`` header of the last successful HTTP response. 
Maintainers MUST add the ``Modified-Since`` header to HTTP their responses, specifying when the list was last updated (presentation added or removed).
If the list is requested, but nothing changed after ``If-Modified-Since``, the maintainer MUST return a 304 Not Modified response.

## 4. Presentation validation

To process a presentation, the following validation steps MUST be performed:

- ``issuanceDate`` of the presentation MUST have passed.
- ``expirationDate`` of the presentation MUST NOT have passed.
- ``expirationDate`` MUST be after ``issuanceDate``.
- all credential issuers MUST be trusted (see section 5). 
- all credentials subjects MUST be the same DID.
- the key used to sign the presentation MUST be owned by the credential subject (see 4.1).

If a validation step fails, the presentation MUST be rejected.

JWT credential and presentation encoding as specified by [VC data model](https://www.w3.org/TR/vc-data-model/#jwt-decoding) MUST be applied.
For instance, ``issuanceDate`` might come from the JWT ``nbf`` claim or the JSON-LD ``issuanceDate`` property.

### 4.1 Signing key validation

Validators MUST check that the key used to sign the presentation is owned by the credential subject.
To validate the key, resolve the DID of the credential subject.
The signing key MUST be listed in the ``assertionMethod`` property of the DID document.

For JWT presentations, the ``kid`` claim MUST be used specify the key used to sign the presentation.
For JSON-LD presentations, the ``verificationMethod`` property  of the ``proof`` object MUST be used to specify the key used to sign the presentation.

### 4.2 Supported formats

The ``did:web`` DID method MUST be supported. Support for other methods is optional.
Presentations and credentials MUST be either in JWT or JSON-LD format.

## 5. Trust

Validators and clients SHOULD share a list of credential issuers they trust.
Issuers MUST be trusted for specific a credential types.
For instance, a driver's license governing body should only be trusted for issuing credentials with type ``DriversLicense``.

## Appendix A - Design Decisions

### Supporting multiple lists

This RFC does not specify nested or multiple lists as this can be achieved by hosting a separate list on an alternate HTTP endpoint.