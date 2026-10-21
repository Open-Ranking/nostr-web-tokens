# NWT Examples

Non-normative examples of how NWTs can be used. These do not change the [specification](README.md).

## Session Authentication with Cookies

A NWT may be used to authenticate a whole browser session with cookies, so the user signs once per session instead of once per request, and requests that can't carry an `Authorization` header (e.g. `<img>`, `<video>`, links, form posts) are still authenticated.

### Login

1. The client signs a NWT with `aud` identifying the server(s), `exp` set to the desired session length (e.g. a few hours), and any custom claims the server requires.
2. The client sends the NWT to the server's login endpoint using the `Authorization` header. The endpoint is application-defined.
3. The server validates the NWT and responds with a cookie containing the same Base64URL-encoded token.

```
POST /login HTTP/1.1
Host: api.example.com
Authorization: Nostr <base64url-no-padding-token>
```

```
HTTP/1.1 204 No Content
Set-Cookie: __Host-nwt=<base64url-no-padding-token>; HttpOnly; Secure; SameSite=Lax; Path=/; Max-Age=<exp - now>
```

On subsequent requests the server reads the NWT from the cookie and validates it following the [validation rules](README.md#validation-rules), exactly as if it was sent in the `Authorization` header. The server does not need to store any session state.

A NWT with multiple `aud` values may be sent to the login endpoint of each of those servers, authenticating a session with all of them using a single user signature.

### Cookie Attributes

The server:

* must set the `HttpOnly` and `Secure` attributes.
* should set `SameSite=Lax` or `SameSite=Strict`.
* should use the `__Host-` cookie name prefix, which prevents other subdomains from setting or overwriting the cookie.
* must not set `Max-Age` or `Expires` beyond the NWT's `exp`.

Base64URL without padding only contains cookie-safe characters, so no further encoding is needed.

If a request contains both an `Authorization` header and a NWT cookie, the server must use the `Authorization` header.

### Cross-Site Request Forgery

Unlike the `Authorization` header, cookies are attached to requests automatically by the browser. Servers should protect state-changing requests (e.g. `POST`, `PUT`, `DELETE`) authenticated by cookie against [CSRF](https://en.wikipedia.org/wiki/Cross-site_request_forgery), by relying on `SameSite` and/or checking the `Origin` header.

### Logout and Renewal

To log out, the server clears the cookie (`Max-Age=0`). Since the NWT remains valid until its `exp`, servers should reject its event `id` until then.

When the NWT expires, the server responds with `401 Unauthorized` and the client signs a new NWT and logs in again.

Servers may require a [challenge](README.md#challenge-response) at the login endpoint, to prevent a captured login NWT from being used to start another session.

## Replacing NIP-98

A service using [NIP-98](https://github.com/nostr-protocol/nips/blob/master/98.md) can switch to NWT with minimal changes, using [HTTP request binding](README.md#http-request-binding).

### Before: NIP-98

```json
{
    "kind": 27235,
    "created_at": 1710000000,
    "tags": [
        ["u", "https://api.example.com/v1/notes"],
        ["method", "POST"],
        ["payload", "5c4b5e2a...e1f3"]
    ],
    "content": ""
}
```

The server accepts the event if `created_at` is within a fixed window (e.g. 60 seconds) of the current time.

### After: request-bound NWT

```json
{
    "kind": 27519,
    "created_at": 1710000000,
    "tags": [
        ["u", "https://api.example.com/v1/notes"],
        ["method", "POST"],
        ["payload", "5c4b5e2a...e1f3"],
        ["exp", "1710000060"]
    ],
    "content": "publish a note"
}
```

The only changes are the `kind` and the `exp` claim. Instead of checking `created_at` against its own window, the server enforces the `exp` chosen by the client.

Both are sent the same way:

```
POST /v1/notes HTTP/1.1
Host: api.example.com
Authorization: Nostr <base64url-no-padding-token>
```

### Next step: session NWT

Request-bound NWTs still require a signature for every request. Once the server accepts NWTs, the client can instead sign a single NWT that is valid for every request to the service for the next hour:

```json
{
    "kind": 27519,
    "created_at": 1710000000,
    "tags": [
        ["aud", "api.example.com"],
        ["exp", "1710003600"]
    ],
    "content": "sign in to api.example.com"
}
```

The client sends the same token in the `Authorization` header of every request until it expires, or exchanges it for a [session cookie](#session-authentication-with-cookies).
