# RFC003 OAuth2 Authorization

|  |  |
| :--- | :--- |
| Nuts foundation | W.M. Slakhorst |
| Request for Comments: 003 | Nedap |
|  | S. van der Vegt |
|  | Nedap |
|  | September 2020 |

## OAuth2 Authorization

### Abstract

This RFC describes authorizing users and or systems in the Nuts network using the [OAuth 2.0 framework](https://oauth.net/2/). OAuth 2.0 is a widely accepted authorization framework well known from w.g., "sign-in with Google". The framework is highly customizable and accepts many _Grant types_. The most used OAuth grant types require clients to be registered up front with the authorization server, tokens to be transferred in advance and do not support zero-knowledge-proofs. In order to use OAuth in the Nuts network we must choose and configure the appropriate grant type. This RFC describes how to identify a system or a user with a custom JWT and then retrieve an access token which then can be used as bearer token for authorization at the resource server.

### Status of document

This document is currently a draft.

### Copyright Notice

![](../.gitbook/assets/license.png)

This document is released under the [Attribution-ShareAlike 4.0 International \(CC BY-SA 4.0\) license](https://creativecommons.org/licenses/by-sa/4.0/).

## 1.  Introduction

In this document we will provide a way of protecting RESTful APIs with use of an OAuth 2.0 access token. Contrary to a regular OAuth flow where a user is forwarded to an authentication server, the access token is obtained by providing a JWT which contains the system or user identity. This document describes the method of obtaining the token using a OAuth2 flow and describes usage of the token during an API request.

## 2. Terminology

* **Client application**: The application that requires access.
* **Resource server**: The application \(a protected resource\) that requires authorized access to its API’s.
* **JWT Grant**: JWT grant contains the user’s identity, subject and custodian and is signed by the acting party. This grant is used as an authorization grant to obtain an OAuth 2 access token.
* **Policy**: A security policy defined by a Bolt. It describes the access to and operations on resources that are allowed.
* **Access token**: An OAuth 2 access token, provided by an authorization Server. This token is handed to the client, so it can authorize itself to a resource server. The contents of the token are opaque to the client. This means that the client does not need to know anything about the content or structure of the token itself.
* **Authorization server**: The authorization server checks the user’s identity and credentials and creates the access token. The authorization server is trusted by the resource server. The resource server can exchange the access token for a JSON document with the user’s identity, subject, custodian and token validity. This mechanism is called token introspection which is described by [RFC7662](https://tools.ietf.org/html/rfc7662).
* **Request context**: The context of a request identified by the access token. The access token refers to this context. The context consists of the **custodian**, **actor**,  **Endpoint reference**: every registered endpoint has a unique reference which is calculated as the hash of the registration document. [RFC006](rfc006-distributed-registry.md) describes endpoint registration.
* **Compound service**: Higher level service as described by [RFC006](rfc006-distributed-registry.md).

Other terminology is taken from the [Nuts Start Architecture](rfc001-nuts-start-architecture.md#nuts-start-architecture).

## 3. OAuth flow

The mechanism of retrieving an access token using a JWT Grant is based on the JSON Web Token \(JWT\) Profile for OAuth 2.0 Client Authentication and Authorization Grants [\[RFC7523\]](https://tools.ietf.org/html/rfc7523).

![](../.gitbook/assets/nuts-oauth-authorization-flow.png)

The diagram shows the oauth flow without the registration part. It consists of three parts: obtaining the user identity, obtaining an access token and using the access token.

### 3.1 Obtain user identity

Part of the Nuts security framework involves the user identity. Data can only be accessed when the user is identified with an approved means. See also the [Authentication Token RFC](rfc002-authentication-token.md). The signed login contract is valid for a longer period of time and can therefore be stored in the user’s session. If the contract has expired, the client application can prompt the user for a new signature. It’s up to the client application when to ask for a signature: as a login method or when elevation is needed. In the case a system is obtaining an access token, this step is skipped.

### 3.2 Obtain access token

The client application MUST obtain an access token before accessing data. This SHOULD be done right before accessing the data. An access token is only valid for seconds/minutes \(determined by the authorization server\). The client application SHOULD refresh the access token only when more data is to be requested.

In order to obtain the access token, the client application MUST construct a signed JWT and send it to the registered authorization server token endpoint. The authorization server MUST validate the JWT and optionally validate if a legal base is present for the given custodian, subject and actor combination. If all is well, the authorization server SHOULD return an access token.

### 3.3 Use access token

When requesting data, the client application MUST add the access token to the Authorization header as a Bearer token as stated in [RFC7523](https://tools.ietf.org/html/rfc7523). The resource server MUST validate the access token with the authorization server. The resource server and authorization server together MUST decide if an access token is valid for the requested resource.

## 4. Client application

### 4.1 Registration

#### 4.1.1 Client registration

In common OAuth2 flows an OAuth client must be registered with the authorization server with its client id and client secret. This way the authorization server knows which requests are made by which party. The registration normally involves manual steps of registering and approving. In a network of trust with countless combinations of authorization servers and clients, this approach does not scale well.

So instead of client secrets, the Nuts OAuth flow binds the request via the JWT using its signature to a known care provider. The key used to sign the JWT is identified by a key identifier \(`kid`\) which can be resolved through the Verifiable Data Registry as defined in [RFC006](rfc006-distributed-registry.md). The key MUST be listed in the `assertionMethod` section of the actor's DID document. The actor is identified by the `iss` field of the JWT.

Example of the actor's DID document:

```javascript
{
  "@context": [ "https://www.w3.org/ns/did/v1" ],
  "id": "did:nuts:123",
  "verificationMethod": [
    {
      "id": "did:nuts:123#_TKzHv2jFIyvdTGF1Dsgwngfdg3SH6TpDv0Ta1aOEkw",
      "controller": "did:nuts:123",
      "type": "JsonWebKey2020",
      "publicKeyJwk": {
        "crv": "P-256",
        "x": "38M1FDts7Oea7urmseiugGW7tWc3mLpJh6rKe7xINZ8",
        "y": "nDQW6XZ7b_u2Sy9slofYLlG03sOEoug3I0aAPQ0exs4",
        "kty": "EC"
      }
    }
  ],
  "assertionMethod": ["did:nuts:123#_TKzHv2jFIyvdTGF1Dsgwngfdg3SH6TpDv0Ta1aOEkw"]  
}
```

#### 4.1.2 Server registration

Each compound service MUST define an `oauth` serviceEndpoint. This endpoint refers to another service in a DID Document. In order for the client to resolve the authorization server endpoint it MUST look up the `oauth` service endpoint of the compound service. A compound service may refer to endpoint services from other DID Documents.

### 4.2 Constructing the JWT

#### 4.2.1 Header

* **typ**: MUST be `JWT`
* **alg**: one of `PS256`, `PS384`, `PS512`, `ES256`, `ES384` or `ES512` \([RFC7518](https://tools.ietf.org/html/rfc7518)\)
* **kid**: MUST contain the identifier of a key published in the actor's DID document, listed in the `assertionMethod` section.  

#### 4.2.2 Payload

* **iss**: The issuer MUST contain the DID of the actor, thus the care organization making the request.
* **sub**: The subject MUST contain the DID of the custodian. The custodian's DID could be used to find the relevant consent \(together with the actor and subject\).
* **vcs**: A list of [Verifiable Credentials](rfc011-verifiable-credential.md). Optional
* **purposeOfUse**: A string containing the intended usage. Defined by a Bolt.   
* **aud**: As per [RFC7523](https://tools.ietf.org/html/rfc7523), the `aud` MUST be an `oauth` service identifier. That service MUST be an absolute endpoint.
* **usi**: User identity signature. The token container according to the [Authentication token RFC](rfc002-authentication-token.md). Base64 encoded. Optional
* **osi**: Ops signature, optional, reserved for future use.
* **exp**: Expiration, MUST NOT be later than 5 seconds after issuing since this call is only used to get an access token. It MUST NOT be after the validity of the Nuts signature validity.
* **iat**: Issued at. NumericDate value of the time at which the JWT was issued.

All other claims may be ignored.

#### 4.2.3 Example JWT

```javascript
{
  "alg": "RS256",
  "typ": "JWT",
  "kid": "did:nuts:123#_TKzHv2jFIyvdTGF1Dsgwngfdg3SH6TpDv0Ta1aOEkw"
}
```

```yaml
{
  "iss": "did:nuts:123",
  "sub": "did:nuts:456"
  "vcs": [...embedded VerifiableCredentials...],
  "purposeOfUse": "test-service",
  "aud": "did:nuts:456#_08934567fgjsdroiuty230467",
  "usi": {...Base64 encoded token container...},
  "osi": {...hardware token sig...},
  "exp": 1578915481,
  "iat": 1578910481
}
```

#### 4.2.4 Posting the JWT

The signed JWT MUST be sent to the registered authorization server oauth endpoint using the HTTP POST method. The `urn:ietf:params:oauth:grant-type:jwt-bearer` _grant\_type_ MUST be used. This profile is described in [RFC7523](https://tools.ietf.org/html/rfc7523). The JWT MUST be present in the assertion field. The scope MUST be **nuts**. Below is a full HTTP example.

```text
POST /token.oauth2 HTTP/1.1
     Host: as.example.com
     Content-Type: application/x-www-form-urlencoded

     grant_type=urn%3Aietf%3Aparams%3Aoauth%3Agrant-type%3Ajwt-bearer
     &scope=nuts
     &assertion=eyJhbGciOiJFUzI1NiIsImtpZCI6IjE2In0.
     eyJpc3Mi[...omitted for brevity...].
     J9l-ZhwP[...omitted for brevity...]
```

The client application MAY also send a JSON body. The authorization server MUST accept this as well.

```text
POST /token.oauth2 HTTP/1.1
     Host: as.example.com
     Content-Type: application/json

     {
          "grant_type": "urn:ietf:params:oauth:grant-type:jwt-bearer",
          "scope": nuts,
          "assertion": "eyJhbGciOiJFUzI1NiIsImtpZCI6IjE2In0.
               eyJpc3Mi[...omitted for brevity...].
               J9l-ZhwP[...omitted for brevity...]"
     }
```

The request MUST be sent to the authorization server via a TLS connection. If all is well, the result MUST be a [RFC6749](https://tools.ietf.org/html/rfc6749) response with a JSON body. It MUST contain the access\_token, token\_type and expires\_in fields.

```text
HTTP/1.1 200 OK
    Content-Type: application/json;charset=UTF-8
    Cache-Control: no-store
    Pragma: no-cache

    {
        "access_token":"2YotnFZFEjr1zCsicMWpAA",
        "token_type":"bearer",
        "expires_in":60     
    }
```

## 5. Authorization server

### 5.1 Registration

The authorization server endpoint needs to be registered for each service that requires authentication in the care organization DID document.

### 5.2 Validation

#### 5.2.1 Validation steps

The following steps MUST all succeed. The order of execution is not relevant although the JWT signature validation SHOULD be done first.

**5.2.1.1 JWT signature validation**

The first step is to validate the JWT, the DID in the **kid** field in the JWT header refers to the public key that is used to sign the JWT. If the signature is invalid, an **invalid\_signature** error is returned.

**5.2.1.2 TLS Client certificate validation**

The client certificate used in the TLS connection must conform the requirements as stated in [RFC008](rfc008-certificate-structure.md).

**5.2.1.3 Issuer validation**

To validate the identity of the issuer, the value of the `kid` MUST be present in the issuer's DID document under the `assertionMethod` section.

**5.2.1.4 JWT validity**

The JWT **iat** and **exp** fields MUST be validated. The timestamp of validation MUST lie between these values. The exp field MAY not be more than 5 seconds after the **iat** field.

**5.2.1.5 Login contract validation**

The **usi** field in the JWT contains the signed login contract. If present it MUST validate according to the [Authentication Token RFC](rfc002-authentication-token.md). The login contract MUST contain the `name` and `city` of the actor. The `name` and `city` MUST exactly \(case sensitive\) match with a valid [Verifiable Credential](rfc011-verifiable-credential.md) issued to the actor. This must be a [NutsOrganizationCredential](rfc012-nuts-organization-credential.md).

**5.2.1.6 Endpoint validation**

The **aud** field MUST match the URL of the registered OAuth endpoint. This prevents the use of the JWT at any other endpoint. The concrete endpoint URL is used for this. \([RFC7523](https://tools.ietf.org/html/rfc7523#section-3)\). The `purposeOfUse` field identifies the compound service which in turn refers to the correct endpoint URL via the `oauth` field. 

**5.2.1.7 Validate verifiable credentials**

The **vcs** field contains the list of verifiable credentials of any type.

If a credential has the `NutsAuthorizationCredential` type it determines whether the accessible resource and is valid according to the following rules \(based on [RFC014](rfc014-authorization-credential.md)\):

* The credential `issuer` equals the **sub** field of the JWT.
* The credential `credentialSubject.id` equals the **iss** field of the JWT.
* The credential proof is valid.
* The credential has not been revoked.
* The credential has not expired.

It MUST be verified that the actor's request conforms to the specified policy and only requests allowed resources.

The **vcs** field is not needed when actor and custodian are the same.

**5.2.1.8 Subject validation**

The **sub** field in the JWT MUST be a known organization. It MUST have been registered by the node operator of the authorization server and it MUST be valid at the time indicated by the **iat** field.

**5.2.1.9 Purpose of use**

The `purposeOfUse` field MUST contain the service type as defined by a Bolt. The `purposeOfUse` entry from every accompanying authorization credential MUST match. The resource server MUST be able to resolve the service type from the access token.

#### 5.2.2 Error responses

Errors are returned as described in [https://tools.ietf.org/html/rfc6749\#section-5.2](https://tools.ietf.org/html/rfc6749#section-5.2):

```text
     HTTP/1.1 400 Bad Request
     Content-Type: application/json;charset=UTF-8
     Cache-Control: no-store
     Pragma: no-cache

     {
       "error":"invalid_request"
     }
```

### 5.3 Access token

There’s no limitation of the type of access token that is issued by the authorization server. The following points do however apply to all forms of access token.

* Any random number MUST at least be 256 bits of length and base64 encoded. The authorization server MUST make sure it’s unique and not reused for a different context. 
* The authorization server MUST store the context associated with the token so the resource server can request it. 
* Tokens MUST NOT be valid for more than 60 seconds.
* The access token MUST be marked to only work for the specific vendor. So when the client application requests resources, the access token together with the TLS client certificate can be matched against the known vendor. This prevents hijacking the resulting access token.

### 5.4 Rate limiting

The access token is used in several flows including automated flows. This can cause considerable strain on the authorization server. A client application SHOULD therefore reuse tokens whenever possible. Whenever an authorization server is under heavy load, it MAY return the `429 Too many requests` status code. It then must also add the `Retry-After` header with the number of seconds the client application MUST wait. The resource server MAY also return this status code if a client application is requesting tokens when previous tokens are still valid. Given the fact that the client may be running in a clustered environment, it MUST not request more than 10 overlapping tokens. A token overlaps if the **iss**, **sub**, **sid** and **usi** fields are the same or when only the **iss** and **sub** fields are the same in the case when a token is used without subject context.

## 6. Resource server

### 6.1 Access token

The access token MUST be present in the Authorization header as bearer token:

```text
GET  /fhir/Patient/1 HTTP/1.1
     Host: resources.example.com
     Authorization: Bearer eyJhbGciOiJFUzI1NiIsImtpZCI6IjE2In0.
     eyJpc3Mi[...omitted for brevity...].
     J9l-ZhwP[...omitted for brevity...]
```

A token MAY be used multiple times unless the returned error code prevents it. A token MUST NOT be used when it has been expired.

### 6.2 Authorization

The resource server MUST validate the validity of the access token. It MAY contact the authorization server to validate the token, or it MAY use existing knowledge to validate the token. For example a JWT can be validated by using the registered public key of the authorization server. The resource server MUST also check if the client certificate used for the TLS connections is from the same party that requested the access token. The next step is to validate if the token may be used to access the requested resource. There are three different cases that MUST be supported:

1. **The requested resource does not contain patient information.** Certain resources do not contain patient information and may therefore be exchanged without user context. Resources that fall in this category MUST be marked as such in the specific use case specification.
2. **The requested resource belongs to a patient.** In this case the resource server MUST validate that user context is present, e.g. an access token has been requested with the _usi_ field. The resource server MUST also verify a Nuts Authorization Credential was used in the access token request for the combination of custodian, actor, subject and resource.
3. **The actor and custodian are the same.** It may be the case that a care organization is using multiple service providers. In that case each service provider acts on behalf of the care organization. Therefore, it's not needed to provide user context. It's up to the service providers to provide the correct enforcement of roles and any auditing duties. Each of the service providers \(actor and custodian\) MAY use different identifiers for the same care organization.

In the first two cases, the resource server MUST check if the policy covers the requested resource.

### 6.3 Error codes

Different protocols return different types of error messages. The format will most likely also differ. This means that error messages have to be specified per use-case. If an error message supports a text-based error code, then it should support the illegal\_access\_token code. If a client receives this error code then it MUST NOT reuse the access token.

## 7. Bolt requirements

Different types of data require different levels of authorization. Because those requirements depend on the data, it's impossible for an RFC to specify these. This passes the requirement on to the Bolts. Therefore the Bolt MUST define the access policy. The policy MUST define which resources may be accessed when no restrictions are given. The resources MAY be separated into 3 categories:

* Personal: personal and/or medical resources.
* Audited: non-personal resources that require user context.
* Organization: non-personal resources that do not require user context. \(e.g. server-to-server logic\)

