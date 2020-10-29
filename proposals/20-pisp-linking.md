---
Change Request ID: 20
Change Request Name: PISP Linking
Prepared By: Lewis Daly (Crosslake)
Solution Proposal Status: In review
Approved/Rejected Date: N/A
Target SIG: Thirdparty SIG
Target APIs: Thirdparty API
---

# Proposal - Allow PISPs to Link End User Accounts with a DFSP

[todo - update]
- [Proposal - Allow PISPs to Link End User Accounts with a DFSP](#proposal---Allow PISPs to Link-End-User-Accounts-with-a-DFSP)
  - [Document History](#document-history)
  - [Supporting Documents](#supporting-documents)
  - [Definitions](#Definitions)
  - [Change Request Background](#change-request-background)
  - [Proposed Solution](#proposed-solution)
    - [`POST /authorizations resource`](#POST-/authorizations-resource)
    - [Request Flow](#request-flow)
    - [Error Handling + Retries](#Error-Handling--Retries)
  - [HTTP Request Descriptions](#HTTP-Request-Descriptions)
  - [Data Models](#data-models)

## Document History

| Version | Date       | Author             | Change Description         |
| ------- | ---------- | ------------------ | -------------------------- |
| 1.0     | 2020-10-29 | Lewis Daly        | Initial draft              |

## Supporting Documents

- [PISP Linking Overview](https://github.com/mojaloop/pisp/tree/master/docs/linking)
- [Proposed Thirdparty-PISP Swagger Explorer](http://docs.mojaloop.io/pisp/?urls.primaryName=thirdparty-pisp-api)
- [Proposed Thirdparty-DFSP Swagger Explorer](http://docs.mojaloop.io/pisp/?urls.primaryName=thirdparty-dfsp-api)

## Definitions

[todo - update]

- `PISP` - Payment initiation service provider. In mojaloop, PISP is a _role_ adopted by a participant participating in a scheme.
- Thirdparty API - A _new_ Mojaloop API. PISPs use the Thirdparty API to talk to the Mojaloop Switch, as opposed to the FSPIOP-API. 
  - The Thirdparty API can be divided into 2 separate parts: Thirdparty-PISP and Thirdparty-DFSP
  - Thirdparty-PISP is the section of the Thirdparty API a PISP uses to communicate with the Mojaloop Switch
  - Thirdparty-DFSP is the section of the Thirdparty API a DFSP uses to communicate with the Mojaloop Switch with respect to Thirdparty functions

- Auth-Service

## Change Request Background

A Payment Initiation Service Provider (PISP) is a new participant _role_ within Mojaloop for participants which can 

As a part of a 3rd Party Transaction Request, we require a PISP to obtain the
explicit consent from their user for each transaction initiated on their behalf.

[todo: more background?]

## Proposed Solution

[todo]


### 1. Linking Flows

#### 1.1 Pre-linking

> Note: see [1.1 pre-linking](https://github.com/mojaloop/pisp/tree/master/docs/linking#11-pre-linking) for more information and sequences.

The PISP asks the switch which DFSPs are available to link customer accounts with. This result can be cached by the PISP, since the list of DFSPs is unlikely to change very often.


#### 1.2 Discovery

> Note: see [1.2 Discovery](https://github.com/mojaloop/pisp/tree/master/docs/linking#12-discovery) for more information and sequences.

After seeing a list of available DFSPs, a user should be able to select their DFSP from a list, and provide an opaque identifier they use to log in with their DFSP. The PISP then calls a `GET /parties/OPAQUE/{id}` request, where the `FSPIOP-Destination` header is pre-specified. See [below](#3-opaque-als-lookup) for more information on this request and why it is necessary.


#### 1.3 Request Consent

> Note: see [1.3 Request Consent](https://github.com/mojaloop/pisp/tree/master/docs/linking#13-request-consent) for more information and sequences.

In this step, a PISP uses a `POST /consentRequests` call to the DFSP to initiate the consent creation process.

There are 2 supported flows for this process: `WEB` and `OTP`.

- `WEB` relies on a DFSP-hosted webpage that the PISP end-user application redirects to for the user to sign in with. This could also be a deeplink to a DFSP's mobile application.
- `OTP` relies on the DFSP to send an OTP to their customer, for example over Email or SMS. This allows DFSPs who may not have front end, public facing web applications to still include their user.

#### 1.4 Authentication

> Note: see [1.4 Authentication](https://github.com/mojaloop/pisp/blob/master/docs/linking/README.md#14-authentication) for more information and sequences.

In the authentication phase, the user is expected to prove their identity to the DFSP. Once this is done, the DFSP will provide the User with some sort of secret (e.g. an OTP or access token). This secret is passed back to the DFSP in the `PUT /consentRequests`, which is verified by the DFSP.

#### 1.5 Grant Consent

> Note: see [1.5 Grant Consent](https://github.com/mojaloop/pisp/blob/master/docs/linking/README.md#15-grant-consent) for more information and sequences.

At this point, we have the 3 way trust relationship between the PISP, DFSP and User. It is up to the DFSP to create a new `Consent` resource, which is a representation of that trust.

In the case of a centrally-hosted auth-service, the Switch updates the auth-service, which is then responsible for registering the consent with the ALS. See [2.2 Consent Object and the ALS](#2.2-Consent-Object-and-the-ALS) for more information.

#### 1.6 Credential Registration

> Note: see [1.6 Credential Registration](https://github.com/mojaloop/pisp/tree/master/docs/linking#16-credential-registration) for more information and sequences.

The PISP then asks the auth-service for a challenge using the `POST /consents/{id}/generateChallenge` request. Since the consent is registered with the ALS, the PISP need not include a `FSPIOP-Destination` header. Instead, the Switch will internally look up the auth-service this request should be routed to, based on an internal `GET /participants` lookup.

The challenge is generated by the auth-service, and then returned to the PISP with a `PUT /consents/{id}` request. 

This is followed by the generation of the credential on the user's device, and the signing of the challenge. see [1.6.2 Finalizing the Credential](https://github.com/mojaloop/pisp/tree/master/docs/linking#162-finalizing-the-credential) for more information

### 2 Auth-Service

The auth-service is a new component we propose alongside the Thirdparty Linking API.

While a `Consent` object is created by the DFSP, the auth-service is considered to be the authority on the consent object. It's responsibilities are:
1. Storing the `Consent` Object
1. Updating the Account Lookup Service with the `participantId` of the consent's auth-service
1. Generating a random challenge for the `Consent.credential`
1. For account linking: Verifying the signed challenge and marking the `Consent.credential` as `VERIFIED`
1. Broadcasting state changes of the `Consent` object to the associated PISP and DFSP
1. For transfers: verifying the signed transaction challenge


For the purposes of request routing and JWS signing, the centrally-hosted auth service should be considered its own participant in a Mojaloop switch. This means it requires:
- A participantId. In this case we use `central-auth`
- JWS keys to sign requests
- TLS certificates


#### 2.1 Modular Design

The design caters for an auth-service to be hosted centrally as a part of the Mojaloop Hub Services, or by a DFSP itself. Depending on a DFSP's size, risk tolerance, etc. they may decide to host the auth-service themselves, or rely upon a centrally hosted auth-service.

#### 2.2 Consent Object and the ALS

To cater for this modular design, we must keep track of _which auth service is responsible for a given `Consent` object_. We propose to use the ALS for this purpose and to reserve the namespace `CONSENTS` as a `{Type}` identifier in the Account Lookup Service.

Upon receiving a `POST /consents` request from the DFSP, the auth-service MUST send a `POST /participants/CONSENTS/{consentId}` to the ALS, with the `fspId` field in the request body set to the auth-services' id (e.g. `central-auth` for a centrally-hosted auth-service, or the `fspId` of the DFSP for the self-hosted case).

### 3 Proposed Conventions

The following are conventions contained within this proposal that are of note. They expand on the principles and conventions outlined in the [FSPIOP-API specification](https://docs.mojaloop.io/mojaloop-specification/documents/API%20Definition%20v1.0.html), and will likely warrant further discussion.

#### 3.1 Broadcast Messages

Whereas the FSPIOP-API is a symmetrical API between 2 parties, the Thirdparty-API breaks with this convention. We now have 3 parties: a PISP, DFSP and auth-service. There are times where a DFSP or auth-service needs to send HTTP requests to 2 parties at a time, such as `CRED-19`, when [the auth-service informs both the PISP and DFSP of a registered credential](https://github.com/mojaloop/pisp/tree/master/docs/linking#162-finalizing-the-credential).

Instead of sending 2 separate HTTP Requests to the switch, we propose that a participant be able to set multiple HTTP Header values for the `FSPIOP-Destination` header. 

For more information, see [pisp#79](https://github.com/mojaloop/pisp/issues/79) for a detailed proposal of this new convention.

#### 3.2 `OPAQUE` ALS lookup

At the start of the linking process, we would like to be able to offer a user experience where, after selecting the DFSP they hold an account with, the user can enter _any_ identifier they use to log in with their DFSP.

This means a modified `GET /parties` call, where the `FSPIOP-Destination` is already set by the PISP.

```
FSPIOP-Source: pispa
FSPIOP-Destination: dfspa

GET /parties/OPAQUE/lewis@email.com  //email 
GET /parties/OPAQUE/+4412345678      //MSISDN
GET /parties/OPAQUE/ldaly            //some username the dfsp knows about
```

The switch should handle this call by forwarding it directly to the DFSP, and not performing the internal `GET /participants` call.

For more information, see this [pisp#28](https://github.com/mojaloop/pisp/issues/28)

#### 3.3 Custom HTTP Methods

In the proposed Thirdparty-API, there are a number of places where we employ [custom methods](https://google.aip.dev/136) to 


## HTTP Request Descriptions

## Data Models