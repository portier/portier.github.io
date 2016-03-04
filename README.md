# Let's Auth!

Let's Auth is an upcoming spiritual successor to Persona.

We are still in the early planning and prototyping stage.

## Project Meetings + Chat

The project meets [every Wednesday at 4 PM UTC](http://arewemeetingyet.com/UTC/2016-03-02/16:00/w/Let's%20Auth%20Weekly%20Meeting) in `#letsauth` on Freenode.

You can also use [our synchronized Gitter room](https://gitter.im/letsauth/LetsAuth) instead of IRC.

## Contributing

There's plenty to do, and everyone is welcome! Check out the open issues and feel free to jump right in.

We're currently focused on two main repositories:

- [letsauth.github.io](https://github.com/letsauth/letsauth.github.io/issues) -
    Project documentation and planning
- [oidc-prototype](https://github.com/letsauth/oidc-prototype/issues) - Prototype Let's Auth daemon that exposes an API compatible with OpenID Connect

## Vision + Goals + Non-Goals

Let's Auth aims to be the simplest way to add email-based authentication (login) to websites, without site-specific passwords.

It frees users from the burden of creating and remembering strong passwords, and by removing the need to handle or store password-derived secrets, it protects developers from risks associated with database breaches.

Let's Auth will be:

-   __Email First__

    Email is the lowest common denominator for identity on the Internet.

-   __Low Friction__

    Where practical, Let's Auth will integrate with protocols like OpenID Connect to provide seamless, in-browser identity verification. In all other cases, it will use traditional confirmation emails.

-   __Free from Lock-in__

    Let's Auth transparently provides the user's verified email address to the website.

    (Technically, we provide [RFC 7565](http://tools.ietf.org/html/rfc7565) account identifiers and make no specific claims as to SMTP-routability. In practice, this distinction should be insignificant.)

-   __Self-hostable__

    Anyone can host their own Let's Auth service; there are no centralized dependencies.

-   __Resilient__

    In addition to self-hosting, sites can arbitrarily switch between Let's Auth services with just a configuration change.

-   __Simple to Deploy__

    Let's Auth 1.0 will ship as a single, statically compiled binary. Pre-1.0, we will use a variety of dynamic languages for prototyping.

-   __Maintainable__

    As a community-driven project, our survival is directly tied to the clarity and simplicity of our code. Every proposed feature will be weighed against its associated maintenance burden.

-   __Language and Framework Agnostic__

    By implementing Let's Auth as a web service, it will integrate with any environment that speaks HTTP.

Equally importantly, Let's Auth _will not_:

-   __Handle Profiles__

    In the interest of simplicity and maintainability, Let's Auth is narrowly focused on authentication, not profile management.

-   __Richly Integrate with Facebook__

    Facebook identities are not naturally thought of in terms of email addresses, so any sort of rich integration would require a bifurcated email-or-facebook workflow. The added maintenance costs and UI/UX complexity are not worth it for the Let's Auth project. Sites and third party libraries are, of course, welcome to combine Let's Auth and Facebook on their own.

-   __Guarantee Users' Privacy from Their Own Email Provider__

    Let's Auth will attempt to protect privacy wherever possible, but future development may require revealing the target website to a user's email provider. We consider this acceptable, as providing any absolute protections is impossible: if a website ever contacts a user, that email implicitly reveals the user's association with the site.

    Privacy-conscious users can exercise control over the providers they trust by choosing a different email provider or self-hosting a domain.

-   __Promise Native Browser Integration__

    We're a small, all-volunteer team. Browser integration would be nice, but it's an enormous amount of work, and universal adoption is so unlikely that sites could never depend on it. Thus, Let's Auth must, first and foremost, work on the Web as it is.

-   __Provide a Single Sign-On Solution__

    Let's Auth is narrowly focused on authenticating email addresses for websites. It would make a great building block for an SSO system, but it is not an SSO system itself.

## Architectural and Design Constraints

We're still early on in the design process, so nearly everything is still up for debate. However, there's decent consensus on the following:

- Let's Auth will exist as a standalone web service, which exposes an HTTP-based API for authenticating email addresses
- The service will only be accessible over secure (SSL/TLS) connections
- A single Let's Auth instance can service requests from an arbitrary number of relying sites
- By default, sites should not need to pre-register with a Let's Auth instance before using it
- Let's Auth instances will not require durable state, aside from static configuration data
- We'll communicate verification claims as signed [JSON Web Tokens](https://jwt.io)

Our initial prototype will aim to be compatible with the OpenID Connect specs, in order to minimize duplication of effort and ease integration with existing libraries.

## Resources

__Inspiration:__

- [Passwordless](https://passwordless.net) has similar goals, but implemented as a Node.js library instead of a standalone web service. We should learn from its design and features. Passwordless does not attempt to integrate with OpenID Connect or other protocols to streamline authentication.

- [Poetica's sign in page](https://poetica.com/signin) is a beautiful example of email-first authentication. Let's Auth should look similar, but be more opinionated. For example, Poetica offers Gmail users an option of OpenID Connect or a confirmation email. Instead of presenting users with a choice, Let's Auth should automatically choose the best strategy.

- [Digits](https://get.digits.com/) does a wonderful job of authenticating users by sending short verification codes over SMS. Our confirmation emails should offer a similar option: either click the link, or type in a code. This would make it possible for a user to complete authentication on a computer by glancing at their smartphone for the confirmation code. This has nice UX and privacy properties since the user isn't actually required to directly sign into their email account on the first device.

__OpenID Connect:__

Our initial prototype, [letsauth/oidc-prototype](https://github.com/letsauth/oidc-prototype), aims to expose an interface that's compatible with OpenID Connect's provider discovery protocol and "implicit" authentication workflow. If successful, this would avoid duplicating effort, and would allow sites to leverage existing OpenID Connect client libraries when integrating Let's Auth.

- [Core 1.0](http://openid.net/specs/openid-connect-core-1_0.html)
- [Discovery 1.0](http://openid.net/specs/openid-connect-discovery-1_0.html)
- [Implicit Flow Client Implementer's Guide 1.0](http://openid.net/specs/openid-connect-implicit-1_0.html)

__JSON Web Tokens (JWTs)__

- [jwt.io](https://jwt.io)
- [RFC 7519](https://tools.ietf.org/html/rfc7519)
