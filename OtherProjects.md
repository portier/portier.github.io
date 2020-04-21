# Portier and Other Projects

Portier considers itself a spiritual successor to [Mozilla
Persona](https://login.persona.org). Though elements of Portier's design appear
in other projects, the specific combination of starting with an email address,
attempting dynamic discovery, and then failing over to traditional email
confirmation appears to be unique to Portier and Persona.

## Email-First, Connected Systems

[Poetica](https://poetica.com) had an excellent email-first, connected login
flow prior to its acquisition by Condé Nast: Users would enter an email address
and then see a filtered menu of compatible authentication options. For example,
Google Sign-In would appear for people using Gmail, but not for those using
Yahoo! Mail.

Portier follows this user experience, but like Persona, automatically selects
the most specific authentication method available, rather than forcing the user
to choose (and remember) a strategy.

## Magic Links

Many contemporary websites support login-via-email workflows:

- Slack has a "[Magic
  Link](http://louiiisechg.tumblr.com/post/130650909766/slack-magic-link)"
- [Appear.in](https://appear.in), [Sandstorm](https://oasis.sandstorm.io/), and
  [Sideway](https://sideway.com/) offer to "Send Login Email."
- For Node.js projects, the [Passwordless](https://passwordless.net/) module
  implements the same.
- [Auth0's Passwordless](https://auth0.com/passwordless) offers this kind of
  experience as a service.

These workflows are all email-only. In contrast, Portier first attempts to use
web-based, federated protocols before falling back to a magic link.

## Confirmation Codes

Many services send information to one device in order to verify interactions on
another: Two-Factor Authentication (2FA) solutions often work by texting a
short code that they must transcribe before users are allowed to sign in.
[Digits](https://get.digits.com), by Twitter, offers this style of SMS-based
[authentication as a
service](https://blog.twitter.com/2015/launching-digits-login-for-web).

Portier uses SMTP, but the principle and user experience are comparable: a user
can prove control of an email address via a code or link sent only to that
address. The ability to manually transcribe codes is important, since it eases
authentication for users with smartphones: glance at the phone, copy the code,
done.

Nevertheless, web-based authentication with a federated protocol is still
preferable when available, since it eliminates context switches for the user.

## Enterprise-Ready Solutions

CoreOS's open source [Dex](https://github.com/coreos/dex) and Red Hat's
[Keycloak](http://keycloak.org) (sold as "[Red Hat Single
Sign-On](https://access.redhat.com/products/red-hat-single-sign-on)") offer
more extensive, enterprise-level feature sets.

These projects may serve as inspiration for future work on Portier.

## Portier Compared to Persona

>   __Note__: Initial interest in Portier will likely come from individuals
>   familiar with the Persona project, hence the detailed retrospective below.
>   In the following section, "we" refers to the Persona core team.

Though imbued with Persona's spirit, Portier is decidedly less ambitious and
more narrowly scoped.

### Where Persona Failed

Persona made choices that, in retrospect, were overly ambitious or even
contradictory to its goal of solving authentication on the Web:

-   __Persona had a user-visible brand__. We encouraged buttons labeled
    "Sign-in with Persona," which confused users. In practice, __Persona was
    just infrastructure__, and should have been (non-)branded as such. "Sign-in
    with Email" better described Persona's function and requirements.
-   __Persona had conflicting missions__. While Persona was initially designed
    to solve authentication for the Web, the team was later tasked with
    additionally solving authentication for Firefox OS handsets and Mozilla's
    corporate infrastructure using the same code. This required adding __opt-in
    centralization and mandatory passwords to a decentralized,
    password-optional protocol__. Other features of centralized systems,
    including support for unverified email addresses and forced invalidation of
    federated credentials, were also implemented. This added complexity slowed
    Persona's development.
-   __Persona tried to protect users from their own email providers__. While
    laudable, meaningful privacy from a user's own email provider is __an
    impossible goal__ that unnecessarily constrained Persona's design: whenever
    a website sends an email, that privacy is violated. Instead, users who are
    concerned about such privacy are empowered to choose providers that they
    trust, or run their own provider, without any degradation in user
    experience.
-   __Persona subverted websites' session management__. If a user's session
    expired on a website, Persona would forcibly log back in. While well
    intentioned, this expanded Persona's scope and intrusively overrode the
    site's own session management. Instead of serving developers, __Persona
    forced websites to conform to it__. Session management also had poor
    failure modes which could trap users in an infinite loop of perpetually
    failing login attempts.

In addition to high-level scope creep, we made tactical decisions that did not
succeed:

-   __Persona had its own, broken account system__. While designed to reduce
    friction, Persona's account system introduced __a point of centralization__
    in an otherwise decentralized system and created __complex state
    interactions__ that proved to be more of a hindrance than convenience.

    For example, each email address could only belong to a single Persona
    account. This made using shared addresses extremely difficult, and it was
    even possible to accidentally split your own addresses between two accounts
    with no clear way to merge them.

-   __Persona had its own passwords__. Because email confirmation loops were
    considered onerous, we asked users to complete a traditional confirmation
    loop once per address and establish a Persona-specific password for future
    interactions. This centralized password meant that users could still
    __authenticate after losing access to an email address__, breaking
    Persona's promise of verified identities.

    This also created user experience challenges: if an email provider added
    support for the BrowserID protocol, Persona would use that instead of
    passwords. Later, if they removed support for BrowserID, Persona would
    suddenly resume asking users for a password they had not used in months.

-   __Persona had a long memory__. Persona remembered every time it saw an
    email provider that supported the BrowserID protocol. If that provider
    later removed support for BrowserID, Persona would wait a week before
    falling back to its own email confirmation system, under the presumption
    that the domain was having a temporary technical problem. This ensured that
    Persona never assumed undue authority over a domain, but also meant that
    __users could be locked out of websites for a week__.

    This also complicated planning for providers wanting to add support for
    BrowserID: Persona's long-lived state meant that reverting a bad deployment
    could inadvertently trigger a week-long lockout. 

-   __The BrowserID protocol was all-or-nothing.__ Because the protocol worked
    at the domain level, it wasn't possible for providers to turn on support
    for a subset of users before rolling it out to everyone at the domain. This
    made deployments and upgrades riskier than necessary.

-   __Persona used pop-ups for its UI.__ This had the benefit of preserving
    page state during log-in. Unfortunately, pop-ups are easily lost when
    juggling windows, hard to associate with a specific tab, and __many users
    have a reflexive aversion to pop-ups__, closing the Persona window before
    it could load. There were also technical issues with pop-up blockers and
    mainstream browsers like Chrome for iOS that did not support the
    `window.opener` property for communicating back to the parent tab.

-   __Persona relied on unfinished specifications.__ Persona was an early
    adopter of the JSON Web Token (JWT) standard and associated JOSE
    specifications. Unfortunately, the specifications were not yet stable, and
    made frequent, backwards incompatible changes. As a result, __we had to
    maintain Persona-specific libraries__, rather than using third party
    libraries that tracked the current versions of the specifications.

-   __Persona required third party cookies__. In the absence of native browser
    support, interactions with Persona happened via invisible iframes rooted at
    external origins. Privacy-conscious users often manually disable browser
    support for third party cookies, or use browser add-ons that do the same.
    This configuration rendered Persona inoperable.

-   __Persona overemphasized native browser support__. There are simply too
    many platforms and too much bureaucratic inertia to effectively pursue
    native implementation in a majority of browsers. __The Web platform has no
    such limitations__. Once we realized this, we pivoted to a Web-first
    strategy, treating browser support as an optional enhancement.

-   __Persona was impossible to self-host or fork__. We inadvertently designed
    __intractable points of centralization__ into Persona, some of which we
    only fully recognized when we tried to map out a future for Persona
    independent of Mozilla.

    For example, in the absence of native browser support, identity providers
    and websites using Persona had to use a third party service to relay
    messages across origins. Both parties had to select the same relay ahead of
    time, *without ever communicating*, otherwise the protocol would stall and
    fail. We "solved" this by expecting everyone to use the same,
    Mozilla-operated relay. However, reliance on this service meant that
    __without native browser support, Persona could not be decentralized__: it
    would fracture into mutually incompatible subnetworks.


### The Fallacy of the Three-Way Cold Start

We often cite a __three-way cold start__ between browser vendors, websites, and
identity providers for Persona's failure. In retrospect, this __incorrectly
evaluates Persona as a consumer product__ that needed growth, visibility, and
network effects to be considered successful, rather than as infrastructure that
simply needed to be reliable and independently useful.

When websites rejected Persona because it didn't have enough users or name
recognition, we should have realized that __we had mis-positioned Persona as a
social authentication button__. We were pursuing the wrong goals.

Similarly, the emphasis on native browser support in the hypothetical three-way
cold start is misplaced: Persona targeted the standard Web platform. It worked
well in all browsers, and for all email addresses. Why, then, would adoption be
contingent on the marginal UX improvements associated with native support? Not
to mention that Mozilla could have independently shipped support in Firefox and
broken this assumed deadlock. That we consciously chose to focus on other tasks
indicates that we did not truly believe native support was critical to
Persona's success.

### What Persona Got Right

Persona *did* get many things right, which Portier preserves:

-   __Email as Identity__. Email is universal, decentralized, and pseudonymous.
    Most sites already collect it during registration, it is commonly used as a
    natural key for accounts, and it allows for out-of-band communication with
    users.
-   __Protocol Bridges__. Developing bespoke integrations with providers like
    Gmail offers an immediately improved experience to hundreds of millions of
    people. Simultaneously, it serves as a direct replacement for existing
    social authentication buttons on websites, but without the UI clutter, API
    keys, or multiple protocols.
-   __Decentralization__. To maintain its sovereignty, the Web needs a modern
    authentication system that isn't controlled by or reliant on a single,
    for-profit, American corporation like Facebook, Google, Twitter, or GitHub.
-   __Self-Certification__. Similarly, to safeguard user choice on the Web,
    domains must be able to certify their own users.
-   __Web First__. The Web is the only universal platform. Solutions must,
    first and foremost, work well on the Web.
-   __Hosted Verifier__. Persona offered a hosted service that would handle the
    more complex steps of verifying identity certificates. This may not be
    necessary now that the JWT spec has stabilized, but the principle of
    delegating computation to an optional, hosted service significantly lowered
    Persona's barrier of adoption.
