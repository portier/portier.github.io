# Portier Authentication Protocol

In Portier, a Relying Party (RP) delegates User authentication to a trusted
Broker. The Broker in turn can optionally delegate authentication to an
Identity Provider (IdP) appointed by the User's email site administrator.

This document describes the protocol used in both of these scenarios. The
protocols used between the RP and the Broker, and the Broker and the IdP are
the same, differing only in trust. An RP is *configured* to trust a specific
Broker, whereas a Broker performs *run-time checks* to verify trust in an IdP.

The protocol described here is based on a strict subset of the OAuth2, OpenID
Connect and JSON Web Token protocols, with some Portier-specific extensions
added. This document will however avoid referencing the specifications of these
underlying protocols, and instead describe each step here.

This document *does* assume familiarity with lower-level protocols such
as [HTTPS] and [JSON].

OAuth2 uses the terms 'Client' and 'Server' to describe the two parties
communicating. To accomodate both scenarios above, this document will do the
same, applying the terms to refer to communication between:

 * an RP as Client and a Broker as Server, or
 * a Broker as Client and an IdP as Server.

Note that both the Client and Server roles are often web applications, running
code on a physical server machine. The terms should not be confused with
networking terminology. A web browser running on the User's workstation is here
referred to as the User Agent (UA).

## Definition of trust

Trust is used in Portier to determine if a Client accepts signed tokens from a
Server for an email address. For the two types of Client defined in this
document, it is applied as follows:

 * An RP Client uses a Broker Server it was configured to use by an
   administrator. In this scenario, the Client trusts signed tokens from the
   Server for *any* email address.

 * A Broker Client finds IdP Servers through a discovery mechanism based on the
   email address. In this scenario, the Client initially trusts signed tokens
   from the Server for *only* the email address it was discovered through.

   A Server can change the email address during authentication, even to another
   domain. Such a change is verified separately by the Client, through
   rerunning the discovery mechanism, to ensure the Server also controls the
   new email address. This allows the Server to, for example, return the
   canonical email address for a User, who instead used an alias as input.

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
URLs and origins. Not doing so may cause token validation to fail.  This
implies using Punycode for the domain in the URL, if necessary.

## Client starts authentication

These steps are performed by a Client that wishes to authenticate an _email_
with the Server at the HTTPS _serverOrigin_, and wishes to have the result
returned in an HTTPS request from the UA to the _redirectUri_ with optionally
some (string) _state_ attached.

For an RP Client, the _serverOrigin_ is configured by an administrator to a
Broker Server. For a Broker Client, the _serverOrigin_ of an IdP Server is
discovered based on _email_.

1. Let _config_ be the result of [Client fetches configuration], with _origin_
   set to _serverOrigin_.

2. Let _responseMode_ be the preferred response mode for the Client, selected
   from the `response_modes_supported` property of _config_.

   * The response mode determines how the request to _redirectUri_ is made.
     See step 1 of [Client completes authentication].

   * The supported response modes for this specification are `form_post` and
     `fragment`. An RP Client MUST support *at least one* of these. A Broker
     Client MUST support *both* of these. The recommended mode is `form_post`,
     because it often saves a round-trip.

   * While the `query` response mode MAY be listed in the configuration, it
     MUST NOT be used to implement this specification. Query parameters may
     leak information through a HTTPS `Referer` header, or in a webserver
     access log.

3. Let _nonce_ be a randomly generated string.

   * A 'nonce' is a number used once. It is later returned unchanged to the
     Client inside the Server-signed token, which the Client must compare with
     its original value. This ensures the token is used only once.

   * The nonce SHOULD contain sufficient random data, to prevent collisions.
     The recommendation is to generate at least 16 bytes of random data from a
     secure generator, and encode it in hexadecimals, [base64url], or some
     other encoding suitable for URL query parameters.

   * The nonce MUST be generated on the system that later verifies the token.
     For example, do NOT generate a nonce on the UA, then use it to verify a
     token on the Client. This would allow a malicious UA to trivially replay a
     token.

4. Store a session record containing both _nonce_ and _email_, in a location it
   can be retrieved from in a later HTTPS request.

   * While this MAY be stored in data associated with the UA, such as a
     cookie-based session, this is NOT recommended. This would prevent the User
     from completing the authentication attempt on another UA (such as another
     device).

   * The optional _state_ may be used to identify the record. For example, it
     can contain an ID for a database record tracking the authentication
     attempt.

