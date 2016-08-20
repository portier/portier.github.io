# Portier and Other Projects

Portier considers itself a spiritual successor to [Mozilla Persona](https://login.persona.org). Though elements of Portier's design appear in other projects, the specific combination of starting with an email address, attempting dynamic discovery, and then failing over to traditional email confirmation appears to be unique to Portier and Persona.

In addition to Persona, Portier takes inspiration from [Poetica](https://poetica.com"), which had an excellent email-first login flow prior to its acquisition by Condé Nast. Specifically, users would enter an email address and then see a menu of compatible authentication options. Google Sign-In wouldn't appear for Yahoo! Mail users, and vice versa. Portier will work similarly, but will automatically select the best available method instead of prompting users. Poetica is also incapable discovering self-hosted providers, forcing users to choose between convenience and data sovereignty.

There are many contemporary examples of login-via-email workflows. Slack has a "[Magic Link](http://louiiisechg.tumblr.com/post/130650909766/slack-magic-link)," while [Sandstorm](https://oasis.sandstorm.io/) and [Sideway](https://sideway.com/) offer to "Send Login Email." The [Passwordless](https://passwordless.net/) module for Node.js implements the same, as does [Auth0's Passwordless](https://auth0.com/passwordless) service. None of these systems are able to integrate with third party, web-based sign-in APIs.

Finally, there are many examples of sending information to one device in order to verify interactions on another: Two-Factor Authentication (2FA) systems often ask users to finish logging in by transcribing a short code sent to their phone via SMS. Though Portier uses SMTP, the principle and user experience are comparable. [Digits](https://get.digits.com), by Twitter, offers this style of SMS-based verification as a service.

## Portier Compared to Persona

Though imbued with Persona's spirit, Portier is decidedly less ambitious and more narrowly scoped.

> FIXME: Write this. Cover things like privacy from IdP, email address as id,
> progressive enhancement via bridges, decentralization, native browser support,
> same origin policy workarounds, three-way cold start, complexity, session
> management, fallback accounts / passwords, the entire notion of an account,
> associating multiple email addresses, visibility as an independent product vs
> an infrastructure implementation detail, web native, custom or off-the-shelf
> protocols, hosted verifier, budget, failure modes, third party cookies,
> self-certification.
