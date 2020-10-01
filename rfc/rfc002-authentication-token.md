# RFC002 Authentication token

|  |  |
| :--- | :--- |
| Nuts foundation | W.M. Slakhorst |
| Request for Comments: 002 | Nedap |
|  | S. van der Vegt |
|  | Nedap |
|  | September 2020 |

## Authentication token

### Abstract

todo

### Status of document

This document is currently a draft.

### Copyright Notice

![](../.gitbook/assets/license.png)

This document is released under the [Attribution-ShareAlike 4.0 International \(CC BY-SA 4.0\) license](https://creativecommons.org/licenses/by-sa/4.0/).

## 1.  Introduction

This document describes the concept of a token which can be used to authenticate a user. The token is a result of an authentication process. Several identification methods can be used, as long as the method of use is capable of creating a digital signature which can be verified by all network participants.

The token proves the relationship between the user and the service provider. This RFC is a specification for [§6.5](rfc001-nuts-start-architecture.md#6-5-user-identity-and-user) from the [Nuts Start Architecture](rfc001-nuts-start-architecture.md#nuts-start-architecture).

The token is the result of signing a login contract. The login contract is a different way of authentication than normally used. Normally a user discloses something only that user knows and the service provider compares this with the information it knows. The login contract is more of a mandate given by the user to the service provider allowing it to do digital requests on behalf of the user. The mandate is signed by the user by means of a digital signature. The digital signature is created with a private key where the public key has been validated by a trusted third party. This is extremely important for the network. Whereas normal disclosure is only of value between the user and service provider, by adding a signature, the resulting token can be used for requests over the network by the service provider.

## 2. Terminology

todo

## 3. Involved parties

### 3.1. User

The user will be a care professional or patient. The user has a need to process data across care providers. It is the party that needs to be authenticated and authorized. It either is the subject and therefore automatically has certain rights to process data or it is a care professional which also has a relation with a care organisation.

### 3.2. Issuer

The issuer is the trusted third party. Trusting the correct Issuers is a manual configuration task for the service provider. Since trust within Nuts has to be cryptographically proven, this can only be in the form of configuring trusted certificates or public keys. A distributed network of service providers will only work if each party trusts the same set of issuers. To further improve the security and robustness of the network, each proof of a relation between the real world and virtual domain needs to be rooted to a different trusted third party.

### 3.3. Service provider

The service provider is the party that offers software services that provides the user with access to the virtual domain. The service provider is responsible for implementing all the specifications in software.

### 3.4. Verifier

The verifier is a service provider that, in the context of this RFC, verifies a token. It could also be the party that holds the desired data for the user. It is responsible for authentication and presumably also the authorization.

## 4. Requirements

### 4.1. Human readable

The login contract must be human readable so the user can make a conscious decision about what it’s agreeing to.

### 4.2. Machine readable

The login contract must also be machine readable. This means that the variation in different contracts must be limited. A fixed set of contracts provides an easy fix.

### 4.3. Single party verification

A verifier must be able to verify the token without contacting external services. Contacting an external service would give that service too much information about care professionals, care organisations and service providers.

### 4.4. User identification

The token must hold the correct attributes on the user identity for the specific scope the token is to be used for.

### 4.5. Cryptographically signed

The token holds a digital signature binding the identity attributes to the login contract. This also makes the token tamper proof.

### 4.6. Time limited

The token is only valid during a limited time. This period of validity must be visible in the login contact so the user is aware of the validity period.

### 4.7. Organisation limited

A token is limited to a single care organisation. The name of the care organisation must therefore be displayed in the login contract.

### 4.8. Non-transferable

A token can only be used by service providers that provide service for the same care organisation stated in the login contract.

### 4.9. Versionable

A login contract must state a version number. This will allow service providers to update versions independently.

### 4.10. I18n

A login contract should be able to support multiple languages.

## 5. Login contract

A mandate usually consists of two parties: the party giving the mandate and the party receiving the mandate. It’s limited in time and limited in scope. The scope for the login contract consists of the care organisation the user works for and a custom scope which could limit the use of the contract for data retrieval, single-sign-on or something else.

The following template is to be used to create a token for a care professional:

```text
EN:PractitionerLogin:v2 Undersigned gives permission to {{service_provider}} to make requests to the Nuts network on behalf of {{care_organisation}} and itself. This permission is valid from {{valid_from}} until {{valid_to}}.
```

Or the Dutch version:

```text
NL:BehandelaarLogin:v2 Ondergetekende geeft toestemming aan {{acting_party}} om namens {{legal_entity}} en ondergetekende het Nuts netwerk te bevragen. Deze toestemming is geldig van {{valid_from}} tot {{valid_to}}.
```

Various placeholders are placed between double curly-braces:

* service\_provider: as defined in this document.
* care\_organisation: the care organisation the user works for.
* valid\_from: a specific ISO formatted date time value. The token is not valid before this moment.
* valid\_to: a specific ISO formatted date time value. The token is not valid after this moment.

A complete example presented to the user would look like:

```text
EN:PractitionerLogin:v2 Undersigned gives permission to Nuts foundation to make requests to the Nuts network on behalf of We Care B.V. and itself. This permission is valid from Monday, 2 January 2006 15:04:05 until Monday, 2 January 2006 16:04:05.
```

### 5.1. Constraints

The time layout used is constructed as: `Monday, 2 January 2006 15:04:05` The name of the service provider must match the name that has been registered for this service provider. The name of the care organisation must match the name within the proof that has been published by the service provider. The signature must be made by cryptographic means which can be connected to the user.

## 6. Authentication Token Container

The Authentication Token MUST be signed by the user. See section 7 for more detail on supported means. In order to support several signing means, the signed token MUST be encapsulated by a Authentication Token Container.

The container MUST be formatted as a JSON object with the following members.

The "type" uniquely identifies the means used for creating the token. The value MUST be a lower case string without spaces. The means identifier gives the validator a hint about the contents of the token.

The "token" member MUST contain the base64 encoded _Authentication Token_. The contents of the token can be anything and must be specified by the means.

Other members than the ones specified here SHOULD NOT be used and MUST be ignored.

Example of a Authentication Token Container:

```yaml
{
  "type": "irma",
  "token": "ewoiQGNvbnRleHQiOiAiaHR...FJJd05QN1N1dlAwSi9NL05WNTFWWnZxQ3lPKzdlRHdKOD0iCiAgfQp9Cn0=0"
}
```

## 7. Supported means

Future means will be added when available.

### 7.1. IRMA

[IRMA](https://irma.app), which stands for “I reveal my attributes” is the name of an app that implements the idemix cryptographic protocol suite. It provides strong authentication as well as privacy-preserving features such as anonymity, the ability to transact without revealing the identity of the transactor, and unlinkability, the ability of a single identity to send multiple transactions without revealing that the transactions were sent by the same identity. More information can be found on the website of the [Privacy by Design foundation](https://privacybydesign.foundation/irma-explanation/).

IRMA works by having the service provider show a challenge which can be cryptographically proven by a user, using it’s mobile. There are three types of challenges IRMA supports:

* Loading attributes
* Disclosing attributes
* Signing

The loading of attributes is required to get the right attributes in the user’s mobile phone. These attributes can later be disclosed to a service provider or can be used to sign data. IRMA supports both desktop and mobile flows. If the user is browsing a website on a desktop, the challenging service provider will show a QR-code to the user. The IRMA app is capable of scanning this QR-code which connects the service provider session to the IRMA app. When browsing a website on the same mobile device, it’s possible to use the IRMA app directly without showing a QR code.

Supported attributes are listed in a so-called scheme. The contents are maintained by the Privacy by design foundation. They ultimately decide if a certain issuer is allowed to join the scheme. This does not mean they control the private keys for each issuer.

Although the IRMA protocol is open, the best way to support IRMA as service provider is to use the available open-source IRMA Go server. The server implementation has been validated and since it contains a lot of complex cryptographical math, it’s better to not build your own implementation.

#### 7.1.1. Attribute selection

Since the goal is to identify the user, the selection of attributes used to sign the login contract is crucial. The combination of attributes must be globally unique and must have been obtained at a high level. The list below shows the selection of attributes required together with their issuer. A complete list of supported attributes can be found on the [IRMA website](https://privacybydesign.foundation/attribute-index/en/).

* `pbdf.gemeente.personalData.initials`
* `pbdf.gemeente.personalData.familyname`
* `pbdf.gemeente.personalData.dateofbirth`
* `pbdf.gemeente.personalData.digidlevel`
* `pbdf.sidn-pbdf.email.email`

The first 3 identify the user but are possibly not unique, therefore the email attribute is added to make the set of attributes unique. The _digidlevel_ is added to assure the attributes were obtained using the correct security level.

#### 7.1.2. Challenge

When the service provider is using the open-source IRMA Go server the following request must be sent to the correct endpoint:

```yaml
{
"@context": "https://irma.app/ld/request/signature/v2",
"Message": "THE LOGIN CONTRACT"
"disclose": [
  [
    [ 
      "pbdf.gemeente.personalData.initials", 
      "pbdf.gemeente.personalData.familyname", 
      "pbdf.gemeente.personalData.dateofbirth", 
      "pbdf.gemeente.personalData.digidlevel"
    ]
  ],
  [
    [
      "pbdf.sidn-pbdf.email.email"
    ]
  ]
]
}
```

The message field must have the full login contract as specified in chapter 6. The server will respond with some data that has to be converted into a QR code or a link that will activate the IRMA app on mobile. The response also contains a token that can be used for polling an endpoint for status changes. Interactions on the mobile phone will trigger certain status changes. When the operation is successful the result can be fetched from an endpoint, see the next paragraph. The [IRMA javascript library](https://irma.app/docs/irmajs/) can help in this process.

#### 7.1.3. Response

Below is the response of a successful IRMA signature challenge

```yaml
{
"@context": "https://irma.app/ld/signature/v2",
"signature": [
  {
    "c": "yIAiQyBhpME2mOEMxLzXur/Kik5+9z9w/CP8z3HWHRc=",
    "A": "RlJ2zRDAJR1GTf0Bm05sDMKw3vGON9VGvH4Y1v+gXdbI71Nh1ReUfsUd2ACurCx2TtKVc6XqylF1G70NimJdt2sA3ItfYz3R5+TxXypwTfCYbXz8AYlgG+MRowzM45xTqqfZMqS4Fs8oIrdAI9ayUeKFDpTQju8b/JpC+v8U5ww=",
    "e_response": "bTHAQd2FYOzTJL3BbWZg2DeW91o9b1WAmQPWnJTwrDYNVHkZ6NVgo1V9paGMtX7OySN5yvO3gR1u",
    "v_response": "BbFK6CKsKnBzCzjYuAAUB9mZfJZ6OA0rnd+mCviL5hlu4v7DXdNrFWyIOqBDfeUce2vESyJnGi59JAbCfLROGK9Ehp+sRTbTvBHazZk5coyalz18nlARj+yi6pnORuU+nAMBhTpQXhnewNSQATuDOTS+KB1mY02Um3KQVXT1jLRhAJlU5yM0jnt1kuuRGh/tTCWmp0TMt5pS3J3yLS1ob909nuJB8Iv6Eaco2YGCFuc6gqNXtTGEhcG4CEHVe7MqLYPh9BC9Uk4bQiTeafZ9d88FBG1Uk2PBvrZMFItFDRmzWnYWU9MiMRSOBJzVtN6t2ofET9/HJ4mP5WfIcdo/",
    "a_responses": {
      "0": "ZyTWJl0205ubywYFH2bGPWp/71sOwAjMEr0ZHW1R9D7xK1tOFO9GYVI8CQ4tD6fZuHrHmwFWBy6QQA2EP9A8kSvuNR0A28MXOiY=",
      "10": "GamYIBWLPlPJ1woU0WrCQ9SQaDW3vK0ikUQCW0097ssuWC62mXm+a9AIjdSxa/AFTr8Q5N7bMkV08N640qdA5wpYgF/uN3Dlw/c=",
      "11": "mEym6gcrFps/XQihA/9gQWYEQszD3+101Kz04cBWJqcBtGl+qjq9ybtaCZiGBqlj8EKom5u6zC8DRsusxuKYCceU4kWKZjEI0I4=",
      "12": "3VZPNJ6xkThx2VJlDzCxCGVwin+6bSY5YwhKISVgphc9oeF8AyMkmKZkvV2JiwfUvNs7KpQlNyiLXw2s8iddpV8zLDlRHcHJDkE=",
      "13": "zieEles1/UQBuP/JH7fmYAAK/7ofiL5bkcUM49dPJaQGU7FuOzVZyBH1nSgocUr/IqhQTelzh2Yy0DDlS6JUSkP4FEhTDaoV4Bc=",
      "14": "GYDhalrXE0+7Qwzk/4Lyyz3kvZceT/gSB/Nxin4f+d4F3weuUnN13FvKDlARVEcIN427ex054dJxq1FtR2mjHK6HY1mmsFzYUi8=",
      "15": "kWH1kZCoiSz/EnbZh0GK7Nohte02KDnPsjjjMPIylSOCzqTi7YlaHy04YqypOLZWKXBHYMoNC6s9lEBpa8eQGMIKgBEOmfwXgj4=",
      "16": "8kO9SIYdGFr1j2dcw81E2poBNmPVCcdeuxMRBIIxVFj6yNOG/VT61B5NNlIruK2c3iLfGB5IzkHfggQiZ/qiamSfSjYeHo3BlRU=",
      "17": "wtN7WXZO7PC2bMpP+BTBIuW3ek4zIe9CagQIv8kZRueqZ8zJjTi6gfG+CmFV7Zv0Wnz1UaR4uk1UUIZM8PNKeFlOKef8t9wbcxU=",
      "18": "ncGLDudDnGucNaPd2KgAzAycVSs8RRtQCXknl7R9LUgf9xNyRT89Y6SdHOS82t9lijF1cnoBJWnafSAAEylqXTsBVDsKkM/ebcc=",
      "19": "iccLNvHK/WcVVUtr/NU59okczziBGyBL4ZlRfQ9tTQwpxnB3Qrew74AIVd+3mnx29NUxMw89Yp0sRqvLSKeQjoFYalD3sr9QLrE=",
      "2": "1TXHps+1ipZL8sb0yk+dQUVeV5Sps0FeRoL9TksPwEQ48zaB7+d3h0ery7ORjIN9F/X5/xfDoN4pTxylSzinkd1r6NrpVkN3sAQ=",
      "7": "r3/vPSuNl5g+azUiPfXXsao4PWuX1MsJgEAR6yQUXEctsokdBNQgprBulsofKMY3+emaW1KQatmIC99AQYO8p8qf8HdlnotrpPw=",
      "8": "OB7YmQXtpbRc/PBMCKIevou8GaAzQ1yx2995hPCNdA7Bzw83XInoQlUV381Tjn3GSi2bFFSEB06EfEvvM8qwcsEWrHjNzw+f5Ks=",
      "9": "PYVsNO5RP3BYMPDHlG88WArTKScAI70Zr7OVCkzIErnAL9AiaVgxGNeDhtsgC6ikHWqlEsalUDYs2vYXy68lYHpiSKT4FrL10VM="
    },
    "a_disclosed": {
      "1": "AwAKOAAaAADzEDKAtyC9EOmkzPGKMnV8",
      "3": "rtLY2MrWyw==",
      "4": "yMs=",
      "5": "hOTq0tTd",
      "6": "rtLY2MrWykDIykCE5OrS1N0="
    }
  }
],
"indices": [
  [
    {
      "cred": 0,
      "attr": 6
    },
    {
      "cred": 0,
      "attr": 3
    },
    {
      "cred": 0,
      "attr": 4
    },
    {
      "cred": 0,
      "attr": 5
    }
  ]
],
"nonce": "VShoLPhVET+5/jq1HyG70w==",
"context": "AQ==",
"message": "NL:BehandelaarLogin:v1 Ondergetekende geeft toestemming aan Demo EHR om namens Zorggroep Nuts en ondergetekende het Nuts netwerk te bevragen. Deze toestemming is geldig van maandag, 24 februari 2020 16:15:47 tot maandag, 24 februari 2020 17:15:47.",
"timestamp": {
  "Time": 1582557442,
  "ServerUrl": "https://keyshare.privacybydesign.foundation/atumd/",
  "Sig": {
    "Alg": "ed25519",
    "Data": "+owJxphmD16dXlfpVfdRGfGnKp84RMU7ewk9lpCxV7DeQfvYgztSEVozqISLvnSz5ABNiQr1zyYOb3bths5TBw==",
    "PublicKey": "MKdXxJxEWPRIwNP7SuvP0J/M/NV51VZvqCyO+7eDwJ8="
  }
}
}
```

The response is quite lengthy and contains tons of information. Luckily the IRMA Go library has functionality to check the signature using information of the well-known IRMA schemes. This will give you information about the validity, the used attributes, the value of the disclosed attributes and at which point in time the signature was created.

The response is used as part of the authorization token in the [OAuth flow](rfc003-oauth2-authorization.md).

#### 7.1.4 Container usages

To use the IRMA _Authentication Token_ in a Authentication_Token Container_, the "type" member of the Authentication_Token Container_ MUST set to the value `irma`

The "token" value MUST be the base64 encoded JSON response.

