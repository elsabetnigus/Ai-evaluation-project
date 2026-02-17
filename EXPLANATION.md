# EXPLANATION

## What was the bug?

`Client.request()` in `app/http_client.py` never sent an `Authorization` header
when `oauth2_token` was set to a plain dict. The class type annotation explicitly
permits `Union[OAuth2Token, Dict[str, Any], None]`, so storing a dict is a valid
state — but the refresh guard silently skipped it, leaving every API call
unauthenticated without raising any error or warning to the caller.

## Why did it happen?

The refresh condition was:

```python
if not self.oauth2_token or (
    isinstance(self.oauth2_token, OAuth2Token) and self.oauth2_token.expired
):
```

A non-empty dict is truthy, so `not self.oauth2_token` evaluates to `False`. The
second branch also evaluates to `False` because a dict is not an `OAuth2Token`
instance. The whole expression is `False`, so `refresh_oauth2()` is never called.
The follow-up `isinstance` guard then also fails, meaning no `Authorization`
header is set at all on the outgoing request.

## Why does your fix actually solve it?

The condition was replaced with:

```python
if not self.oauth2_token or not isinstance(self.oauth2_token, OAuth2Token) or self.oauth2_token.expired:
```

The new `not isinstance(self.oauth2_token, OAuth2Token)` clause catches every
token value that is not a proper `OAuth2Token` — regardless of its truthiness —
and forces a refresh. After the refresh the token is a valid `OAuth2Token` and
the `Authorization` header is set correctly on every API request.

## One realistic edge case the tests still don't cover

Concurrent requests fired at the same instant when the token is expired. Both
threads could independently evaluate `expired` as `True`, both call
`refresh_oauth2()`, and the second write would silently overwrite the first fresh token. Depending on timing,
one request might still use a stale header. No existing test covers this parallel refresh
race condition.
