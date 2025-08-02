---
title: "The Sneaky DRF Authentication Bug: When 401 Turns into 403"
classes: wide
header:
  teaser: /assets/images/posts/drf-auth-bug/teaser.png
  overlay_image: /assets/images/posts/drf-auth-bug/teaser.png
  overlay_filter: 0.3
ribbon: Crimson
excerpt: "A silent bug in Django Rest Framework caused my API to return 403 instead of 401. The culprit? Authentication class order. Here's what I discovered deep in the source."
description: "How DRF's authentication class order causes incorrect HTTP response codes - and why it's arguably a bug"
categories:
  - Django Rest Framework
  - Bugs
  - Blog
tags:
  - Django
  - DRF
  - Authentication
  - HTTP Status Codes
  - Bug Hunt
  - Python
toc: true
toc_sticky: true
toc_label: "DRF Auth Internals"
toc_icon: "bug"
---

# The sneaky bug

Not all bugs scream at you. Some whisper in 403s.

I recently spent a lot of time debugging a strange behavior in a Django Rest Framework (DRF) API. When a request failed authentication, it returned a **403 Forbidden** instead of **401 Unauthorized** - and this subtle difference broke parts of my application logic.

After triple-checking everything and diving deep into DRF’s internals, I found the unexpected root cause: **the order of authentication classes in your settings matters more than you think**.

Let me walk you through what happened, what I learned, and why I believe this is a real bug in DRF.

## Background: DRF Authentication Flow

In DRF, authentication is handled by a list of classes defined in `DEFAULT_AUTHENTICATION_CLASSES`. Each class tries to authenticate the request, raising `AuthenticationFailed` on failure or returning `None` if it chooses not to act.

When authentication fails, DRF must return either a **401 Unauthorized** or **403 Forbidden**. According to spec, a 401 response **must** include a `WWW-Authenticate` header. DRF checks each class’s `authenticate_header()` method to decide which status to send.

If no class provides a header-that is, if all return `None`-DRF defaults to 403. Otherwise, it returns 401.

## This Is Known-and Here’s Why (but I Still Think It’s a Bug)

DRF documentation clearly states:

> "Although multiple authentication schemes may be in use, only one scheme may be used to determine the type of response. ***The first*** authentication class set on the view is used when determining the type of response." [DRF docs](https://www.django-rest-framework.org/api-guide/authentication/#unauthorized-and-forbidden-responses)

And [GitHub issue #3800](https://github.com/encode/django-rest-framework/issues/3800) shows this has been an intentional design decision for years.

### Why Was This Designed This Way?

Session-based authentication doesn’t define `authenticate_header()`, so returning a 401 with no header would violate RFC 7235. DRF therefore treats session auth failure as 403.

Because DRF uses the **first** authenticator’s header presence to decide status, even if a later class actually raises `AuthenticationFailed`, its header gets ignored if an earlier class returns `None`.

The rationale: session first → no header → 403; token-first → header present → 401.

## My Setup

I had this configuration:

```python
REST_FRAMEWORK = {
    'DEFAULT_AUTHENTICATION_CLASSES': (
        rest_framework.authentication.SessionAuthentication,
        rest_framework.authentication.TokenAuthentication,
        myapp.authentication.BearerAuthentication,
    )
}
```

My `BearerAuthentication` inherits from `BaseAuthentication` and returns `"Bearer"` in `authenticate_header()`.

Yet failed bearer requests returned **403**, not **401**-because `SessionAuthentication` came first.

## DRF Source Code Digging

In `rest_framework/views.py` (around line 189), DRF handles exceptions like this:

```python
# simplified from rest_framework.views APIView.handle_exception()

# WWW-Authenticate header for 401 responses, else coerce to 403
auth_header = self.get_authenticate_header(self.request)

if auth_header:
    exc.auth_header = auth_header
else:
    exc.status_code = status.HTTP_403_FORBIDDEN
```

While `auth_header` equals:

```python
# authentication_classes = DEFAULT_AUTHENTICATION_CLASSES or per-view
authenticators = [auth() for auth in self.authentication_classes]
if authenticators:
    return authenticators[0].authenticate_header(request) # returns the first no matter what
```
That means DRF picks the **first** `authenticate_header()` among all authenticators, no matter which authenticator actually failed.

Because `SessionAuthentication.authenticate_header()` returns `None`, every failure came through as 403-even when the error originated from my bearer auth.

Once I reordered my settings:

```python
DEFAULT_AUTHENTICATION_CLASSES = (
    myapp.authentication.BearerAuthentication,
    rest_framework.authentication.SessionAuthentication,
    rest_framework.authentication.TokenAuthentication,
)
```

...failed bearer requests finally returned **401** as expected.

---

### Why It Still Matters

DRF treats this behavior as expected and arguably spec‑compliant-but it still goes against intuitive expectations. If a specific authentication scheme fails, clients should receive a response tied to that scheme’s semantics.

Importantly, returning the `WWW-Authenticate` header of the **class that raised** `AuthenticationFailed` would **not violate RFC 7235**. The spec requires that a **401 Unauthorized** response *must* include at least one `WWW-Authenticate` header identifying the correct authentication challenge. Choosing the header from the authenticator that failed actually aligns better with that rule, precisely informing the client which type of credentials are required-without risking a spec violation.([http.dev](https://http.dev/www-authenticate))

In contrast, DRF’s current approach-relying on the first listed authenticator-can lead to misleading 403 responses when a later scheme actually fails.

## Proposed Fix and Upcoming PR

I plan to open a pull request addressing issue #3800 with:

* A minimal reproducer test case
* A patch that captures the `authenticate_header()` from the authenticator that *raised* `AuthenticationFailed`, not just from the first in the list
* An explanation of how this improves DRF behavior to better match RFC 7235 and developer expectations

Until then, the only reliable workaround is:

👉 **Always put the authenticator likely to raise errors (e.g. your BearerAuthentication) *first* in the list, or implement your own exception handler**

## Conclusion

This subtle auth-status behavior can sneakily break API logic or client-side handling. DRF’s current design serves spec compliance-but sacrifices developer intuition when multiple authentication schemes are involved.

I hope this deep dive helps you avoid the same trap.