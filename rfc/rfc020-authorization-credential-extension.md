# RFC020 Authorization credential extension

|                           |                 |
|:--------------------------|:----------------|
| Nuts foundation           | W.M. Slakhorst  |
| Request for Comments: 020 | Nedap           |
| Amends: RFC014            | April 2023      |

## Authorization credential extension

### Abstract

A `assuranceLevel` field is added to the `NutsAuthorizationCredential`. It can be used within the context of a `resource` to indicate the required assurance level of the authentication.

This RFC is an addition to the means listed in [RFC014](./rfc014-authorization-credential.md)

### Status of document

This document is currently in draft.

### Copyright Notice

![](../.gitbook/assets/license.png)

This document is released under the [Attribution-ShareAlike 4.0 International \(CC BY-SA 4.0\) license](https://creativecommons.org/licenses/by-sa/4.0/).

## 1.  Introduction

A resource server should be able to provide information about the authentication assurance level that is used to access resources.
With the introduction of [RFC019](./rfc019-employee-identity-means.md) an authentication means with a low assurance level has been introduced.
This authentication means should not be used on resources that require a high assurance level. 
An additional field in the NutsAuthorizationCredential allows a resource server to indicate which level of assurance it requires.

## 2. Terminology

* **Authorization server**: The application that evaluates access token requests and creates access tokens.
* **Resource server**: The application that requires authorized access to its APIs.

## 3. AssurenceLevel field

The additional field is called `assuranceLevel`. It MUST contain one of the following values: `low`, `substantial` or `high`.
The field is optional. When present it COULD be used by the authorization server to verify the access token request. 
The field is located within a resource. A resource is located in the `resources` list.

The following example shows the location of the new field, other fields have been omitted for brevity:

```json
{
  ...
  "credentialSubject": {
    "id": "did:nuts:SjkuVHVqZndMVVJwcnUzbjhuZklhODB1M1M0LW9LcWY0WUs5S2",
    "resources": [
      {
        "path": "/DocumentReference/f2aeec97-fc0d-42bf-8ca7-0548192d4231",
        "operations": ["read"],
        "userContext": true,
        "assuranceLevel": "low"
      }
    ],
    "purposeOfUse": "test-service"
  },
  ...
}
```