# RFC021 VP Token Grant Type

|                           |                  |
|:--------------------------|:-----------------|
| Nuts foundation           | W.M. Slakhorst   |
| Request for Comments: 021 | Nedap            |
|                           | September 2023   |

## VP Token Grant Type

### Abstract

This specification defines the use of a Verifiable Presentation Bearer
Token as a means for requesting an OAuth 2.0 access token.

### Status of document

This document is currently in draft.

### Copyright Notice

![](../.gitbook/assets/license.png)

This document is released under the [Attribution-ShareAlike 4.0 International \(CC BY-SA 4.0\) license](https://creativecommons.org/licenses/by-sa/4.0/).

## 1.  Introduction

A Verifiable Presentation [VP] is an encoding that enables identity and security  information base  on Verifiable Credentials to be shared  across security domains. 
A security token is generally issued by an Identity Provider and consumed by a Relying Party that relies on its content to identify  the token's subject for security-related purposes. 
In the case of  Verifiable Credentials, the identity provider role is fulfilled by  the holder using a wallet. The relying party role equals the verifier  role as defined by the Verifiable Credentials specification.

The OAuth 2.0 Authorization Framework [RFC6749] provides a method for making authenticated HTTP requests to a resource using an access token. 
Access tokens are issued to third-party clients by an authorization server (AS) with the (sometimes implicit) approval of the resource owner. 
In OAuth, an authorization grant is an abstract term used to describe intermediate credentials that represent the resource owner authorization. 
An authorization grant is used by the client to obtain an access token.  Several authorization grant types are defined to support a wide range of client types and user experiences. 
OAuth also allows for the definition of new extension grant types to support additional clients or to provide a bridge between OAuth and other trust frameworks. 
Finally, OAuth allows the definition of additional authentication mechanisms to be used by clients when interacting with the authorization server.

"Assertion Framework for OAuth 2.0 Client Authentication and Authorization Grants" [RFC7521] is an abstract extension to OAuth 2.0 that provides a general framework for the use of assertions (a.k.a. security tokens) as client credentials and/or authorization grants with OAuth 2.0. 
This specification profiles the OAuth Assertion Framework [RFC7521] to define an extension grant type that uses a Verifiable Presentation Bearer Token to request an OAuth 2.0 access token.

## 2. Terminology

All terms are as defined in the following specifications: "The OAuth 2.0 Authorization Framework" [RFC6749], the OAuth Assertion Framework [RFC7521] "JSON Web Token (JWT)" [JWT], Verifiable Credentials Data Model 1.1 [VC-DATA-MODEL], Verifiable Credentials Presentation Exchange [PE], JSON Web Signature 2020 [JSONWebSignature2020], Decentralized Identifiers 1.0 [DID].

## 3. HTTP Parameter Bindings for Transporting Assertions

The OAuth Assertion Framework [RFC7521] defines generic HTTP parameters for transporting assertions (a.k.a. security tokens) during interactions with a token endpoint. 
This section defines specific parameters and treatments of those parameters for use with VP Bearer Tokens as Authorization Grants.

To use a VP as an authorization grant, the client uses an access token request as defined in Section 4 of the OAuth Assertion Framework [RFC7521] with the following specific parameter values and encodings.

The value of the "grant_type" is "urn:ietf:params:oauth:grant-type:vp_token-bearer".

The value of the "assertion" parameter MUST contain a Verifiable Presentation. The Verifiable Presentation MUST be encoded as either:

* The Verifiable Presentation is encoded as JSON using the application/x-www-form-urlencoded content type.
* The Verifiable Presentation is encoded as JWT according to section §6.3.1 of the Verifiable Credentials Data Model 1.1 [VC-DATA-MODEL].

The "scope" parameter MUST be used, as defined in the OAuth Assertion Framework [RFC7521], to indicate the requested scope. 
The scope parameter determines the set of credentials the authorization server expects.

The "presentation_submission" parameter MUST be used, as defined in Presentation Exchange [PE], to indicate how the VP matches the requested Verifiable Credentials.
The Presentation Submission MUST be encoded using the application/x-www-form-urlencoded content type.

The following example demonstrates an access token request with a VP as an authorization grant (with extra line breaks for display purposes only):

     POST /token.oauth2 HTTP/1.1
     Host: as.example.com
     Content-Type: application/x-www-form-urlencoded

     grant_type=urn%3Aietf%3Aparams%3Aoauth%3Agrant-type%3Avp_token-bearer
     &assertion=eyJhbGciOiJFUzI1NiIsImtpZCI6IjE2In0.
     eyJpc3Mi[...omitted for brevity...].
     J9l-ZhwP[...omitted for brevity...]
     &presentation_submission= {..}

## 3.1 Assertion Encoding

As stated in the previous section, the assertion parameter can either be encoded in JSON or JWT.
The authorization server determines the encoding by adding the correct parameters to the authorization server metadata [RFC8414]].
The following parameter MUST be added:

* `vp_formats`: An object defining the formats and proof types of Verifiable Presentations and Verifiable Credentials that a Verifier supports as stated by §9 of the OpenID for Verifiable Presentations specification [OIDC4VP].
  (ldp_vp or jwt_vp)

When processing the assertion the authorization server can detect the used format by checking the initial character.
A JSON encoded assertion starts with a '{' character or '%7B' when urlencoded.

## 4. Processing requirements

In order to issue an access token response as described in OAuth 2.0 [RFC6749], the authorization server MUST validate the VP.
The validation requirements are split into three paragraphs. The first paragraph describes the requirements that apply to all VP's.
The second paragraph describes the requirements that apply to JWT encoded VP's. The third paragraph describes the requirements that apply to JSON encoded VP's.

### 4.1 General requirements

