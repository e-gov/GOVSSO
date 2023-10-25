---
permalink: TechnicalSpecification
---

<img src='img/eu_regional_development_fund_horizontal.jpg' width="350" height="200" alt="European Union European Regional Development Fund"/>

# Technical specification
{: .no_toc}
v2.2, 2023-10-26

- TOC
{:toc}

## 1 Overview

This document describes the technical characteristics of the State Single Sign-On (GovSSO) service protocol and includes advice for implementing client application interfaces. 

GovSSO protocol was designed following the best practices of OpenID Connect protocol and is heavily influenced of the current TARA service protocol [[TARA](https://e-gov.github.io/TARA-Doku/TechnicalSpecification)].
GovSSO will provide, in addition to user authentication (which is provided by TARA), user session management functionality.

The terminology in this document follows primarily terminology used in OpenID Connect protocol ([[OIDC-CORE](https://openid.net/specs/openid-connect-core-1_0.html)]). In non OpenID Connect specific cases [[TARA](https://e-gov.github.io/TARA-Doku/TechnicalSpecification)] terminology is followed.

## 2 OpenID Connect compliance

GovSSO protocol has been designed to follow the OpenID Connect protocol standard as closely as possible. The intent is to create a protocol that is compatible with a wide range of OpenID Foundation certified OIDC client applications and standard libraries. OIDC is by itself a very diverse protocol, providing multiple different workflows for a wide array of use-cases. To limit the scope of the development required to implement GovSSO service and client applications, the full scope of OIDC protocol will not be supported. This chapter describes the main customizations that are done to GovSSO as compared to the full OIDC protocol:

- The service supports only the authorization code flow of OIDC ([[OIDC-CORE](https://openid.net/specs/openid-connect-core-1_0.html)] "3.1.  Authentication using the Authorization Code Flow"). The authorization code flow is deemed the most secure option and is thus appropriate for public services.
- All information about an authenticated user is transferred to the application with ID Token ([[OIDC-CORE](https://openid.net/specs/openid-connect-core-1_0.html)] "2.  ID Token and 3.1.3.6.  ID Token").
- The eIDAS level of assurance (LoA) is returned within the `acr` claim in ID Token, using custom claim values `high`, `substantial`, `low`. [[OIDC-CORE](https://openid.net/specs/openid-connect-core-1_0.html)] "2.  ID Token"
- The requested minimum authentication level of assurance is selected by the client application in the beginning of GovSSO session (on initial authentication). [[OIDC-CORE](https://openid.net/specs/openid-connect-core-1_0.html)] "5.5.1.1.  Requesting the "acr" Claim".
- Available authentication methods for the user are provided based on the requested minimum authentication level of assurance.
- GovSSO supports only a single default scope that will return person authentication data: given name, family name, birthdate, person identifier. The scope remains the same during the entire GovSSO authentication session.
- Single-sign-on (SSO) is supported. Client applications are expected to perform session status checks to keep the authentication session alive.
- Refresh Token mechanism is used to keep the GovSSO session alive and retrieve new ID Token. ID Token is always present on successful response. Access Token mechanism is not used. ([[OIDC-CORE](https://openid.net/specs/openid-connect-core-1_0.html)] "12.  Using Refresh Tokens".)
- Central logout is supported according to OIDC Back-Channel logout specification [[OIDC-BACK](https://openid.net/specs/openid-connect-backchannel-1_0.html)].

## 3 SSO session

GovSSO will create an SSO session for each successful user authentication and stores it in its internal session storage. The SSO session is used to store user personal information and various parameters about the authentication request (for example authentication time, authentication level of assurance, authentication method used). The authentication information and parameters are not allowed to change during the authentication session.

The SSO session is linked to the user agent via a session cookie. The user is allowed to create a single SSO session per user agent but may create concurrent sessions across multiple user agents.

SSO session has a limited validity period of 15 minutes. During the SSO session validity period the user is not required to re-authenticate while logging into different client applications. Instead, they are given the option to use the authentication information stored in SSO session. GovSSO issues ID Tokens as proof of an active SSO session. The ID Token contains the user personal information as well as the session information. All issued ID Tokens are linked to an SSO session via a session id (`sid`) claim and have the same validity period as the SSO session.

The SSO session validity period is extended by 15 minutes every time a new client application authentication request is received, or an existing authenticated client application performs an SSO session update request. This means that all client applications are contributing to keep the SSO session valid for the duration the user is still using the client applications.

SSO session expires when no new authentication requests or session update requests have been received in the last 15 minutes. This is a safety feature assuring that user SSO session is not kept alive too long after the user has finished using the client applications.

SSO session validity period is determined by GovSSO service. It is currently 15 minutes by default, but may change as needed. Client applications must not hard code this value, but instead should use ID Token's `exp` claim if they need to determine when SSO session is going to expire.

The SSO session may also be terminated before the end of its expiry time by the user. When the user initiates a logout from one client application the client application must inform GovSSO of the logout event. The user is then given an option to terminate the SSO session and log out of all client applications related to the same SSO session.

## 4 Process flows

### 4.1 Authentication process

In order to log a user into a client application, the client application must acquire an ID Token from GovSSO. The ID Token will contain relevant user personal information as well as information of the SSO session. If the SSO session does not exist prior to the authentication request, a new session is created. If the session already exists but its level of assurance is lower than requested in the new request, then the user will be prompted that upon continuing the previous SSO session will be terminated and a new SSO session will be created.

<p style='text-align:left;'><img src='img/govsso_authentication_flow.png' style='width:1000px'></p>

1. User requests protected page from client application.
2. Client application checks whether the user has been authenticated in the client application.
3. Since the user is not authenticated in the client application, the client application will construct a GovSSO authentication request URL and redirects user agent to it. This is specified in [6.1 Authentication request](#61-authentication-request).
4. GovSSO checks whether there is a valid SSO session for given user agent.
    1. If SSO session exists and its level of assurance is equal or higher than requested the process will continue from step 8.
    2. Otherwise, GovSSO will automatically terminate the existing session and the process continues from step 5.
5. SSO session does not exist, therefore, GovSSO needs the user to be authenticated with TARA service. GovSSO constructs TARA authentication request and redirects user agent to it.
6. User is securely authenticated in TARA service. The detailed authentication process is described in TARA technical specification [[TARA](https://e-gov.github.io/TARA-Doku/TechnicalSpecification)]. Once the user has been authenticated TARA will redirect the user agent back to GovSSO with the TARA authorization code.
7. GovSSO uses the authorization code to acquire ID Token from TARA service (this request happens in GovSSO backend system). GovSSO will store the user identification information in its session storage.
8. GovSSO will optionally display a user consent form. This form is displayed only when a previous SSO session was used for authentication. Meaning that in step 4.1 a valid existing session was found.
9. GovSSO will construct its own authorization code and redirects the user agent back to client application URL.
10. The client application will use the authorization code to acquire a GovSSO ID Token. This is done in client application backend by sending an ID Token request to GovSSO. This is specified in [6.2 ID Token request](#62-id-token-request).
11. GovSSO will respond to the client application with a GovSSO ID Token and Refresh Token. GovSSO will also internally update the SSO session expiration time to `currentTime + N minutes`. The client application's back-end server validates the ID Token, then stores the ID Token and the Refresh Token into its session storage. The client application now has the user authentication information and can display the protected page. The client application's back-end server signals it's UI that next session update request should be scheduled at ID Token's `exp` minus 2 minutes.

### 4.2 Session update process

Once the user has been authenticated and an SSO session created, the client application must periodically perform SSO session update requests to keep the SSO session alive in GovSSO. In GovSSO protocol this is done by acquiring a new ID Token from GovSSO service using Refresh Token ([[OIDC-CORE](https://openid.net/specs/openid-connect-core-1_0.html)] "12.  Using Refresh Tokens").
NB! GovSSO has the following specifics compared to general Refresh Token approach in OpenID Connect and OAuth standards:
* Access Token mechanism is not used in GovSSO. Refresh Token mechanism is used to retrieve new ID Token instead.
* Refresh Token is always returned, without requesting `offline_access` scope. Client application must not add `offline_access` scope to authentication request.
* Refresh Token expires at the same time as the ID Token that was received with the same request.
* `id_token` field is always present in successful response of Refresh Request.
* The same validation rules apply to the ID Token as in retrieving the ID Token on authentication ([7.1 Verification of the ID Token and Logout Token](#71-verification-of-the-id-token-and-logout-token)).

SSO session update request can be made only when the client application knows the active Refresh Token and a valid SSO session exists.

SSO session update must be triggered from the user agent. This ensures that GovSSO session update is performed only when the user agent still has the client application page open, not when the user may have closed the client application page without clicking log out. 

Client application must initiate SSO session update request 2 minutes before ID Token's expiration (`exp` claim value). This leaves a reasonable buffer for slow or problematic network conditions. Refresh Token expires at the same time as ID Token expires.

If the SSO session update request returns with an OpenID Connect error code, then client application must terminate it's session and inform the user that they have been logged out. If the SSO session update request is unsuccessful because of a network error, client application may retry the request until latest ID Token's expiration time. Retrying the request after latest ID Token's expiration time is unnecessary because GovSSO refuses to process it.

<p style='text-align:left;'><img src='img/govsso_session_update_flow.png' style='width:700px'></p>

1. User is browsing protected content in the client application, but the latest ID Token is about to expire in 2 minutes from the current time. Client application's UI performs a background request with JavaScript to client application's back-end server to signal that the user agent still has the client application page open and GovSSO session needs to be updated.
2. The client application's back-end server verifies whether the user has an active client application session and that client application session storage contains a valid (not expired) GovSSO ID Token.
3. The client application's back-end server performs a GovSSO session update request that contains the latest Refresh Token. This is specified in [6.3 Session update request](#63-session-update-request).
4. GovSSO verifies that an SSO session is still active for the given Refresh Token.
5. GovSSO responds to the client application with a GovSSO ID Token and Refresh Token. GovSSO also internally updates the SSO session expiration time to `currentTime + N minutes`.
6. The client application's back-end server stores the new ID Token and the new Refresh Token into its session storage (replacing previous ones) and signals client application's UI that the session update request succeeded and the next GovSSO session update request should be scheduled at new ID Token's `exp` minus 2 minutes.

### 4.3 Logout process

It is expected that client applications terminate GovSSO session as soon as the user has finished using client application(s). If a user has requested to log out of a client application, then information about this event must be sent to GovSSO as well. GovSSO will either terminate the SSO session automatically (if only one client application is linked to the same SSO session) or display a logout consent form. The consent form allows user to log out from all client applications at once or to continue SSO session.

After a successful logout the user agent is redirected back to the client application. If a technical error occurs or the logout request is invalid (for example `post_logout_redirect_uri` does not match registered post-logout redirect URL in GovSSO), then the user is redirected to GovSSO default error page.

<p style='text-align:left;'><img src='img/govsso_logout_flow.png' style='width:800px'></p>

1. User requested to log out of client application
2. Client application does its internal application session termination procedures.
3. Client application redirects user agent to GovSSO logout URL. The redirect must include the previous ID Token (as `id_token_hint` parameter) and a redirect URI (as `post_logout_redirect_uri` parameter) where the user agent must be redirected to after logout. This is specified in [6.4 Logout request](#64-logout-request).
4. GovSSO validates the logout request. If the ID Token is not linked to the active SSO session of given user agent, then nothing is done, and the user agent is redirected back to the `post_logout_redirect_uri`. GovSSO will unlink the client application from SSO session in its session store.
5. If more client applications are linked to the same SSO session, then GovSSO will show a logout consent page.
6. User can either select to log out from all other client applications that are linked to the same SSO session or to continue SSO session with other client applications.
7. If the user selected to log out from all other client applications, GovSSO will send a back-channel logout request to each linked client application and unlink client applications from GovSSO session. This is specified in [6.5 Back-channel logout request](#65-back-channel-logout-request).
8. If no client applications remain linked to SSO session, GovSSO will terminate the session. User has been logged out of all client applications.
9. User agent is redirected back to the `post_logout_redirect_uri` of the client application which initiated the logout procedure.

### 4.4 Back-channel logout notifications

Each GovSSO client application must declare support to the OIDC Back-Channel logout specification ([[OIDC-BACK](https://openid.net/specs/openid-connect-backchannel-1_0.html)]) whose relevant parts are specified in [6.5 Back-channel logout request](#65-back-channel-logout-request). Each client must provide a back-channel logout endpoint URL as part of their registration information. GovSSO will send an out-of-band POST request to client application back-channel logout endpoint every time an SSO session ends.

The logout request contains a Logout Token. The Logout Token must be validated according to OIDC Back-Channel logout specification [[OIDC-BACK](https://openid.net/specs/openid-connect-backchannel-1_0.html)] "2.6.  Logout Token Validation".  After receiving a valid Logout Token from the GovSSO, the client application locates the session identified by the `sid` Claim. The client application then clears any state associated with the identified session. If the identified user is already logged out at the client application when the logout request is received, the logout is considered to have succeeded.

If the logout succeeded, the RP MUST respond with HTTP 200 OK.

Access to client application's back-channel logout endpoint should be restricted to GovSSO outgoing IP address specified in [9 Environments](#9-environments). For example, client application can serve users at `https://client.example.com/` and may provide back-channel logout endpoint on the same domain and port, for example `https://client.example.com/aaa/bbb/back-channel-logout`, but should filter requests to this endpoint by GovSSO IP address. 

Back-channel logout endpoint must support TLSv1.2 and/or TLSv1.3 protocol. Back-channel logout endpoint must present a valid TLS certificate that is signed by a certificate authority (CA) that is participating in the Mozilla Root Program.

## 5 Tokens

### 5.1 ID Token

In GovSSO protocol the ID Token is used as a certificate of the fact that authentication was performed. The ID Token has a concrete issuance and expiration date between which it is to be considered valid.

The ID Token is issued in JSON Web Token [[JWT](https://tools.ietf.org/html/rfc7519)] form.

***Example GovSSO ID Token***

````
{
  "jti": "663a35d8-92ec-4a8d-95e7-fc6ca90ebda2",
  "iss": "https://govsso.ria.ee/",
  "aud": [
    "sso-client-1"
  ],
  "exp": 1591709871,
  "iat": 1591709811,
  "sub": "EE60001018800",
  "birthdate": "2000-01-01",
  "given_name": "MARY ÄNN",
  "family_name": "O’CONNEŽ-ŠUSLIK TESTNUMBER",
  "amr": [
    "mID"
  ],
  "nonce": "POYXXoyDo49deYC3o5_rG-ig3U4o-dtKgcym5SyHfCM",
  "acr": "high",
  "at_hash": "AKIDtvBT2JS_02tkl_DvuA",
  "sid": "f5ab396c-1490-4042-b073-ae8e003c7258"
}
````
**ID Token claims**

| ID Token element (claim)   | example           |     explanation       |
|----------------------------|------------------ |-----------------------|
| jti | `"jti": "663a35d8-92ec-4a8d-95e7-fc6ca90ebda2"` |  ID Token unique identifier ([[JWT](https://tools.ietf.org/html/rfc7519)] "4.1.7.  jti (JWT ID) Claim"). |
| iss | `"iss": "https://govsso.ria.ee/"` |  Issuer Identifier, as specified in  [[OIDC-CORE](https://openid.net/specs/openid-connect-core-1_0.html)]. |
| aud | `"aud": [`<br> `"sso-client-1"` <br>`]` <br><br> or<br><br> `"aud": "sso-client-1"` |  Unique ID of a client application in GovSSO client database. ID belongs to the client that requested authentication (the value of `client_id` field is specified in authentication request). <br><br> String or array of strings. A single `aud` value is present in GovSSO tokens. |
| exp | `"exp": 1591709871` |  The expiration time of the ID Token (in Unix _epoch_ format). This also denotes the expiration time of the corresponding Refresh Token. |
| iat | `"iat": 1591709811` |  The time of issue of the ID Token (in Unix _epoch_ format). |
| sub | `"sub": "EE60001018800"` |  The identifier of the authenticated user (personal identification code or eIDAS identifier) with the prefix of the country code of the citizen (country codes based on the ISO 3166-1 alpha-2 standard). The subject identifier format is set by TARA authentication service ID Token [[TARA](https://e-gov.github.io/TARA-Doku/TechnicalSpecification)] "4.3.1 Identity token". NB! in case of eIDAS authentication the maximum length is 256 characters.|
| birthdate | `"birthdate": "2000-01-01"` |  The date of birth of the authenticated user in the ISO_8601 format. Only sent in the case of persons with Estonian personal identification code and in the case of eIDAS authentication. |
| given_name | `"given_name": "MARY ÄNN"` |  The first name of the authenticated user (the test name was chosen because it includes special characters). |
| family_name | `"family_name": "O’CONNEŽ-ŠUSLIK TESTNUMBER"` |  The surname of the authenticated user (the test name was selected because it includes special characters). |
| amr | `"amr": [ "mID" ]` |  Authentication method reference. The authentication method used for user authentication. A single `amr` value is present in GovSSO tokens. Possible values:<br><br> `mID` - Mobile-ID<br> `idcard` - Estonian ID card<br> `eIDAS` - European cross-border authentication<br> `smartid` - Smart-ID<br><br> Available authentication methods depend on TARA authentication service and the list may be extended in the future [[TARA](https://e-gov.github.io/TARA-Doku/TechnicalSpecification)] "4.1 Authentication request". |
| nonce | `"nonce": "POYXXoyDo49deYC3o5_rG-ig3U4o-dtKgcym5SyHfCM"` |  Security element. The authentication request’s `nonce` parameter value. Value is present only in case the `nonce` parameter was sent in the authentication request. |
| acr | `"acr": "high"` |  Authentication Context Class Reference. Signals the level of assurance of the authentication method that was used. Possible values: `low`, `substantial`, `high`. The element is not used if the level of assurance is not applicable or is unknown. |
| at_hash | `"at_hash": "AKIDtvBT2JS_02tkl_DvuA"` |  The access token hash calculated as described in OIDC specification [[OIDC-CORE](https://openid.net/specs/openid-connect-core-1_0.html)]. |
| sid | `"sid": "f5ab396c-1490-4042-b073-ae8e003c7258"` |  Session ID - String identifier for a GovSSO session. This represents a session of a User Agent. Different sid values are used to identify distinct sessions at GovSSO. |

ID Token may consist other OpenID Connect protocol based fields that are not supported in GovSSO.

**Phone number claims**
If authentication request is performed with `phone` scope (e.g. `scope=openid%20phone`) and user's phone number is known (currently only when user has authenticated with Mobile-ID), then GovSSO will issue ID Tokens with the following additional claims. If authentication request is not performed with `phone` scope or user's phone number is not known, these claims are not included in GovSSO ID Token.

| ID Token element (claim)   | example           |     explanation       |
|----------------------------|------------------ |-----------------------|
| phone_number | `"phone_number": "+37200000766"` | User's phone number in E.164 format. |
| phone_number_verified | `"phone_number_verified": true` | Always `true`, because user's phone number was verified during Mobile-ID authentication. |

Obtaining user's phone number can be useful if client application also provides digital signing functionality with Mobile-ID, then phone number can be used as an input parameter in Mobile-ID signing request.

### 5.2 Logout Token

GovSSO sends a JWT similar to an ID Token to client applications called a Logout Token ([OIDC-BACK](https://openid.net/specs/openid-connect-backchannel-1_0.html) "2.4.  Logout Token") to request that they log out a GovSSO session or user. The token issuer is GovSSO and the audience is the client application. Upon receiving a Logout Token the client application is expected to validate the token to make sure that it is a valid logout request. In case the token is valid the client application session (session between client application and user agent) must be ended.

Issued Logout Tokens contain a `sid` claim and are linked to ID Tokens via that `sid` claim. Each client application is expected to internally keep track of the ID Token `sid` claim and client application session relation. Upon receiving and validating the Logout Token, the client application must log out the corresponding client application session that is related to the same `sid` value.

OIDC Logout Tokens can be encrypted but GovSSO Logout Tokens are not encrypted.

***Example GovSSO Logout Token***

````
{
  "aud": [
    "sso-client-1"
  ],
  "events": {
    "http://schemas.openid.net/event/backchannel-logout": {}
  },
  "iat": 1591958452,
  "iss": "https://govsso.ria.ee/",
  "jti": "c0cfc91a-cdf5-4706-ad26-847b3a3fb937",
  "sid": "f5ab396c-1490-4042-b073-ae8e003c7258"
}
````
***Logout Token claims***

| Logout Token element (claim)   | compulsory       |    example        |     explanation       |
|--------------------------------|------------------|------------------ |-----------------------|
| iss | yes |  `"iss": "https://govsso.ria.ee/"` |  Issuer Identifier, as specified in  [[OIDC-CORE](https://openid.net/specs/openid-connect-core-1_0.html)]. |
| events | yes | `"events": {`<br> `"http://schemas.openid.net/event/backchannel-logout": {}`<br> `}` |  Claim whose value is a JSON object containing the member name `http://schemas.openid.net/event/backchannel-logout.` This declares that the JWT is a Logout Token. The corresponding member value MUST be a JSON object and SHOULD be the empty JSON object `{}`. [OIDC-BACK](https://openid.net/specs/openid-connect-backchannel-1_0.html) "2.4.  Logout Token"|
| aud | yes |  `"aud": [`<br> `"sso-client-1"` <br>`]` <br><br> or<br><br> `"aud": "sso-client-1"` | Audience(s), as specified in [[OIDC-CORE](https://openid.net/specs/openid-connect-core-1_0.html)]. |
| iat | yes |  `"iat": 1591958452` |  Issued at time, as specified IN [[OIDC-CORE](https://openid.net/specs/openid-connect-core-1_0.html)]. |
| jti | yes |  `"jti": "c0cfc91a-cdf5-4706-ad26-847b3a3fb937"` |  Unique identifier for the token, as specified in [[OIDC-CORE](https://openid.net/specs/openid-connect-core-1_0.html)]. |
| sid | yes | `"sid": "f5ab396c-1490-4042-b073-ae8e003c7258"` |  Session ID - String identifier for a GovSSO session. This represents a session of a User Agent. Different sid values are used to identify distinct sessions at GovSSO. |

## 6 Requests

### 6.1 Authentication request

**Request**

An authentication request is an HTTP GET request by which the user is redirected from the client application to GovSSO for authentication. After completing authentication request in user agent (specified in current chapter), client application's back-end server must perform ID Token request specified in [6.2 ID Token request](#62-id-token-request).

***Example GovSSO authentication request***
````
GET https://govsso.ria.ee/oauth2/auth?
 
redirect_uri=https%3A%2F%2Fclient.example.com%2Fcallback&
scope=openid&
state=hkMVY7vjuN7xyLl5&
response_type=code&
client_id=58e7ba35aab5b4f1671a&
ui_locales=en&
nonce=fsdsfwrerhtry3qeewq&
acr_values=substantial
````
(for better readability, the parts of the HTTP request are divided onto several lines)

***Request parameters***

| URL element   | compulsory       |    example        |     explanation       |
|---------------|------------------|------------------ |-----------------------|
| protocol, host, port and path | yes |  `https://govsso.ria.ee/oauth2/auth` |  `/oauth2/auth` is the OpenID Connect-based authentication endpoint of the GovSSO service (the concept of ‘authorization’ originates from the OAuth 2.0 standard protocol).<br><br> The URL is provided from OIDC server public discovery service: `https://govsso.ria.ee/.well-known/openid-configuration` as `authorization_endpoint` parameter. |
| client_id | yes | `client_id=58e7ba35aab5b4f1671a`  |  Client identifier. The client identifier is issued by RIA upon registration of the client application as a user of the authentication service. |
| redirect_uri | yes | `redirect_uri=https%3A%2F%2Fclient.example.com%2Fcallback`  |  Redirect URL ([[OAUTH](https://tools.ietf.org/html/rfc6749)] "3.1.2. Redirection Endpoint"). The redirect URL is selected by the institution. The redirect URL may include the query component. URL encoding should be used, if necessary [[URLENC](https://en.wikipedia.org/wiki/Percent-encoding)].<br> It is not permitted to use the URI fragment component (`#` and the following component; [[URI](https://tools.ietf.org/html/rfc3986)] "3.5. Fragment").<br> The URL protocol, host, port and path must match one of the pre-registered redirect URLs of given client application registration metadata (see `client_id` parameter). |
| scope | yes | `scope=openid` |  The authentication scope. Space delimited list of requested scopes.<br><br> `openid` scope is compulsory to signal that this is an OIDC authentication request.<br> In the default `scope` of openid GovSSO will issue ID Tokens with the following claims:<br><br> `sub` (physical person identifier)<br> `given_name`<br> `family_name`<br> `birthdate`<br><br> Presence of given attribute values will depend on the amount of information that is returned within TARA ID Tokens. <br><br>`phone` scope is optional, it can be used to request user's phone number if it is known. See **Phone number claims** chapter for details. |
| state | yes |  `state=hkMVY7vjuN7xyLl5` |  Security code against false request attacks (cross-site request forgery, CSRF). Read more about formation and verification of state under "7.3 Protection against false request attacks". |
| response_type | yes |  `response_type=code` |  Determines the manner of communication of the authentication result to the server. Only value `code` is allowed as only authorization code flow is supported by GovSSO |
| ui_locales | no |  `ui_locales=et` |  Selection of the user interface language. The following languages are supported: `et`, `en`, `ru`. By default, the user interface is in Estonian language. The client can select the desired language. This will also set the GUI language for TARA service views. |
| nonce | no |  `nonce=fsdsfwrerhtry3qeewq` |  A unique parameter which helps to prevent replay attacks based on the OIDC protocol ([[OIDC-CORE](https://openid.net/specs/openid-connect-core-1_0.html)] "3.1.2.1. Authentication Request"). |
| acr_values | no |  `acr_values=substantial` |  The minimum required level of authentication based on the eIDAS level of assurance (LoA). Allowed values are: `low`, `substantial`, `high`. `high` is used by default if the value has not been set in request.<br><br> GovSSO will store the authentication level of assurance in the SSO session object as an Authentication Context Class Reference (`acr`) claim, from TARA ID Token response. Upon each GovSSO authentication request, GovSSO will check that the requested level of assurance (`acr_values` parameter value) is lower or equal to the `acr` claim value of the GovSSO session. If the SSO session `acr` value (level of assurance) is lower than requested, the previous GovSSO session is automatically terminated and a new authentication is requested from TARA. After successful authentication a new SSO session is created. |

**Response**

GovSSO server will respond with an HTTP 302 Found message to redirect user agent back to client application URL.

The user agent is redirected to the `redirect_uri` included in the authentication request sent by the client application. In the redirect request, an authorization code (`code` parameter) is appended to the `redirect_uri` by GovSSO. Using the received code the client application can then request ID Token from GovSSO. Technically, an HTTP status code `302 Found` redirect request is used for redirecting.

The state value will also be returned to the client application, making it possible to perform CSRF validation on client application side.

***Example GovSSO authentication redirect***
````
GET https://client.example.com/callback?
 
code=71ed5797c3d957817d31&
state=hkMVY7vjuN7xyLl5
````
(for better readability, the parts of the HTTP request are divided onto several lines)

***Request parameters***

| URL element   | example           |     explanation       |
|---------------|------------------ |-----------------------|
| protocol, host, port and path	 | `https://client.example.com/callback` |  Matches the `redirect_uri` value sent in the authentication request. |
| code | `code=71ed579...` |  The authorization code to request the ID Token. |
| state | `state=hkMVY7vjuN7xyLl5` |  Security code against false request attacks. The security code received in the authentication request is mirrored back. Read more about forming and verifying `state` from ‘Protection against false request attacks’. |

Request might contain other URL parameters, that client application must ignore.

**Error response**

If GovSSO is unable to process an authentication request – there is an error in the request, TARA was unable to authenticate user or another error has occurred – GovSSO transfers an error message (URL parameter `error`) and the description of the error (URL parameter `error_description`) in the redirect request.

GovSSO relies on the OpenID Connect standard error messages [[OIDC-CORE](https://openid.net/specs/openid-connect-core-1_0.html)] "3.1.2.6. Authentication Error Response" and [[OAUTH](https://tools.ietf.org/html/rfc6749)] "4.1.2.1. Error Response". The error messages are always displayed in English.

Additionally, the following custom error codes are returned:
* 

`state` is also returned but no authorization code (`code`) is sent.

***Example GovSSO authentication error***
````
GET https://client.example.com/callback?
 
error=invalid_scope&
error_description=Required+scope+%3Copenid%3E+not+provided.+GovSSO+does+not+allow+this+request+to+be+processed&
state=hkMVY7vjuN7xyLl5
````
(for better readability, the parts of the HTTP request are divided onto several lines)

**Authentication termination response**

The user may also return to the client application without choosing an authentication method and completing the authentication process (via ‘Back to the service provider’ link). This option is provided for the cases in which the user has clicked ‘Log in’ in the client application but does not actually wish to log in. Then GovSSO returns an error response with `user_cancel` error code.

***Example GovSSO authentication termination response***
````
GET https://client.example.com/callback?
 
error=user_cancel&
error_description=User+canceled+the+authentication+process.&
state=hkMVY7vjuN7xyLl5
````
(for better readability, the parts of the HTTP request are divided onto several lines)

### 6.2 ID Token request

**Request**

The ID Token request is an HTTP POST request which is used by the client application to request the ID Token from the GovSSO service. It must be performed by client application's back-end server which knows `client_secret` value.

***Example GovSSO ID Token request***
````
POST /oauth2/token HTTP/1.1
Host: govsso.ria.ee
Content-Type: application/x-www-form-urlencoded
Authorization: Basic czZCaGRSa3F0MzpnWDFmQmF0M2JW
 
grant_type=authorization_code&
code=SplxlOBeZQQYbYS6WxSbIA&
redirect_uri=https%3A%2F%2client.example.com%2Fcallback
````
(for better readability, the body of the HTTP POST request is divided over several lines)

The client secret code must be provided in the ID Token request. For this purpose, the request must include the Authorization request header with the value formed of the word Basic, a space, and a string `<client_id>:<client_secret>` encoded in the Base64 format ([[HTTP-AUTH](https://datatracker.ietf.org/doc/html/rfc2617)] "2 Basic Authentication Scheme").

The body of the HTTP POST request must be presented in a serialized format based on the OpenID Connect protocol.

***Request parameters***

| Parameter   | parameter type     |    example        |     explanation       |
|-------------|--------------------|------------------ |-----------------------|
| protocol, host, port and path | query |  `https://govsso.ria.ee/oauth2/token` |  GovSSO server token endpoint URL. Published in GovSSO discovery endpoint `token_endpoint` parameter value. |
| grant_type | body |  `grant_type=authorization_code` |  The `authorization_code` value required based on the protocol. [[OIDC-CORE](https://openid.net/specs/openid-connect-core-1_0.html)] "3.1.3.1.  Token Request" |
| code | body |  `code=SplxlOBeZQQYbYS6WxSbIA` |  The authorization code received from the authentication service. |
| redirect_uri | body |  `redirect_uri=https%3A%2F%2client.example.com%2Fcallback` |  The redirect URL sent in the authentication request. |

**Response**

GovSSO server verifies that the ID Token is requested by the right application and issues the ID Token included in the response body (HTTP response body).

***Example GovSSO token endpoint response***
````
HTTP/1.1 200 OK
Content-Type: application/json
Cache-Control: no-store
Pragma: no-cache
 
{
 "access_token": "EKN-4fXC4n1RdkegKk-M0DRxZ8RwJYZ_EwW-9zLCYcA.7GT7Xq2deLvWzrrFq6f0DNwL6INW2PYRDPPEFMbws1o",
 "token_type": "bearer",
 "expires_in": 1,
 "refresh_token": "1kYI7zBU_WEGoMCVxSraXLcuA906szL9hxC2qq7bgso.uq1VHIByywr0Q9fk-V9Jp1BmLLQihoqXctHHHY8b3bQ",
 "id_token": "eyJhbGciOiJSUzI1NiIsImtpZCI6InB1YmxpYzoyMDQ0YmVlOC03MmY0LTQxYTMtYTRl
              Zi1hODQ5Y2Y1MmIzNzAiLCJ0eXAiOiJKV1QifQ.eyJhY3IiOiJoaWdoIiwiYW1yIjoib
              UlEIiwiYXRfaGFzaCI6Ik1Edl9MYzlFWmNpalZUWWJPMXBQdnciLCJhdWQiOlsic3NvL
              WNsaWVudC0xIl0sImF1dGhfdGltZSI6MTU5MTcwOTgxMCwiZXhwIjoxNTkxNzE2NTk4L
              CJpYXQiOjE1OTE3MTY1MzgsImlzcyI6Imh0dHBzOi8vc3NvLW9pZGMtc2VydmVyLmRld
              i5raXQ6ODA4MC8iLCJqdGkiOiI2YTk4MGVkMC1hMDFhLTQzMjgtOWNkOC1hYzIxOTA2Y
              mFkZTQiLCJub25jZSI6ImxSdWQwaVBxTVF2WGJBWHRmZ0VPVjJhYk9Mc3RJZEpSajRPS
              WxoRDF2NGciLCJwcm9maWxlX2F0dHJpYnV0ZXMiOnsiZGF0ZV9vZl9iaXJ0aCI6IjIwM
              DAtMDEtMDEiLCJmYW1pbHlfbmFtZSI6Ik_igJlDT05ORcW9LcWgVVNMSUsgVEVTVE5VT
              UJFUiIsImdpdmVuX25hbWUiOiJNQVJZIMOETk4ifSwicmF0IjoxNTkxNzE2NTM1LCJza
              WQiOiJmNWFiMzk2Yy0xNDkwLTQwNDItYjA3My1hZThlMDAzYzcyNTgiLCJzdWIiOiJFR
              TYwMDAxMDE4ODAwIn0.VHymIxnGlPSqKZl9fXG6ggKekFTy7-p95vEdhPDWEske7zNS_
              LELzK3pnWdsMNbO2IFf146ir6V-WmfYTRCN15IMsRhBOgqg_FacilqvK2B25fD8LxFoC
              DwYRFjFFEs1U4j6SA0OrFh-aWbZ6xYhOlPErYLFgKt5gl6dkAdO34UM09gx5ASzrkW4d
              gsfUcZ8YktDw9n_iq6TDtb17RMEqeIprRLCQ-fLEKaHe4GBAZc6RfwIzWLCmcwUL0sCq
              vMrHBagM99lrzorkpbUA9MmUCBel6QbskIoZQE_hXjnR7H18kuhjwZ2_mWwYj9zb-4aM
              HGL0dQn0eoz72lfjfh18NEFuLHyYHooxsN4H_8TonWPz_QvbCjFUkpm44lkAeaLM_9eO
              VX7m_iaqNXHAbZUBzInSNyF8Fb7yZGCgSxWq4_HyPxnTaZfuh7P5xdK_mQ5aJXA0kee6
              UAZNGg9dk-lqC3epVFUkgYDrr5Np8fNaqfbJJ_FGF0jY0GRGq7Ip-800Qko62m1ooTOj
              -iP-3qW-bpmivpnFIWGzlZSbXE67Z1oEQZrNCOZNx6hSguNC2LgwQpfYg5UkXHk4rVk3
              Vjc6MEZ5ZWIzdrvWoX5CEn1POF_r_JrqsTK1KkwxL9km0g5qP4jihcRKJ7HU0Ov_nalC
              iia9Cl7qaszTRmyQbWCTvE"
}
````
(for better readability, the body of the HTTP response is divided over several lines)

***Response parameters***

| Parameter   |    explanation       |
|-------------|----------------------|
| access_token |  OAuth 2.0 access token. With the access token the client application can request authenticated user’s data from userinfo endpoint.<br> **Not used in GovSSO because GovSSO session management is purely ID Token dependent. All user data is already available in the ID Token.** |
| token_type |  OAuth 2.0 access token type with `bearer` value. Not used in GovSSO. |
| expires_in |  The validity period of the OAuth 2.0 access token. Not used in GovSSO. |
| refresh_token | Refresh Token to be used for the next GovSSO session update request. An opaque string value. Refresh Token expires at the same time as `exp` value of `id_token`. |
| id_token |  ID Token, encapsulated in JWS Compact Serialization form ([[JWS](https://tools.ietf.org/html/rfc7515)] chapter 3.1). The ID Token itself is issued in JSON Web Token [[JWT](https://tools.ietf.org/html/rfc7519)] format .|

Response body might contain other fields, that client application must ignore.

**Error response**

In case the token endpoint encounters an error and can not issue valid tokens, an error response is generated according to OIDC Core specification [[OIDC-CORE](https://openid.net/specs/openid-connect-core-1_0.html)].

### 6.3 Session update request

**Request**

Client applications must periodically check the SSO authentication session validity on GovSSO server. Session update requests will also signal GovSSO server that the user is still active in the client application and the authentication session expiration time can be extended.

The process of acquiring a new authentication token is similar to initial ID Token request after user authentication, but `grant_type=refresh_token` and `refresh_token=` with a value of the previously received Refresh Token are used. 

***Example GovSSO session update request***
````
POST /oauth2/token HTTP/1.1
Host: govsso.ria.ee
Content-Type: application/x-www-form-urlencoded
Authorization: Basic czZCaGRSa3F0MzpnWDFmQmF0M2JW
 
grant_type=refresh_token&
refresh_token=1kYI7zBU_WEGoMCVxSraXLcuA906szL9hxC2qq7bgso.uq1VHIByywr0Q9fk-V9Jp1BmLLQihoqXctHHHY8b3bQ
````
(for better readability, the parts of the HTTP request are divided onto several lines)

***Request parameters***

| Parameter   | parameter type     |    example        |     explanation       |
|-------------|--------------------|------------------ |-----------------------|
| protocol, host, port and path | query |  `https://govsso.ria.ee/oauth2/token` |  GovSSO server token endpoint URL. Published in GovSSO discovery endpoint `token_endpoint` parameter value. |
| grant_type | body |  `grant_type=refresh_token` |  The `refresh_token` value required based on the protocol. [[OIDC-CORE](https://openid.net/specs/openid-connect-core-1_0.html)] "12.  Using Refresh Tokens"|
| refresh_token | body |  `refresh_token=1kY...3bQ` |  The Refresh Token value previously received from the GovSSO service. |

**Response**

GovSSO server verifies that the session update is requested by the right application and issues new ID Token and new Refresh Token in response body.

***Example GovSSO token endpoint response***
````
HTTP/1.1 200 OK
Content-Type: application/json
Cache-Control: no-store
Pragma: no-cache
 
{
 "access_token": "DysAY_mxlBX6epOugCgJeDRWBCQ90ORNfNL4qErwNKw.rzI0_mguIZuksDLY-72g_VSv_V_7vYjlrIq1eLTi7B0",
 "token_type": "bearer",
 "expires_in": 1,
 "refresh_token": "ruovXWcllnYmP9u4YZCgvN7sTB_FZdPgQKxiXsdUZCo.MYuYBKvRVl6-P2VU4Qk_gecA8LY5jjAGnNOpspLC5VA",
 "id_token": "eyJhbGciOiJSUzI1NiIsImtpZCI6InB1YmxpYzoyMDQ0YmVlOC03MmY0LTQxYTMtYTRl
              Zi1hODQ5Y2Y1MmIzNzAiLCJ0eXAiOiJKV1QifQ.eyJhY3IiOiJoaWdoIiwiYW1yIjoib
              UlEIiwiYXRfaGFzaCI6Ik1Edl9MYzlFWmNpalZUWWJPMXBQdnciLCJhdWQiOlsic3NvL
              WNsaWVudC0xIl0sImF1dGhfdGltZSI6MTU5MTcwOTgxMCwiZXhwIjoxNTkxNzE2NTk4L
              CJpYXQiOjE1OTE3MTY1MzgsImlzcyI6Imh0dHBzOi8vc3NvLW9pZGMtc2VydmVyLmRld
              i5raXQ6ODA4MC8iLCJqdGkiOiI2YTk4MGVkMC1hMDFhLTQzMjgtOWNkOC1hYzIxOTA2Y
              mFkZTQiLCJub25jZSI6ImxSdWQwaVBxTVF2WGJBWHRmZ0VPVjJhYk9Mc3RJZEpSajRPS
              WxoRDF2NGciLCJwcm9maWxlX2F0dHJpYnV0ZXMiOnsiZGF0ZV9vZl9iaXJ0aCI6IjIwM
              DAtMDEtMDEiLCJmYW1pbHlfbmFtZSI6Ik_igJlDT05ORcW9LcWgVVNMSUsgVEVTVE5VT
              UJFUiIsImdpdmVuX25hbWUiOiJNQVJZIMOETk4ifSwicmF0IjoxNTkxNzE2NTM1LCJza
              WQiOiJmNWFiMzk2Yy0xNDkwLTQwNDItYjA3My1hZThlMDAzYzcyNTgiLCJzdWIiOiJFR
              TYwMDAxMDE4ODAwIn0.VHymIxnGlPSqKZl9fXG6ggKekFTy7-p95vEdhPDWEske7zNS_
              LELzK3pnWdsMNbO2IFf146ir6V-WmfYTRCN15IMsRhBOgqg_FacilqvK2B25fD8LxFoC
              DwYRFjFFEs1U4j6SA0OrFh-aWbZ6xYhOlPErYLFgKt5gl6dkAdO34UM09gx5ASzrkW4d
              gsfUcZ8YktDw9n_iq6TDtb17RMEqeIprRLCQ-fLEKaHe4GBAZc6RfwIzWLCmcwUL0sCq
              vMrHBagM99lrzorkpbUA9MmUCBel6QbskIoZQE_hXjnR7H18kuhjwZ2_mWwYj9zb-4aM
              HGL0dQn0eoz72lfjfh18NEFuLHyYHooxsN4H_8TonWPz_QvbCjFUkpm44lkAeaLM_9eO
              VX7m_iaqNXHAbZUBzInSNyF8Fb7yZGCgSxWq4_HyPxnTaZfuh7P5xdK_mQ5aJXA0kee6
              UAZNGg9dk-lqC3epVFUkgYDrr5Np8fNaqfbJJ_FGF0jY0GRGq7Ip-800Qko62m1ooTOj
              -iP-3qW-bpmivpnFIWGzlZSbXE67Z1oEQZrNCOZNx6hSguNC2LgwQpfYg5UkXHk4rVk3
              Vjc6MEZ5ZWIzdrvWoX5CEn1POF_r_JrqsTK1KkwxL9km0g5qP4jihcRKJ7HU0Ov_nalC
              iia9Cl7qaszTRmyQbWCTvE"
}
````
(for better readability, the body of the HTTP response is divided over several lines)

***Response parameters***

| Parameter   |    explanation       |
|-------------|----------------------|
| access_token |  OAuth 2.0 access token. With the access token the client application can request authenticated user’s data from userinfo endpoint.<br> **Not used in GovSSO because GovSSO session management is purely ID Token dependent. All user data is already available in the ID Token.** |
| token_type |  OAuth 2.0 access token type with `bearer` value. Not used in GovSSO. |
| expires_in |  The validity period of the OAuth 2.0 access token. Not used in GovSSO. |
| refresh_token | New Refresh Token to be used for the next GovSSO session update request. An opaque string value. Refresh Token expires at the same time as `exp` value of `id_token`. |
| id_token |  New ID Token which is valid for a specified duration. ID Token claims are identical to previously issued ID Tokens, except for `jti`, `iat`, `exp`, `at_hash`. ID Token is encapsulated in JWS Compact Serialization form ([[JWS](https://tools.ietf.org/html/rfc7515)] chapter 3.1). The ID Token itself is issued in JSON Web Token [[JWT](https://tools.ietf.org/html/rfc7519)] format .|

Response body might contain other fields, that client application must ignore.

**Error response**

In case the token endpoint encounters an error and can not issue valid tokens, an error response is generated according to OIDC Core specification [[OIDC-CORE](https://openid.net/specs/openid-connect-core-1_0.html)].

### 6.4 Logout request

**Request**

A client application must notify the GovSSO that the user has logged out of client application and might want to log out of GovSSO as well. In this case, the client application, after having logged the user out of the client application, redirects the user's User Agent to GovSSO's logout endpoint URL. This URL is normally obtained via the `end_session_endpoint` element of GovSSO Discovery response or may be learned via other mechanisms.

***Example GovSSO logout request***
````
GET https://govsso.ria.ee/oauth2/sessions/logout?
 
id_token_hint=eyJhbGciOiJSUzI1NiIsImtpZCI6InB1YmxpYzo3Njc2MG...VkDzh0LYvs
post_logout_redirect_uri=https%3A%2F%2Fclient.example.com%2Floggedout&
state=0dHJpYnV0ZXMiOnsiZGF0ZV9vZl9iaXJ&
ui_locales=et
````
(for better readability, the parts of the HTTP request are divided onto several lines)

***Request parameters***

| URL element   | compulsory       |    example        |     explanation       |
|---------------|------------------|------------------ |-----------------------|
| protocol, host, port and path | yes |  `https://govsso.ria.ee/oauth2/sessions/logout` |  `/oauth2/auth` is the OpenID Connect-based logout endpoint of the GovSSO service. Described in OIDC session management specification [[OIDC-SESSION](https://openid.net/specs/openid-connect-session-1_0.html)] "2.1.  OpenID Provider Discovery Metadata" <br><br> The URL is provided from OIDC server public discovery service: `https://govsso.ria.ee/.well-known/openid-configuration end_session_endpoint` parameter. |
| post_logout_redirect_uri | yes | `post_logout_redirect_uri=https%3A%2F%2Fclient.example.com%2Floggedout` |  Post-logout redirect URL. The redirect URL is selected by the institution. The redirect URL may include the query component. URL encoding should be used, if necessary [[https://en.wikipedia.org/wiki/Percent-encoding](https://en.wikipedia.org/wiki/Percent-encoding).<br> It is not permitted ([[OAUTH](https://tools.ietf.org/html/rfc6749)] "3.1.2. Redirection Endpoint") to use the URI fragment component (`#` and the following component; [[URI](https://tools.ietf.org/html/rfc3986)] "3.5. Fragment".<br> The URL protocol, host, port and path must match one of the pre-registered redirect URLs of given client application. Client application is determined by the contents of the ID Token (token audience must belong to a registered GovSSO client application).<br> Different from OIDC session management specification, this parameter is considered mandatory in GovSSO. In GovSSO user logout flow we expect that the user is always redirected back to the client application that initiated the logout process. The `post_logout_redirect_uri` should point to the client application front page or a client application internal redirect url. |
| id_token_hint | yes |  `id_token_hint=eyJhbGciOiJSU...TvE` |  ID Token previously issued by GovSSO being passed as a hint about the user's current or past authenticated session with the client application. If the user identified by the ID Token is not logged in or is logged in by the request, then GovSSO returns a positive response; otherwise, it WILL return an error. **`id_token_hint` encryption is not supported.** |
| state | no |  `state=hkMVY7vjuN7xyLl5` |  Security code against false request attacks (cross-site request forgery, CSRF). Length of `state` parameter must be minimally 8 characters. Read more about formation and verification of state under 'Protection against false request attacks'.<br> If included in the logout request, the TARA passes this value back to the RP using the state query parameter when redirecting the user agent back to the client application.<br> Using the state parameter is not mandatory for login callbacks. It is expected that the user was already logged out of the client application before calling GovSSO logout endpoint. |
| ui_locales | no |  `ui_locales=et	` |  Selection of the user interface language. The following languages are supported: `et`, `en`, `ru`. By default, the user interface is in Estonian language.<br> If the user was logged into a single client application, then no GUI prompt will be displayed to the user. |

**Response**

After successful logout request, GovSSO will redirect the user agent back to the client application.

***Example GovSSO logout redirect***
````
GET https://client.example.com/loggedout?state=hkMVY7vjuN7xyLl5
````

Request might contain other URL parameters, that client application must ignore.

**Error response**

If the logout request processing is unsuccessful (for example the client and post_logout_redirect_uri do not match or a technical error in GovSSO prevents user logout), the user agent will be redirected to GovSSO error URL. According to OIDC the logout request does not provide means to redirect the user back to client application with an error message. Therefore, the user logout flow will end in GovSSO instead of the client application. GovSSO should display some form of request correlation id to aid customer support.

### 6.5 Back-channel logout request

**Request**

All GovSSO clients must support OIDC Back-channel logout functionality. According to back-channel logout specification [[OIDC-BACK](https://openid.net/specs/openid-connect-backchannel-1_0.html)] GovSSO server will send an out-of-band logout request to each client application when the user has requested to end the GovSSO session. GovSSO will include a Logout Token with each request so that the client applications can verify the logout event.

Relying Parties supporting back-channel-based logout register a back-channel logout URI with GovSSO as part of their client registration [[OIDC-BACK](https://openid.net/specs/openid-connect-backchannel-1_0.html)] "2.2.  Indicating RP Support for Back-Channel Logout".

The back-channel logout URI must be an absolute URI as defined by Section 4.3 of  [[URI](https://tools.ietf.org/html/rfc3986)]. The back-channel logout includes an `application/x-www-form-urlencoded` formatted query component. The back-channel logout URI will not include a fragment component.

Client applications must perform Logout Token validation to check if the received request is a valid GovSSO logout request. Validation must be performed according to OIDC protocol [[OIDC-BACK](https://openid.net/specs/openid-connect-backchannel-1_0.html)] "2.6. Logout Token Validation". If at any point token validation fails, the client application is expected to respond with an HTTP status code 400 Bad Request. If a session with a matching session ID is not found but the Logout Token is otherwise valid, then the client application should respond with HTTP status code 200 OK.

***Example GovSSO back-channel logout request***
````
POST /back-channel-logout HTTP/1.1
Host: client.example.com
Content-Type: application/x-www-form-urlencoded
 
logout_token=eyJhbGciOiJSUzI1NiIsImtpZCI...p-wczlqgp1dg
````
***Request parameters***

| Parameter   | compulsory     |    example        |     explanation       |
|-------------|----------------|------------------ |-----------------------|
| protocol, host, port and path | yes |  `https://client.example.com:443/back-channel-logout` |  Client application must authorize access to GovSSO to an internal URL and port. Access to the port should be limited based on IP address. The port must be protected with TLS. GovSSO must trust the logout endpoint server certificate. |
| logout_token | yes |  `logout_token=eyJhbGciOiJSUz...qgp1dg` |  GovSSO sends a JWT token similar to an ID Token to client applications called a Logout Token to request that they log out. Logout Token will give the client application exact information about the session (see the sid claim in ID Token) that should be logged out. The token is signed by GovSSO with the same secret key that is used for singing issued ID Tokens. |

Request might contain other URL parameters or body parameters, that client application must ignore.

## 7 Security operations

### 7.1 Verification of the ID Token and Logout Token

The client application must always verify the ID Token and Logout Token. The client application must implement all the verifications based on OpenID Connect ([[OIDC-CORE](https://openid.net/specs/openid-connect-core-1_0.html)] and [[OIDC-BACK](https://openid.net/specs/openid-connect-backchannel-1_0.html)]) and OAuth 2.0 protocols ([OAUTH](https://tools.ietf.org/html/rfc6749)).

The client must verify token’s:

- signature
- issuer
- addressee
- validity
- eIDAS level of assurance of (in case of ID Token)


#### 7.1.1 Verifying the signature

The ID Token and Logout Tokens are signed by the GovSSO authentication service. The signature meets the JWT standard ([JWT](https://tools.ietf.org/html/rfc7519)). GovSSO uses the same key for signing ID Token and Logout Token.

`RS256` signature algorithm is used, the client application must be able to verify the signature using this algorithm. It would be reasonable to use a standard JWT library which supports all JWT algorithms. The change of algorithm is considered unlikely, but possible in case a security vulnerability is detected in the `RS256`.

For the signature verification the GovSSO public signature key must be used. The public signature key is published at the public signature key endpoint (see chapter [8 Endpoints](#8-endpoints)).

The public signature key is stable - the public signature key will be changed according to security recommendations. However, the key can be changed without prior notice for security reasons. Key exchange is carried out based on [OIDC-CORE](https://openid.net/specs/openid-connect-core-1_0.html) standard.

The public signature key has an identifier (`kid`). The key identifier is aligned with the best practices of [OIDC-CORE](https://openid.net/specs/openid-connect-core-1_0.html#Signing) and OAuth 2.0 ([OAUTH](https://tools.ietf.org/html/rfc6749)) that enables the key exchange without the service interruption.

We recommend buffering the GovSSO public key (together with `kid` value) in order to decrease the amount of requests made to GovSSO server. However, it is allowed to request the key on each validation.

For signature validation following checks needs to be performed on client application:

1. Read the `kid` value from the JWT header. 
2. a. If the client application do not buffer the public key, make request to public signature key endpoint and select key corresponding to `kid` value received from JWT header. <br>
b. If client application buffers the public key (it needs to be buffered together with `kid` value), it needs to compare the `kid` value from JWT header with buffered `kid` value. If they match, buffered key can be used. If not client application needs to make request to public signature key endpoint and select key corresponding to `kid` value received from JWT header and buffer it. 
3. Validate the signature using the key corresponding to `kid` value from the JWT header.

NB! "Hard coding" the key to client application configuration must be avoided. The key change will be typically communicated (except of urgent security reasons), but manual key handling will result downtime in client application for the period when GovSSO is already using the new key until the new key is taken use in client application.

#### 7.1.2 Verifying the TLS connection to endpoints

Certificates described in this chapter will be applied in `govsso-demo.ria.ee` environment on 2023-11-07 and in `govsso.ria.ee` environment on 2023-11-14. Until that time certificates described in the [previous version](https://github.com/e-gov/GOVSSO/blob/02559ae5bd0cea4477f378a5624d3e89b17ba985/TechnicalSpecification.md#712-verifying-the-tls-connection-to-endpoints) are in effect.
{: .adv}

When making requests from the client application to GovSSO (to all endpoints, i.e., server discovery endpoint, public signature key endpoint, token endpoint), all necessary checks must be correctly performed during the initiation of the TLS connection. It is recommended not to implement these checks yourself, but to use a library with HTTPS or TLS connection functionality.

The trust anchor for the TLS connection must be set to the [DigiCert Global Root G2](https://cacerts.digicert.com/DigiCertGlobalRootG2.crt.pem) root certificate only. It is not advisable to trust certificates from other CAs or the entire operating system's CA certificate store. If necessary, instead of the DigiCert Global Root G2, the trust anchor for the TLS connection can be set to the [end-entity certificate](https://github.com/e-gov/TARA-Doku/blob/master/certificates/star_ria_ee_valid_until_2024-11-17.crt), which is replaced at least once a year.

The HTTPS or TLS connection functionality library must perform the following checks for each connection initiation:
* check whether a certificate chain with valid signatures is formed, ending with the [DigiCert Global Root G2](https://cacerts.digicert.com/DigiCertGlobalRootG2.crt.pem) root certificate;
* check whether the hostname in the request (`govsso.ria.ee` or `govsso-demo.ria.ee`) matches the CN field in the server's presented certificate or is included in the SAN field;
* check the start and end validity values for all certificates in the chain;
* check the constraints defined in the certificates (basic, name, key usage, critical extensions).

#### 7.1.3 Verifying the issuer of tokens

The `iss` value of the ID Token element must be `https://govsso-demo.ria.ee/` (for GovSSO demo environment) or `https://govsso.ria.ee/` (for GovSSO production environment).

#### 7.1.4 Verifying the addressee of the tokens

The client application must verify whether the token received was issued for them. For this purpose, it must be made sure that the `aud` value of the ID Token element matches the `client_id` issued upon registration of the client application.

#### 7.1.5 Verifying the validity of the tokens

***ID Token***

The verification is done using `iat` and `exp` elements in the ID Token. The client application uses its own clock to verify the validity. The following details should be verified:

* that token issuing time has been reached:

`iat` <= (current time + permitted difference between clocks)

* that the “expired” time has not been reached:

`exp` > (current time - permitted difference between clocks)

The application must choose the permitted difference between clocks value.

***Logout Token***

The verification is done using the `iat` claim value of the Logout Token. The following details should be verified:

* that token issuing time has been reached 

`iat` <= (current time + permitted difference between clocks)

The application must choose the permitted difference between clocks value.

#### 7.1.6 Verifying the eIDAS level of assurance

In order to prevent access with authentication tools with a lower security level, it must be verified that the authentication level in the `acr` claim of ID Token is not lower than the minimum level of assurance allowed.

For example, if the client application wants to use only authentication methods with eIDAS level of assurance `high` and has specified the value in the `acr_values` parameter, then only ID Tokens with `acr` claim with value `high` can be accepted.

In case the level of assurance in the authentication request using `acr_values` parameter is not specified, the ID Token must be equal to a level of assurance `high`.

### 7.2 Creating a session

After a successful verification of the ID Token, the client application will create a session with the user. The client application is responsible for creating and holding the sessions. The methods for doing this are not included in the scope of the GovSSO authentication service.

ID Tokens issued to the client application will contain a `sid` claim. This claim can be used to link client application session to SSO session. Client applications must store the latest ID Token and Refresh Token issued for each session. Refresh Token is required to perform session update requests and ID Token is required to create logout requests.

The GovSSO session refresh interval in client application should be slightly shorter than GovSSO ID Token expiration time. This way the client application is forced to always request a new ID Token before the last ID Token expires. GovSSO server will reject requests that contain Refresh Tokens for expired GovSSO sessions.

Logout Tokens usually contain the same `sid` claim. When a Logout Token is received the client application must find all application sessions that contain ID Tokens with the same `sid` claim value and terminate them (force the user to log out on the same user agent).

When application session was terminated internally by GovSSO, the Logout Token may instead contain only a `sub` claim. In this case the client application is expected to terminate all session that contain ID Tokens with matching `sub` value (force the user to log out on all user agents).

### 7.3 Protection against false request attacks

The client application must implement protective measures against false request attacks (cross-site request forgery, CSRF). This can be achieved by using `state` and `nonce` security codes. Using `state` is compulsory; using `nonce` is optional. The procedure of using `state` parameter with client side cookie (in this case the client application do not need to store the state itself) is given below.

The `state` security code is used to combat falsification of the redirect request following the authentication request. The client application must perform the following steps:

1. Generate random nonce (not to confuse with `nonce` parameter) word `R`, for example of the length of 16 characters: `XoD2LIie4KZRgmyc`.
2. Calculate hash `H` from the `R` (`H = hash(R)`), for example by using the SHA256 hash algorithm and by converting the result to the Base64 format: `vCg0HahTdjiYZsI+yxsuhm/0BJNDgvVkT6BAFNU394A=`. Length of `state` parameter must be minimally 8 characters.
3. Directly before sending the authentication request set a cookie to client service domain with the value `R`, for example:
   `Set-Cookie CLIENTSERVICE=XoD2LIie4KZRgmyc; HttpOnly`,
   where `CLIENTSERVICE` is a freely selected cookie name. The HttpOnly attribute must be applied to the cookie.
4. On making the authentication request set the value `H` for the `state` parameter:
   `GET ... state=vCg0HahTdjiYZsI+yxsuhm/0BJNDgvVkT6BAFNU394A=`
 
In the course of processing the redirect request, the client application must:

1. Take the `CLIENTSERVICE` value of the cookie received with the request (with callback request two values are received: cookie with the nonce word `R` and the nonce hash `H` in `state` parameter).
2. Calculate the hash based on the cookie value and convert it to Base64.
3. Verify that the hash matches the `state` value mirrored back in the redirect request.

The redirect request may only be accepted if the checks described above are successful.

The key element of the process described above is connection of the `state` value with the session. This is achieved by using a cookie. (This is a temporary authentication session. The work session will be created by the client application after the successful completion of the authentication).

Unfortunately, this topic is not presented clearly in the OpenID Connect protocol [[OIDC-CORE](https://openid.net/specs/openid-connect-core-1_0.html)].

### 7.4 Logging

Logging must enable the reconstruction of the course of the communication between GovSSO and the client application for each occasion of using GovSSO. For this purpose, all following requests and responses must be logged by GovSSO as well as by the client application: authentication_request, authentication_redirect, token_request, session_update_request, session_update_redirect, logout_request, logout_redirect, backchannel_logout. As the volumes of data transferred are not large, the URL as well as the ID Token must be logged in full. The retention periods of the logs should be determined based on the importance of the client application. We advise using 1 … 7 years. In case of any issue, please submit an excerpt from the log (Which requests were sent to GovSSO? Which responses were received?).

## 8 Endpoints

| Endpoint description | Endpoint | Comment |
|----------------------|----------|---------|
| server discovery | `/.well-known/openid-configuration` |Public endpoint for GovSSO server OpenID Connect configuration information. Usually provided as standard endpoint for OIDC implementations that support service discovery [[OIDC-DISCOVERY](https://openid.net/specs/openid-connect-discovery-1_0.html)] "4.1 OpenID Provider Configuration Request". |
| key info | `/.well-known/jwks.json` |  JSON Web Key Set document for GovSSO service. Publishes at minimum the public key that client applications must use to validate ID Token and Logout Token signatures [[JWK](https://tools.ietf.org/html/draft-ietf-jose-json-web-key-41)]. |
| authorization | `/oauth2/auth` |  OAuth 2.0 authorization endpoint. Used for GovSSO session update requests and authentication requests. [[OAUTH](https://tools.ietf.org/html/rfc6749)] "3.1.  Authorization Endpoint". |
| token | `/oauth2/token` | GovSSO endpoint to obtain ID Token [[OIDC-CORE](https://openid.net/specs/openid-connect-core-1_0.html)] "3.1.3.  Token Endpoint". In addition access tokens are returned for OAuth 2.0 compliance but their use in GovSSO protocol is not required. |
| logout | `/oauth2/sessions/logout` | GovSSO client application initiated logout endpoint. [[OIDC-SESSION](https://openid.net/specs/openid-connect-session-1_0.html)] "5. RP-Initiated Logout". |

## 9 Environments

| Environment | OpenID Connect Issuer URL | Originating IP address of GovSSO back-channel logout requests (from GovSSO to client application) |
|-------------|---------------------------|---------------------------------------------------------------------------------------------------|
| Demo | `https://govsso-demo.ria.ee/` | 195.80.127.38 (govsso-demo.ria.ee) |
| Production | `https://govsso.ria.ee/` | 195.80.127.3 (govsso.ria.ee) |

## Change history

| Version, Date    | Description |
|------------------|-------------|
| 2.2, 2023-10-26  | TLS trust anchor change (same as in TARA Technical Specification). |
| 2.1, 2023-04-11  | Elaborated TLS validation requirements and specified TLS trust anchor (same as in TARA Technical Specification). |
| 2.0, 2023-03-20  | Replaced `prompt=none&id_token_hint=...` session update process with Refresh Token session update process. |
| 1.1, 2023-01-24  | Notification about upcoming changes in session update process. |
| 1.0, 2022-09-19  | Added `phone` scope, `phone_number` claim, and `phone_number_verified` claim. Specified authentication termination response with `user_cancel` error code. Fixed `post_logout_redirect_uri` references. Specified production environment address. |
| 0.3, 2022-03-27  | Clarifications: session update request must be performed 2 minutes before ID Token's expiration (`exp` claim value), 15 minutes must not be hard coded; if session update request returns with an OIDC error code, session must be terminated; if session update request fails with a network error, it may be retried. |
| 0.2, 2022-03-23  | Clarifications: client can't verify ID Token's authentication method, because it cannot be given as input parameter to authentication request; requests might contain other URL parameters, that client application must ignore; session update request must be performed in user agent's background. Specify originating IP addresses of GovSSO back-channel logout requests. |
| 0.1, 2021-12-28  | Preliminary protocol changes |
| 0.01, 2021-10-26 | Initial version |
