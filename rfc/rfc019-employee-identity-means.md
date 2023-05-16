# RFC019 Employee Identity Authentication Means

|                           |                 |
|:--------------------------|:----------------|
| Nuts foundation           | W.M. Slakhorst  |
| Request for Comments: 019 | Nedap           |
|                           | S. van der Vegt |
|                           | Nedap           |
|                           | April 2023      |

## Employee Identity Authentication Means

### Abstract

The NutsEmployeeIdentity method is an authentication _means_ which allows an organisation to assert the identity of the current user.
The organisation uses the information of the current user session to present the user with a statement about its identity and asks the user to confirm this.
After confirmation this bundle of information is signed with the organisation's private key.

This method makes it possible to assert a user identity without the need for a personal authentication means.
This puts the organisation in the role of identity issuer instead of a trusted third party.
A use-case can choose to allow this authentication under certain (low risk) circumstances which still require a person to be authenticated in order to comply with the applicable legislation.

This RFC is an addition to the means listed in [RFC002](./rfc002-authentication-token.md#7-supported-means)

### Status of document

This document is currently in draft.

### Copyright Notice

![](../.gitbook/assets/license.png)

This document is released under the [Attribution-ShareAlike 4.0 International \(CC BY-SA 4.0\) license](https://creativecommons.org/licenses/by-sa/4.0/).

## 1.  Introduction

Introducing new authentication means in an organizational setting can have quite an impact, even more when mainstream identity providers have not yet adjusted to new standards.
New feature or applications require a higher assurance level for authentication than the current situation.
This introduces an additional authentication step in the process of users. This often happens at a place in the workflow where they don't expect it.
This is a situation that remain until the main login is adapted to the newer/higher security standard.
This RFC bridges the gap so the software can already be changed to accept newer protocols and the user can be made aware of upcoming changes without going through an authentication step using additional means.

## 2. Terminology

* **Client application**: The application that requires access.
* **Node**: A piece of software implementing the Nuts RFCs.
* **Resource server**: The application \(a protected resource\) that requires authorized access to its API’s.
* **VerifiableCredential**: Verifiable Credential according to the [Verifiable Credential Data Model](https://www.w3.org/TR/vc-data-model/).
* **VerifiablePresentation**: Verifiable Presentation according to the [Verifiable Credential Data Model](https://www.w3.org/TR/vc-data-model/).

## 3. Nuts Employee Identity

The authentication flow is as follows:

- a user requests access to a resource that is hosted at a *resource server* and requires additional authentication.
- the *client application* starts a session on the *node* with the current user data.
- the user is directed to a web page where it accepts the given conditions by pressing a button.
- this button sends the confirmation to the *node* for the given session.
- upon receiving the request, the node will issue a `NutsEmployeeCredential` from and to the organization DID associated with the existing user session.
- the credential is used to create a VerifiablePresentation.
- the VerifiablePresentation is used in the access token request as defined in [RFC003](./rfc003-oauth2-authorization.md#422-payload).

### 3.1 User data

In order for the resource server to comply to local legislation the following data MUST be added to the credential:

- **Initials**: The initials of the user.
- **Family name**: The family name of the user.
- **Identifier**: A unique identifier for the user within the context of the organization. This may be an email address or an employee number. 

The user data MAY also contain a `roleName` that contains the user role within the organization.

### 3.2 HTML based challenge

After a session has been started by the node, the client application MUST present the user a web page that is hosted by the node.
The node MUST make sure that the web page is linked to the session and that the give user data can not be altered.
The web page MUST present the user with a contract as stated in [RFC002](./rfc002-authentication-token.md#5-login-contract) and show which user data will be exposed to the resource server. 
The web page MUST also provide a checkbox that requires the user to acknowledge the challenge and confirm the data to be shared.
The web page MUST present a button that allows the user to submit the response to the node via an HTTP POST call.
The session identifier used to link the user response to the session MUST be a random secure token with a minimum length of 16 bytes.
The token lifetime MUST NOT be longer than 15 minutes.

### 3.3 VerifiableCredential

A `NutsEmployeeCredential` is issued after the user submitted the response. This credential MUST only be used to create the verifiable presentation. 
Below is an example of a `NutsEmployeeCredential`. The proof has been omitted for brevity.

```json
{
  "@context": [
    "https://www.w3.org/2018/credentials/v1",
    "https://w3c-ccg.github.io/lds-jws2020/contexts/lds-jws2020-v1.json",
    "https://nuts.nl/credentials/v1"
  ],
  "id": "did:nuts:8NYzfsndZJHh6GqzKiSBpyERrFxuX64z6tE5raa7nEjm#dde77e76-7e3c-483f-a813-2b851a6a969c",
  "type": [
    "NutsEmployeeCredential",
    "VerifiableCredential"
  ],
  "issuer": "did:nuts:8NYzfsndZJHh6GqzKiSBpyERrFxuX64z6tE5raa7nEjm",
  "issuanceDate": "2023-04-20T08:52:45.941461+02:00",
  "expirationDate": "2023-04-21T08:52:45.941461+02:00",
  "credentialSubject": [
    {
      "id": "did:nuts:8NYzfsndZJHh6GqzKiSBpyERrFxuX64z6tE5raa7nEjm",
      "member": {
        "identifier": "user@example.com",
        "member": {
          "familyName": "Tester",
          "initials": "T",
          "type": "Person"
        },
        "roleName": "Verpleegkundige niveau 2",
        "type": "EmployeeRole"
      },
      "type": "Organization"
    }
  ],
  "proof": [{...}]
}
```

The credential has the following requirements:

- The type MUST contain `NutsEmployeeCredential` in addition to the default verifiable credential type.
- The `@context` MUST contain `https://nuts.nl/credentials/v1` and `https://w3c-ccg.github.io/lds-jws2020/contexts/lds-jws2020-v1.json` in addition to the default context.
- The `issuer` and `credentialSubject.id` MUST be the same.
- The `expirationDate` MUST NOT be more than one day after the `issuanceDate`.
- `credentialSubject.type` MUST be `Organization`.
- `credentialSubject.member.type` MUST be `EmployeeRole`.
- `credentialSubject.member.member.type` MUST be `Person`.
- `credentialSubject.member.identifier` MUST contain the user identifier.
- `credentialSubject.member.member.initials` MUST contain the user initials.
- `credentialSubject.member.member.familyName` MUST contain the user family name.
- `credentialSubject.member.roleName` is optional and MAY contain the user role.
- The `proof.type` MUST be a `JsonWebSignature2020`.
- The `proof.proofPurpose` MUST be `assertionMethod`.

### 3.4 VerifiablePresentation

The node MUST create a verifiable presentation with the newly created credential and return it to the client application.
Below is an example of a `NutsSelfSignedPresentation`. The credential has been omitted for brevity.

```json
{
  "@context": [
    "https://w3c-ccg.github.io/lds-jws2020/contexts/lds-jws2020-v1.json",
    "https://nuts.nl/credentials/v1",
    "https://www.w3.org/2018/credentials/v1"
  ],
  "type": [
    "VerifiablePresentation",
    "NutsSelfSignedPresentation"
  ],
  "verifiableCredential": [
    {...}
  ],
  "proof": [
    {
      "challenge": "EN:PractitionerLogin:v3 I hereby declare to act on behalf of CareBears located in Caretown. This declaration is valid from Wednesday, 19 April 2023 12:20:00 until Thursday, 20 April 2023 13:20:00.",
      "created": "2023-04-20T09:53:03.783007+02:00",
      "expires": "2023-04-20T13:20:00+02:00",
      "jws": "eyJhbGciOiJFUzI1NiIsImI2NCI6ZmFsc2UsImNyaXQiOlsiYjY0Il0sImtpZCI6ImRpZDpudXRzOjhOWXpmc25kWkpIaDZHcXpLaVNCcHlFUnJGeHVYNjR6NnRFNXJhYTduRWptI2JZY3VldDZFSG9qTWxhTXF3Tm9DM2M2ZXRLbFVIb0o5clJ2VXUzWktFRXcifQ..5NvfnxxZ9QT11NAYgqHybThR4Ygcd8jXjAsNy8lko3uajC-v6YiJw18a4qJEqZ7xYg3b-aBYt7UbXFbMqVlXFA",
      "proofPurpose": "authentication",
      "type": "JsonWebSignature2020",
      "verificationMethod": "did:nuts:8NYzfsndZJHh6GqzKiSBpyERrFxuX64z6tE5raa7nEjm#bYcuet6EHojMlaMqwNoC3c6etKlUHoJ9rRvUu3ZKEEw"
    }
  ]
}
```

The presentation has the following requirements:

- The type MUST contain `NutsSelfSignedPresentation` in addition to the default verifiable presentation type.
- The `@context` MUST contain `https://nuts.nl/credentials/v1` and `https://w3c-ccg.github.io/lds-jws2020/contexts/lds-jws2020-v1.json` in addition to the default context.
- The `verifiableCredential` field MUST contain exactly one credential as described in the previous paragraph.
- The `challenge` MUST contain a contract as specified by [RFC002](./rfc002-authentication-token.md#5-login-contract).
- `proof.expires` MUST be filled.
- The `proof.type` MUST be a `JsonWebSignature2020`.
- The `proof.proofPurpose` MUST be `authentication`.

## 4. Security considerations

The _means_ is designed in a way that it is interchangeable with (other) means with a higher trust level, (using a personal authentication device).
Even it is technically possible, this means MUST NOT be used to perform machine to machine interactions which are not initiated by the current user, e.g. background jobs, fetching data automatically etc.
By doing so, a use-case can not update the trust level of a means without braking functionality.
This means MUST only be used to claim the identity of actual and identifying natural persons working under the responsibility of the organisation, and MUST NOT identify e.g. shared accounts, teams, administrators, dummy or test users.