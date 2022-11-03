---
permalink: Testing
---

# Testing
{: .no_toc}

- TOC
  {:toc}

## 1 Overview

This page holds guidelines for GovSSO clients to ensure a secure and working integration with GovSSO.

The information contained here focuses on testing integration with GovSSO from the point of view of the client application.

If you need further advice than present in GovSSO documentation or wish to report a possible bug, please send an e-mail to [help@ria.ee](help@ria.ee).

## 2 Prerequisites

In order to perform testing, the integrating party has to first [submit an application to join the GovSSO demo environment](Application).

The integrator should beforehand acquaint with GovSSO [technical specifications](TechnicalSpecification).

Demo GovSSO uses same authentication methods as demo [TARA](https://e-gov.github.io/TARA-Doku/), and therefore it is also important to be aware of [different authentication methods and test accounts](https://www.id.ee/en/rubriik/for-developer/).

## 3 Testing

Once your client application has been integrated with demo GovSSO and you have familiarized yourself with other prerequisites, you are ready to start the testing process.

Please note that the processes described below include both workflows through the user interface and backend processes.

The backend processes mainly include processes dealing with ID tokens, logout tokens and security checks.

For backend processes the integrator should ensure conformity through static testing, code reviews and unit tests.

NB! Demo and production GovSSO environments **should not** be used for performance or load testing.

### 3.1 Main workflows

#### 3.1.1 Authentication

In order to allow a user to log in to the client application, it is mandatory for the client to first verify the ID token.

Compulsory verification includes verifying:

- signature
- issuer
- addressee
- validity
- eIDAS level of assurance

For a more detailed overview of security operations please refer to the [technical specifications page](TechnicalSpecification#71-verification-of-the-id-token-and-logout-token).

Please note that the client application is allowed to use the ID token only if **all** security checks pass.

If all security checks pass, only then can the client application consider the ID token valid.

The integrator should then test that the user can indeed access the client application.

#### 3.1.2 Session update

In order for the GovSSO session to stay active, a session update is required before the ID token expires.

The current validity period of an ID token is 15 minutes. However, this value should not be hardcoded as it may be subject to change.  

To avoid any issues arising from, for example, network problems, it is advised that the client application initiates session update 2 minutes before ID token's expiration (`exp`) time.

It is important for the integrator to ensure that session update is triggered as a background request so that it would be seamless for the user and would not interfere with the user's activities.

If the session update request is unsuccessful due to a network error, the client application should retry the request.

If the session update request returns and OpenID Connect error, the client application must terminate the user's session immediately and inform the user about it.

After a successful session update request, a new ID token is issued. Before the client application can consider the session updated, the same ID token checks have to be conducted as described in the [authentication step](#311-Authentication). If any of the security checks fail, the client application should immediately terminate the user's session.

In case a session has to be terminated for any reason, the user must be logged out and informed accordingly.

If the session update process is successful and security checks passed, the integrator should test that the user can indeed continue without any issues.

#### 3.1.3 Session continuation

If the user already has an active GovSSO session in another GovSSO client application, the user can:

- continue the session in another GovSSO client application and skip TARA authentication
- terminate the existing session and reauthenticate in TARA

A session cannot be continued if the existing session has been created with a lower eIDAS level-of-assurance ([submitted as `acr_values` parameter in the authentication request](TechnicalSpecification#61-authentication-request) and [displayed as `acr` value in ID token](TechnicalSpecification#51-id-token)) than required by the new client application. In this case, GovSSO will allow the user to access the new client application only by reauthenticating.

If an existing session is continued in another client application, the new client application receives an ID token. The ID token must be verified the same way as in the [authentication step](#311-Authentication) before the client application can use the token.

The integrator must ensure that the user can only access the client application once ID token validation has passed.

#### 3.1.4 Logging out

There are three possible scenarios regarding logging out from a GovSSO session:

- User is logged in to only one client application and logs out;
- User is logged in to more than one client application and logs out only from one;
- User is logged in to more than one client application and logs out from all client applications by terminating the active GovSSO session;

The integrator has to ensure that no matter the reason for user logging out of the client application, that this information is also submitted to GovSSO.

In all the above-mentioned cases the application(s) being logged out from will be sent a logout token to the back-channel logout endpoint URL specified in the submitted GovSSO joining application.

The integrator has to ensure that the client application can handle user logout when it is triggered from another client application tied to the active session. For example, user is logged in to the client application A and another application B. The user terminates the GovSSO session by logging out from application B and then chooses to log out from all services. Application A receives a logout token and must now be able to also terminate the user's login.  

As with ID tokens, the integrator has to validate the logout token before actually logging out the user. For a more detailed overview of logout token verification, please refer to [technical specifications](TechnicalSpecification#71-verification-of-the-id-token-and-logout-token).

### 3.2 Additional aspects to test

Beside the main workflows, there are additional aspects related to integrating with GovSSO to keep in mind while testing the client application.

#### 3.2.1 Multiple browser tabs

A user might have open several tabs of the same client application in a browser. This means that the integrator should ensure proper handling of different tabs.

For **authentication** the integrator has to ensure that after a successful login from one tab, the user can also access the client application from other tabs.

For **session update** the integrator should ensure that session update requests are only sent from one tab.

For **logging out** the integrator has to ensure that after logging out from one tab, the user is also logged out from other tabs of the same application.

If the client application encounters an issue which leads to termination of the user session (e.g. security checks fail for a new ID token), the integrator has to ensure that the user is logged out of all tabs of the same application.

#### 3.2.2 State and nonce parameters

GovSSO uses the compulsory `state` and optional `nonce` parameters to protect against [false request attacks](TechnicalSpecification#73-protection-against-false-request-attacks).

The integrator should ensure the following:
- `state` and `nonce` parameters are created according to the standard described in technical specification, i.e. the value of these parameters is a Base64 encoded hash of at least a 16 byte random string;
- The client application conducts a `state` parameter check on callback request from GovSSO. If the validation fails, the client side authentication must also fail.
- If the optional `nonce` parameter is used, then `nonce` validation is incorporated into ID token validation.

#### 3.2.3 Error response handling

Errors can occur for various reasons, and it is important for the integrator to handle these cases with care.

Please refer to [requests subsection in technical specifications](TechnicalSpecification#6-requests) for a more detailed overview of error causes for different requests.

The integrator has to ensure that if an error is returned and user is redirected back to the client application, then the application displays the user proper information regarding the error.

If an error is returned for logout or session update requests, the user session should be terminated in the client application.

#### 3.2.4 Public signature key identifier usage

The public signature key (`kid`) is used for JWT [signature verification](TechnicalSpecification#71-verification-of-the-id-token-and-logout-token) and can be obtained from the public signature key [endpoint](TechnicalSpecification#8-endpoints).

The integrator should ensure that the `kid` value is not hardcoded on the client application side. If the key is hardcoded and should change, then client application users will be unable to log in.

It is recommended to buffer the key on the client side. If JWT validation fails due to a key change, then the client application should check the public signature key endpoint. If there is a new `kid`, the client application should buffer the new value and revalidate the JWT.

#### 3.2.5 User language preference

GovSSO supports Estonian, English and Russian. For better user experience, it is advised to add the `ui_locales` parameter with [authentication](TechnicalSpecification#61-authentication-request) and [logout](TechnicalSpecification#64-logout-request) requests.

When using the parameter, the integrator should test that after making requests to GovSSO the client application is shown in accordance with the submitted user's language preference.

#### 3.2.6 Client logo

GovSSO clients have the option of submitting their logo to GovSSO. This logo will be displayed in GovSSO and TARA UI.

If a logo is used, the integrator should test that the logo is displayed properly in TARA and GovSSO UI, i.e. that the logo scales correctly and there are no visual anomalies.

Please note that client logos are not displayed in mobile view.

#### 3.2.7 Browsers and devices

The integrator should test whether the client application works with GovSSO with a combination of browsers and devices supported by the client.