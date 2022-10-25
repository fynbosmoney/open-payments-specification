---
title: "Open Payments"
category: info

docname: draft-hopebailie-open-payments-latest
submissiontype: IETF  # also: "independent", "IAB", or "IRTF"
number:
date:
consensus: true
v: 3
# area: AREA
# workgroup: WG Working Group
keyword:
 - payments
 - gnap
 - digitial wallet
venue:
#  group: WG
#  type: Working Group
#  mail: WG@example.com
#  arch: https://example.com/WG
  github: "adrianhopebailie/open-payments-specification"
  latest: "https://adrianhopebailie.github.io/open-payments-specification/draft-hopebailie-open-payments.html"

author:
 -
    fullname: Adrian Hope-Bailie
    organization: Fynbos
    email: adrian@fynbos.dev

normative:
   I-D.ietf-gnap-core-protocol-10: GNAP
   I-D.ietf-httpbis-message-signatures-13: HTTP-Message-Signatures
   RFC7517: RFC7517

informative:


--- abstract

Open Payments is an HTTP-based protocol for interacting with an online digital wallet over the Internet for the purpose of sending and receiving payments from or to the wallet. Open Payments defines APIs for interacting with the wallet and also a profile of the Grant Negotiation and Agreement Protocol {{-GNAP}} to negotiate access to those APIs and access to select data about the wallet's owner.

--- middle

# Introduction

Open Payments is a protocol, based on HTTP, that can be implemented by any digital wallet to enable access to features of the wallet via APIs. It uses a profile of the Grant Negotiation and Agreement Protocol {{-GNAP}} to define a mechanism by which clients get authorization to use the APIs and get data about the wallet owner.

A unique feature of the protocol is that the wallet identifier (a URL called a payment pointer) is also the API entry point, as described in {{payment-pointers}}.

## Goals

The goal of Open Payments is to define a standard API and protocol that can be implemented by a digital wallet for the purpose of:

  - sharing information about the wallet owner and underlying accounts and transactions.
  - allowing clients to create an **incoming payment**(((incoming payment))) at the wallet which acts as a placeholder for a future incoming payment (including relevant metadata such as a payment reference or invoice number) in return the wallet provides details to the sender on how to make the incoming payment such that it is automatically reconciled with this metadata.
  - allowing clients to create an **outgoing payment**(((outgoing payment))) which is an instruction to debit one of the underlying accounts and make a payment to a counterparty.
  - allowing clients to get a **quote**(((quote)))for executing an outgoing payment that indicates the amount that will be debited from one of the underlying accounts and the amount that will be credited to the counterparty if the payment is executed before the quote expires.

By defining an open standard it should be possible for an application to connect to any digital wallet that implements the standard without requiring custom integrations or aggregators.

Using fine grained access control through grants, wallet owners can have very specific control over the permissions they grant to applications that connect to their wallet. This enables powerful use cases such as third-party payment initiation and delegated authorisation without compromising the security of the underlying accounts or requiring that wallet owners share any sensitive account information or credentials.

## Notational Conventions

{::boilerplate bcp14-tagged}

## Terminology

In this document the term `Open Payments server` (sometimes shortened to `server`) refers to the services offered by an implementor of the Open Payment APIs. These APIs may be served by multiple components (e.g. a resource server and an authorization server).

TODO - other definitions

# Payment Pointers {#payment-pointers}

Open Payments is designed to operate over the Web with no need for access to private or proprietary networks.

Every digital wallet that is accessible via the Open Payments APIs is identified by one or more URLs called payment pointers. These URLs not only identify the wallet but are also the entry point for the APIs served by the wallet.

Clients also identify themselves using payment pointers. All client requests in Open Payments are signed and the keys used to sign the requests are published as a URL relative to the client's payment pointer as defined in {{client-auth}}.

## Normalisation of a payment pointer {#normalise}

Payment pointers are URLs but they have an alternative representation that makes them easy to transcribe and differentiate from other URLs. 

The shortened form of a payment pointer uses the `$` (dollar) character in place of the `https://` prefix to shorten the string and also make a payment pointer easy to recognise.

