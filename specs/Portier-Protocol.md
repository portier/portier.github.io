# Portier Authentication Protocol

In Portier, a Relying Party (RP) delegates User authentication to a trusted
Broker. The Broker in turn can optionally delegate authentication to an
Identity Provider (IdP) appointed by the User's email site administrator.

This document describes the protocol used in both of these scenarios. The
protocols used between the RP and the Broker, and the Broker and the IdP are
the same, differing only in trust. An RP is _configured_ to trust a specific
Broker, whereas a Broker performs _run-time checks_ to verify trust in an IdP.

The protocol described here is based on a strict subset of the OAuth2, OpenID
Connect and JSON Web Token protocols, with some optional Portier-specific
extensions. This document will however avoid referencing the specifications of
these underlying protocols, and instead describe each step here.

This document _does_ assume familiarity with lower-level protocols such
as [HTTPS] and [JSON].

OAuth2 uses the terms 'Client' and 'Server' to describe the two parties
communicating. To accomodate both scenarios above, this document will do the
same, applying the terms to refer to communication between:

- an RP as Client and a Broker as Server, or
- a Broker as Client and an IdP as Server.

Note that both the Client and Server roles are often web applications, running
code on a physical server machine. The terms should not be confused with
networking terminology. A web browser running on the User's workstation is here
referred to as the User Agent (UA).

## Definition of trust

Trust is used in Portier to determine if a Client accepts signed tokens from a
Server for an email address. For the two types of Client defined in this
document, it is applied as follows:

- An RP Client uses a Broker Server it was configured to use by an
  administrator. In this scenario, the Client trusts signed tokens from the
  Server for _any_ email address.

- A Broker Client finds IdP Servers through a discovery mechanism based on the
  email address. In this scenario, the Client trusts signed tokens from the
  Server for _only_ the email address it was discovered through.

These rules are reflected in the steps below.

## Security considerations

Communication between all parties (the UA, the Client and the Server) happens
through HTTPS requests and responses. This means URLs MUST have the `https`
scheme, and use protocols standardized for use with this URL scheme. A Client
on the open internet environment MUST ensure connections with the Server and UA
use only these secure protocols.

In testing or local deployments, a Client MAY choose to use insecure HTTP
connections with the UA, the Server, or both, when a connection spans only
trusted networks. The remainder of this document will however NOT account for
these scenarios, and will describe HTTPS as mandatory everywhere.

## Interationalized Domain Name considerations

Portier supports Internationalized Domain Names (IDN), and allows the user to
input an email address with non-ASCII characters. The Broker automatically
applies [email normalization] to all addresses, which is mostly transparent to
the RP and IdP.

However, RPs and IdPs MUST use ASCII-serialization for the origin when handling
URLs and origins. Not doing so may cause token validation to fail. This
implies using Punycode for the domain in the URL, if necessary.

## Client starts authentication

These steps are performed by a Client that wishes to authenticate an _email_
with the Server at the HTTPS _serverOrigin_, and wishes to have the result
returned in an HTTPS request from the UA to the _redirectUri_ with optionally
some (string) _state_ attached.

For an RP Client, the _serverOrigin_ is configured by an administrator to a
Broker Server. For a Broker Client, the _serverOrigin_ of an IdP Server is
discovered based on _email_.

For an RP Client, the _email_ MAY be normalized. For a Broker Client, the
_email_ MUST be normalized. A Broker does this on behalf of RPs when
necessary.

1. Let _config_ be the result of [Client fetches configuration], with _origin_
   set to _serverOrigin_.

2. Let _responseMode_ be the preferred response mode for the Client, selected
   from the `response_modes_supported` property of _config_.

   - The response mode determines how the request to _redirectUri_ is made.
     See step 1 of [Client completes authentication].

   - The supported response modes for this specification are `form_post` and
     `fragment`. An RP Client MUST support _at least one_ of these. A Broker
     Client MUST support _both_ of these. The recommended mode is `form_post`,
     because it often saves a round-trip.

   - While the `query` response mode MAY be listed in the configuration, it
     SHOULD NOT be used to implement this specification. Query parameters may
     leak information through a HTTPS `Referer` header, or in a webserver
     access log.

