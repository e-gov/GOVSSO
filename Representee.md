---
permalink: Representee
---

<img src='img/eu_regional_development_fund_horizontal.jpg' width="350" height="200" alt="European Union European Regional Development Fund"/>

# Technical Specification: Representee

{: .no_toc}
v1.0, 2025-02-13

- TOC
{:toc}

## 1 Overview

This document is an addition to the [core technical specification](TechnicalSpecification), specifying details about the optional representee feature.

[Central authorisations management information system Pääsuke](https://www.ria.ee/en/state-information-system/central-platforms-provision-public-services/authorisations-management) can provide client applications data about representation rights. An application can register as a client of Pääsuke service. In that case, that application can directly query representation rights (legal representation rights loaded from the Estonian e-Business Register and authorisations created from the Pääsuke user interface in eesti.ee) by calling Pääsuke X-Road services.

GovSSO offers brokering representation rights data from Pääsuke to its client application. That way, the GovSSO client application does not have to query Pääsuke X-Road services but can get representation rights data from GovSSO. GovSSO does not display representation rights data to the end-user on the GovSSO page.

## 2 Enabling and configuring the GovSSO representee feature

For a client application to obtain representation rights via GovSSO, the representee feature must be explicitly enabled and configured for that client application on the GovSSO side.

1. **Prerequisite:** the client application must register as a client of Pääsuke service [see registration form](https://www.ria.ee/riigi-infosusteem/kesksed-platvormid-avalike-e-teenuste-pakkumiseks/paasuke#liitumine). GovSSO can broker representation rights only if the client has a contract with Pääsuke service.
2. For a GovSSO client application with a certain `client_id`, provide RIA with the following configuration values (on the initial client application registration form or later via [help@ria.ee](help@ria.ee)):

| Configuration parameter | example | explanation |
|-------------------------|---------|-------------|
| Parameters for Pääsuke request | `ns=BR_REPRIGHT&role=AGENCY-Q:Edit.submit` | URL query parameters that GovSSO transmits to Pääsuke service for all representation rights requests. Must be in the format described in Pääsuke's technical documentation chapter [X-Road services offered by Pääsuke](https://github.com/e-gov/PH?tab=readme-ov-file#x-road-services-offered-by-p%C3%A4%C3%A4suke). Client application must specify the same URL query parameters here that it would use if it would perform requests to Pääsuke's X-Road services directly. URL query parameters must limit the queriable representation rights by a list of namespaces and/or roles or other criteria that Pääsuke supports. As of [Pääsuke specification 0.5.3](https://github.com/e-gov/PH/blob/main/spec/x-road_services_provided_by_paasuke.v0.5.3.pdf), at least one `ns` and/or `role` parameter must be included, `representeeType` is optional. |

## 3 Requesting representation rights

GovSSO client application can request representation rights by adding `representee.*` and/or `representee_list` to the initial [authentication request](TechnicalSpecification#61-authentication-request) scope values.

- The `representee_list` scope enables the client application for the currently authenticated user to get the list representations (the list of all persons the authenticated user is allowed to represent). The client application can use this information to display representation choices to the end-user on the client application page. GovSSO does not use `representee_list` to display additional information to the end-user on the GovSSO page. 
The `representee.*` scope enables the client application to request a specific representation (the list of representation rights granted by a particular person to the authenticated user) after initial authentication. The client application can request details of a specific representation with [session update requests](TechnicalSpecification#63-session-update-request).
- The `representee.{subject}` scope enables client applications to get detailed representation data of a person from the `representee_list` with [session update requests](TechnicalSpecification#63-session-update-request).

### 3.1 Authentication request with representation scopes
 
A client application cannot request representation information later during the user's session if it didn't request representation scopes in the initial authentication request.

***Example authentication request with representation scopes***
````
GET /oauth2/auth?
 
redirect_uri=https%3A%2F%2Fclient.example.com%2Fcallback&
scope=openid%20representee.*%20representee_list&
state=hkMVY7vjuN7xyLl5&
response_type=code&
client_id=58e7ba35aab5b4f1671a&
ui_locales=en&
nonce=fsdsfwrerhtry3qeewq&
acr_values=substantial&
````
(for better readability, the parts of the HTTP request are divided onto several lines)

### 3.2 Session update requests with representation data

By default, representation data will not be carried over to subsequent ID Tokens. If needed, the client application should request representation data with the following [session update requests](TechnicalSpecification#63-session-update-request).


#### 3.2.1 Session update requests with `representee_list` scope

To get a new up-to-date `representee_list`, the client application needs to add scope to the session update request.

***Example GovSSO session update request with `representee_list` scope***
````
POST /oauth2/token HTTP/1.1
Host: govsso.ria.ee
Content-Type: application/x-www-form-urlencoded
Authorization: Basic czZCaGRSa3F0MzpnWDFmQmF0M2JW
 
grant_type=refresh_token&
refresh_token=1kYI7zBU_WEGoMCVxSraXLcuA906szL9hxC2qq7bgso.uq1VHIByywr0Q9fk-V9Jp1BmLLQihoqXctHHHY8b3bQ&
scope=openid%20representee_list
````
(for better readability, the parts of the HTTP request are divided onto several lines)

#### 3.2.2 Session update request with `representee.*` scope

The client application must add `representee.{subject}` scope to the session update request to get detailed representation information for a specific person.

The `{subject}` value must be the ID code of a natural person or the registry code of a legal person, prefixed by the country code of the person's origin.

The client application can only request detailed representation information about persons listed in the `representee_list`. Only one representee can be requested at once.

***Example GovSSO session update request with `representee.*` scope***
````
POST /oauth2/token HTTP/1.1
Host: govsso.ria.ee
Content-Type: application/x-www-form-urlencoded
Authorization: Basic czZCaGRSa3F0MzpnWDFmQmF0M2JW
 
grant_type=refresh_token&
refresh_token=1kYI7zBU_WEGoMCVxSraXLcuA906szL9hxC2qq7bgso.uq1VHIByywr0Q9fk-V9Jp1BmLLQihoqXctHHHY8b3bQ&
scope=openid%20representee.EE12345678901
````
(for better readability, the parts of the HTTP request are divided onto several lines)

## 4 Receiving representation data

All information regarding representation will be provided to the client application in the [ID Token](TechnicalSpecification#51-id-token) in addition to the default GovSSO claims.

If the GovSSO client application has also enabled the optional [Access Token configuration](AccessToken), then `representee.*` scope specific claims will be present in the Access Token as well. Note that Access Token does not hold `representee_list` claims.

The `representee` claim is omitted from ID Tokens and Access Tokens if the authenticated user represents oneself.

### 4.1 `representee_list` claims

If `representee_list` scope was requested with the initial authentication request or subsequent session update requests, the ID Token will hold additional information regarding users representations.

***Example `representee_list` claims in an ID Token***
````
...
"representee_list": {
    "status": "REPRESENTEE_LIST_CURRENT",
    "list": [
      {
        "sub": "EE12345678",
        "type": "LEGAL_PERSON",      
        "name": "AS Legal Person"
      },
      {
        "sub": "EE12345678901",
        "type": "NATURAL_PERSON"
        "given_name": "Max"
        "family_name": "Stonewood"
      }
    ]
},
...
````

| ID Token element (claim) | example | explanation |
|--------------------------|---------|-------------|
| representee_list.status | `REPRESENTEE_LIST_CURRENT` or `SERVICE_NOT_AVAILABLE`| `REPRESENTEE_LIST_CURRENT` when the request from GovSSO to Pääsuke succeeded, therefore the nested `list` claim contains the currently valid list of representations for the currently authenticated user. The nested `list` claim is an empty array when the currently authenticated user has no representations. The list of representations for the currently authenticated user is filtered on the Pääsuke service's side by the namespaces, roles, or other criteria configured with "Parameters for Pääsuke request" in [chapter 2](2-enabling-and-configuring-representee-feature).<br> `SERVICE_NOT_AVAILABLE` when the request from GovSSO to Pääsuke failed (temporary problem); therefore, the nested `list` claim is omitted. |
| representee_list.list.sub | `EE12345678901` | The ID code of a natural person or the registry code of a legal person. Prefixed by the country code of the person's origin. |
| representee_list.list.type | `NATURAL_PERSON` or `LEGAL_PERSON` | Type of the representee (person the user has a right to represent). |
| representee_list.list.name | `AS Legal Person` | Legal person's official name. |
| representee_list.list.given_name | `Max` | Natural person's given name. |
| representee_list.list.family_name | `Stonewood` | Natural person's family name. |

### 4.2 `representee.{subject}` claims

If `representee.{subject}` scope was requested with session update requests, the ID Token will hold additional information regarding users specific representation.

***Example `representee.{subject}` claims for a natural person in an ID Token***
````
...
"representee": {
    "status": "REQUESTED_REPRESENTEE_CURRENT", 
    "sub": "EE12345678901",
    "type": "NATURAL_PERSON",       
    "given_name": "Max",
    "family_name": "Stonewood",
    "mandates": [
      {
        "role": "BR_REPRIGHT:ROLE_IN_BOARD"
      },
      {
        "role": "AGENCY-Q:Edit"
      }
    ]
},
...
````

***Example `representee.{subject}` claims for a legal person in an ID Token***
````
...
"representee": {
    "status": "REQUESTED_REPRESENTEE_CURRENT",
    "sub": "EE12345678",
    "type": "LEGAL_PERSON",
    "name": "AS Legal Person",    
    "mandates": [
      {
        "role": "BR_REPRIGHT:ROLE_IN_BOARD"
      },
      {
        "role": "AGENCY-Q:Edit"
      }
    ]
},
...
````

| ID Token and Access Token element (claim) | example | explanation |
|-------------------------------------------|---------|-------------|
| representee.status | `REQUESTED_REPRESENTEE_CURRENT`, `REQUESTED_REPRESENTEE_NOT_ALLOWED` or `SERVICE_NOT_AVAILABLE` | `REQUESTED_REPRESENTEE_CURRENT` when the request from GovSSO to Pääsuke succeeded and currently authenticated user is allowed to represent the requested representee, therefore other nested claims contain details about the requested representee.<br> `REQUESTED_REPRESENTEE_NOT_ALLOWED` when the request from GovSSO to Pääsuke succeeded and currently authenticated user is not allowed to represent the requested representee, therefore other nested claims are omitted. Available representees for the currently authenticated user are filtered on Pääsuke service's side by the namespaces, roles, or other criteria configured with "Parameters for Pääsuke request" in [chapter 2](2-enabling-and-configuring-representee-feature).<br> `SERVICE_NOT_AVAILABLE` when the request from GovSSO to Pääsuke failed (temporary problem), therefore other nested claims are omitted. |
| representee.sub | `EE12345678901` | The ID code of a natural person or the registry code of a legal person. Prefixed by the country code of the person's origin. |
| representee.type | `NATURAL_PERSON` or `LEGAL_PERSON` | Type of the representee. |
| representee.name | `AS Legal Person` | Legal person's official name. |
| representee.given_name | `Max` | Natural person's given name. |
| representee.family_name | `Stonewood` | Natural person's family name. |
| representee.mandates.role | `"role": "BR_REPRIGHT:ROLE_IN_BOARD"` | Role codes of authorisations and legal representation rights that the currently authenticated user has been granted to represent that representee. For each representee returned by GovSSO, there exists at least one role. |

## 5 Logout request

If the last valid ID Token, i.e., the `id_token_hint` value included in the [logout request](TechnicalSpecification#64-logout-request), contains a `representee_list` claim, then the user's browser must be directed to the GovSSO logout endpoint with a `POST` request (using Form Serialization method) because the `id_token_hint` value may be longer than what the browser would support in a `GET` request. For the GovSSO logout endpoint, a `POST` request can always be used, even when the `id_token_hint` does not contain a `representee_list` claim. At the GovSSO logout endpoint, a `GET` request may only be used if the `id_token_hint` does not contain a `representee_list` claim.

## 6 Environments

GovSSO demo and production environments are integrated with Pääsuke over following X-road environments:

| GovSSO environment | Pääsuke X-road environment |
|--------------------|----------------------------|
| Demo | Stage - `ee-test/GOV/70006317/volitused/oraakel-stage` |
| Production | Production - `EE/GOV/70006317/volitused/oraakel` |

## Change history

| Version, Date | Description |
|---------------|-------------|
| 1.0, 2025-02-13  | Initial version |
