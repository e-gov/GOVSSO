---
permalink: AccessToken
---

<img src='img/eu_regional_development_fund_horizontal.jpg' width="350" height="200" alt="European Union European Regional Development Fund"/>

NB! The usage principles for the GovSSO Access Token are currently under review by the state architect's board and may change in the near future. Updates will be published as they become available.
{: .adv}

# Technical Specification: Access Token Feature
{: .no_toc}
v1.0, 2025-02-10

- TOC
{:toc}

## 1 Overview

This document is an addition to the [core technical specification](TechnicalSpecification), specifying details about the optional JWT Access Token feature, based on [RFC 9068](https://datatracker.ietf.org/doc/html/rfc9068).

By default, GovSSO returns an opaque (random string value) Access Token on every token endpoint request for all GovSSO client applications. The random string value is not usable in any meaningful way, but it cannot be omitted by GovSSO because of OpenID Connect Core requirements.

After enabling JWT Access Token feature for the specified client application registration, GovSSO will return Access Tokens in JWT format for that client application. JWT Access Token always contains the same authentication data (user details) as the JWT ID Token, but the Access Token has a different purpose and a different lifetime.

GovSSO client applications may transmit JWT Access Tokens to other services (resource servers in OAuth terms) which themselves don't have to be GovSSO client applications. Those other services may provide end-user access to its resources/API-s based on the JWT Access Token that is transmitted to them with the request.

ID Tokens must be consumed only by the same GovSSO client application for which they are issued. ID Tokens must not be transmitted to other services. If the client application doesn't need to transmit a JWT Access Token to other services, the JWT Access Token feature should not be enabled for that client application.

## 2 Enabling and configuring JWT Access Token feature

For a client application to be able to obtain JWT Access Tokens from GovSSO, the JWT Access Token feature must be explicitly enabled and configured for that client application registration on the GovSSO side. For a client application with a certain `client_id`, provide RIA with the following configuration values (on the initial client application registration form or later via [klient@ria.ee](mailto:klient@ria.ee)):

| Configuration parameter | example | explanation |
|-------------------------|---------|-------------|
| Audience URLs | `"https://example.com", "https://example.org/end/point"` | URLs that are used as `aud` claim values in Access Tokens. These values must be agreed between the GovSSO client application and the resource servers that the client application transmits JWT Access Tokens to. Values may or may not be existing URLs on the Internet, may or may not be related to the client application or to other services. For GovSSO, these values are opaque - GovSSO does not use these values in any way. These values are only useful for agreements between the client application and the resource servers that the client application transmits JWT Access Tokens to. <br> * Each value must conform to URL standard. <br> * Registration must include at least one URL. <br> * URL must use HTTPS protocol. <br> * URL must not contain userinfo or fragment components. |
| Access Token lifespan | `5m` | Access Token validity time period in minutes and/or seconds. Minimal allowed validity period is 1 second and maximum period 15 minutes. Access Token's validity period is not connected with ID Token and Refresh Token validity periods which are 15 minutes by default and cannot be changed by the GovSSO client.<br> Access Tokens cannot be revoked. If the user logs out of GovSSO, any issued Access Tokens will remain valid until their expiration. However, it should not be too short, to prevent frequent token requests and significant overhead. |

## 3 Obtaining Access Tokens

A client can obtain Access Tokens with [general ID Token requests](TechnicalSpecification#62-id-token-request) and subsequent [session update requests](TechnicalSpecification#63-session-update-request). Access Token is always returned on `/oauth2/token` response.

If an Access Token is set to expire before the ID Token, but the client application needs a valid Access Token, make a session update request before the Access Token expires.

### 3.1 Requesting Access Token with specific audience

If the client application has registered more than 1 Access Token audience URL but wants to explicitly get an Access Token with a specific audience URL, then this can be done by adding `audience={registered URL}` to the initial authentication request query. If the `audience` parameter is omitted, then all pre-registered audience URLs are returned in the Access Token.

***Example authentication request with specific audience URL***
````
GET /oauth2/auth?
 
redirect_uri=https%3A%2F%2Fclient.example.com%2Fcallback&
scope=openid&
state=hkMVY7vjuN7xyLl5&
response_type=code&
client_id=58e7ba35aab5b4f1671a&
ui_locales=en&
nonce=fsdsfwrerhtry3qeewq&
acr_values=substantial&
audience=https%3A%2F%2Fexample.com
````
(for better readability, the parts of the HTTP request are divided onto several lines)

### 3.2 Access Token request

***Example initial Access Token request***
````
POST /oauth2/token HTTP/1.1
Host: govsso.ria.ee
Content-Type: application/x-www-form-urlencoded
Authorization: Basic Zm9vYmFyYmF6OkhYUkhIYzV1RzB2N2VROTNIcDJ2N0poME9jSVBKRVM=
 
grant_type=authorization_code&
code=8mnzSdFyJM1LyBvBNvynthUbbNma9Byrao19gOIVcgM.RCsRqqR1BaNJRB_ZsUQxmHhPLr9IgffM2JUrrwCUBIo&
redirect_uri=https%3A%2F%2client.example.com%2Fcallback
````
(for better readability, the parts of the HTTP request are divided onto several lines)

***Example Access Token request with session update***
````
POST /oauth2/token HTTP/1.1
Host: govsso.ria.ee
Content-Type: application/x-www-form-urlencoded
Authorization: Basic Zm9vYmFyYmF6OkhYUkhIYzV1RzB2N2VROTNIcDJ2N0poME9jSVBKRVM=
 
grant_type=refresh_token&
refresh_token=1kYI7zBU_WEGoMCVxSraXLcuA906szL9hxC2qq7bgso.uq1VHIByywr0Q9fk-V9Jp1BmLLQihoqXctHHHY8b3bQ
````
(for better readability, the parts of the HTTP request are divided onto several lines)

### 3.3 Access Token response

***Example Access Token response***
````
HTTP/1.1 200 OK
Content-Type: application/json
Cache-Control: no-store
Pragma: no-cache
  
{
 "access_token": "eyJhbGciOiJSUzI1NiIsImtpZCI6Ijk5NGQ4OWU3LTA1YzAtNGY5My1hNGFh
                  LTZkNjJlMTRkY2ZiZiIsInR5cCI6IkpXVCJ9.eyJhY3IiOiJoaWdoIiwiYW1
                  yIjpbInNtYXJ0aWQiXSwiYXVkIjpbImh0dHBzOi8vdGVzdDEudGVzdCIsImh
                  0dHBzOi8vdGVzdDIudGVzdCJdLCJiaXJ0aGRhdGUiOiIxOTAzLTAzLTAzIiw
                  iY2xpZW50X2lkIjoiZWYxN2E1NDUtOGJmOS00OTc4LTgxNDUtNjA0MDk5Nzk
                  wMGFjIiwiZXhwIjoxNzM4OTQzNDQ3LCJmYW1pbHlfbmFtZSI6IlRFU1ROVU1
                  CRVIiLCJnaXZlbl9uYW1lIjoiT0siLCJpYXQiOjE3Mzg5NDMxNDYsImlzcyI
                  6Imh0dHBzOi8vZ292c3NvLWRlbW8ucmlhLmVlLyIsImp0aSI6IjE1NzhhYzA
                  wLWE4MWMtNGI0Mi1hODhhLTM3M2IxMjllYmJmNCIsInN1YiI6IkVFMzAzMDM
                  wMzk5MTQifQ.XjLnpKXSX29kT2Z0Epr_Eu3_aBytRxD927w78S_93n0rnZmT
                  Wcs8LZQHrmcJiygWGjY09oeXoqavsF6SFG6LdO_q1E5X1XsJ5VfACWMNQd4t
                  MGlCeIYXcpjuwB1Z-eNuy6rBUMwCZqID08bvShegFk7JgBwSy-TgtVXXxLFA
                  DEsFdYs8GG2Yj9wdkFrwVg_ALVOzQNQXJ6nNxPfsckZox05BhNcz4jz1NZH-
                  IgvFDD3ONl-24yTWGNZw1AqkQSDjP3vdHshYsWZTuc2F0l-M7yUxOat_qkfW
                  jTUHsIrk1OneAp0s4ojVenDm6nLdjHXAZCBOIGgZOV2Ydsug2uHrKlvT6_H4
                  s9XE8jHo9J2_QXWvLNwfRusjya5_bnhOax1HzrWenC81dF-SbB2wUD6DZTp_
                  uff0jIf2HD3BOaQP9PVixJ96tlJqraOHUq96r7pTUZ44gAl6xd6bjPiW1yTk
                  nYIp-Cjh3WgfeGH-neX0ZWhi0KqsO1fEJzaQfPmVOnWEraq_yT_igasC2xIZ
                  t56_IjviWMExWb10Gmn_FvlxIYY-p7JFYx6xymQGbf-a-Q3MCIso83N8aVMz
                  joG1KMrpp-BFmUlY4BC7pc2I_mPSDlP_m6sY-vvdEaVx7KWT0_lNMhpF5jlB
                  zTF_n_-wQela7b0aysCw7ixVthOz_qCoH4E",
 "token_type": "bearer",
 "expires_in": 300,
 "id_token": "eyJhbGciOiJSUzI1NiIsImtpZCI6ImNkZjQ2OGYwLWVlNGUtNGE3Yy1iMTQ1
              LTNjNjFhYTg1NjliNyIsInR5cCI6IkpXVCJ9.eyJhY3IiOiJoaWdoIiwiYW1
              yIjpbInNtYXJ0aWQiXSwiYXRfaGFzaCI6InRKczdxZGl0b2o4OEF3VTFIUml
              GdHciLCJhdWQiOlsiZWYxN2E1NDUtOGJmOS00OTc4LTgxNDUtNjA0MDk5Nzk
              wMGFjIl0sImF1dGhfdGltZSI6MTczODk0MTE2NywiYmlydGhkYXRlIjoiMTk
              wMy0wMy0wMyIsImV4cCI6MTczODk0NDA0NiwiZmFtaWx5X25hbWUiOiJURVN
              UTlVNQkVSIiwiZ2l2ZW5fbmFtZSI6Ik9LIiwiaWF0IjoxNzM4OTQzMTQ2LCJ
              pc3MiOiJodHRwczovL2dvdnNzby1kZW1vLnJpYS5lZS8iLCJqdGkiOiJhODE
              4OGE5NS00Yzg1LTRjMWMtOGM4Ny1lMzU0OWViMDdmYjYiLCJub25jZSI6Ill
              pci1qRnlDQUJIbDVRdzU4LWQtQnZybUlPbDZqNTlqODlYNVktZnptRnMiLCJ
              yYXQiOjE3Mzg5NDExNTMsInNpZCI6ImFhZTg0ZDA5LTNmMTMtNDM1ZS04MTU
              xLTFkMDQ1ZmU5NzhjNyIsInN1YiI6IkVFMzAzMDMwMzk5MTQifQ.mo-MDIN3
              B3wAUVeEroNpi5_qNLx7DIDBS9OVrOsywvP31hIi5PUC88-1f8y4d55J7x90
              tcxit4VZXSGBgyy0nzo9rNTu2bjfkLrkgN6NLpPOVuZz210ZZ8H7xWNbRVyp
              yp8lqgR6q9VbjuEFkvLqBUzxaU3nrEfD8_IUYn0Y5t1WnFiE9mx4KylnU_CF
              IRYKd8pY01njh5yQfVFYYWPgb-VJfj-hcyIYGcBLOQQShNKwK6df_j32g-ix
              9wnt1mC7DZGdO8ees8CjV9lDNaJPQtQtg-92pZedHzy7xP1FoUxlEvgXQrVv
              c1UVLVoNRlx2GVfvpCVDbkXDGv9mbOl6SnXBcXkopCbRrXAwOUrly10nKeuB
              2zRjjvFHiWdJHQO1jSxvUmz0-5sAEE8s-LvIxHXoMTjMkWKvN0J25o-9FqA9
              kNkSZLzFlxeODLXKIpIurWsMFKSQP3VRGToBvHw926IEEy5z-urBQ6M0gKDW
              rfTGdlmFVIYa2uPY12ReGj1Cu5v4wA1iPBl0nfETVQzajGt-PYzv_cU9Bw3n
              XenlTKlZTAbOS4ZgRkr1h_dZ5pr2JFoJKQDqdMqfSDpeRHa_Dxlv6078AeNs
              aloHizQIf5afNEOqvx49Lan5w060wUyUYBz-Qq0apgdKUk83oEkpnGzxrTpn
              npNnn9l3sctlJvo",
 "refresh_token": "1kYI7zBU_WEGoMCVxSraXLcuA906szL9hxC2qq7bgso.uq1VHIByywr0Q9fk-V9Jp1BmLLQihoqXctHHHY8b3bQ"
}
````

Other response parameters other than `access_token` are returned exactly the same as described in [core technical specification](TechnicalSpecification#62-id-token-request), but access_token contains a JWT value instead of an opaque (random string) value.

## 4 Access Token details

***Example GovSSO Access Token***
````
{
  "jti": "7816a5ca-bafe-4fb1-80e8-4b5f6991bfbe",
  "client_id": "sso-client-1",
  "aud": [
    "https://example.com",
    "https://example.org/example"
  ],
  "iss": "https://govsso.ria.ee/",
  "exp": 1732030361,
  "iat": 1732029460,
  "sub": "EE30303039914"
  "birthdate": "1903-03-03",
  "given_name": "OK",
  "family_name": "TESTNUMBER",
  "amr": [
    "smartid"
  ],
  "acr": "high"
}
````
**Access Token claims**

| Access Token element (claim) | example | explanation |
|------------------------------|---------|-------------|
| jti | `"jti": "663a35d8-92ec-4a8d-95e7-fc6ca90ebda2"` | Access Token unique identifier ([[JWT](https://tools.ietf.org/html/rfc7519)] "4.1.7.  jti (JWT ID) Claim"). |
| client_id | `"client_id": "GovSSO client"` | Unique ID belonging to the client that requested authentication (the value of `client_id` field is specified in authentication request). |
| aud | `"aud": [`<br> `"https://example.com",` <br><br> `"https://example.org/example"` <br>`]` | List of Access Token audience URLs. Compared to ID Token, Access Token `aud` claim does not hold GovSSO `client_id` value. Instead, the audience URLs are preregistered values (see explanation in [2.1 Configuration parameters](21-configuration-parameters). |
| iss | `"iss": "https://govsso.ria.ee/"` | Issuer Identifier, as specified in  [[OIDC-CORE](https://openid.net/specs/openid-connect-core-1_0.html)]. |
| exp | `"exp": 1591709871` | The expiration time of the Access Token (in Unix _epoch_ format). |
| iat | `"iat": 1591709811` | The time of issuance of the Access Token (in Unix _epoch_ format). |
| sub | `"sub": "EE60001018800"` |  The identifier of the authenticated user (personal identification code or eIDAS identifier) with the prefix of the country code of the citizen (country codes based on the ISO 3166-1 alpha-2 standard). NB! in case of eIDAS authentication the maximum length is 256 characters.|
| birthdate | `"birthdate": "2000-01-01"` | The date of birth of the authenticated user in the ISO_8601 format. Only sent in the case of persons with Estonian personal identification code and in the case of eIDAS authentication. |
| given_name | `"given_name": "MARY ÄNN"` | The first name of the authenticated user. |
| family_name | `"family_name": "O’CONNEŽ-ŠUSLIK TESTNUMBER"` | The surname of the authenticated user. |
| amr | `"amr": [ "mID" ]` | Authentication method reference. The authentication method used for user authentication. A single `amr` value is present in Access Token. Possible values:<br><br> `idcard` - Estonian ID card<br> `mID` - Mobile-ID<br> `smartid` - Smart-ID<br> `eidas` - European cross-border authentication |
| acr | `"acr": "high"` | Authentication Context Class Reference. Signals the level of assurance of the authentication method that was used. Possible values: `low`, `substantial`, `high`. The element is not used if the level of assurance is not applicable or is unknown. |

Compared to [ID Token claims](TechnicalSpecification#51-id-token):
- Access Token has different `jti` and `aud` values; `exp` value may be different depending on configuration (see chapter 2); `iat` value should be same or near value depending on request timing; 
- Access Token has additional `client_id` claim, that doesn't exist in ID Token; 
- Access Token doesn't have `at_hash`, `nonce` and `sid` claims, that exist in ID Token; 
- Access Token always has identical values for `iss`, `birthdate`, `given_name`, `family_name`, `amr` and `acr` claims as the ID Token that is returned with the same `/oauth2/token` response; 
- Access Token also has identical values for `phone_number` and `phone_number_verified` claims if these claims exist in the ID Token;

**Phone number claims**
If authentication request is performed with `phone` scope (e.g. `scope=openid%20phone`) and user's phone number is known (currently only when user has authenticated with Mobile-ID), then GovSSO will issue Access Tokens with the following additional claims.

| Access Token element (claim) | example | explanation |
|------------------------------|---------|-------------|
| phone_number | `"phone_number": "+37200000766"` | User's phone number in E.164 format. |
| phone_number_verified | `"phone_number_verified": true` | Always `true`, because user's phone number was verified during Mobile-ID authentication. |

If authentication request is not performed with `phone` scope or user's phone number is not known, these claims are not included in GovSSO Access Token.

Additionally, Access Token can also hold claims related to the optional [representee functionality](Representee).

Compared to Access Tokens, ID Tokens may hold other OpenID Connect protocol based claims not supported by GovSSO.

## 5 Security operations for client applications

If the client application performed the initial authentication request with a [specific `audience`](3-obtaining-access-tokens), then the client application must validate that the `aud` claim value in the Access Token corresponds to the requested `audience`.

If the client application performed the initial authentication request without specific `audience` value, then the validation can be omitted.

## 6 Security operations for resource servers

Similar to [ID Token security checks](TechnicalSpecification#7-security-operations), a resource server receiving an Access Token must perform similar verifications.

The resource server must conduct the following Access Token verifications:
- Verify signature;
- Verify issuer;
- Verify client ID;
- Verify that the token's audience holds a matching value to the resource server;
- Verify that the token is current;
- Verify the eIDAS level of assurance (`acr`) if lower than `high` level of assurance is not acceptable;

These validations are the same as in [RFC 9068 standard](https://datatracker.ietf.org/doc/html/rfc9068#name-validating-jwt-access-token) except for `typ` header validation which GovSSO does not support and with the inclusion of `client_id` validation.

### 6.1 Signature verification

This verification is identical to [ID Token's signature verification](TechnicalSpecification#711-verifying-the-signature).

The GovSSO authentication service signs Access Tokens. The signature meets the JWT standard ([JWT](https://tools.ietf.org/html/rfc7519)). GovSSO may use a different key for signing Access Tokens than signing ID Tokens and Logout Tokens.

The signature uses the `RS256` algorithm. The resource server must be able to verify the signature using this algorithm. It would be reasonable to use a standard JWT library which supports all JWT algorithms. Change of signing algorithm is considered unlikely, but possible for security considerations.

GovSSO public signature key must be used for the signature verification. The public signature key is published at the public key info endpoint (see chapter [8 Endpoints](TechnicalSpecification#8-endpoints) in GovSSO technical specification).

When performing requests to the GovSSO key info endpoint, TLS connection must be verified in the same way as described in [Technical specification](TechnicalSpecification#712-verifying-the-tls-connection-to-endpoints).

The public signature key remains stable under normal conditions. However, it may be changed without prior notice if required for security considerations. Key exchange is carried out based on [OIDC-CORE](https://openid.net/specs/openid-connect-core-1_0.html) standard.

The public signature key has an identifier (`kid`). The key identifier is aligned with the best practices of [OIDC-CORE](https://openid.net/specs/openid-connect-core-1_0.html#Signing) and OAuth 2.0 ([OAUTH](https://tools.ietf.org/html/rfc6749)) that enables the key exchange without service interruption.

We recommend buffering the GovSSO public key (together with `kid` value) in order to decrease the amount of requests made to GovSSO server.

The resource server must perform the following signature validation steps:

1. Read the `kid` value from Access Token header.
2. a. If the client application does not buffer the public key, make a request to GovSSO public key info endpoint and select key corresponding to `kid` value received from JWT header. <br>
   b. If client application buffers the public key (it needs to be buffered together with `kid` value), it must compare the `kid` value from Access Token header with buffered `kid` value. If they match, buffered key can be used. If not, then client application needs to make a request to public key info endpoint and select the key corresponding to `kid` value received from Access Token header and buffer it.
3. Validate the signature using the key corresponding to `kid` value from the Access Token header.

NB! "Hard coding" the key to client application configuration must be avoided. The key change will be typically communicated (except of urgent security reasons), but manual key handling will result downtime in client application for the period when GovSSO is already using the new key until the new key is taken use in client application.

### 6.2 Token issuer verification

This verification is identical to [ID Token's issuer verification](TechnicalSpecification#713-verifying-the-issuer-of-tokens).

The `iss` value of the Access Token element must be `https://govsso-demo.ria.ee/` (for GovSSO demo environment) or `https://govsso.ria.ee/` (for GovSSO production environment).

### 6.3 Client ID verification

The `client_id` in the Access Token must match the registered client ID of the GovSSO client application that is forwarding the access tokens.

### 6.4 Audience verification

The `aud` claim in the Access Token must contain the resource server's identifier.

### 6.5 Currentness verification

This verification is identical to [ID Token's validity verification](TechnicalSpecification#715-verifying-the-validity-of-the-tokens).

The verification is done using `iat` and `exp` elements in the Access Token. The client application uses its own clock to verify the validity. The following details should be verified:

* that token issuing time has been reached:

`iat` <= (current time + permitted difference between clocks)

* that the “expired” time has not been reached:

`exp` > (current time - permitted difference between clocks)

The resource server must choose the permitted difference between clock values.

### 6.6 eIDAS level of assurance (LoA) verification

This verification is identical to [ID Token's LoA verification](TechnicalSpecification#716-verifying-the-eidas-level-of-assurance).

Estonian authentication methods (ID-card, Mobile-ID, Smart-ID) have been defined currently as `high` LoA.

However, foreigners authenticating to Estonian services through eIDAS might authenticate by methods with any LoA (`high`, `substantial` or `low`). The resource server must verify the `acr` claim if it wishes to exclude authentication methods with lower LoAs (`substantial` or `low`).

## Change history

| Version, Date    | Description |
|------------------|-------------|
| 1.0, 2025-02-10  | First version |