3. Let _clientId_ be the origin of _redirectUri_.

4. The Client may now optionally extend _clientId_ with parameters. If so, the
   origin in _clientId_ is immediately followed by U+003F (?) and then
   name-value pairs. Pairs are separated by U+0026 (&), and each name-value
   pair is split at the first U+003D (=).

   - This format is similar to the [application/x-www-form-urlencoded] format,
     and implementations MAY use a full parser/serializer for this format.
     However, this version of the Portier specification does not require more
     elaborate parsing than what is described above.

   The Server MUST ignore parameters it does not understand. It is the
   responsibility of Clients to only send parameters the Server understands by
   inspecting _config_.

   This specification defines only one optional parameter:

   - `id_token_signed_response_alg` may be set to one of the values indicated
     by the `id_token_signing_alg_values_supported` property of _config_ (a
     list of strings), if available.

     Some algorithms may use several types of keys. For example, the EdDSA
     algorithm may use Ed25519 or Ed448 keys, indicated by the `crv` property
     in the key JSON. In these situations, the Client SHOULD also perform
     [Client fetches keys], inspect the result, and refrain from selecting the
     algorithm if it finds key types it cannot support.

     If the Client does not make a selection, the authentication flow proceeds
     with the default value `RS256`.

5. Let _nonce_ be a randomly generated string.

   - A 'nonce' is a number used once. It is later returned unchanged to the
     Client inside the Server-signed token, which the Client must compare with
     its original value. This ensures the token is used only once.

   - The nonce SHOULD contain sufficient random data, to prevent collisions.
     The recommendation is to generate at least 16 bytes of random data from a
     secure generator, and encode it in hexadecimals, [base64url], or some
     other encoding suitable for URL query parameters.

   - The nonce MUST be generated on the system that later verifies the token.
     For example, do NOT generate a nonce on the UA, then use it to verify a
     token on the Client. This would allow a malicious UA to trivially replay a
     token.

6. Store a session record containing _nonce_, _clientId_ and _email_, in a
   location it can be retrieved from in a later HTTPS request.

   - While this MAY be stored in data associated with the UA, such as a
     cookie-based session, this is NOT recommended. This would prevent the User
     from completing the authentication attempt on another UA (such as another
     device).

   - The optional _state_ may be used to identify the record. For example, it
     can contain an ID for a database record tracking the authentication
     attempt.

   - The Client MAY omit _clientId_ from the record if its value is constant.
     For example, if it does not intend to use parameters, and _clientId_ is
     thus always simply the origin of _redirectUri_. The remainder of this
     specification will ignore this detail.

     What matters is that, at the time the Client receives the callback to
     _redirectUri_, it still knows the value of _clientId_ used to start the
     authentication flow, and is not influenced by changes the Server may have
     made to its configuration in the mean time.

7. Let _authUrl_ be the URL from the `authorization_endpoint` property of
   _config_, with the following query parameters appended:

   - `login_hint` set to _email_ (OPTIONAL, user is prompted if missing)

   - `scope` set to the string `openid email`

   - `nonce` set to _nonce_

   - `state` optionally set to _state_

   - `response_type` set to the string `id_token`

   - `client_id` set to _clientId_

   - `redirect_uri` set to _redirectUri_

   - `response_mode` set to _responseMode_, optional if this is `fragment`

8. Redirect the UA to _authUrl_.

   - The Client SHOULD use the `303 See Other` HTTPS status code.

## Server performs authentication

A Server that receives an HTTPS `GET` request from a UA at its authorization
endpoint runs these steps:

**TODO**

## Client completes authentication

A Client that receives an HTTPS request at _redirectUri_ from a UA runs these
steps:

