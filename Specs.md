# Portier Specifications

The Portier protocol is composed of several smaller documents listed on this page.

## Terminology

Some common terminology is used everywhere we try define the Portier protocol:

- User: The person that wants to authenticate with a Relying Party.

- User Agent (UA): A web browser, or similar application, being directly
  controlled by the User.

- Relying Party (RP): A website that wants to authenticate Users by their email
  address, using Portier.

- Identity: A handle that identifies the online presence of a User. In Portier,
  this is an email address.

- Identity Provider (IdP): A website appointed by the User to control their
  Identity.

- Identity Provider Discovery: The method with which the Identity Provider is
  discovered, based on the User email address.

- Broker: A website that implements User authentication for the Relying Party,
  optionally using Identity Provider Discovery and delagating to an Identity
  Provider.

## Documents

- [Email Normalization](specs/Email-Normalization.md), relevant to broker
  implementors, and possibly relying parties that need to compare email
  addresses.

- Relying Party Authentication Protocol (TODO)

- Identity Provider Discovery Protocol (TODO)

- Identity Provider Authentication Protocol (TODO)

## Referenced External Standards

To avoid reinventing the wheel, Portier is built atop existing, proven
standards. Specifically, the Portier Broker exposes itself to websites as a
discoverable OpenID Connect (OIDC) Identity Provider supporting the "implicit"
authentication workflow.

> FIXME: Add more detail, link to specific sections of specs, use more precise
> language when defining what exactly Portier implements

## OpenID Connect

The relevant OpenID Connect specifications are:

- [Core 1.0](http://openid.net/specs/openid-connect-core-1_0.html), the main
  OpenID Connect specification.
- [Discovery 1.0](http://openid.net/specs/openid-connect-discovery-1_0.html),
  which describes how websites can dynamically discover and interact with
  OpenID Connect Identity Providers, including the Portier Broker.

## OAuth 2.0

OpenID Connect is built atop the OAuth 2.0 Core specification, with a few extension specifications that are used by Portier.

- [Core](https://tools.ietf.org/html/rfc6749) (RFC 6749), the protocol atop
  which OpenID Connect was built.

- [Multiple Respone Type Encoding Practices](http://openid.net/specs/oauth-v2-multiple-response-types-1_0.html),
  an extension by the OpenID Foundation, defines the `response_mode` request
  parameter and several new `response_type` values, including `id_token` and
  `none`.

- [Form Post Response Mode](http://openid.net/specs/oauth-v2-form-post-response-mode-1_0.html),
  another extension by the OpenID Foundation, defines the new `form_post`
  response mode, which Portier uses.

## JSON Web Tokens (JWTs)

Much of the data that Portier exchanges is in the form of JSON Web Tokens (JWTs), which are cryptographically signed JSON documents.

- [RFC 7519](https://tools.ietf.org/html/rfc7519) defines JSON Web Tokens.

- [jwt.io](https://jwt.io) offers learning materials and a great interactive
  tool for parsing and verifying JWTs.