Payment pointer URLs MUST never have an empty path component. This design allows domain owners to use their domain as their payment pointer and not create conflicts with their website which is likely hosted at the same domain. To accommodate this, any payment pointer that is being normalised from the short form to the full URL form and has an empty path is assigned the path `/.well-know/pay`.

When a payment pointer is used as an identifier it MUST be normalised into the full URL form. The following algorithm can be followed to **normalise a payment pointer**(((normalise a payment pointer))) (PP) to the full URL form:

      if (PP starts with the character '$') then
        PP = trim_first_character(PP);
        PP = string_concatenate('https://', PP);
      endif;

      if (PP does not end with the character '/') then
        PP = string_concatenate(PP, '/');
      endif;

      URL = parse_url_per_rfc3986(PP)

      if (URL.scheme not equals 'https') then
        abort
      endif;

      if (URL.path equals '/') then
        PP = string_concatenate(PP, '.well-known/pay/');
      endif;

If at any point this algorithm throws an error then the input is not a valid payment pointer.

# Clients

The Open Payments protocol is a client server protocol. The client will interface with one or more HTTP endpoints hosted by an Open Payments server to complete an operation, always starting with the payment pointer of the wallet on which the client is operating.

## Client Identification {#client-id}

All clients MUST identify themselves when interacting with an Open Payments server.

Clients identify themselves using either an access token they have been granted by the server during a previous grant negotiation or by passing their payment pointer to the server when initiating a grant negotiation.

Clients initiating a grant negotiation with an Open Payments server MUST make a {{-GNAP}} request to the server and identify themselves by reference as described in {{Section 2.3.1 of -GNAP}}. The client instance identifier MUST be a payment pointer owned by the client. The client proves ownership of the payment pointer as described in {{client-auth}}.

When a server receives a request initiating a new grant negotiation it MUST identify the client by the provided payment pointer. The server MUST {{normalise}} the payment pointer before using it to uniquely identify the client.

## Client Authentication {#client-auth}

Clients make both authenticated and unauthenticated requests as part of the protocol. All authenticated requests MUST be signed by the client. Client authentication follows {{Section 2.3.3 of -GNAP}} and security of the authentication follows a profile of {{Section 7 of -GNAP}} limited to only {{-HTTP-Message-Signatures}} as a proofing mechanism and ED25519 keys, passed by reference, for signing.

A client MUST publish the public keys they use to sign requests in a JSON Web Key Set document as defined in {{Section 5 of RFC7517}}.

The absolute URL of the published document MUST be constructed by appending the suffix 'jwks.json' to the normalised payment pointer the client uses to identify itself.

For a client payment pointer (PP), the following pseudocode describes an algorithm for transforming PP into the URL (K) that references the JWKS document of the client:

      PP = normalise(PP)
      K = string_concatenate(PP, 'jwks.json')

A client making an authenticated request MUST use a key published in the client's JSON Web Key Set document to sign the request. The signature on the request MUST follow the signing mechanism defined in {{-HTTP-Message-Signatures}} as described further in {{request-signing}}.

Servers that process incoming authenticated requests MUST identify the client making the request and authenticate the client by verifying the request signature.

The binding of the client keys to the client identity is based upon the presence of the JWKS document relative to the payment pointer.

It is up to the server to make a determination as to how much trust it places in this binding. Servers that receive requests from a new client SHOULD have policies for calculating the level of trust they place in the client identity and the binding between the client and the JWKS document.

The level of trust that the server places in the identity of the client SHOULD be considered when evaluating the security risk of the operations being requested by the client.

## DDoS Protection

The authentication of a client request by a server requires two steps, getting the list of keys for the payment pointer, and verifying the signature on the client request using one of the keys. This exposes servers to being part of a distributed denial of service attack if a malicious client makes requests to multiple servers using the same payment pointer URL that points to the target of the attack.

While an attacker will not be able to provide a valid signature for a payment pointer that they don't control, each server that is verifying a client's keys will fetch the client's JWKS document before it is able to verify the signature. To mitigate these key requests being used as a denial of service attack clients SHOULD serve their JWKS document from an appropriate cache and servers SHOULD track failed signature verifications and throttle or block requests from IP addresses that have a highly percentage of failures especially if requests from the same IP address claim multiple payment pointer identities.