1. Let _params_ be the parameters extracted from the request according to the
   response mode of the request.

   - If the Client supports the `form_post` response mode, and the request used
     the `POST` method, the parameters are extracted by [form-urldecoding] the
     request body.

   - If the Client supports the `fragment` response mode, and the request used
     the `GET` method, the parameters are extracted by [form-urldecoding] the
     fragment part of the URL.

     Note that this part of the URL is only available to the UA, and it is up
     to the implementor to transport these to the Client if necessary.

2. Let _token_, _state_, _error_ and _errorDescription_ be the values of the
   parameters `id_token`, `state`, `error` and `error_description` in _params_
   respectively, any of which may be unset.

   - The `state` parameter contains the state originally provided by the Client
     when it started authentication, but MUST be treated as untrusted input.

3. If _error_ is set, or if _token_ is not set, return failure.

   - The value of _error_ indicates the type of failure, one of:

     - `invalid_request`, equivalent to HTTPS status code 400.
     - `temporarily unavailable`, equivalent to HTTPS status code 503.
     - `server_error`, equivalent to HTTPS status code 500.

   - The optional _errorDescription_ may contain a more detailed reason for the
     failure. Its value SHOULD NOT be relied on for comparison, and it MAY be
     localized for the user.

4. Let _serverOrigin_ be the origin of the Server that is expected to have
   instructed the UA to make this request to the Client.

   - For an RP Client, this is simply the configured Broker Server origin.

   - For a Broker Client, this is usually determined by looking at a session
     record retrieved through _state_.

     While alternatively the session record MAY be associated with the UA, such
     as a cookie-based session, this is NOT recommended. This would prevent the
     User from completing the authentication attempt on another UA (such as
     another device).

5. Let _config_ be the result of [Client fetches configuration], with _origin_
   set to _serverOrigin_.

6. Let _jwks_ be the result of [Client fetches keys] with _url_ set to the URL
   from the `jwks_uri` property of _config_.

7. Let _encodedHeader_, _encodedPayload_, and _encodedSignature_ be the result
   of splitting _token_ at every occurence of U+002E (.), throwing away the
   separators. If _token_ is not exactly 3 parts, return failure.

8. Let _header_ be the result of [JSON]-decoding the result of
   [base64url]-decoding _encodedHeader_.

9. Verify _header_ is a JSON object with the following properties:

   - `alg` MUST be one of the signing algorithms supported by the Client.
     Clients that do not implement any algorithm selection logic MUST verify
     this is the default value `RS256`.

   - `kid` MUST be a non-empty string.

10. Let _key_ be the matching JSON object from the JSON array _keys_, whose
    property `kid` has the same value as the property `kid` from _header_. If
    no match is found, return failure.

11. Let _signature_ be the result of [base64url]-decoding _encodedSignature_.

12. Let _signedPart_ be _encodedHeader_, U+002E (.), and _encodedPayload_
    concatenated.

13. Verify _signature_ is a valid signature for _signedPart_ according the
    signing algorithm specified in the `alg` property of _header_.

    For `RS256`, the Client follows the steps in
    [Client verifies an RS256 signature] with _message_ set to _signedPart_,
    _signature_ set to _signature_, and _key_ set to _key_.

    For `EdDSA`, the Client follows the steps in
    [Client verifies an EdDSA signature] with _message_ set to _signedPart_,
    _signature_ set to _signature_, and _key_ set to _key_.

    Implementations MAY add support for other algorithms outside of this
    specification. For these advanced use-cases this document defers to the
    full OpenID Connect specifications.

14. Let _payload_ be the result of [JSON]-decoding the result of
    [base64url]-decoding _encodedPayload_.

