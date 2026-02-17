````markdown
# HTTP Client OAuth2 Token Refresh Bug Explanation

## What was the bug?

In `app/http_client.py`, the logic that decides when to refresh the OAuth2 token inside the `request()` method only triggered a refresh under very specific conditions: either when the token was `None`/falsy, **or** when it was an instance of `OAuth2Token` that had expired.  

However, the class also allows tokens to be provided as a **dictionary** (`Dict[str, Any]`), per its type annotation:

```python
Union[OAuth2Token, Dict[str, Any], None]
````

If a dictionary token was passed, the original condition never triggered a refresh. For example, a token like `{"access_token": "stale", "expires_at": 0}` is a non-empty dict, which evaluates as truthy. The first check (`not self.oauth2_token`) fails, and the second check (`isinstance(self.oauth2_token, OAuth2Token) and self.oauth2_token.expired`) also fails because the token isn’t an `OAuth2Token` instance. As a result, the refresh logic is completely skipped, and the subsequent `Authorization` header is never set.

---

## Why did it happen?

The original conditional looked like this:

```python
if not self.oauth2_token or (
    isinstance(self.oauth2_token, OAuth2Token) and self.oauth2_token.expired
):
```

Breaking it down:

1. `not self.oauth2_token` → Only triggers for `None` or empty values.
2. `isinstance(self.oauth2_token, OAuth2Token) and self.oauth2_token.expired` → Only triggers if the token is a proper `OAuth2Token` instance and it has expired.

So any token that is a non-empty dictionary passes both checks, causing the refresh logic to be bypassed entirely.

---

## How the fix solves it

The revised condition is:

```python
if not self.oauth2_token or not isinstance(self.oauth2_token, OAuth2Token) or self.oauth2_token.expired:
```

The key addition is `not isinstance(self.oauth2_token, OAuth2Token)`.

* Now, **any token that is not a proper `OAuth2Token` instance**, including dictionaries, triggers the refresh.
* After this refresh, the token is guaranteed to be a valid `OAuth2Token` instance, so the `Authorization` header is set correctly on subsequent requests.
* This ensures that both dictionary-based tokens and expired `OAuth2Token` instances are handled consistently.

---

## Edge case still not covered by tests

One subtle scenario that isn’t tested: an `OAuth2Token` whose `expires_at` is exactly equal to the current Unix timestamp — essentially expiring *right now*.

* The `expired` property uses `>=`, so the token is treated as expired and triggers a refresh.
* However, on a very slow system, the system clock might tick forward between the `expired` check and the `refresh_oauth2()` call. In that split second, the refreshed token could also be considered expired, creating a potential race condition.

This timing-sensitive edge case is not yet captured by the automated tests.

```