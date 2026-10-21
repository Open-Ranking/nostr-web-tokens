# Nostr Web Token (NWT)

`draft` `optional`

## Overview

A **Nostr Web Token (NWT)** is a **signed Nostr event** as defined by [NIP-01](https://github.com/nostr-protocol/nips/blob/master/01.md) used to convey signed claims between parties on the web.

Claims are pieces of information asserted about a subject. A claim is identified by a name, and associated with one or more values.

A NWT is not a JSON Web Token (JWT), but adopts its semantics where applicable.

## Event

A NWT has a `kind 27519`, in reference to [REC-7519](https://datatracker.ietf.org/doc/html/rfc7519) that defines the JWT.

Claims are represented as `tags`, following JWT naming conventions and semantics where applicable. Claims that allow a single value MUST appear only once; claims that allow multiple values are represented by repeated tags.

### Example

```jsonc
{
    "id": <32-bytes hex-encoded sha256>,
    "pubkey": <32-bytes hex-encoded public key>,
    "created_at": <unix timestamp in seconds>,
    "kind": 27519,
    "tags": [
        ["aud", "blossom.example.com"],
        ["aud", "cdn.another.com"],
        ["aud", "cdn.last.one"],
        ["exp", "1710003600"],
        ["nbf", "1710000000"],
        ["action", "upload"],               // custom claim
        ["payload", "b1674191a...dsad"]     // custom claim
    ],
    "content": "upload bitcoin.pdf",
    "sig": <64-bytes hex-encoded signature>
}
```

### Registered Claims

| Claim     |   Tag  |   Status   | Type |     Cardinality     |
|-----------|--------|------------|------ | --------------------|
| Issuer | `iss` | Optional | string | single |
| Subject | `sub` | Optional | string | single |
| Audience | `aud` | Recommended | string | multi | 
| Issued At | `iat` | Optional | stringified timestamp* | single |
| Expiration | `exp` | Recommended | stringified timestamp* | single |
| Not Before | `nbf` | Optional | stringified timestamp* | single |

\* Timestamp values MUST be base-10 non-negative integer seconds since the Unix epoch. Verifiers SHOULD allow a small clock skew (e.g. ±60 seconds).

#### Issuer `iss`

The `iss` (issuer) claim identifies the entity that issued the NWT.  
This tag is OPTIONAL. If not present, the issuer of the NWT is assumed to be the Nostr public key (`pubkey`) that signed the event.

#### Subject `sub`

The `sub` (subject) claim identifies the entity that is the subject of the NWT.  
This tag is OPTIONAL. If not present, the subject of the NWT is assumed to be the Nostr public key (`pubkey`) that signed the event.

#### Audience `aud`

The `aud` (audience) claim identifies the recipients that the NWT is intended for.  
Recipients MAY be of various types, such as pubkeys, domain names or even specific endpoints. A verifier MUST reject a NWT if it doesn't identify itself with a value in the audience claim. The interpretation of "identify itself" is application-specific.  
This tag is OPTIONAL, but recommended. If not present, the audience of the NWT is assumed to be everyone, which may cause [security issues](#security-considerations).

#### Issued At `iat`

The `iat` (issued at) claim identifies the time at which the NWT was issued.  
This tag is OPTIONAL. If not present, the issued timestamp is assumed to be the timestamp of the Nostr event (`created_at`). If present, verifiers MUST use `iat` instead of `created_at`.

#### Expiration `exp`

The `exp` (expiration) claim identifies the expiration time on or after which the NWT MUST NOT be accepted for processing.  
This tag is OPTIONAL, but recommended. If not present, the expiration is assumed to never occur, which may cause [security issues](#security-considerations).

#### Not Before `nbf`

The `nbf` (not before) claim identifies the time before which the NWT MUST NOT be accepted for processing.  
This tag is OPTIONAL.

### Additional Claims

A producer and consumer of a NWT MAY agree to use additional claims (e.g. `roles`, `scope` etc.). These additional claims MUST be encoded as `tags`, however, their definition and intended use is application specific and is not defined here.


### Content

The `content` field SHOULD contain a human readable message intended to be displayed to the signer of the NWT, explaining its intended use. For example, `authorize deletion of fiat.pdf`.


## Validation Rules

A verifier MUST:

1. Verify the Nostr event id.
2. Verify the Nostr event signature.
3. Verify the event `kind` is `27519`.
4. Enforce `exp`, `nbf`, and `aud` claims when present; apply default behaviors when absent.
5. Enforce [request claims](#http-request-binding) (`u`, `method`, `payload`, `challenge`) when present.
6. Validate `pubkey` and `iss` trust (application-defined).


## Transport Encoding

A NWT MAY be transported using any non-destructive encoding.  
Its primary use, however, is transmission via an HTTP `Authorization` header using the `Nostr` scheme, where the token is encoded using `Base64URL` (URL-safe base 64) without padding.

Example:

```
Authorization: Nostr <base64url-no-padding-token>
```


## HTTP Request Binding

A NWT MAY be bound to a single HTTP request, so that it cannot be used for any other request. This provides the same guarantees as [NIP-98](https://github.com/nostr-protocol/nips/blob/master/98.md), and exists to help services migrate off of NIP-98.

Request-bound NWTs are NOT RECOMMENDED. Every request requires a new user signature, which can significantly slow down applications making many requests (e.g. hundreds of requests taking a second each to sign). Services SHOULD instead accept NWTs scoped with `aud`, `exp` and custom claims that authenticate the user for a whole session (see [Session Authentication](EXAMPLES.md#session-authentication-with-cookies)).

### Request Claims

| Claim     |   Tag  |   Status   | Type |     Cardinality     |
|-----------|--------|------------|------ | --------------------|
| URL | `u` | Optional | string | single |
| Method | `method` | Optional | string | single |
| Payload Hash | `payload` | Optional | hex-encoded sha256 | single |
| Challenge | `challenge` | Optional | string | single |

A NWT containing both `u` and `method` is a **request-bound** NWT.

#### URL `u`

The `u` (URL) claim identifies the absolute URL of the request, including query parameters.  
If present, a verifier MUST reject the NWT if it doesn't exactly match the request URL. Since `u` already identifies the recipient, `aud` MAY be omitted from request-bound NWTs.

#### Method `method`

The `method` claim identifies the HTTP method of the request (e.g. `GET`, `POST`), in uppercase.  
If present, a verifier MUST reject the NWT if it doesn't match the request method.

#### Payload Hash `payload`

The `payload` claim is the lowercase hex-encoded SHA-256 hash of the request body.  
If present, a verifier MUST reject the NWT if it doesn't match the hash of the request body. Clients SHOULD include it for requests with a body, and verifiers MAY require it.

#### Challenge `challenge`

The `challenge` claim contains a nonce previously issued by the verifier (see [Challenge-Response](#challenge-response)).  
If present, a verifier MUST reject the NWT if the challenge was not issued by itself, has expired, or has already been used.

### Freshness

Request-bound NWTs MUST include an `exp` claim, and it SHOULD be no more than 60 seconds after `iat`. Verifiers MAY reject request-bound NWTs whose validity window exceeds their own policy.

### Challenge-Response

To guarantee a NWT is used only once, a server MAY require a challenge. Like request binding, this requires a new user signature for every request and is NOT RECOMMENDED for general use. It is, however, well suited to [session login](EXAMPLES.md#session-authentication-with-cookies), which only requires one signature per session.

1. The server responds to an unauthenticated request with `401 Unauthorized` and a `WWW-Authenticate` header containing a challenge:
   ```
   WWW-Authenticate: Nostr challenge="<nonce>"
   ```
2. The client signs a NWT with the `challenge` claim set to the nonce, and retries the request.
3. The server validates the NWT and marks the challenge as used.

Challenges MUST be unpredictable (e.g. at least 16 random bytes) and SHOULD expire shortly after being issued. A server MUST invalidate a challenge after its first successful use.


## HTTP Error Codes

Servers responding to requests authenticated with a NWT SHOULD use the following HTTP status codes:

| Code | Meaning | When to use |
|------|---------|-------------|
| `401 Unauthorized` | Missing or invalid token | The `Authorization` header is absent, the token is malformed, the signature is invalid, the token is expired, a [request claim](#http-request-binding) doesn't match the request, or the challenge is invalid |
| `403 Forbidden` | Insufficient permissions | The token is valid but does not grant access to the requested resource (e.g. wrong audience, missing claim) |


## Security Considerations

NWTs are primarily intended for authentication and/or authorization purposes. Just like JWTs, they are bearer tokens; if a malicious actor is able to get a NWT, it may use it to impersonate the user.

Because of that, it is recommended to:

* Use short expiration times (e.g. 5 minutes)
* Restrict the audience for destructive operations to avoid [replay attacks](https://en.wikipedia.org/wiki/Replay_attack)
* Use the NWT event `id` if replay detection if needed
* When storing a NWT in a cookie, use the `HttpOnly`, `Secure` and `SameSite` attributes and protect against CSRF (see [Session Authentication](EXAMPLES.md#session-authentication-with-cookies))

## Libraries

Here is a list of libraries implementing NWT.

- **Go**: https://github.com/Open-Ranking/nwt 

## Comparisons

Authentication and authorization schemes aren't new in Nostr. Here we compare NWT with other popular schemes, highlighting the superior flexibility and control that NWT gives to the end users.

### NWT vs NIP-98

[NIP-98](https://github.com/nostr-protocol/nips/blob/master/98.md) defines the use of Nostr event kind `27235` to perform http authentication and/or authorization.

Example event:

```json
{
    "id": <32-bytes hex-encoded sha256>,
    "pubkey": <32-bytes hex-encoded public key>,
    "created_at": <unix timestamp in seconds>,
    "kind": 27235,
    "tags": [
        ["u", "https://api.snort.social/api/v1/n5sp/list"],
        ["method", "GET"]
    ],
    "content": "",
    "sig": <64-bytes hex-encoded signature>
}
```

NWT is a superset of NIP-98: a [request-bound](#http-request-binding) NWT uses the same `u`, `method` and `payload` tags, so the event above becomes a NWT by changing its kind to `27519` and adding an `exp` claim. This gives NIP-98 services a direct migration path, after which they SHOULD move to session-scoped NWTs:

```json
{
    "id": <32-bytes hex-encoded sha256>,
    "pubkey": <32-bytes hex-encoded public key>,
    "created_at": <unix timestamp in seconds>,
    "kind": 27519,
    "tags": [
        ["u", "https://api.snort.social/api/v1/n5sp/list"],
        ["method", "GET"],
        ["exp", "1710000060"]
    ],
    "content": "",
    "sig": <64-bytes hex-encoded signature>
}
```

Beyond that, NWT improves on NIP-98 in several ways:

* NIP-98 requires a signing round for every http request, which can become cumbersome in multi-server scenarios (e.g. blossom). A NWT can instead target many servers with `aud`.
* NIP-98 relies on server-side freshness checks (`created_at` within a fixed window) rather than user-defined validity time-windows (`exp`, `nbf`).
* NIP-98 has no protection against replay within its freshness window. NWT supports single-use tokens via [challenge-response](#challenge-response).
* NIP-98 doesn't allow app developers to add custom claims.


### NWT vs Blossom Auth

[Blossom](https://github.com/hzrd149/blossom/blob/master/buds/01.md#authorization-events) defines the use of event kind `24242` to perform authorization of its protocol-specific endpoints.

Example event:

```json
{
    "id": <32-bytes hex-encoded sha256>,
    "pubkey": <32-bytes hex-encoded public key>,
    "created_at": <unix timestamp in seconds>,
    "kind": 24242,
    "tags": [
        ["t", "upload"],
        ["x", "b1674191a88...4f553"],
        ["server", "blossom.example.com"],
        ["expiration", "1708858680"]
    ],
    "content": "",
    "sig": <64-bytes hex-encoded signature>
}
```

Before [BUD-11](https://github.com/hzrd149/blossom/blob/master/buds/11.md), Blossom auth was subject to the following replay attack:

* Alice sends a DELETE to Server 1, with the required authorization event
* Before expiration, Server 1 can impersonate Alice and forward the DELETE to Server 2

This attack could occur in very practical scenarios, such as Alice wanting to switch blossom server providers.
However, this was fixed with the introduction of `server` tags, which map exactly to the `aud` tag in NWT.

The current main drawback of Blossom auth is that it doesn't allow users to specify "not before" timestamps, nor allows app developers to add custom claims.