15. Verify _payload_ is a JSON object with the following properties:

    - `iss` MUST be equal to _serverOrigin_

    - `aud` MUST be a non-empty string.

    - `exp`: MUST be a Unix timestamp in seconds, indicating a time later than
      the current system time, allowing for leeway no more than 5 minutes.

    - `iat` MUST be a Unix timestamp in seconds, indicating a time equal to or
      preceding the current system time, allowing for leeway no more than 5
      minutes.

    - `email` MUST be an email address, and it MUST be in normalized form as
      per [email normalization].

    - `email_original` is optional, and if set MUST be an email address (but
      not necessarily in normalized form). When not set, it defaults to the
      value of the `email` property.

    - `nonce` MUST be a non-empty string

16. Let _nonce_ be the value of the `nonce` property from _payload_.

17. Let _originalEmail_ be the value of the `email_original` property (or its
    default value) from _payload_.

18. Let _clientId_ be the value of the `aud` property from _payload_.

19. Verify a session record exists matching _nonce_, _clientId_, and
    _originalEmail_ exactly, as recorded by the Client when it started the
    authentication attempt.

    - This refers to step 3 through 6 of [Client starts authentication].

    - The optional _state_ may be used to identify the record.

20. Invalidate the session record from the previous step, so that it cannot be
    used in future attempts.

    - This is meant to ensure that the nonce (number used once) is in fact only
      used once.

21. Verify the `alg` property of _header_ matches the signing algorithm
    selection made when Client started the authentication attempt.

    - This can be retrieved from `id_token_signed_response_alg` parameter in
      _clientId_. If not present, or if the Client does not implement any
      algorithm selection logic, it MUST verify this is the default value
      `RS256`.

22. Let _email_ be the value of the _email_ property from _payload_.

23. Verify _serverOrigin_ is trusted to sign tokens for _email_, or return
    failure.

    - For an RP Client, the Server is the configured Broker. An RP Client
      SHOULD simply trust _any_ email address from its configured Broker.

    - For a Broker Client, the Server is a discovered IdP, and _email_ was
      already normalized by the Broker Client. If _email_ differs from
      _originalEmail_, this algorithm MUST return failure.

24. Return _email_.

    - This value can now be treated as trusted, as we've verified the Server's
      signature of the token, all claims in the signed token, and that we
      accept signed tokens from the Server for this email address.

## Subroutines

### Client requests HTTPS resource

For various purposes, a Client sometimes needs to fetch an external HTTPS
resource. These resources are addressed using an URL with the `https` scheme,
and MUST be retrieved using the standardized connection protocols for this URL
scheme. The result is the resource body on success. A response with an error
status code (4xx or 5xx) results in a failure.

The actual implementation is up to the implementor, but a Client MUST follow
these additional considerations:

- The security parameters of a connection MUST always be verified. This
  includes (but is not limited to) verifying certificates and ensuring the
  cryptographic algorithms used are acceptable for use on the open internet.
  (In most cases, this is already ensured by up-to-date HTTPS implementations
  of libraries and operating systems commonly used.)

- The resource can respond with a redirect. If it does, the Client MUST verify
  the redirection is to another resource with the `https` URL scheme. New
  connections MUST be verified to be secure, as described in the previous
  point.

- The resource can include cache validators in the response (and intermediate
  redirect responses) which SHOULD be followed. For details on HTTPS caching,
  see [section 13 of RFC 2616].

### Client fetches configuration

A Client that needs to fetch the configuration document for an _origin_
performs the following steps:

1. Let _url_ be the _origin_ with the path `/.well-known/openid-configuration`
   appended.

2. Let _data_ be the result of [Client requests HTTPS resource] for _url_.

3. Let _config_ be the result of [JSON]-decoding _data_.

