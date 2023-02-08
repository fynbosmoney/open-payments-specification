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
  github: "fynbos-dev/open-payments-specification"
  latest: "https://fynbos-dev.github.io/open-payments-specification/draft-hopebailie-open-payments.html"

author:
 -
    fullname: Adrian Hope-Bailie
    organization: Fynbos
    email: adrian@fynbos.dev

normative:
   I-D.ietf-gnap-core-protocol-12: GNAP
   I-D.ietf-httpbis-message-signatures-15: HTTP-Message-Signatures
   RFC7517: RFC7517
   RFC9110: RFC9110
   RFC2397: RFC2397

informative:


--- abstract

Open Payments is an HTTP-based protocol for interacting with an digital wallet over the Internet for the purpose of sending and receiving payments from or to the wallet. Open Payments defines APIs for interacting with the wallet and also a profile of the Grant Negotiation and Agreement Protocol {{-GNAP}} to negotiate access to those APIs and access to select data about the wallet's owner.

--- middle

# Introduction

Open Payments is an HTTP-based API that can be implemented by any digital wallet to enable clients to interface with the wallet. It uses a profile of the Grant Negotiation and Agreement Protocol {{-GNAP}} to define a mechanism by which clients get authorization to use the APIs and get data about the wallet owner.

For the purposes of this specification, a digital wallet is any Internet connected system that acts as an agent of a person or business that is capable of sending and receiving payments from or to that person or business.

## Goals

The goal of Open Payments is to define a standard API and protocol that can be implemented by a digital wallet for the purpose of:

  - sharing information about the wallet owner and underlying accounts and transactions.
  - receiving payments into the wallet (or any underlying accounts).
  - sending payments from the wallet (or any underlying account).

As with any technical standard, the ultimate goal is interoperability.

The standard is intentionally simple with only a small number of primitives. It should be possible to use these primitives to complete a comprehensive set of payments use cases.

Using fine grained access control through grants, wallet owners can have very specific control over the permissions they grant to applications that access their wallet. This enables powerful use cases such as third-party payment initiation and delegated authorisation without compromising the security of the underlying accounts or requiring that wallet owners share any sensitive account information or credentials.

## Notational Conventions

{::boilerplate bcp14-tagged}

# Terminology and Core Concepts

This specification adheres to the terminology and core concepts of {{-RFC9110}} and {{-GNAP}}.

## Resources {#resources}

There are 4 resource types exposed via the Open Payments interface. Resources are all identified by a URL and accessible at that URL via an HTTP GET request.

The JSON representation of a resource MUST have a top level property **type** which indicates the type per the type identifiers below.

### Client

A client is an entity that connects to the Open Payments interface through an application referred to as a client instance in {{-GNAP}} or a user agent in {{-RFC9110}}.

The client resource contains details of the entity that controls the client instance or user agent. These details are fetched by other clients in order to identify and authenticate the client.

The client resource has one or more public key sub-resources that are used to authenticate the client when it makes signed HTTP requests to an Open Payments server. This is described in detail in {clients}.

The type identifier for the client resource is **client**.

A client resource is read-only and cannot be created, updated or deleted via the Open Payments interface.

The following properties of a client resource SHOULD be accessible to anonymous clients. These correspond directly to the properties of the `display` field in the `client` section of a grant request as defined in {{-GNAP}} :

`id` (string):
: The identifier of the resource. This MUST be the URL at which the resource can be accessed via the Open payments interface. REQUIRED.

`name` (string):
: Display name of the client software. RECOMMENDED.

`uri` (string):
: User-facing web page of the client software. This URI MUST be an absolute URI. OPTIONAL.

`logo_uri` (string)
: Display image to represent the client software. This URI MUST be an absolute URI. The logo MAY be passed by value by using a data: URI {{!RFC2397}} referencing an image mediatype. OPTIONAL.

~~~ json
{
  "id": "https://wallet.example/client/123",
  "type": "client",
  "name": "Example Client",
  "uri": "https://example.net",
  "logo_uri": "data:image/png;base64,Eeww...="
}
~~~

### Wallet

