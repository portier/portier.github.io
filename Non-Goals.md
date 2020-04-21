# Portier's Non-Goals

Equally important to what Portier will do is what it _will not_ do. Many of
these decisions are taken in the interest of simplicity and maintainability: we
want Portier to be as well-defined, understandable, and maintainable as
possible. Thus, we explicitly reject features that would dramatically expand
Portier's scope, increase its complexity, or require significant changes to its
user experience.

Portier is a distinct tool, not an omnibus toolkit.

Specifically, Portier will not:

*   __Provide profile metadata__: Portier is narrowly focused on authenticating
    email addresses. Names, photos, and other metadata are out of scope.

*   __Integrate with Facebook__: People do not naturally think of their
    Facebook accounts as email addresses, while Portier is fundamentally driven
    by the notion of email address as identity. As such, direct integration
    with Facebook does not make sense for Portier. Websites may, of course,
    offer Facebook Connect as an authentication option alongside Portier.

*   __Be a Single Sign-On service__: Though Portier could be useful when build
    an SSO system, it is not itself an SSO system.

*   __Remember multiple identities__: Portier only verifies email addresses. It
    doesn't remember those addresses or make claims about associations between
    multiple addresses.

*   __Promise native browser integration__: Portier must work for everyone on
    the Web, regardless of browser. The complexity, bureaucracy, and time
    required for native integration is not currently worth its ill-defined
    benefits, especially with promising standards from the [FIDO
    Alliance](https://fidoalliance.org/) on the horizon. In contrast, Portier
    is relatively simple and designed to solve a specific problem _today_.

*   __Protect users from their own email providers__: A malicious email
    provider is able to impersonate users within its own domain, and may be
    able to observe where its users are logging in. However, this is true of
    _all_ authentication systems with email-based account recovery workflows.
    Portier is not in a position to properly solve this problem.

    Privacy-conscious users can control which parties they must trust by
    selecting a different email provider, or self-hosting their own domain.