4. Verify _config_ is a JSON object with the following required properties:

   - `authorization_endpoint` MUST be a valid URL with the `https` scheme

   - `jwks_uri` MUST be a valid URL with the `https` scheme

   - `response_modes_supported` is optional, and if set MUST be an array of
     strings. When not set, it defaults to `["fragment"]`.

     - The supported response modes for this specification are `form_post` and
       `fragment`. An RP Client MUST support _at least one_ of these. A Broker
       Client MUST support _both_ of these. The recommended mode is
       `form_post`, because it often saves a round-trip.

     - While the `query` response mode MAY be listed in the configuration, it
       MUST NOT be used to implement this specification. Query parameters may
       leak information through a HTTPS `Referer` header, or in a webserver
       access log.

5. Return _config_.

### Client fetches keys

A Client that needs to fetch the public keys from a _url_ performs the
following steps:

1. Let _data_ be the result of [Client requests HTTPS resource] for _url_.

2. Let _doc_ be the result of [JSON]-decoding _data_.

3. Let _keys_ be the _keys_ property of _doc_, or return failure if missing.

4. Verify _keys_ is an array of objects.

5. Remove items from _keys_ that do not have a string property `kid`.

   - The Server MUST provide a unique `kid` for each item. However, Clients
     SHOULD NOT fail these steps if this constraint is violated. This
     specification leaves the behavior of Clients mostly undefined in this
     case, except that Clients MUST pick only one key when matching `kid`
     during signature verification steps.

5. Remove items from _keys_ that do not have a property `use` with value `sig`.

4. Return the value of _keys_.

### Client verifies an RS256 signature

A Client that needs to verify an RS256 _signature_ for _message_ is correct for
the given public _key_ in JSON JWK format performs the following steps:

1. Verify the `kty` property of _key_ is set to `RSA`, or return failure.

2. Let _n_ be the result of [base64url]-decoding the `n` property of _key_.
   If the property is missing, return failure.

3. Let _e_ be the result of [base64url]-decoding the `e` property of _key_.
   If the property is missing, return failure.

4. Let _keyObject_ be the [RSA public key] composed of _n_ and _e_.

5. Verify _signature_ is a valid signature for _message_ according to
   [RSASSA-PKCS1-v1_5] using hash algorithm SHA256 and the public key
   _keyObject_.

### Client verifies an EdDSA signature

A Client that needs to verify an EdDSA _signature_ for _message_ is correct for
the given public _key_ in JSON JWK format performs the following steps:

1. Verify the `kty` property of _key_ is set to `OKP`, or return failure.

2. Verify the value of the `crv` property of _key_ is an identifier for an
   Edwards curve supported by the Client EdDSA implementation, or return
   failure.

   - This specification recommends implementations use `Ed25519`.

3. Let _x_ be the result of [base64url]-decoding the `x` property of _key_.
   If the property is missing, return failure.

4. Verify _signature_ is a valid signature for _message_ according to
   [EdDSA] using the Edwards curve from the value of the `crv` property of
   _key_ and the public key _x_.

[application/x-www-form-urlencoded]: https://url.spec.whatwg.org/#application/x-www-form-urlencoded
[base64url]: https://tools.ietf.org/html/rfc4648#section-5
[client completes authentication]: #client-completes-authentication
[client fetches configuration]: #client-fetches-configuration
[client fetches keys]: #client-fetches-keys
[client requests https resource]: #client-requests-https-resource
[client starts authentication]: #client-starts-authentication
[client verifies an eddsa signature]: #client-verifies-an-eddsa-signature
[client verifies an rs256 signature]: #client-verifies-an-rs256-signature
[eddsa]: https://ed25519.cr.yp.to/
[email normalization]: ./Email-Normalization.md
[form-urldecoding]: https://url.spec.whatwg.org/#application/x-www-form-urlencoded
[https]: https://tools.ietf.org/html/rfc2616
[json]: https://tools.ietf.org/html/rfc8259
[rsa public key]: https://tools.ietf.org/html/rfc3447#section-3.1
[rsassa-pkcs1-v1_5]: https://tools.ietf.org/html/rfc3447#section-8.2
[section 13 of rfc 2616]: https://tools.ietf.org/html/rfc2616#section-13
[simple https fetch]: #simple-https-fetch