1. The authorization server MUST validate the Presentation Submission according to section 6 of the Presentation Exchange [PE] specification.
2. The authorization server MUST validate the VP according to section 6.1 of the Presentation Exchange [PE] specification.
3. The Presentation Submission MUST be an answer to the Presentation Definition that corresponds with the requested scope. 
   How an authorization server determines the Presentation Definition based on scope is out of scope of this specification.
   However, a request to the Presentation Definition endpoint MUST result to the same Presentation Definition as the Presentation Definition that is used to validate the VP.
4. The Verifiable Presentation MUST be not be valid for more than 15 minutes.
5. A clock skew of no more than 5 seconds MAY be applied when validating the Verifiable Presentation.

### 4.2 JWT requirements

1. The assertion MUST be a valid JWT according to §6.3.1 of the Verifiable Credentials Data Model 1.1 [VC-DATA-MODEL].
2. The `vp` field of the JWT MUST be valid according to §4.10 the Verifiable Credentials Data Model 1.1 [VC-DATA-MODEL].
3. The `nonce` of the JWT MUST be a string that is unique for each request.
4. The `kid` field MUST be a DID URL.
5. The `iss` field MUST be a Decentralized Identifier [DID] and MUST correspond to the `kid`.
6. The `sub` field MUST match the `credentialSubject.ID` field from all the Verifiable Credentials that are used to request the access token.
7. The `aud` field MUST be a DID under control of the Authorization Server.

### 4.3 JSON requirements

1. The assertion MUST be a valid JSON object according to §4.10 the Verifiable Credentials Data Model 1.1 [VC-DATA-MODEL].
2. The proof of the VP MUST be a valid [JSONWebSignature2020] object.
3. The `challenge` field of the JSON object MUST be a string that is unique for each request.
4. The `domain` field of the JSON object MUST be a DID under control of the Authorization Server.
5. The `issuer` field of the JSON object MUST be a Decentralized Identifier [DID].
6. The `verificationMethod` field of the Proof MUST be a DID URL and correspond to the `issuer` field.
7. The `holder` field if present MUST match the `credentialSubject.ID` field from all the Verifiable Credentials that are used to request the access token.
8. If the `holder` field is not present then the `issuer` field MUST match the `credentialSubject.ID` field from all the Verifiable Credentials that are used to request the access token.

## 5. Presentation Definition endpoint

In order for a client to know which Presentation Definition [PE] to use, the authorization server MUST provide a Presentation Definition endpoint.
The Presentation Definition endpoint MUST be registered as a `presentation_definition_endpoint` in the authorization server metadata [RFC8414].
The Presentation Definition endpoint MUST return a Presentation Definition that corresponds with the requested scope.

## 6. Access Token Introspection

The authorization server MUST support the access token introspection endpoint as defined in OAuth 2.0 [RFC7662].
The introspection endpoint MUST return the following fields:

* `iss`: The issuer of the access token.
* `sub`: The subject of the access token.
* `exp`: The expiration time of the access token.
* `iat`: The time the access token was issued.
* `scope`: The granted scope.
* `vcs`: The Verifiable Credentials that were used to request the access token.

## 7. Security Considerations

The nonce/challenge (nonce) is used to prevent replay attacks. The nonce is a random string that is generated by the client.
The nonce is included in the signed data. The authorization server MUST reject any request that uses a nonce that has been used before.
The authorization server MAY reduce storage requirements by only storing the nonce for the lifetime of the Verifiable Presentation.

## 8. References

* [RFC6749] D. Hardt, "The OAuth 2.0 Authorization Framework", RFC 6749, October 2012, <https://www.rfc-editor.org/info/rfc6749>.
* [RFC7521] D. Campbell, Ed., B. Zaninovich, Ed., "Assertion Framework for OAuth 2.0 Client Authentication and Authorization Grants", RFC 7521, DOI 10.17487/RFC7521, May 2015, <https://www.rfc-editor.org/info/rfc7521>.
* [JWT] M. Jones, J. Bradley, N. Sakimura, "JSON Web Token (JWT)", RFC 7519, May 2015, <https://www.rfc-editor.org/info/rfc7519>.
* [VC-DATA-MODEL] M. Sporny, D. Longley, D. Chadwick, "Verifiable Credentials Data Model 1.1", W3C Recommendation, 3 March 2022, <https://www.w3.org/TR/vc-data-model/>.
* [PE] D. Buchner, B. Zundel, M. Riedel, K.H. Duffy, "Presentation Exchange 2.X.X", 12 September 2023, <https://identity.foundation/presentation-exchange>.
* [JSONWebSignature2020] O. Steel, M. Prorock, C.E. Lehner, "JSON Web Signature 2020", W3C Community Group Report, 21 July 2022, <https://w3c-ccg.github.io/lds-jws2020/>.
* [OIDC4VP] O. Terbu, T. Lodderstedt, K. Yasuda, T. Looker, "OpenID for Verifiable Presentations Draft 18", OpenID connect working group, 21 April 2023, <https://openid.net/specs/openid-4-verifiable-presentations-1_0.html>.
* [RFC8414] M. Jones, N. Sakimura, J. Bradley, "OAuth 2.0 Authorization Server Metadata", RFC 8414, June 2018, <https://www.rfc-editor.org/info/rfc8414>.
* [RFC7662] J. Richer, "OAuth 2.0 Token Introspection", RFC 7662, October 2015, <https://www.rfc-editor.org/info/rfc7662>.
* [DID] M. Sporny, D. Longley, M. Sabadello, D. Reed, O. Steele, C. Allen, "Decentralized Identifiers (DIDs) v1.0", W3C Recommendation, 19 July 2022, <https://www.w3.org/TR/did-core/>.