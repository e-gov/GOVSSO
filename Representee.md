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

[Central authorisations management information system Pääsuke](https://www.ria.ee/en/state-information-system/central-platforms-provision-public-services/authorisations-management) can provide client applications data about representation rights. If a client application registers as a client of Pääsuke service, that client application can directly query X-Road services offered by Pääsuke.

GovSSO offers brokering the data about representation rights from Pääsuke to GovSSO's client application. That way GovSSO's client application itself does not have to query Pääsuke's X-Road services, but can get the necessary data about representation rights from GovSSO. GovSSO itself does not display representations to the end-user on the GovSSO page.

## 2 Enabling and configuring representee feature

For a client application to be able to obtain representation rights via GovSSO, the representee feature must be explicitly enabled and configured for that client application registration on the GovSSO side.

1. **Prerequisite:** the client application must register as a client of Pääsuke service, [see registration form](https://www.ria.ee/riigi-infosusteem/kesksed-platvormid-avalike-e-teenuste-pakkumiseks/paasuke#liitumine). GovSSO can broker representation rights only if the client has a contract with Pääsuke service.
2. For a GovSSO client application with a certain `client_id`, provide RIA with the following configuration values (on the initial client application registration form or later via [help@ria.ee](help@ria.ee)):

| Configuration parameter | example | explanation |
|-------------------------|---------|-------------|
| Parameters for Pääsuke request | `ns=BR_REPRIGHT&role=AGENCY-Q:Edit.submit` | URL query parameters that GovSSO transmits to Pääsuke service for all representation rights requests. Must be in the format described in Pääsuke's technical documentation's chapter [X-Road services offered by Pääsuke](https://github.com/e-gov/PH?tab=readme-ov-file#x-road-services-offered-by-p%C3%A4%C3%A4suke). Client application must specify the same URL query parameters here that it would use if it would perform requests to Pääsuke's X-Road services directly. URL query parameters must limit the queriable representation rights by namespace, role, or other criteria that is supported by Pääsuke. As of [Pääsuke specification 0.5.3](https://github.com/e-gov/PH/blob/main/spec/x-road_services_provided_by_paasuke.v0.5.3.pdf), at least one `ns` and/or `role` parameter must be included, `representeeType` is optional. |

## 3 Requesting representation data

GovSSO client application can request representation data by adding `representee.*` and/or `representee_list` to the initial [authentication request](TechnicalSpecification#61-authentication-request) scope values.

- `representee_list` scope enables the client application to get the list of all representations of the currently authenticated user. The client application can use this information to display representation choices to the end-user on the client application page. GovSSO does not use `representee_list` itself to display any additional information to the end-user on the GovSSO page. 
- `representee.*` scope enables the client application to request specific representation of the authenticated user after initial authentication. Details of a specific representation can only be requested with [session update requests](TechnicalSpecification#63-session-update-request).
- `representee.{subject}` scope enables client applications to get detailed representation data of a person from the `representee_list` with [session update requests](TechnicalSpecification#63-session-update-request).

### 3.1 Authentication request with representation scopes
 
Client application cannot request representation information later during the user's session if representation scopes were not requested in the initial authentication request.

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

### 3.2 Session update requests with representation data

Representation data will not carry on by default to subsequent ID Tokens. The client application should request representation data with following [session update requests](TechnicalSpecification#63-session-update-request) if needed.

#### 3.2.1 Session update requests with `representee_list` scope

To get a new up-to-date `representee_list`, the scope needs to be added to session update request.

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

#### 3.2.2 Session update request with `representee.*` scope

To get detailed representation information for a specific person, `representee.{subject}` scope needs to be added to session update request.

The `{subject}` value must be the ID code of a natural person or the registry code of a legal person, prefixed by the country code of the person's origin.

The client application can only request detailed representation info about persons listed in the `representee_list`. Only one representee can be requested at once.

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

## 4 Receiving representation data

All information regarding representation will be provided to the client application in the [ID Token](TechnicalSpecification#51-id-token) in addition to the default GovSSO claims.

If the GovSSO client application has also enabled the optional [Access Token configuration](AccessToken), then `representee.*` scope specific claims will be present in the Access Token as well. Note that Access Token does not hold `representee_list` claims.

`representee` claim is omitted from ID Tokens and Access Tokens if the user is representing themselves.

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
| representee_list.status | `REPRESENTEE_LIST_CURRENT` or `SERVICE_NOT_AVAILABLE`| `REPRESENTEE_LIST_CURRENT` when the request from GovSSO to Pääsuke succeeded, therefore the nested `list` claim contains the currently valid list of representees for the currently authenticated user. When the currently authenticated user does not have any representees, the nested `list` claim is an empty array. The list of representees for the currently authenticated user is filtered on Pääsuke service's side by the namespaces, roles, or other criteria configured with "Parameters for Pääsuke request" in [chapter 2](2-enabling-and-configuring-representee-feature).<br> `SERVICE_NOT_AVAILABLE` when the request from GovSSO to Pääsuke failed (temporary problem), therefore the nested `list` claim is omitted. |
| representee_list.list.sub | `EE12345678901` | The ID code of a natural person or the registry code of a legal person. Prefixed by the country code of the person's origin. |
| representee_list.list.type | `NATURAL_PERSON` or `LEGAL_PERSON` | Type of the representee. |
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
        "role": "BR_REPRIGHT:MANAGEMENT"
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
        "role": "BR_REPRIGHT:MANAGEMENT"
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
| representee.mandates.role | `"role": "BR_REPRIGHT:MANAGEMENT"` | Roles that have been granted to the currently authenticated user for representing that representee. At least one role exists for each representee. |

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