5. Let _authUrl_ be the URL from the `authorization_endpoint` property of
   _config_, with the following query parameters appended:

   * `login_hint` set to _email_ (OPTIONAL, user is prompted if missing)

   * `scope` set to the string `openid email`

   * `nonce` set to _nonce_

   * `state` optionally set to _state_

   * `response_type` set to the string `id_token`

   * `client_id` set to the origin of _redirectUri_

   * `redirect_uri` set to _redirectUri_

   * `response_mode` set to _responseMode_, optional if this is `fragment`

6. Redirect the UA to _authUrl_.

   * The Client SHOULD use the `303 See Other` HTTPS status code.

## Server performs authentication

A Server that receives an HTTPS `GET` request from a UA at its authorization
endpoint runs these steps:

**TODO**

## Client completes authentication

A Client that receives an HTTPS request at _redirectUri_ from a UA runs these
steps:

1. Let _params_ be the parameters extracted from the request according to the
   response mode of the request.

   * If the Client supports the `form_post` response mode, and the request used
     the `POST` method, the parameters are extracted by [form-urldecoding] the
     request body.

   * If the Client supports the `fragment` response mode, and the request used
     the `GET` method, the parameters are extracted by [form-urldecoding] the
     fragment part of the URL.

     Note that this part of the URL is only available to the UA, and it is up
     to the implementor to transport these to the Client if necessary.

2. Let _token_, _state_, _error_ and _errorDescription_ be the values of the
   parameters `id_token`, `state`, `error` and `error_description` in _params_
   respectively, any of which may be unset.

   * The `state` parameter contains the state originally provided by the Client
     when it started authentication, but MUST be treated as untrusted input.

3. If _error_ is set, or if _token_ is not set, return failure.

   * The value of _error_ indicates the type of failure, one of:

     * `invalid_request`, equivalent to HTTPS status code 400.
     * `temporarily unavailable`, equivalent to HTTPS status code 503.
     * `server_error`, equivalent to HTTPS status code 500.

   * The optional _errorDescription_ may contain a more detailed reason for the
     failure. Its value SHOULD NOT be relied on for comparison, and it MAY be
     localized for the user.

4. Let _serverOrigin_ be the origin of the Server that is expected to have
   instructed the UA to make this request to the Client.

   * For an RP Client, this is simply the configured Broker Server origin.

   * For a Broker Client, this is usually determined by looking at a session
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

   * `alg` MUST be the string `RS256`

   * `kid` MUST be a non-empty string

10. Let _key_ be the matching JSON object from the JSON array _keys_, whose
    property `kid` has the same value as the property `kid` from _header_. If
    no match is found, return failure.

11. Let _n_ be the result of [base64url]-decoding the `n` property of _key_.

12. Let _e_ be the result of [base64url]-decoding the `e` property of _key_.

13. Let _keyObject_ be the [RSA public key] composed of _n_ and _e_.

14. Let _signature_ be the result of [base64url]-decoding _encodedSignature_.

15. Let _signedPart_ be _encodedHeader_, U+002E (.), and _encodedPayload_
    concatenated.

16. Verify _signature_ is a valid signature for _signedPart_ according to
    [RSASSA-PKCS1-v1_5] using hash algorithm SHA256 and the public key
    _keyObject_.

17. Let _payload_ be the result of [JSON]-decoding the result of
    [base64url]-decoding _encodedPayload_.

18. Verify _payload_ is a JSON object with the following properties:

    * `iss` MUST be equal to _serverOrigin_

    * `aud` MUST be equal to the origin of _redirectUri_
      (Typically the Client origin.)

    * `exp`: MUST be a Unix timestamp in seconds, indicating a time later than
      the current system time, allowing for leeway no more than 5 minutes.

    * `iat` MUST be a Unix timestamp in seconds, indicating a time equal to or
      preceding the current system time, allowing for leeway no more than 5
      minutes.

    * `email` MUST be an email address, and it MUST be in normalized form as
      per [email normalization].

    * `email_original` is optional, and if set MUST be an email address (but
      not necessarily in normalized form). When not set, it defaults to the
      value of the `email` property.

    * `nonce` MUST be a non-empty string

