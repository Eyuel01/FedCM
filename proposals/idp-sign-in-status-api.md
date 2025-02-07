# Explainer for IDP Login Status API (formerly known as IDP Sign-in Status API)

## Background

The FedCM API lets a website make cross-origin credentialed requests. We want
to limit websites’ ability to make such requests silently without causing user
annoyance; in particular, we do not want FedCM calls to show a "Sign in with
IDPX" dialog every time a user visits a website if the user is not logged in to
IDPX.

The problem with allowing such requests silently is a privacy issue as follows.
In a world without third-party cookies, it is generally impossible to make
cross-site credentialed requests because they would allow tracking users. FedCM
introduces a credentialed request to the accounts endpoint. This request does
not send data identifying the requestor and also does not allow passing through
data provided by the RP, however if the RP (perhaps through an SDK of some kind
provided by the tracker) makes an uncredentialed request providing its URL or
other data, then the server can correlate the uncredentialed request with the
credentialed request (stochastically) using the IP address and timing, or other
fingerprinting data. See [these
slides](https://github.com/fedidcg/FedCM/blob/main/meetings/2022/FedCM_%20Options%20for%20the%20Timing%20Attack%20Problem%202022-08-31.pdf)
for more details. 

To solve this problem (and to slightly optimize network traffic), we propose
this API to let identity providers (IDPs) tell the browser when the user logs
in to and out of the IDP. The IDP Sign-in Status API does not grant the
identity provider any permissions. Using the IDP login status API is not a
way for a site to prove that it is an identity provider. The purpose of the
IDP Sign-in Status API is to enable identity providers to disable the FedCM API
for their IDP in order to deliver a better user experience.

In addition, with the browser gaining knowledge about IDPs, this may allow
finer-grained user control over which IDPs are available to websites, e.g. a
settings page allowing the user to disable certain IDPs for use with FedCM.


## Proposed API

The "Headers" and "JS API" sections describe the proposed
[Login Status API](https://github.com/fedidcg/login-status). We describe them
in this document as well for easier readability.

### Headers

```http
Set-Login: logged-in
Set-Login: logged-out
```

These headers can be sent on the toplevel load as well as subresources such as
XMLHttpRequest (this is necessary for at least one IDP).

The logout header should only be sent when no accounts remain logged in
to the IDP, i.e., when this action has logged out all accounts or if this
was the last/only account getting logged out.

This will be parsed as a [structured field](https://fetch.spec.whatwg.org/#concept-header-list-get-structured-header)
with type `item`. For now, we only use bare item part, but this lets us extend
the header to support parameters in the future.

### JS API

```idl

enum LoginStatus {
  "logged-in",
  "logged-out",
};

interface NavigatorLogin {
  Promise<void> setStatus(LoginStatus status);
};

partial interface Navigator {
  [SecureContext] readonly NavigatorLogin login;
};
```

Alternatively, an IdP can call the IdP Sign-in Status API via JS calls through
the static functions `navigator.login.setStatus("logged-in")` and
`navigator.login.setStatus("logged-out")`. These are to be called from the IDP's
origin, and mark the current origin as logged-in or logged-out.

```idl
[Exposed=Window]
interface IdentityProvider {
  static void close();
}
```

In addition, an `IdentityProvider.close()` function is provided to signal to the browser that the
login flow is finished. The reason for this function in addition to the header
is that even when the user is already logged in, the login flow may not be
finished yet; for example, an IDP may want to prompt the user to verify their
phone number. To allow for such flows, the IDP must call
`IdentityProvider.close()` when the flow is fully done.

### Config file addition

This proposal adds a new `login_url` field to [the config file](https://fedidcg.github.io/FedCM/#dictdef-identityproviderapiconfig):


```
partial dictionary IdentityProviderAPIConfig {
	USVString login_url;
}
```

See further below for a description of the semantics.


### Semantics

For each IDP (identified by its config URL) the browser keeps a tri-state
variable representing the login state with possible values “logged-in”,
“logged-out”, and “unknown”, defaulting to “unknown”.

When receiving the login header, the state will be set to “logged in”. In
case of subresources, to limit abuse, the header is only processed if the
resource is same-origin with the document.

Similar for the logout header.

In some cases, a user can get logged out server-side while the user is not on
the IDP website. For example, the IDP may require re-authentication every N
days, or the user may have changed their password (or deleted their account) on
a different browser, forcing re-login. This proposal does not have special
handling in this situation; we would show the error dialog mentioned below.

There is [some discussion](https://crbug.com/1381505) on whether the login header should require user activation; however, right now in Chrome it does not.


### Effect on FedCM requests

When an RP calls navigator.credentials.get():

* If the login state on the provided config URL is “logged out”, no
network requests will be made and the promise is rejected (with a delay
as usual (step 3 of
[the algorithm](https://fedidcg.github.io/FedCM/#dom-identitycredential-discoverfromexternalsource-slot)))
* Otherwise, network requests are made as usual

When the accounts endpoint response is successful and has at least one account:

* The login state is set to “logged-in” if it was previously “unknown”


When an error is received from the accounts endpoint or no accounts are returned:

* If the login state was unknown, the login state is set to “logged out”. No UI is displayed and the promise is rejected as usual
    * This is used when launching this API, when the browser has no stored IDP login data, and also when an IDP starts supporting FedCM, where the user can also be logged in without the login status being set. This allows us to handle these cases without being required to show UI when the user is not logged in
    * This does incur a one-time timing attack per IDP. Since this can only happen once per IDP/browser profile, it seems impractical for an attacker to rely on.
        * An alternative solution is to show the login UI even in this case
* If the login state was “logged in”, the login state is set to “logged out”. An error dialog is displayed that also allows the user to login to the IDP. The exact UI is TBD; the dialog may not explicitly say something like “we thought you were logged in to the IDP”.
    * The primary case where this will happen is if the session is invalidated server-side, either because of session-length settings, because the user forced logout on other devices, or other reasons.
    * We show a dialog in this situation to discourage trackers using this
    * This dialog is why there is a login URL being added in this proposal, so that the user has a way to recover instead of being presented with a useless dialog. However, having this URL is also useful for other UI enhancements.


## Alternatives considered

### Big picture

See [https://github.com/fedidcg/FedCM/blob/main/meetings/2022/FedCM_%20Options%20for%20the%20Timing%20Attack%20Problem%202022-08-31.pdf](https://github.com/fedidcg/FedCM/blob/main/meetings/2022/FedCM_%20Options%20for%20the%20Timing%20Attack%20Problem%202022-08-31.pdf) 

### Header syntax

We could instead or in addition have allowed notifying the user agent of
individual accounts being logged in/out, such as:

```
Set-Login: logged-in; count=2; type=idp
Set-Login: logged-out; new-count=1; type=idp
```

Or

```
Set-Login: logged-in; accountid=foo@bar.com; type=idp
Set-Login: logged-out; accountid=foo@bar.com; type=idp
```

However, we decided to go with the simpler syntax because we do not currently
have a use case that requires the extra information.

Additionally, the second option would require the browser to track which
specific account IDs are logged in, so that it can tell when there no
more logged in accounts for this IDP. This introduces extra complexity,
whereas the IDP already knows how many accounts are logged in and thus
whether no accounts remain after this logout action.