A wallet is an Internet-connected software application. It provides an abstraction over one or more financial accounts and provides the Open Payments interface through which external clients can interact with the accounts via the wallet.

The wallet resource contains details of the entity that is the legal owner of the wallet and additional details about the features, currencies, payment methods and operations supported by the wallet.

The type identifier for the wallet resource is **wallet**.

A wallet resource is read-only and cannot be created, updated or deleted via the Open Payments interface.

The following properties of a wallet resource SHOULD be accessible to anonymous clients:

`id` (string):
: The identifier of the resource. This MUST be the URL at which the resource can be accessed via the Open payments interface. REQUIRED.

`name` (string):
: Display name of the wallet. RECOMMENDED.

`avatar_uri` (string)
: Display image to represent the wallet. This URI MUST be an absolute URI. The image MAY be passed by value by using a data: URI {{!RFC2397}} referencing an image mediatype. OPTIONAL.

~~~ json
{
  "id": "https://wallet.example/wallet/456",
  "type": "wallet",
  "name": "Alice's Wallet",
  "avatar_uri": "data:image/png;base64,Eeww...="
}
~~~

### Incoming Payment

An incoming payment is a resource that describes a payment into a wallet. An incoming payment resource is created via the Open Payments interface before a payment is made into the wallet.

It contains data about the payment such as the amount expected, the amount received, an expiry timestamp and metadata to facilitate reconciliation with linked accounts and the payment methods used to make the payment.

When an incoming payment is created the wallet does any internal processing required to accept the incoming payment. The specific processing required will depend on the payment methods that are available to make the payment.

The type identifier for the incoming payment resource is **incoming-payment**.

Incoming payment resources can be created, read and updated via the Open Payments interface by clients with an appropriate grant.

The following properties of an incoming payment resource SHOULD be accessible to anonymous clients:

`id` (string):
: The identifier of the resource. This MUST be the URL at which the resource can be accessed via the Open payments interface. REQUIRED.

`wallet` (string):
: The identifier of the wallet that is the target of this incoming payment. REQUIRED.

`incoming_amount` (object):
: The amount that is expected to be paid into the wallet. OPTIONAL.

~~~ json
{
  "id": "https://wallet.example/payments/xyz",
  "type": "incoming-payment",
  "wallet": "https://wallet.example/wallet/456",
  "incoming_amount": {
    "amount": "100.00",
    "currency": "USD"
  }
}
~~~

TODO - add more properties

### Outgoing Payment

An outgoing payment is a resource that describes a payment out of a wallet. An outgoing payment is created as a placeholder for a payment that will be made out of the wallet.

It contains data about the payment such as the receiver, the amount to send, an expiry timestamp and metadata to facilitate reconciliation with linked accounts and the payment methods used to make the payment.

The type identifier for the outgoing payment resource is **outgoing-payment**.

Outgoing payment resources can be created, read and updated via the Open Payments interface by clients with an appropriate grant.

The following properties of an outgoing payment resource SHOULD be accessible to anonymous clients:

`id` (string):
: The identifier of the resource. This MUST be the URL at which the resource can be accessed via the Open payments interface. REQUIRED.

`wallet` (string):
: The identifier of the wallet that is the source of this outgoing payment. REQUIRED.

`send_amount` (object):
: The amount that is expected to be sent from the wallet. OPTIONAL.

~~~ json
{
  "id": "https://wallet.example/payments/xyz",
  "type": "outgoing-payment",
  "wallet": "https://wallet.example/wallet/456",
  "send_amount": {
    "amount": "100.00",
    "currency": "USD"
  }
}
~~~

TODO - add more properties

## Representations

All resources in this specification can be represented as JSON objects for consumption by a client or as HTML for display to an end-user in a Web browser.

Servers MUST support proactive content-type negotiation and accept requests and return responses with the `text/html` and `application/json` media types. All messages MUST use the UTF-8 character set.

## User Agents

The concept of a user agent as defined in {{-RFC9110}} and a client instance in {{-GNAP}} are very similar. In this specification a client is understood to mean the entity in control of the client instance or user agent.

# Payment Pointers {#payment-pointers}