19. Let _originalEmail_ be the value of the `email_original` property (or its
    default value) from _payload_.

20. Let _nonce_ be the value of the `nonce` property from _payload_.

21. Verify a session record exists matching both _nonce_ and _originalEmail_
    exactly, as recorded by the Client when it started the authentication
    attempt.

    * This refers to step 4 of [Client starts authentication].

    * The optional _state_ may be used to identify the record.

22. Invalidate the session record from the previous step, so that it cannot be
    used in future attempts.

    * This is meant to ensure that the nonce (number used once) is in fact only
      used once.

23. Let _email_ be the value of the _email_ property from _payload_.

24. Verify _serverOrigin_ is trusted to sign tokens for _email_, or return
    failure.

    * For an RP Client, the Server is the configured Broker. An RP Client
      SHOULD simply trust *any* email address from its configured Broker.

    * For a Broker Client, the Server is a discovered IdP. If _email_ differs
      from _originalEmail_, the Broker MUST use its discovery mechanism to
      establish that the same IdP Server indeed also controls _email_.

25. Return _email_.

    * This value can now be treated as trusted, as we've verified the Server's
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

* The security parameters of a connection MUST always be verified. This
  includes (but is not limited to) verifying certificates and ensuring the
  cryptographic algorithms used are acceptable for use on the open internet.
  (In most cases, this is already ensured by up-to-date HTTPS implementations
  of libraries and operating systems commonly used.)

* The resource can respond with a redirect. If it does, the Client MUST verify
  the redirection is to another resource with the `https` URL scheme. New
  connections MUST be verified to be secure, as described in the previous
  point.

* The resource can include cache validators in the response (and intermediate
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

   * `authorization_endpoint` MUST be a valid URL with the `https` scheme

   * `jwks_uri` MUST be a valid URL with the `https` scheme

   * `response_modes_supported` is optional, and if set MUST be an array of
     strings. When not set, it defaults to `["fragment"]`.

     * The supported response modes for this specification are `form_post` and
       `fragment`. An RP Client MUST support *at least one* of these. A Broker
       Client MUST support *both* of these. The recommended mode is
       `form_post`, because it often saves a round-trip.

     * While the `query` response mode MAY be listed in the configuration, it
       MUST NOT be used to implement this specification. Query parameters may
       leak information through a HTTPS `Referer` header, or in a webserver
       access log.

5. Return _config_.

### Client fetches keys

A Client that needs to fetch the public keys from a _url_ performs the
following steps:

1. Let _data_ be the result of [Client requests HTTPS resource] for _url_.

3. Let _keys_ be the result of [JSON]-decoding _data_.

4. Verify _keys_ is a JSON object with the following required property:

   * `keys` MUST be a JSON array of objects, with the following required
     properties:

     * `kty` MUST be the string `RSA`

     * `alg` MUST be the string `RS256`

     * `use` MUST be the string `sig`

     * `kid` MUST be a non-empty string unique among objects in the array

     * `n` MUST be a string containing a [base64url]-encoded big integer

     * `e` MUST be a string containing a [base64url]-encoded big integer

5. Return the JSON array from the `keys` property of _keys_.

 [Client completes authentication]: #client-completes-authentication
 [Client fetches configuration]: #client-fetches-configuration
 [Client fetches keys]: #client-fetches-keys
 [Client requests HTTPS resource]: #client-requests-https-resource
 [Client starts authentication]: #client-starts-authentication
 [Simple HTTPS fetch]: #simple-https-fetch

 [Email normalization]: ./Email-Normalization.md

 [Base64url]: https://tools.ietf.org/html/rfc4648#section-5
 [Form-urldecoding]: https://url.spec.whatwg.org/#application/x-www-form-urlencoded
 [HTTPS]: https://tools.ietf.org/html/rfc2616
 [JSON]: https://tools.ietf.org/html/rfc8259
 [RSA public key]: https://tools.ietf.org/html/rfc3447#section-3.1
 [RSASSA-PKCS1-v1_5]: https://tools.ietf.org/html/rfc3447#section-8.2
 [Section 13 of RFC 2616]: https://tools.ietf.org/html/rfc2616#section-13
