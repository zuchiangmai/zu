# Request Authentication

A specification for how software proves to a host which DID it acts for.

This document is not adapted from an AT Protocol specification. It follows the same idea as atproto's authentication between services and Nostr's HTTP Auth (NIP-98): a short-lived statement, signed with the account's own signing key and bound to the service it is sent to. It is a draft; the questions still open are listed under Possible Future Changes.

Most data in Zu authenticates itself. A [record](record.md) is signed, so a host can accept a valid record from anyone, and needs no proof of who delivered it. Some requests are different: the host has to know who is asking. Examples are asking a city to confirm a handle for a DID, and a limit that a host applies to each individual. With such a request, software sends an **authentication statement**: a small signed map that says which DID is asking, which host is asked, and when.

This document covers the statement, how it is sent with an HTTP request, and how a host verifies it. Which requests a host asks a statement for, and what a host allows each DID to do, are left to other documents.

## Statement

A statement is a map in the [Data Model](data-model.md) with these fields, all of them required:

- `$type` (string): the fixed value `auth`.
- `v` (integer): the fixed value `1`.
- `by` (string): the DID the software acts for (see [DIDs](did.md)).
- `aud` (string): the host the statement is for, as the origin of the URL the request is sent to: scheme, hostname, and the port when it is not the default one. For example `https://pol.4seas.example`.
- `at` (integer): when the statement was signed, in milliseconds since 1970-01-01T00:00:00Z, by the clock of the signing software.
- `nonce` (bytes): at least 16 random bytes, made anew for each statement.
- `pub` (string): the public key of the Zu signing key that signed the statement, in multikey encoding (see [Cryptography](cryptography.md)).
- `sig` (bytes): the signature.

The statement is signed as described in [Cryptography](cryptography.md): the map without `sig` is encoded in DRISL, and the bytes are signed with the Zu signing key whose public key is `pub`.

A statement is used for one request. Software signs a new one for the next request.

In JSON, a statement looks like this:

```json
{
  "$type": "auth",
  "v": 1,
  "by": "did:zu:ewvi7nxzyoun6zhxrhs64oiz",
  "aud": "https://pol.4seas.example",
  "at": 1794528000000,
  "nonce": { "$bytes": "…" },
  "pub": "z6Mk…",
  "sig": { "$bytes": "…" }
}
```

## Sending a Statement

The statement, including `sig`, is encoded in DRISL, and the bytes are encoded in base64url ([RFC 4648, section 5](https://datatracker.ietf.org/doc/html/rfc4648#section-5)) without padding. The result is sent in the `Authorization` header of the HTTP request, with the scheme `Zu`:

```
Authorization: Zu <base64url of the DRISL-encoded statement>
```

Secure HTTPS is required for all real-world requests that carry a statement. Unsecured HTTP can only be used for local development and testing.

## Verifying a Statement

A host accepts a statement only if all of the following hold:

- it is a DRISL map with every required field, each of the right type; `$type` is `auth` and `v` is `1`
- `aud` is an origin under which the host itself received the request
- `at` is within 5 minutes of the host's own clock, before or after
- the host has not seen the same `nonce` with the same `by` before
- the signature in `sig` is valid for the statement without `sig`, under the public key in `pub`, by the rules in [Cryptography](cryptography.md)
- `pub` is a Zu signing key in the current DID document of `by`

A host remembers the nonces it has accepted for at least 10 minutes, the whole span in which a statement's `at` can be accepted.

Unlike a record, a statement is checked only against the keys in the current DID document. A key that has been removed from the document no longer authenticates.

When the statement is missing, or does not pass, the host answers with HTTP status 401.

## Usage and Implementation Guidelines

Software asks for a signature only when a request needs one. Fetching public content by CID (see [Retrieval](retrieval.md)) and delivering signed records carry no statement.

The same statement serves a host or a city that makes a request to another host: `by` is then the DID of the host or city, and the statement is signed with its Zu signing key.

After it has verified a statement, a host may give the software a session of its own, for example a cookie, so that later requests need no new signature. Such sessions are the host's own matter and are not specified here.

Test vectors for this specification will be published in [`vectors/`](../vectors/).

## Security and Privacy Considerations

A statement is bound to one host by `aud`. A host that receives a statement cannot use it to act for the person at another host, because the other host is not named in it.

A statement can be used once, within 10 minutes. Someone who intercepts it cannot use it again at the same host as long as the host keeps its nonces, and HTTPS keeps it from being intercepted on the way.

A statement names the host but not the request. A host that receives a statement together with one request could attach it to a different request of its own making; it gains nothing by that, since it is the host that would act on the request.

A statement tells the host which DID is asking. Software should not send one where the request can be made without.

A person's Zu signing key may be held in several web origins and devices (see [DIDs](did.md)). Each of them can authenticate as the person, and a statement does not show which one signed it.

The clocks of devices are often wrong by minutes. A device whose clock is off by more than 5 minutes cannot authenticate until the clock is corrected.

## Possible Future Changes

Questions still open in this draft:

- Whether a statement also names the request it is sent with, its method and path, so that it cannot be used with a different request to the same host.