Any URL that identifies and locates a resource as defined in {resources} is called a `payment pointer`. Payment pointers always use the https URI scheme.

## Normalisation of a payment pointer {#normalise}

Payment pointers are URLs but they have an alternative representation that makes them easy to transcribe and differentiate from other URLs.

The shortened form of a payment pointer uses the `$` (dollar) character in place of the `https://` prefix to shorten the string and also make a payment pointer easy to recognise. This short-form MUST only be used for display purposes or data entry from an end-user.

Payment pointer URLs never have an empty path component. This design allows domain owners to use their domain as their payment pointer and not create conflicts with their website which is likely hosted at the same domain. To accommodate this, any payment pointer that is being normalised from the short form to the full URL form and has an empty path is assigned the path `/.well-know/pay`.

When a payment pointer is used as an identifier it MUST be normalised into the full URL form and then further normalised per the rules for normalization and comparison in {{-RFC9110}}.

The following algorithm can be followed to **normalise a payment pointer**(((normalise a payment pointer))) (PP) to the full URL form:

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

# Clients {#clients}

The Open Payments protocol is a client server protocol. The client instance or user agent, acting on on behalf of a client, will interface with one or more HTTP endpoints hosted by an Open Payments server.

## Client Identification {#client-id}

All clients MUST identify themselves when interacting with an Open Payments server.

Clients identify themselves using either an access token they have been granted by the server during a previous grant negotiation or by passing their payment pointer to the server when initiating a grant negotiation.

Clients initiating a grant negotiation with an Open Payments server MUST make a {{-GNAP}} request to the server and identify themselves by reference as described in {{Section 2.3.1 of -GNAP}}. The client instance identifier MUST be a payment pointer (in full URL form) that identifies a resource of type **client**, describing the client making the request. The client proves ownership of the payment pointer as described in {{client-auth}}.

When a server receives a request initiating a new grant negotiation it MUST identify the client by the provided payment pointer. The server MUST {{normalise}} the payment pointer before using it to uniquely identify the client.

## Client Authentication {#client-auth}

Clients make both authenticated and unauthenticated requests as part of the protocol. All authenticated requests MUST be signed by the client. Client authentication follows {{Section 2.3.3 of -GNAP}} and security of the authentication follows a profile of {{Section 7 of -GNAP}} limited to only {{-HTTP-Message-Signatures}} as a proofing mechanism and ED25519 keys, passed by reference, for signing.

A client MUST publish the public keys they use to sign requests in a JSON Web Key Set document as defined in {{Section 5 of RFC7517}}.

The absolute URL of the published document MUST be constructed by appending the suffix 'jwks.json' to the normalised payment pointer the client uses to identify itself.

For a client payment pointer (PP), the following pseudocode describes an algorithm for transforming a payment pointer (PP) into the URL (K) that references the JWKS document of the client:

      PP = normalise(PP)
      K = string_concatenate(PP, 'jwks.json')

A client making an authenticated request MUST use a key published in the client's JSON Web Key Set document to sign the request. The signature on the request MUST follow the signing mechanism defined in {{-HTTP-Message-Signatures}} as described further in {{request-signing}}.

Servers that process incoming authenticated requests MUST identify the client making the request and authenticate the client by verifying the request signature.

The binding of the client keys to the client identity is based upon the presence of the JWKS document relative to the payment pointer.

It is up to the server to make a determination as to how much trust it places in this binding. Servers that receive requests from a new client SHOULD have policies for calculating the level of trust they place in the client identity and the binding between the client and the JWKS document.

The level of trust that the server places in the identity of the client SHOULD be considered when evaluating the security risk of the operations being requested by the client.

## DDoS Protection

The authentication of a client request by a server requires two steps, getting the list of keys for the payment pointer, and verifying the signature on the client request using one of the keys. This exposes servers to being part of a distributed denial of service attack if a malicious client makes requests to multiple servers using the same payment pointer URL that points to the target of the attack.

While an attacker will not be able to provide a valid signature for a payment pointer that they don't control, each server that is verifying a client's keys will fetch the client's JWKS document before it is able to verify the signature. To mitigate these key requests being used as a denial of service attack clients SHOULD serve their JWKS document from an appropriate cache and servers SHOULD track failed signature verifications and throttle or block requests from IP addresses that have a high percentage of failures, especially if requests from the same IP address claim multiple payment pointer identities.

