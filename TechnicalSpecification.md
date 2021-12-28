---
permalink: TechnicalSpecification
---

This documentation is in DRAFT state and can be changed during the development of GOVSSO. Changes in API may take place until finalization of this document. 
{: .adv}

# Technical specification
{: .no_toc}
v0.01, 26.10.2021

- TOC
{:toc}

## 1 Overview

This document describes the technical characteristics of the State Single Sign-On (GOVSSO) service protocol and includes advice for implementing client application interfaces. 

GOVSSO protocol was designed following the best practices of OpenID Connect protocol and is heavily influenced of the current TARA service protocol [[TARA](https://e-gov.github.io/TARA-Doku/TechnicalSpecification)].
GOVSSO will provide, in addition to user authentication (which is provided by TARA), user session management functionality.

The terminology in this document follows primarily terminology used in OpenID Connect protocol ([[OIDC-CORE](https://openid.net/specs/openid-connect-core-1_0.html)]). In non OpenID Connect specific cases [[TARA](https://e-gov.github.io/TARA-Doku/TechnicalSpecification)] terminology is followed.

## 2 OpenID Connect compliance

GOVSSO protocol has been designed to follow the OpenID Connect protocol standard as closely as possible. The intent is to create a protocol that is compatible with a wide range of OpenID Foundation certified OIDC client applications and standard libraries. OIDC is by itself a very diverse protocol, providing multiple different workflows for a wide array of use-cases. To limit the scope of the development required to implement GOVSSO service and client applications, the full scope of OIDC protocol will not be supported. This chapter describes the main customizations that are done to GOVSSO as compared to the full OIDC protocol:

- The service supports only the authorization code flow of OIDC ([[OIDC-CORE](https://openid.net/specs/openid-connect-core-1_0.html)] "3.1.  Authentication using the Authorization Code Flow"). The authorization code flow is deemed the most secure option and is thus appropriate for public services.
- All information about an authenticated user is transferred to the application with ID Token ([[OIDC-CORE](https://openid.net/specs/openid-connect-core-1_0.html)] "2.  ID Token and 3.1.3.6.  ID Token").
- The eIDAS level of assurance (loa) is returned in the `acr` claim in ID Token, using custom claim values `high`, `substantial`, `low`. [[OIDC-CORE](https://openid.net/specs/openid-connect-core-1_0.html)] "2.  ID Token"
- The requested minimum authentication level of assurance is selected by the client application in the beginning of GOVSSO session (on initial authentication). [[OIDC-CORE](https://openid.net/specs/openid-connect-core-1_0.html)] "5.5.1.1.  Requesting the "acr" Claim".
- Available authentication methods for the user are provided based on the requested minimum authentication level of assurance.
- GOVSSO supports only a single default scope that will return person authentication data: given name, family name, birthdate, person identifier. The scope remains the same during the entire GOVSSO authentication session.
- Single-sign-on (SSO) is supported. Client applications are expected to perform session status checks to keep the authentication session alive.
- Central logout is supported according to OIDC Back-Channel logout specification [[OIDC-BACK](https://openid.net/specs/openid-connect-backchannel-1_0.html)].
- Client applications must always prove knowledge of previous ID Token to check session status or end session in GOVSSO. Different from OIDC standard protocol, the `id_token_hint` parameter is a mandatory parameter in GOVSSO requests. [[OIDC-CORE](https://openid.net/specs/openid-connect-core-1_0.html)] "3.1.2.1.  Authentication Request"

## 3 SSO session

GOVSSO will create a SSO session for each successful user authentication and stores it in its internal session storage. The SSO session is used to store user personal information and various parameters about the authentication request (for example authentication time, authentication level of assurance, authentication method used). The authentication information and parameters are not allowed to change during the authentication session.

The SSO session is linked to the user agent via a session cookie. The user is allowed to create a single SSO session per user agent but may create concurrent sessions across multiple user agents.

SSO session has a limited validity period of 15 minutes. During the SSO session validity period the user is not required to re-authenticate while logging into different client applications. Instead, they are given the option to use the authentication information stored in SSO session. GOVSSO issues ID Tokens as proof of an active SSO session. The ID Token contains the user personal information as well as the session information. All issued ID Tokens are linked to an SSO session via a session id (`sid`) claim and have the same validity period as the SSO session.

The SSO session validity period is extended by 15 minutes every time a new client application authentication request is received, or an existing authenticated client application performs a SSO session update request. This means that all client applications are contributing to keep the SSO session valid for the duration the user is still using the client applications.

SSO session expires when no new authentication requests or session update requests have been received in the last 15 minutes. This is a safety feature assuring that user SSO session is not kept alive too long after the user has finished using the client applications.

The SSO session may also be terminated before the end of its expiry time by the user. When the user initiates a logout from one client application the client application must inform GOVSSO of the logout event. The user is then given an option to terminate the SSO session and log out of all client applications related to the same SSO session.

## 4 GOVSSO process flows

### 4.1 Authentication process

In order to log a user into a client application, the client application must acquire an ID Token from GOVSSO. The ID Token will contain relevant user personal information as well as information of the SSO session. If the SSO session does not exist prior to the authentication request, a new session is created. If the session already exists but its level of assurance is lower than requested in the new request, then the previous SSO session will be terminated and a new SSO session will be created.

<p style='text-align:left;'><img src='img/govsso_tech_auth_flow.png' style='width:1000px'></p>

1. User requests protected page from client application.
2. Client application checks whether the user has been authenticated in the client application.
3. Since the user is not authenticated in the client application, the client application will construct a GOVSSO authentication request URL and redirects user agent to it.
4. GOVSSO checks whether there is a valid SSO session for given user agent.
    1. If SSO session exists and its level of assurance is equal or higher than requested the process will continue from step 8.
    2. Otherwise GOVSSO will automatically terminate the existing session and the process continues from step 5.
5. SSO session does not exist, therefore, GOVSSO needs the user to be authenticated with TARA service. GOVSSO constructs TARA authentication request and redirects user agent to it.
6. User is securely authenticated in TARA service. The detailed authentication process is described in TARA technical specification [[TARA](https://e-gov.github.io/TARA-Doku/TechnicalSpecification)]. Once the user has been authenticated TARA will redirect the user agent back to GOVSSO with the TARA authorization code.
7. GOVSSO uses the authorization code to acquire ID Token from TARA service (this request happens in GOVSSO backend system). GOVSSO will store the user identification information in its session storage.
8. GOVSSO will optionally display a user consent form. This form is displayed only when a previous SSO session was used for authentication. Meaning that in step 4.1 a valid existing session was found.
9. GOVSSO will construct its own authorization code and redirects the user agent back to client application URL.
10. The client application will use the authorization code to acquire a GOVSSO ID Token. This is done in client application backend by sending an ID Token request to GOVSSO.
11. GOVSSO will respond to the client application with a GOVSSO ID Token. GOVSSO will also internally update the SSO session expiration time to `currentTime + 15 minutes`. The client application now has the user authentication information and can display the protected page.

### 4.2 SSO session update process

Once the user has been authenticated and an SSO session created, the client application must periodically perform SSO session update requests to keep the SSO session alive in GOVSSO. In GOVSSO protocol this is done by acquiring a new token from GOVSSO service. GOVSSO session update requests are very similar to authentication requests. The only difference is that GOVSSO will not display any graphical page to the user when the user agent is redirected. To differentiate SSO session update requests from SSO authentication requests, the client application will add `prompt` parameter to the request (`prompt=none`). ([[OIDC-CORE](https://openid.net/specs/openid-connect-core-1_0.html)] "3.1.2.1.  Authentication Request".

SSO Session update request can be made only when the client application knows the previous ID Token and a valid SSO session exists. Client application must prove that it has knowledge of the SSO session context by including the `id_token_hint` parameter to request containing previous ID Token. This is a security measure and helps to assure that GOVSSO and client application have the same knowledge about the authenticated user.

If the SSO session update request fails for any reason, then the client application must perform a new SSO authentication process to get a new ID Token.

<p style='text-align:left;'><img src='img/govsso_session_update_flow.png' style='width:700px'></p>

1. User wants to access protected content in client application.
2. Client application verifies whether user has active client application session and that client application session storage contains a valid (not expired) GOVSSO ID Token.
3. Client application finds that user ID Token is about to expire and redirects user agent to GOVSSO with a valid SSO session update request. The request must include `id_token_hint` and `prompt=none` parameters.
4. GOVSSO validates the request
    1. Verifies that an SSO session is still active for user agent
    2. Verifies that the SSO session subject matches the subject in the received ID Token.
5. If all validations passed, GOVSSO will issue a new authorization code to the client application.
6. Using the authorization code, the client application makes an ID Token request to GOVSSO. GOVSSO responds with a new ID Token and a directive for the user agent to update SSO session cookie expiration date to `currentTime + 15 minutes`.
7. Client application stores ID Token into its session storage and shows protected content to user.

### 4.3 Logout process

It is expected that client applications terminate GOVSSO session as soon as the user has finished using client application(s). If a user has requested to log out of a client application, then information about this event must be sent to GOVSSO as well. GOVSSO will either terminate the SSO session automatically (if only one other client application is linked to the same SSO session) or display a logout consent form. The consent form allows user to log out from all client applications at once or just from single application.

After a successful logout the user agent is redirected back to the client application. If a technical error occurs or the logout request is invalid (for example `redirect_uri` does not match registered redirect URL in GOVSSO), then the user is redirected to GOVSSO default error page.

<p style='text-align:left;'><img src='img/govsso_logout_flow.png' style='width:800px'></p>

1. User requested to log out of client application
2. Client application does its internal application session termination procedures.
3. Client application redirects user agent to GOVSSO logout URL. The redirect must include the previous ID Token (as `id_token_hint` parameter) and a redirect URI (as `post_logout_redirect_uri` parameter) where the user agent must be redirected to after logout.
4. GOVSSO validates the logout request. If the ID Token is not linked to the active SSO session of given user agent, then nothing is done, and the user agent is redirected back to the `post_logout_redirect_uri`. GOVSSO will unlink the client application from SSO session in its session store.
5. If more client applications are linked to the same SSO session, then GOVSSO will show a logout consent page.
6. User can either select to log out of only the client application that made the logout request or from all client applications that are linked to the same SSO session.
7. If the user selected to log out from all client applications, GOVSSO will send a back-channel logout request to each linked client application and unlink client applications from GOVSSO session ([[OIDC-BACK](https://openid.net/specs/openid-connect-backchannel-1_0.html)]). 
8. If no client applications remain linked to SSO session, GOVSSO will terminate the session. User has been logged out of all client applications.
9. User agent is redirected back to the `post_logout_redirect_uri` of the client application which initiated the logout procedure.

### 4.4 Back-channel logout notifications

Each GOVSSO client application must declare support to the OIDC Back-Channel logout specification ([[OIDC-BACK](https://openid.net/specs/openid-connect-backchannel-1_0.html)]). Each client must provide a back-channel logout endpoint URL as part of their registration information. GOVSSO will send an out-of-band POST request to client application back-channel logout endpoint every time an SSO session ends.

The logout request contains a Logout Token. The Logout Token must be validated according to OIDC Back-Channel logout specification [[OIDC-BACK](https://openid.net/specs/openid-connect-backchannel-1_0.html)] "2.6.  Logout Token Validation".  After receiving a valid Logout Token from the GOVSSO, the client application locates the session(s) identified by the `iss` and `sub` Claims and/or the `sid` Claim. The client application then clears any state associated with the identified session(s). If the identified user is already logged out at the client application when the logout request is received, the logout is considered to have succeeded.

If the logout succeeded, the RP MUST respond with HTTP 200 OK.

Access to the back-channel logout endpoint should be restricted. Only GOVSSO needs to have access to given endpoint.

## 5 Tokens

### 5.1 ID Token

In GOVSSO protocol the ID Token is used as a certificate of the fact that authentication was performed. The ID Token has a concrete issuance and expiration date between which it is to be considered valid.

The ID Token is issued in JSON Web Token [[JWT](https://tools.ietf.org/html/rfc7519)] form.

**Example GOVSSO ID Token:**

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
  "family_name": "O’CONNEŽ-ŠUSLIK TESTNUMBER",
  "given_name": "MARY ÄNN",
  "amr": [
    "mID"
  ],
  "acr": "high",
  "at_hash": "AKIDtvBT2JS_02tkl_DvuA",
  "nonce": "POYXXoyDo49deYC3o5_rG-ig3U4o-dtKgcym5SyHfCM",
  "sid": "f5ab396c-1490-4042-b073-ae8e003c7258",
}
````
**ID Token claims**

| ID Token element (claim)   | example           |     explanation       |
|----------------------------|------------------ |-----------------------|
| jti | `"jti":"663a35d8-92ec-4a8d-95e7-fc6ca90ebda2"` |  ID Token unique identifier ([[JWT](https://tools.ietf.org/html/rfc7519)] "4.1.7.  jti (JWT ID) Claim"). |
| iss | `"iss":"https://govsso.ria.ee/"` |  Issuer Identifier, as specified in  [[OIDC-CORE](https://openid.net/specs/openid-connect-core-1_0.html)]. |
| aud | `"aud": [`<br> `"sso-client-1"` <br>`]` <br><br> or<br><br> `"aud": "sso-client-1"` |  Unique ID of a client application in GOVSSO client database. ID belongs to the client that requested authentication (the value of `client_id` field is specified in authentication request). <br><br> String or array of strings. A single aud value is present in GOVSSO tokens. |
| exp | `"exp": 1591709871` |  The expiration time of the ID Token (in Unix _epoch_ format). |
| iat | `"iat": 1591709811` |  The time of issue of the ID Token (in Unix _epoch_ format). |
| sub | `"sub": "EE60001018800"` |  The identifier of the authenticated user (personal identification code or eIDAS identifier) with the prefix of the country code of the citizen (country codes based on the ISO 3166-1 alpha-2 standard). The subject identifier format is set by TARA authentication service ID Token [[TARA](https://e-gov.github.io/TARA-Doku/TechnicalSpecification)] "4.3.1 Identity token". NB! in case of eIDAS authentication the maximum length is 256 characters.|
| birthdate | `"birthdate": "2000-01-01"` |  The date of birth of the authenticated user in the ISO_8601 format. Only sent in the case of persons with Estonian personal identification code and in the case of eIDAS authentication. |
| given_name | `"given_name": "MARY ÄNN"` |  The first name of the authenticated user (the test name was chosen because it consists special characters). |
| family_name | `"family_name": "O’CONNEŽ-ŠUSLIK TESTNUMBER"` |  The surname of the authenticated user (the test name was selected because it includes special characters). |
| amr | `"amr": [ "mID" ]` |  Authentication method reference. The authentication method used for user authentication. A single `amr` value is present in GOVSSO tokens. Possible values:<br><br> `mID` - Mobile-ID<br> `idcard` - Estonian ID card<br> `eIDAS` - European cross-border authentication<br> `smartid` - Smart-ID<br><br> Available authentication methods depend on TARA authentication service and the list may be extended in the future [[TARA](https://e-gov.github.io/TARA-Doku/TechnicalSpecification)] "4.1 Authentication request". |
| nonce | `"nonce": "POYXXoyDo49deYC3o5_rG-ig3U4o-dtKgcym5SyHfCM"` |  Security element. The authentication request’s `nonce` parameter value. Value is present only in case the `nonce` parameter was sent in the authentication request. |
| acr | `"acr": "high"` |  Authentication Context Class Reference. Signals the level of assurance of the authentication method that was used. Possible values: `low`, `substantial`, `high`. The element is not used if the level of assurance is not applicable or is unknown. |
| at_hash | `"at_hash": "AKIDtvBT2JS_02tkl_DvuA"` |  The access token hash calculated as described in OIDC specification [[OIDC-CORE](https://openid.net/specs/openid-connect-core-1_0.html)]. |
| sid | `"sid": "f5ab396c-1490-4042-b073-ae8e003c7258"` |  Session ID - String identifier for a GOVSSO session. This represents a session of a User Agent. Different sid values are used to identify distinct sessions at GOVSSO. |

ID Token may consist other OpenID Connect protocol based fields that are not supported in GOVSSO.

### 5.2 Logout Token

GOVSSO sends a JWT similar to an ID Token to client applications called a Logout Token ([OIDC-BACK](https://openid.net/specs/openid-connect-backchannel-1_0.html) "2.4.  Logout Token") to request that they log out a GOVSSO session or user. The token issuer is GOVSSO and the audience is the client application. Upon receiving a Logout Token the client application is expected to validate the token to make sure that it is a valid logout request. In case the token is valid the client application session (session between client application and user agent) must be ended.

Issued Logout Tokens are linked to ID Tokens via the `sid` claim. Each client application is expected to internally keep track of the ID Token `sid` claim and client application session relations. Meaning that each application session during which GOVSSO authentication was used, must hold the `sub` and `sid` values of ID Tokens that were issued during that application session.

A Logout Token contains a `sub` claim, a `sid` claim, or both. If a `sid` claim is present, the client application must log out all client application sessions that contain the same `sid` value. If only a `sub` claim is present, then all client application sessions with the same `sub` value must be logged out. Logout Tokens with only a `sub` value are issued only if the logout was initiated by GOVSSO server (not a client application).

OIDC Logout Tokens can be encrypted but GOVSSO Logout Tokens are not encrypted.

***Example GOVSSO Logout Token payload***

````
{
  "aud": [
    "sso-client-1"
  ],
  "events": {
    "http://schemas.openid.net/event/backchannel-logout": {}
  },
  "sub": "EE60001018800",
  "iat": 1591958452,
  "iss": "https://govsso.ria.ee/",
  "jti": "c0cfc91a-cdf5-4706-ad26-847b3a3fb937",
  "sid": "9038c51d-719f-40e1-9322-a7920a2087c8"
}
````
***Logout Token claims***

| Logout Token element (claim)   | compulsory       |    example        |     explanation       |
|--------------------------------|------------------|------------------ |-----------------------|
| iss | yes |  `"iss": "https://govsso.ria.ee/"` |  Issuer Identifier, as specified in  [[OIDC-CORE](https://openid.net/specs/openid-connect-core-1_0.html)]. |
| sub | no |  `"sub": "EE60001018800"` | The identifier of the authenticated user (personal identification code or eIDAS identifier) with the prefix of the country code of the citizen (country codes based on the ISO 3166-1 alpha-2 standard). The subject identifier format is set by TARA authentication service ID Token [[TARA](https://e-gov.github.io/TARA-Doku/TechnicalSpecification)] "4.3.1 Identity token". NB! in case of eIDAS authentication the maximum length is 256 characters.  |
| events | yes | `"events": {`<br> `"http://schemas.openid.net/event/backchannel-logout": {}`<br> `}` |  Claim whose value is a JSON object containing the member name `http://schemas.openid.net/event/backchannel-logout.` This declares that the JWT is a Logout Token. The corresponding member value MUST be a JSON object and SHOULD be the empty JSON object `{}`. [OIDC-BACK](https://openid.net/specs/openid-connect-backchannel-1_0.html) "2.4.  Logout Token"|
| aud | yes |  `"aud": [`<br> `"sso-client-1"` <br>`]` <br><br> or<br><br> `"aud": "sso-client-1"` | Audience(s), as specified in [[OIDC-CORE](https://openid.net/specs/openid-connect-core-1_0.html)]. |
| iat | yes |  `"iat": 1591958452` |  Issued at time, as specified IN [[OIDC-CORE](https://openid.net/specs/openid-connect-core-1_0.html)]. |
| jti | yes |  `"jti": "c0cfc91a-cdf5-4706-ad26-847b3a3fb937"` |  Unique identifier for the token, as specified in [[OIDC-CORE](https://openid.net/specs/openid-connect-core-1_0.html)]. |

## 6 Requests

### 6.1 Authentication request

**Request**

An authentication request is a HTTP GET request by which the user is redirected from the client application to GOVSSO for authentication.

***Example GOVSSO authentication request***
````
GET https://govsso.ria.ee/oauth2/auth?
 
redirect_uri=https%3A%2F%2Fclient.example.com%2Fcallback&
scope=openid&
state=hkMVY7vjuN7xyLl5&|
response_type=code&
client_id=58e7ba35aab5b4f1671a&
ui_locales=en&
nonce=fsdsfwrerhtry3qeewq&
prompt=consent&
acr_values=substantial
````
(for better readability, the parts of the HTTP request are divided onto several lines)

***Request parameters***

| URL element   | compulsory       |    example        |     explanation       |
|---------------|------------------|------------------ |-----------------------|
| protocol, host, port and path | yes |  `https://govsso.ria.ee/oauth2/auth` |  `/oauth2/auth` is the OpenID Connect-based authentication endpoint of the GOVSSO service (the concept of ‘authorization’ originates from the OAuth 2.0 standard protocol).<br><br> The URL is provided from OIDC server public discovery service: `https://govsso.ria.ee/.well-known/openid-configuration` as `authorization_endpoint` parameter. |
| client_id | yes | `client_id=58e7ba35aab5b4f1671a`  |  Client identifier. The client identifier is issued by RIA upon registration of the client application as a user of the authentication service. |
| redirect_uri | yes | `redirect_uri=https%3A%2F%2Fclient.example.com%2Fcallback`  |  Redirect URL ([[OAUTH](https://tools.ietf.org/html/rfc6749)] "3.1.2. Redirection Endpoint"). The redirect URL is selected by the institution. The redirect URL may include the query component. URL encoding should be used, if necessary [[URLENC](https://en.wikipedia.org/wiki/Percent-encoding)].<br> It is not permitted to use the URI fragment component (`#` and the following component; [[URI](https://tools.ietf.org/html/rfc3986)] "3.5. Fragment").<br> The URL protocol, host, port and path must match one of the pre-registered redirect URLs of given client application registration metadata (see `client_id` parameter). |
| scope | yes | `scope=openid` |  The authentication scope. Space delimited list of requested scopes.<br><br> `openid` scope is compulsory to signal that this is an OIDC authentication request.<br> In the default `scope` of openid GOVSSO will issue ID Tokens with the following attributes:<br><br> `sub` (physical person identifier)<br> `given_name`<br> `family_name`<br> `birthdate`<br><br> Presence of given attribute values will depend on the amount of information that is returned within TARA ID Tokens. |
| state | yes |  `state=hkMVY7vjuN7xyLl5` |  Security code against false request attacks (cross-site request forgery, CSRF). Read more about formation and verification of state under "7.3 Protection against false request attacks". |
| response_type | yes |  `response_type=code` |  Determines the manner of communication of the authentication result to the server. Only value `code` is allowed as only authorization code flow is supported by GOVSSO |
| ui_locales | no |  `ui_locales=et` |  Selection of the user interface language. The following languages are supported: `et`, `en`, `ru`. By default, the user interface is in Estonian language. The client can select the desired language. This will also set the GUI language for TARA service views. |
| nonce | no |  `nonce=fsdsfwrerhtry3qeewq` |  A unique parameter which helps to prevent replay attacks based on the OIDC protocol ([[OIDC-CORE](https://openid.net/specs/openid-connect-core-1_0.html)] "3.1.2.1. Authentication Request"). |
| acr_values | no |  `acr_values=substantial` |  The minimum required level of authentication based on the eIDAS level of assurance (LoA). Allowed values are: `low`, `substantial`, `high`. `substantial` is used by default if the value has not been set in request.<br><br> GOVSSO will store the authentication level of assurance in the SSO session object as an Authentication Context Class Reference (`acr`) claim, from TARA ID Token response. Upon each GOVSSO authentication request, GOVSSO will check that the requested level of assurance (`acr_values` parameter value) is lower or equal to the `acr` claim value of the GOVSSO session. If the SSO session `acr` value (level of assurance) is lower than requested, the previous GOVSSO session is automatically terminated and a new authentication is requested from TARA. After successful authentication a new SSO session is created. |
| prompt | no |  `prompt=consent` |  Required parameter signalling GOVSSO that user consent should be asked before returning information to the client application. If GOVSSO cannot obtain consent, it will return an error, typically `consent_required`. |

**Response**

GOVSSO server will respond with a HTTP 302 Found message to redirect user agent back to client application URL.

The user agent is redirected to the `redirect_uri` included in the authentication request sent by the client application. In the redirect request, an authorization code (`code` parameter) is appended to the `redirect_uri` by GOVSSO. Using the received code the client application can then request ID Token from GOVSSO. Technically, a HTTP status code `302 Found` redirect request is used for redirecting.

The state value will also be returned to the client application, making it possible to perform CSRF validation on client application side.

***Example GOVSSO authentication redirect***
````
HTTP GET https://client.example.com/callback?
 
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

If GOVSSO is unable to process an authentication request – there is an error in the request, TARA was unable to authenticate user or another error has occurred – GOVSSO transfers an error message (URL parameter `error`) and the description of the error (URL parameter `error_description`) in the redirect request.

GOVSSO relies on the OpenID Connect standard error messages [[OIDC-CORE](https://openid.net/specs/openid-connect-core-1_0.html)] "3.1.2.6. Authentication Error Response" and [[OAUTH](https://tools.ietf.org/html/rfc6749)] "4.1.2.1. Error Response". The error messages are always displayed in English.

`state` is also returned but no authorization code (`code`) is sent.

***Example GOVSSO authentication error***
````
HTTP GET https://client.example.com/callback?
 
error=invalid_scope&
error_description=Required+scope+%3Copenid%3E+not+provided.+GOVSSO+does+not+allow+this+request+to+be+processed&
state=hkMVY7vjuN7xyLl5
````
(for better readability, the parts of the HTTP request are divided onto several lines)

**Authentication termination response**

The user may also return to the client application without choosing an authentication method and completing the authentication process (via ‘Back to the service provider’ link). This option is provided for the cases in which the user has clicked ‘Log in’ in the client application but does not actually wish to log in. The URL where to direct the user must be defined upon registering for GOVSSO service. NB! The OpenID Connect protocol `redirect_url` and the URL described here have different meanings.

### 6.2 ID Token request

**Request**

The ID Token request is a HTTP POST request which is used by the client application to request the ID Token from the GOVSSO service.

***Example GOVSSO token request***
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
| protocol, host, port and path | query |  `https://govsso.ria.ee/oauth2/token` |  GOVSSO server token endpoint URL. Published in GOVSSO discovery endpoint `token_endpoint` parameter value. |
| grant_type | body |  `grant_type=authorization_code` |  The `authorization_code` value required based on the protocol. [[OIDC-CORE](https://openid.net/specs/openid-connect-core-1_0.html)] "3.1.3.1.  Token Request" |
| code | body |  `code=SplxlOBeZQQYbYS6WxSbIA` |  The authorization code received from the authentication service. |
| redirect_uri | body |  `redirect_uri=https%3A%2F%2client.example.com%2Fcallback` |  The redirect URL sent in the authentication request. |

**Response**

GOVSSO server verifies that the ID Token is requested by the right application and issues the ID Token included in the response body (HTTP response body).

***Example GOVSSO token endpoint response***
````
HTTP/1.1 200 OK
Content-Type: application/json
Cache-Control: no-store
Pragma: no-cache
 
{
 "access_token": "EKN-4fXC4n1RdkegKk-M0DRxZ8RwJYZ_EwW-9zLCYcA.7GT7Xq2deLvWzrrFq6f0DNwL6INW2PYRDPPEFMbws1o",
 "token_type": "Bearer",
 "expires_in": 3600,
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
| access_token |  OAuth 2.0 access token. With the access token the client application can request authenticated user’s data from userinfo endpoint.<br> **Not used in GOVSSO because GOVSSO session management is purely ID Token dependent. All user data is already available in the ID Token.** |
| token_type |  OAuth 2.0 access token type with `bearer` value. Not used in GOVSSO. |
| expires_in |  The validity period of the OAuth 2.0 access token. Not used in GOVSSO. |
| id_token |  ID Token, encapsulated in JWS Compact Serialization form ([[JWS](https://tools.ietf.org/html/rfc7515)] chapter 3.1). The ID Token itself is issued in JSON Web Token [[JWT](https://tools.ietf.org/html/rfc7519)] format .|

Response body might contain other fields, that client application must ignore.

**Error response**

In case the token endpoint encounters an error and can not issue valid tokens, an error response is generated according to OIDC Core specification [[OIDC-CORE](https://openid.net/specs/openid-connect-core-1_0.html)].

### 6.3 SSO session update request

**Request**

Client applications must periodically check the SSO authentication session validity on GOVSSO server. Session update requests will also signal GOVSSO server that the user is still active in the client application and the authentication session expiration time can be extended.

The process of acquiring a new authentication token is similar to initial user authentication.

1. The client application redirects user agent to GOVSSO authorization endpoint for an authorization code.
2. Upon receiving the authorization code, the client application makes a request to the GOVSSO token endpoint.

***Example GOVSSO session update request***
````
GET https://govsso.ria.ee/oauth2/auth?
 
redirect_uri=https%3A%2F%2client.example.com%2Fcallback&
scope=openid&
state=hkMVY7vjuN7xyLl5&|
response_type=code&
client_id=58e7ba35aab5b4f1671a&
ui_locales=en&
nonce=fsdsfwrerhtry3qeewq&
acr_values=substantial&
prompt=none&
id_token_hint=eyJhbGciOiJSUzI1NiIsImtpZCI6InB1YmxpYzo...TvE
````
***Request parameters***

| URL element   | compulsory       |    example        |     explanation       |
|---------------|------------------|------------------ |-----------------------|
| protocol, host, port and path | yes |  `https://govsso.ria.ee/oauth2/auth` |  `/oauth2/auth` is the OpenID Connect-based authentication endpoint of the GOVSSO service (the concept of ‘authorization’ originates from the OAuth 2.0 standard protocol [[OAUTH](https://tools.ietf.org/html/rfc6749)]).<br><br> The URL is provided from OIDC server public discovery service: `https://govsso.ria.ee/.well-known/openid-configuration` `authorization_endpoint` parameter. |
| client_id | yes |   |  Application identifier. The application identifier is issued to the institution by RIA upon registration of the client application as a user of the authentication service. |
| redirect_uri | yes | `redirect_uri=https%3A%2F%2Fclient.example.com%2Fcallback`  |  Redirect URL. The redirect URL is selected by the institution. The redirect URL may include the query component. URL encoding should be used, if necessary [[URLENC](https://en.wikipedia.org/wiki/Percent-encoding)].<br> It is not permitted ([[OAUTH](https://tools.ietf.org/html/rfc6749)] "3.1.2. Redirection Endpoint") to use the URI fragment component (`#` and the following component; [[URI](https://tools.ietf.org/html/rfc3986)] "3.5. Fragment").<br> The URL protocol, host, port and path must match one of the pre-registered redirect URLs of given client application registration metadata. |
| scope | yes | `scope=openid` |  The authentication scope.<br><br> `openid` scope is compulsory to signal that this is an OIDC authentication request. Scope values are case sensitive. Only openid is allowed, other values cause an error with code `invalid_scope` to be returned<br> In the default scope of `openid` GOVSSO will issue ID Tokens with the following attributes:<br><br> `sub` (physical person identifier)<br> `given_name`<br> `family_name`<br> `birthdate`<br><br> Presence of given attribute values will depend on the amount of information that is returned within TARA ID Tokens. |
| state | yes |  `state=hkMVY7vjuN7xyLl5` |  Security code against false request attacks (cross-site request forgery, CSRF). Length of `state` parameter must be minimally 8 characters. Read more about formation and verification of state under 'Protection against false request attacks'. |
| response_type | yes |  `response_type=code` |  Determines the manner of communication of the authentication result to the server. The method of authorization code is supported (authorization flow of the OpenID Connect protocol) and it is referred to the code value. |
| ui_locales | no |  `ui_locales=et` |  Selection of the user interface language. The following languages are supported: `et`, `en`, `ru`. By default, the user interface is in Estonian language. The client can select the desired language. This will also set the GUI language for TARA service views. |
| nonce | no |  `nonce=fsdsfwrerhtry3qeewq` |  A unique parameter which helps to prevent replay attacks based on the OIDC protocol [[OIDC-CORE](https://openid.net/specs/openid-connect-core-1_0.html)] "3.1.2.1. Authentication Request". The nonce parameter is not compulsory. |
| acr_values | no |  `acr_values=substantial` |  The minimum required level of authentication based on the eIDAS level of assurance (LoA). Allowed values are: `low`, `substantial`, `high`. `substantial` is used by default if the value has not been set in request.<br><br> GOVSSO will store the authentication level of assurance in the SSO session object as an Authentication Context Class Reference (`acr`) claim, from TARA ID Token response. Upon each GOVSSO authentication request, GOVSSO will check that the requested level of assurance (`acr_values` parameter value) is lower or equal to the `acr` claim value of the GOVSSO session. If the SSO session `acr` value (level of assurance) is lower than requested, the previous GOVSSO session is automatically terminated and a new authentication is requested from TARA. After successful authentication a new SSO session is created. |
| prompt | yes |  `prompt=none` |  Signals GOVSSO server that it MUST NOT display any authentication or consent view to the user. An error is returned if user is not already authenticated in GOVSSO or the client application does not have pre-configured consent for the requested scope, acr_values or does not fulfill other conditions for processing the request. The error code will typically be login_required, interaction_required, or another code defined in OIDC standard ([[OIDC-CORE](https://openid.net/specs/openid-connect-core-1_0.html)] "3.1.2.6. Authentication Error Response"). |
| id_token_hint | yes |  `id_token_hint=eyJhbGciOiJSUzI1NiIsImtpZCI6InB1YmxpYzo...TvE` |  ID Token previously issued by GOVSSO being passed as a hint about the user's current or past authenticated session with the client application. If the user identified by the ID Token is logged in or is logged in by the request, then GOVSSO returns a positive response; otherwise, it WILL return an error. **Encryption of the id_token_hint parameter is not supported in GOVSSO.** |

**Response**

GOVSSO server will respond with a HTTP 302 Found message to redirect user agent back to client application URL.

The user agent is redirected to the `redirect_uri` included in the authentication request sent by the client application. In the redirect request, an authorization code (`code` parameter) is appended to the `redirect_uri` by GOVSSO. Using the received code the client application can then request ID Token from GOVSSO. Technically, a HTTP status code `302 Found` redirect request is used for redirecting.

The state value will also be returned to the client application, making it possible to perform CSRF validation on client application side.

***Example GOVSSO session update redirect***
````
HTTP GET https://client.example.com/callback?
 
code=71ed5797c3d957817d31&
state=hkMVY7vjuN7xyLl5
````
***Request parameters***

| URL element   |    example       |    explanation    |
|---------------|------------------|------------------ |
| protocol, host, port and path |  `https://client.example.com/callback` |  Matches the `redirect_uri` value sent in the authentication request. |
| code |  `code=71ed579...` |  The authorization code is a single ‘permission note’ to receive the ID Token. |
| state |  `state=hkMVY7vjuN7xyLl5` |  Security code against false request attacks. The security code received in the authentication request is mirrored back. Read more about forming and verifying `state` from ‘Protection against false request attacks’. |

**Error response**

If GOVSSO is unable to process an session update request – there is an error in the request or another error has occurred – GOVSSO transfers an error message (URL parameter `error`) and the description of the error (URL parameter `error_description`) in the redirect request.

GOVSSO relies on the OpenID Connect standard on error messages [[OIDC-CORE](https://openid.net/specs/openid-connect-core-1_0.html)] "3.1.2.6. Authentication Error Response" and [[OAUTH](https://tools.ietf.org/html/rfc6749)] "4.1.2.1. Error Response". The error messages are always displayed in English.

`state` is also redirected but no authorization code (`code`) is sent. E.g.:

***Example GOVSSO session update error***
````
HTTP GET https://client.example.com/callback?
 
error=authentication_required&
error_description=Authenticated+subject+does+not+match+provided+id_token_hint&
state=hkMVY7vjuN7xyLl5
````

### 6.4 SSO logout request

**Request**

A client application can notify the GOVSSO that the user has logged out of client application and might want to log out of GOVSSO as well. In this case, the client application, after having logged the user out of the client application, redirects the user's User Agent to GOVSSO's logout endpoint URL. This URL is normally obtained via the `end_session_endpoint` element of GOVSSO Discovery response or may be learned via other mechanisms.

***Example GOVSSO logout request***
````
GET https://govsso.ria.ee/oauth2/sessions/logout?
 
 
id_token_hint=eyJhbGciOiJSUzI1NiIsImtpZCI6InB1YmxpYzo3Njc2MG...VkDzh0LYvs
post_logout_redirect_uri=https%3A%2F%2Fclient.example.com%2Fcallback&
state=0dHJpYnV0ZXMiOnsiZGF0ZV9vZl9iaXJ&
ui_locales=et
````
***Request parameters***

| URL element   | compulsory       |    example        |     explanation       |
|---------------|------------------|------------------ |-----------------------|
| protocol, host, port and path | yes |  `https://govsso.ria.ee/oauth2/sessions/logout` |  `/oauth2/auth` is the OpenID Connect-based logout endpoint of the GOVSSO service. Described in OIDC session management specification [[OIDC-SESSION](https://openid.net/specs/openid-connect-session-1_0.html)] "2.1.  OpenID Provider Discovery Metadata" <br><br> The URL is provided from OIDC server public discovery service: `https://govsso.ria.ee/.well-known/openid-configuration end_session_endpoint` parameter. |
| post_logout_redirect_uri | yes | `redirect_uri=https%3A%2F%2Fclient.example.com%2Fcallback` |  Redirect URL. The redirect URL is selected by the institution. The redirect URL may include the query component. URL encoding should be used, if necessary [[https://en.wikipedia.org/wiki/Percent-encoding](https://en.wikipedia.org/wiki/Percent-encoding).<br> It is not permitted ([[OAUTH](https://tools.ietf.org/html/rfc6749)] "3.1.2. Redirection Endpoint") to use the URI fragment component (`#` and the following component; [[URI](https://tools.ietf.org/html/rfc3986)] "3.5. Fragment".<br> The URL protocol, host, port and path must match one of the pre-registered redirect URLs of given client application. Client application is determined by the contents of the ID Token (token audience must belong to a registered GOVSSO client application).<br> Different from OIDC session management specification, this parameter is considered mandatory in GOVSSO. In GOVSSO user logout flow we expect that the user is always redirected back to the client application that initiated the logout process. The `post_logout_redirect_uri` should point to the client application front page or a client application internal redirect url. |
| id_token_hint | yes |  `id_token_hint=eyJhbGciOiJSU...TvE` |  ID Token previously issued by GOVSSO being passed as a hint about the user's current or past authenticated session with the client application. If the user identified by the ID Token is not logged in or is logged in by the request, then GOVSSO returns a positive response; otherwise, it WILL return an error. **`id_token_hint` encryption is not supported.** |
| state | no |  `state=hkMVY7vjuN7xyLl5` |  Security code against false request attacks (cross-site request forgery, CSRF). Length of `state` parameter must be minimally 8 characters. Read more about formation and verification of state under 'Protection against false request attacks'.<br> If included in the logout request, the TARA passes this value back to the RP using the state query parameter when redirecting the user agent back to the client application.<br> Using the state parameter is not mandatory for login callbacks. It is expected that the user was already logged out of the client application before calling GOVSSO logout endpoint. |
| ui_locales | no |  `ui_locales=et	` |  Selection of the user interface language. The following languages are supported: `et`, `en`, `ru`. By default, the user interface is in Estonian language.<br> If the user was logged into a single client application, then no GUI prompt will be displayed to the user. |

**Response**

After successful logout request, GOVSSO will redirect the user agent back to the client application.

***Example GOVSSO logout redirect***
````
GET https://client.example.com/callback?state=hkMVY7vjuN7xyLl5
````
**Error response**

If the logout request processing is unsuccessful (for example the client and post_logout_redirect_uri do not match or a technical error in GOVSSO prevents user logout), the user agent will be redirected to GOVSSO error URL. According to OIDC the logout request does not provide means to redirect the user back to client application with an error message. Therefore, the user logout flow will end in GOVSSO instead of the client application. GOVSSO should display some form of request correlation id to aid customer support.

### 6.5 Back-channel logout request

**Request**

All GOVSSO clients must support OIDC Back-channel logout functionality. According to back-channel logout specification [[OIDC-BACK](https://openid.net/specs/openid-connect-backchannel-1_0.html)] GOVSSO server will send an out-of-band logout request to each client application when the user has requested to end the GOVSSO session. GOVSSO will include a Logout Token with each request so that the client applications can verify the logout event.

Relying Parties supporting back-channel-based logout register a back-channel logout URI with GOVSSO as part of their client registration [[OIDC-BACK](https://openid.net/specs/openid-connect-backchannel-1_0.html)] "2.2.  Indicating RP Support for Back-Channel Logout".

The back-channel logout URI must be an absolute URI as defined by Section 4.3 of  [[URI](https://tools.ietf.org/html/rfc3986)]. The back-channel logout includes an `application/x-www-form-urlencoded` formatted query component. The back-channel logout URI will not include a fragment component.

Client applications must perform Logout Token validation to check if the received request is a valid GOVSSO logout request. Validation must be performed according to OIDC protocol [[OIDC-BACK](https://openid.net/specs/openid-connect-backchannel-1_0.html)] "2.6. Logout Token Validation". If at any point token validation fails, the client application is expected to respond with a HTTP status code 400 Bad Request. If a session with a matching session ID is not found but the Logout Token is otherwise valid, then the client application should respond with HTTP status code 200 OK.

***Example GOVSSO back-channel logout request***
````
POST /back-channel-logout HTTP/1.1
Host: client.example.com
Content-Type: application/x-www-form-urlencoded
 
logout_token=eyJhbGciOiJSUzI1NiIsImtpZCI...p-wczlqgp1dg
````
***Request parameters***

| Parameter   | compulsory     |    example        |     explanation       |
|-------------|----------------|------------------ |-----------------------|
| protocol, host, port and path | yes |  `https://client.example.com:443/back-channel-logout` |  Client application must authorize access to GOVSSO to an internal URL and port. Access to the port should be limited based on IP address. The port must be protected with TLS. GOVSSO must trust the logout endpoint server certificate. |
| logout_token | yes |  `logout_token=eyJhbGciOiJSUz...qgp1dg` |  GOVSSO sends a JWT token similar to an ID Token to client applications called a Logout Token to request that they log out. Logout Token will give the client application exact information about the subject and the session (see the sid claim in ID Token) that should be logged out. The token is signed by GOVSSO with the same secret key that is used for singing issued ID Tokens. |

## 7 Security operations

### 7.1 Verification of the ID Token and Logout Token

The client application must always verify the ID Token and Logout Token. The client application must implement all the verifications based on OpenID Connect ([[OIDC-CORE](https://openid.net/specs/openid-connect-core-1_0.html)] and [[OIDC-BACK](https://openid.net/specs/openid-connect-backchannel-1_0.html)]) and OAuth 2.0 protocols ([OAUTH](https://tools.ietf.org/html/rfc6749)).

The client must verify token’s:

- signature
- issuer
- addressee
- validity
- authentication method (in case of ID Token)
- eIDAS level of assurance of (in case of ID Token)


**Verifying the signature**

The ID Token and Logout Tokens are signed by the GOVSSO authentication service. The signature meets the JWT standard ([JWT](https://tools.ietf.org/html/rfc7519)). GOVSSO uses the same key for signing ID Token and Logout Token.

`RS256` signature algorithm is used, the client application must be able to verify the signature using this algorithm. It would be reasonable to use a standard JWT library which supports all JWT algorithms. The change of algorithm is considered unlikely, but possible in case a security vulnerability is detected in the `RS256`.

For the signature verification the GOVSSO public signature key must be used. The public signature key is published at the public signature key endpoint (see chapter “8. GOVSSO endpoints”).

The public signature key is stable - the public signature key will be changed according to security recommendations. However, the key can be changed without prior notice for security reasons. Key exchange is carried out based on [OIDC-CORE](https://openid.net/specs/openid-connect-core-1_0.html) standard.

The public signature key has an identifier (`kid`). The key identifier is aligned with the best practices of [OIDC-CORE](https://openid.net/specs/openid-connect-core-1_0.html#Signing) and OAuth 2.0 ([OAUTH](https://tools.ietf.org/html/rfc6749)) that enables the key exchange without the service interruption.

We recommend buffering the GOVSSO public key (together with `kid` value) in order to decrease the amount of requests made to GOVSSO server. However, it is allowed to request the key on each validation.

For signature validation following checks needs to be performed on client application:

1. Read the `kid` value from the JWT header. 
2. a. If the client application do not buffer the public key, make request to public signature key endpoint and select key corresponding to `kid` value received from JWT header. <br>
b. If client application buffers the public key (it needs to be buffered together with `kid` value), it needs to compare the `kid` value from JWT header with buffered `kid` value. If they match, buffered key can be used. If not client application needs to make request to public signature key endpoint and select key corresponding to `kid` value received from JWT header and buffer it. 
3. Validate the signature using the key corresponding to `kid` value from the JWT header.

NB! "Hardcoding" the key to client application configuration must be avoided. The key change will be typically communicated (except of urgent security reasons), but manual key handling will result downtime in client application for the period when GOVSSO is already using the new key until the new key is taken use in client application.

**Trust of the public signature key endpoint**

HTTPS must be used on requests to GOVSSO server. The client application must verify GOVSSO server’s certificate.

**Verifying the issuer of tokens**

The `iss` value of the ID Token element must be `https://govsso-demo.ria.ee/` (for GOVSSO test environment) or `https://govsso.ria.ee/` (for GOVSSO production environment).

**Verifying the addressee of the tokens**

The client application must verify whether the token received was issued for them. For this purpose, it must be made sure that the `aud` value of the ID Token element matches the `client_id` issued upon registration of the client application.

**Verifying the validity of the tokens**

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

**Verifying the eIDAS level of assurance**

In order to prevent access to cross-border authentication tools with a lower security level, it must be verified that the authentication level in the `acr` claim of ID Token is not lower than the minimum level of assurance allowed.

For example, if the client application wants to use only authentication methods with eIDAS level of assurance `high` and has specified the value in the `acr_values` parameter, then only ID Tokens with `acr` claim with value `high` can be accepted.

In case the level of assurance in the authentication request using `acr_values` parameter is not specified, the ID Token must be equal to a level of assurance `substantial` or `high`.

### 7.2 Creating a session

After a successful verification of the ID Token, the client application will create a session with the user. The client application is responsible for creating and holding the sessions. The methods for doing this are not included in the scope of the GOVSSO authentication service.

All tokens issued to the client application will contain a `sid` claim. This claim can be used to link client application session to SSO session. Client applications must store the latest ID Token issued for each session. The previous token is required to create session update requests and logout requests.

The client application session expiration time should be slightly shorter than GOVSSO ID Token expiration time. This way the client application is forced to always request a new ID Token before the last ID Token expires. GOVSSO server will reject requests that contain ID Tokens that have expired.

Logout Tokens usually contain the same `sid` claim. When a Logout Token is received the client application must find all application sessions that contain ID Tokens with the same `sid` claim value and terminate them (force the user to log out on the same user agent).

When application session was terminated internally by GOVSSO, the Logout Token may instead contain only a `sub` claim. In this case the client application is expected to terminate all session that contain ID Tokens with matching `sub` value (force the user to log out on all user agents).

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

Logging must enable the reconstruction of the course of the communication between GOVSSO and the client application for each occasion of using GOVSSO. For this purpose, all following requests and responses must be logged by GOVSSO as well as by the client application: authentication_request, authentication_redirect, token_request, session_update_request, session_update_redirect, logout_request, logout_redirect, backchannel_logout. As the volumes of data transferred are not large, the URL as well as the ID Token must be logged in full. The retention periods of the logs should be determined based on the importance of the client application. We advise using 1 … 7 years. In case of any issue, please submit an excerpt from the log (Which requests were sent to GOVSSO? Which responses were received?).

## 8 GOVSSO endpoints

| Endpoint description | Endpoint | Comment |
|----------------------|----------|---------|
| server discovery | /.well-known/openid-configuration |Public endpoint for GOVSSO server OpenID Connect configuration information. Usually provided as standard endpoint for OIDC implementations that support service discovery [[OIDC-DISCOVERY](https://openid.net/specs/openid-connect-discovery-1_0.html)] "4.1 OpenID Provider Configuration Request". |
| key info | /jwks |  JSON Web Key Set document for GOVSSO service. Publishes at minimum the public key that client applications must use to validate ID Token and Logout Token signatures [[JWK](https://tools.ietf.org/html/draft-ietf-jose-json-web-key-41)]. |
| authorization | /auth |  OAuth 2.0 authorization endpoint. Used for GOVSSO session update requests and authentication requests. [[OAUTH](https://tools.ietf.org/html/rfc6749)] "3.1.  Authorization Endpoint". |
| token | /token | GOVSSO endpoint to obtain ID Token [[OIDC-CORE](https://openid.net/specs/openid-connect-core-1_0.html)] "3.1.3.  Token Endpoint". In addition access tokens are returned for OAuth 2.0 compliance but their use in GOVSSO protocol is not required. |
| logout | /sessions/logout | GOVSSO client application initiated logout endpoint. [[OIDC-SESSION](https://openid.net/specs/openid-connect-session-1_0.html)] "5. RP-Initiated Logout". |

## Change history

| Version, Date | Description |
|---------------|-------------|
| 0.01, 26.10.2021 | Initial version |