## Request Signing {#request-signing}

All authenticated requests from the client MUST be signed using {{-HTTP-Message-Signatures}} as described in {{Section 7.3.1 of -GNAP}} with a value of `Ed25519` for the `alg` parameter.

The signing key MUST be a JWK present in the JSON Web Key Set of the client as described in {#client-auth}. The `keyid` parameter of the signature MUST be set to the `kid` value of the JWK, the signing algorithm used MUST be the JWS algorithm denoted by the key's alg field, and the explicit alg signature parameter MUST NOT be included.

# Grants

A client that wishes to perform any operation MUST get a grant from the server to perform that operation. Grants are requested following the {{-GNAP}} protocol and are issued with one or more access tokens that are passed in the subsequent request to perform the operation.

## Access

The Open Payment APIs are resource oriented and {operations} are initiated by creating a new resource. The state of the operation is determined by reading the state of the resource and an operation is modified by updating the resource.

The `access` property of each access token returned to the client in a grant response defines what operations are permitted by the client when using that access token. The schema of this data is based on {{Section 8 of -GNAP}}.

### Type {#access-type}

Each `access` object MUST have a `type` property that has one of the following values:

  - `incoming-payment`
  - `outgoing-payment`
  - `quote`

The `type` of the grant indicates the type of resource that the client is permitted to perform actions on at the server. These resources and the operations they represent are defined further in {operations}.

### Actions {#access-actions}

Each `access` object MUST also have an `actions` property which is an array of actions that the grant permits the client to perform on resources of the type defined in the `type` property.

Depending on the type of resource these MAY include:

  - `create`
  - `read`
  - `read-all`
  - `list`
  - `list-all`

#### Create {#access-actions-create}

The `create` action permits the client to create a new resource of the type specified in `type`. Creating a new resource is the mechanism by which an operation is initiated. Creating a resource is done by making an HTTP `POST` request to the API endpoint specified in `locations` which is the URL of the resource collection for resources of the specified type.

Servers MUST return a JSON representation of the newly created resource in the response along with the HTTP response code `201 Created`. Servers MUST assign a unique URL as the identifier of any newly created resource and return this URL as the `id` property in the response.

#### Read {#access-actions-read}

The `read` action permits the client to read the current state of a resource of the type specified in `type` if that resource was created by the client. Servers MUST be capable of identifying which client created a resource and ensuring that a client with a grant to `read` resources is only given access to the resources it has created.

Reading a resource is done by making an HTTP `GET` request to the URL that is also the unique identifier of the resource represented by the `id` property of the resource.

#### Read All {#access-actions-read-all}

The `read-all` action permits the client to read the current state of a resource of the type specified in `type` even if that resource was not created by the client.

#### List {#access-actions-list}

The `list` action permits the client to get a list of all the resources of the type specified in `type`. The server MUST filter the results to only return resources created by the client.

Listing resources is done by making an HTTP `GET` request to the URL of the resource collection.

#### List All {#access-actions-list-all}

The `list-all` action permits the client to get a list of all the resources of the type specified in `type` even if that resource was not created by the client.

### Locations {#access-locations}

Each `access` object MUST have an `actions` property which is an array of actions that the grant permits the client to perform on resources of the type defined in the `type` property.

# Operations {#operations}

The operations that can be performed by a client are represented by resources that can be created on the server. The API endpoints for these resource sets are relative to the URL of the payment pointer of the wallet.

All operations are initiated by creating a new resource through an HTTP `POST` request to the appropriate resource collection endpoint. The request body MUST contain a JSON representation of the resource being created and the request `content-type` header MUST be `application/json`.

The resource created is identified by a URL that is returned in the response under the key `id`. Clients MUST make an HTTP `GET` request to that URL to `read` the state of the operation.

## Anonymous Requests

All client requests MUST be authenticated as described in {client-auth} with the exception of GET requests to a payment pointer.

## Incoming Payment {#incoming-payment}

TODO

## Quote {#quote}

TODO

## Outgoing Payment {#outgoing-payment}

TODO

# Security Considerations

TODO

# IANA Considerations

This document has no IANA actions.


--- back

# Acknowledgments
{:numbered="false"}

TODO acknowledge.
