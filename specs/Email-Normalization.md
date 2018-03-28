# Portier Email Normalization

As one of the first steps during authentication, a Portier broker
implementation normalizes the email address as provided by the user. This
document describes the normalization process.

This document is relevant to the implementation of a broker. It may also be
relevant to the implementation of a Relying Party (RP), when a verified email
address (provided to the RP by its broker) needs to be compared with input
obtained by other means.

The algorithm in this document may also be useful in general when comparing two
email addresses obtained from separate input.

This document builds on the [Unicode] and [WHATWG URL] specifications. If
implementation of the algorithm proves difficult in a specific runtime, this
document also defines a normalization API over HTTPS, typically implemented by
the broker.

 [Unicode]: https://www.unicode.org/versions/Unicode10.0.0/
 [WHATWG URL]: https://url.spec.whatwg.org/

## Normalization algorithm

Normalization takes a string _input_, and runs these steps:

1. Find the first occurence of U+0040 (@) in _input_. Let _user_ be the part
   before this point, and _domain_ be the part after this point.

2. Ensure that _user_ and _domain_ are not empty, otherwise return failure.

3. Let _asciiDomain_ be the result of running [domain to ASCII], as per [WHATWG
   URL], on _domain_.

   * Client implementations may also use the full [host parsing] algorithm, but
     MUST ensure the result is a domain. An IPv4 address or IPv6 address alawys
     results in failure.

     Servers (such as a broker) MUST NOT use the full algorithm.

4. If _asciiDomain_ is failure, return failure.

5. If _asciiDomain_ contains a [forbidden host code point], as per [WHATWG
   URL], return failure.

6. Let _ipv4Host_ be the result of [IPv4 parsing] _asciiDomain_.

7. If _ipv4Host_ is an IPv4 address or failure, return failure.

8. Let _lowercaseUser_ be the result of mapping each code point _c_ in _user_
   using toLowercase(_c_), according to [Unicode] Default Case Conversion
   algorithm.

   * Notably, implementors MUST ensure their Unicode library provides the full
     lowercase mapping algorithm, and not just the simple variant.

9. Let _output_ be _lowercaseUser_, U+0040 (@), and _asciiDomain_ concatenated.

10. Return _output_.

 [domain to ASCII]: https://url.spec.whatwg.org/#concept-domain-to-ascii
 [host parsing]: https://url.spec.whatwg.org/#host-parsing
 [IPv4 parsing]: https://url.spec.whatwg.org/#concept-ipv4-parser
 [forbidden host code point]: https://url.spec.whatwg.org/#forbidden-host-code-point

## Normalized form

In some validation steps of the Portier protocol, we mandate that an input
email address is in 'normalized form'. Here 'normalized form' means the email
address MUST conform to the output format of the normalization algorithm.

In other words, when given an email address in normalized form as input,
running the normalization algorithm on it MUST result in an output that doesn't
change from / exactly matches the input.

## Normalization API

A normalization API over HTTPS is defined here for use from runtime
environments that cannot fully support an implementation of the normalization
algorithm. This API is typically implemented by the broker.

* Note that secure connections are a requirement. Clients and servers MUST use
  HTTPS to implement the API, with the exception being test deployments that
  are not in any way exposed to real-world usage.

A client may send an HTTPS request `POST /normalize` with a header
`Content-Type: text/plain; charset=utf-8` and a plain text entity containing
one email address per line.

A server will process each line from the request entity, normalize the address
using the above algorithm, and build a plain text response entity containing
one normalized address per line, preserving input order. Failure of the
algorithm results in an empty line in the reponse entity at the matching input
position.

A server response always has status code 200 under normal circumstances, even
when the request entity was empty. The response MUST include a header
`Content-Type: text/plain; charset=utf-8`.

For both requests and responses, lines are separated by a line feed (U+000A),
or a carriage return and line feed (U+000D U+000A). A trailing line separator
is allowed and MAY be ignored.

Both parties MUST ensure no additional whitespace is present around addresses.