## Request Signing {#request-signing}

All authenticated requests from the client MUST be signed using {{-HTTP-Message-Signatures}} as described in {{Section 7.3.1 of -GNAP}} with a value of `Ed25519` for the `alg` parameter.

The signing key MUST be a JWK present in the JSON Web Key Set of the client as described in {#client-auth}. The `keyid` parameter of the signature MUST be set to the `kid` value of the JWK, the signing algorithm used MUST be the JWS algorithm denoted by the key's alg field, and the explicit alg signature parameter MUST NOT be included.

# Grants

A client that wishes to perform any operation MUST get a grant from the server to perform that operation. Grants are requested following the {{-GNAP}} protocol and are issued with one or more access tokens that are passed in the subsequent request to perform the operation.

## Access

The Open Payment APIs are resource oriented and {operations} are initiated by creating a new resource. The state of the operation is determined by reading the state of the resource and the state of an operation is modified by updating the resource.

The `access` property of each access token returned to the client in a grant response defines what operations are permitted by the client when using that access token. The schema of this data is based on {{Section 8 of -GNAP}}.

### Type {#access-type}

Each `access` object MUST have a `type` property that has one of the following values:

  - `incoming-payment`
  - `outgoing-payment`

The `type` of the grant indicates the `type` of resource that the client is permitted to create or access at the server. The operations these resources represent are defined further in {operations}.

### Actions {#access-actions}

Each `access` object MUST also have an `actions` property which is an array of actions that the grant permits the client to perform on resources of the type defined in the `type` property.

Depending on the type of resource these MAY include:

  - `create`
  - `read`
  - `read-all`
  - `list`
  - `list-all`

#### Create {#access-actions-create}

The `create` action permits the client to create a new resource of the type specified in `type`. Creating a new resource is the mechanism by which an operation is initiated. Creating a resource is done by making an HTTP `POST` request to an API endpoint specified in `locations` which is the URL of the resource collection for resources of the specified type.

Servers MUST return a JSON representation of the newly created resource in the response along with the HTTP response code `201 Created`. Servers MUST assign a unique URL as the identifier of any newly created resource and return this URL as the `id` property in the response.

#### Read {#access-actions-read}

The `read` action permits the client to read the current state of a resource of the type specified in `type` if that resource was created by the client. Servers MUST be capable of identifying which client created a resource and ensuring that a client with a grant to `read` resources is only given access to the resources it has created.

Reading a resource is done by making an HTTP `GET` request to the URL that is also the unique identifier of the resource represented by the `id` property of the resource.

#### Read All {#access-actions-read-all}

The `read-all` action permits the client to read the current state of a resource of the type specified in `type` even if that resource was not created by the client.

#### List {#access-actions-list}

The `list` action permits the client to get a list of all the resources of the type specified in `type`. The server MUST filter the results to only return resources created by the client.

Listing resources is done by making an HTTP `GET` request to the URL of the resource collection which is returned in `locations` in the grant response.

#### List All {#access-actions-list-all}

The `list-all` action permits the client to get a list of all the resources of the type specified in `type` even if that resource was not created by the client.

### Locations {#access-locations}

TODO - provide details of how `locations` indicates the resource collection URL for the grant when create and list are used.

# Operations {#operations}

The operations that can be performed by a client are represented by resources that can be created on the server.

All operations are initiated by creating a new resource through an HTTP `POST` request to the appropriate resource collection endpoint. The request body MUST contain a JSON representation of the resource being created and the request `content-type` header MUST be `application/json`.

The resource created is identified by a URL that is returned in the response under the key `id`. Clients MUST make an HTTP `GET` request to that URL to `read` the state of the operation.

## Anonymous Requests

All client requests MUST be authenticated as described in {client-auth} with the exception of GET requests to a payment pointer.

# Security Considerations

TODO

# IANA Considerations

This document has no IANA actions.


--- back

# Acknowledgments
{:numbered="false"}

TODO acknowledge.
