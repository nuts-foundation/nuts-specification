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

OAuth is widely accepted, but unfortunately almost all OAuth flows require tokens to be transfered in advance and do not support zero-knowledge-proofs. This RFC describes how a custom JWT can be used as bearer token for authorization at the authorization server. Different parts of the JWT then identify the vendor, care organisation and user.

### Status of document

This document is currently a draft.

### Copyright Notice

![](../.gitbook/assets/license.png)

This document is released under the [Attribution-ShareAlike 4.0 International \(CC BY-SA 4.0\) license](https://creativecommons.org/licenses/by-sa/4.0/).

## 1.  Introduction

In this document we will provide a way of protecting RESTful APIs with use of an OAuth2 access token. Contrary to a regular OAuth flow where a user is forwarded to an authentication server, the access token is obtained by providing a JWT which contains the system or user identity. This document describes the method of obtaining the token using a OAuth2 flow and describes usage of the token during an API request.

## 2. Terminology

* **Client application**: The application that requires access.
* **Resource server**: The application (a protected resource) that requires authorized access to its API’s.
* **JWT bearer token**: JWT encoded bearer token contains the user’s identity, subject and custodian and is signed by the acting party. This token is used to obtain an OAuth 2 access token.
* **Access token**: An OAuth 2 access token, provided by an authorization Server. This token is handed to the client so it can authorize itself to a resource server. The contents of the token are opaque to the client. This means that the client does not need to know anything about the content or structure of the token itself.
* **Authorization server**: The authorization server checks the user’s identity and credentials and creates the access token. The authorization server is trusted by the resource server. The resource server can exchange the access token for a JSON document with the user’s identity, subject, custodian, token validity and scope. This mechanism is called token introspection which is described by [RFC7662](%20https://tools.ietf.org/html/rfc7662).
* **Request context**: The context of a request identified by the access token. The access token refers to this context. The context consists of the **custodian**, **actor**, **subject** and scope of the request like, in this case, an SSO. A context can be retrieved by performing token inspection.

Other terminology is taken from the [Nuts Start Architecture](rfc001-nuts-start-architecture.md#nuts-start-architecture).

## 3. OAuth flow

The mechanism of retrieving an access token using a JWT is based on the [JSON Web Token \(JWT\) Profile for OAuth 2.0 Client Authentication and Authorization Grants](%20https://tools.ietf.org/html/rfc7523).

![](../.gitbook/assets/nuts-oauth-authorization-flow.png)

The diagram shows the oauth flow without the registration part. It consists of three parts: obtaining the user identity, obtaining an access token and using the access token.

### 3.1. Obtain user identity

Part of the Nuts security framework involves the user identity. Data can only be accessed when the user is identified with an approved means. See also the [Authentication Token RFC](rfc002-authentication-token.md). The signed login contract is valid for a longer period of time and can therefore be stored in the user’s session. If the contract has expired, the client application can prompt the user for a new signature. It’s up to the client application when to ask for a signature: as a login method or when elevation is needed. In the case a system is obtaining an access token, this step is skipped.

### 3.2. Obtain access token

The client application MUST obtain an access token before accessing data. This SHOULD be done right before accessing the data. An access token is only valid for seconds \(determined by the authorization server\). The client application SHOULD refresh the access token only when more data is to be requested.

In order to obtain the access token, the client application MUST construct a signed JWT and send it to the registered authorization server token endpoint. The authorization server MUST validate the JWT and optionally validate if a legal base is present for the given custodian, subject and actor combination. If all is well, the authorization server SHOULD return an access token. There are no restrictions on the returned access token.

### 3.3. Use access token

When requesting data, the client application MUST add the access token to the Authorization header as a Bearer token. The resource server MUST validate the access token with the authorization server. The resource server and authorization server together MUST decide if an access token is valid for the requested resource.

## 4. Client application

### 4.1. Registration

In common OAuth2 flows a client application must be registered by an authorization server with a client id and client secret. In a network of trust with many OAuth servers, this approach is difficult because it would mean every node needs to exchange secrets with every other node. Instead, the JWT signing public key needs to be approved by the client applications vendor. A client application needs to generate a key pair. The vendor should sign a certificate for the client application with its vendor CA certificate. The vendor CA certificate MUST be known by both client application and authorization server. The resulting certificate and the vendor CA MUST be added to the x5c header field. The certificate MUST not be valid for longer than 4 days.

At this point there’s no need to use a client certificate for the TLS connection.

### 4.2. Constructing the JWT

#### 4.2.1. Header

* **typ**: MUST be `JWT`
* **alg**: one of `RS256`, `RS384`, `RS512`, `ES256`, `ES384` or `ES512`
* **x5c**: contains both the signing certificate and the vendor CA. The first certificate MUST be the signing certificate. \([RFC7515](%20https://tools.ietf.org/html/rfc7515)\)

#### 4.2.2. Payload

* **iss**: The issuer in the JWT is always the actor, thus the care organization doing the request.
* **sub**: The subject contains the urn of the custodian. The custodian information could be used to find the relevant consent \(together with actor and subject\).
* **sid**: The Nuts subject id, patient identifier in the form of an oid encoded BSN. Optional
* **aud**: As per [RFC7523](%20https://tools.ietf.org/html/rfc7523), the aud MUST be the token endpoint. This can be taken from the Nuts registry. This is very important to prevent relay attacks.
* **usi**: User identity signature. The signed login contract according to the [Authentication token RFC](rfc002-authentication-token.md). Base64 encoded. Optional
* **osi**: Ops signature, optional, reserved for future use.
* **exp**: Expiration, SHOULD be set relatively short since this call is only used to get an access token. It MUST NOT be after the validity of the Nuts signature validity.
* **iat**: Issued at. NumericDate value of the time at which the JWT was issued.
* **jti**: Unique identifier, secure random number to prevent replay attacks. The authorization server MUST check this.

#### 4.2.3. Example JWT

```yaml
{
  "alg": "RS256",
  "typ": "JWT",
  "X5c": ["MIIE3jCCA8agAwIBAgICAwEwDQYJKoZIh...abbrevated...0aBsXBTWVU+4=",
 "MIIE+zCCBGSgAwIBAgICAQ0wDQYJKoZIh...abbrevated...9saWNkr09VZw="]
}
```

```yaml
{
  "iss": "urn:oid:2.16.840.1.113883.2.4.6.1:48000000",
  "sub": "urn:oid:2.16.840.1.113883.2.4.6.1:12481248",
  "sid": "urn:oid:2.16.840.1.113883.2.4.6.3:9999990",
  "aud": "https://as.example.com/token.oauth2",
  "usi": {...Base64 encoded login contract...},
  "osi": {...hardware token sig...},
  "exp": 1578915481,
  "iat": 1578910481,
  "jti": {unique-identifier}
}
```

#### 4.2.4. Posting the JWT

The signed JWT MUST be sent to the registered authorization server oauth endpoint using the HTTP POST method. The `urn:ietf:params:oauth:grant-type:jwt-bearer` grant\_type MUST be used. This profile is described in [RFC7523](%20https://tools.ietf.org/html/rfc7523). The JWT MUST be present in the assertion field. Below is a full HTTP example.

```text
POST /token.oauth2 HTTP/1.1
     Host: as.example.com
     Content-Type: application/x-www-form-urlencoded

     grant_type=urn%3Aietf%3Aparams%3Aoauth%3Agrant-type%3Ajwt-bearer
     &assertion=eyJhbGciOiJFUzI1NiIsImtpZCI6IjE2In0.
     eyJpc3Mi[...omitted for brevity...].
     J9l-ZhwP[...omitted for brevity...]
```

The request MUST be sent to the authorization server via a TLS connection. If all is well, the resulting access token MUST be present as bearer token in the authorization header.

## 5. Authorization server

### 5.1. Registration

The authorization server endpoint needs to be registered for each vendor/care organisation combination.

### 5.2. Validation

#### 5.2.1. Validation steps

**5.2.1.1. JWT signature validation**

The first step is to validate the JWT, the **x5c** field in the JWT header holds the public key and chain that is used to sign the JWT. If the signature is invalid, an invalid\_signature error is returned.

**5.2.1.2. Issuer validation**

The actor from the **iss** field must be known to the authorization server, a vendor must have registered it using a signing certificate signed by the same vendor CA as in the **x5c** field. The registered vendor CA MUST match the root of the **x5c** header field. It MAY be the case that the vendor CA has been renewed, in that case a previous valid certificate from the same vendor MAY have been used to register the actor. It that case the vendor CA MUST have been valid at the time the signing certificate had been signed.

**5.2.1.3. JWT validity**

The JWT **iat** and **exp** fields MUST be validated. The timestamp of validation MUST lie between these values. The exp field MAY not be more than 5 seconds after the **iat** field.

**5.2.1.4. Login contract validation**

The **usi** field in the JWT contains the signed login contract. If present it MUST validate according to the [Authentication Token RFC](rfc002-authentication-token.md).

**5.2.1.5. Employee proof validation**

Future work

**5.2.1.6. Endpoint validation**

The **aud** field MUST match the registered endpoint from which the authorization server is responding. This prevents the use of the JWT at any other endpoint.

**5.2.1.7. Validate legal base**

The **iss** fields contains the identifier of the actor, the **sub** field contains the identifier of the custodian and the **sid** field contains the identifier for the subject. A known legal base SHOULD be present at the authorization server/resource server side for this triple. If a particular resource may be accessed MUST be checked when accessing the resource. If no **sid** field is present, this check can be skipped.

#### 5.2.2. Error responses

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

### 5.3. Access token

There’s no limitation of the type of access token that is issued by the authorization server. A random number MUST at least be 256 bits of length and base64 encoded. The authorization server MUST make sure it’s unique and not reused for a different context. The authorization server SHOULD store the context associated with the token so the resource server can request it. The authorization server MAY also issue a signed JWT as access token with the context in the token. In that case the resource server MUST have access to the corresponding public key. In the case of a signed JWT, the JWT MUST contain the **jti**, **exp** and **iat** fields. The **jti** value MUST not be reused. The **exp** field MUST not be more than 60 seconds after the **iat** field.

The access token MUST be marked to only work for the specific vendor. So when the client application requests resources, the access token together with the TLS client certificate can be matched against the known vendor. This prevents hijacking the resulting access token.

### 5.4. Rate limiting

The access token is used in several flows including automated flows. This can cause considerable strain on the authorization server. A client application SHOULD therefore reuse tokens whenever possible. Whenever an authorization server is under heavy load, it MAY return the `429 Too many requests` status code. It then must also add the `Retry-After` header with the number of seconds the client application MUST wait. The resource server MAY also return this status code if a client application is requesting tokens when previous tokens are still valid. Given the fact that the client may be running in a clustered environment, it MUST not request more than 10 overlapping tokens. A token overlaps if the **iss**, **sub**, **sid** and **usi** fields are the same or when only the **iss** and **sub** fields are the same in the case when a token is used without subject context.

## 6. Resource server

### 6.1. Access token

The access token MUST be present in the Authorization header as bearer token:

```text
GET  /fhir/Patient/1 HTTP/1.1
     Host: resources.example.com
     Authorization: Bearer eyJhbGciOiJFUzI1NiIsImtpZCI6IjE2In0.
     eyJpc3Mi[...omitted for brevity...].
     J9l-ZhwP[...omitted for brevity...]
```

A token MAY be used multiple times unless the returned error code prevents it. A token MUST NOT be used when it has been expired.

### 6.2. Error codes

Different protocols return different types of error messages. The format will most likely also differ. This means that error messages have to be specified per use-case. If an error message supports a text-based error code, then it should support the illegal\_access\_token code. If a client receives this error code then it MUST NOT reuse the access token.

