# Paths

A specification for addressing a record by its individual, its type and its key, and for writing that address as a `zu://` URI.

This document is adapted, with changes, from the AT Protocol [AT URI Scheme](https://atproto.com/specs/at-uri-scheme) and [Record Key](https://atproto.com/specs/record-key) specifications (bluesky-social/atproto-website, commit `e079bd4`), which are licensed under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/). It is a draft; the changes that may still come are listed under Possible Future Changes.

Every record in Zu has a **path** of three segments:

```
individual > type > key
```

- The **individual** is the DID the record belongs to (see [DIDs](did.md)). People, cities and hosts are all individuals. A city's public server keeps its data under its own DID, in the same way as a person.
- The **type** is an [NSID](nsid.md), such as `com.example.note`.
- The **key** names one record among the individual's records of that type.

The communities a record belongs to, and the application that wrote it, are not part of its path.

A path is written as a `zu://` URI, for example:

```
zu://did:zu:ewvi7nxzyoun6zhxrhs64oiz/com.example.note/self
```

This document covers the path, the `zu://` URI syntax and the key syntax. How a record is signed, and which record a path refers to when it has been written more than once, are specified in [Records](record.md). Which keys a type uses is stated in its definition (see [Types](type.md)).

## `zu://` URI Structure

The structure of a `zu://` URI is:

```
"zu://" AUTHORITY [ "/" TYPE [ "/" KEY ] ]
```

The **authority** is required, and should be normalized. It is either a DID or a handle (see [Map](map.md)), and stands for the individual. It is syntactically valid to have `zu://` URIs with DID methods that Zu does not accept, though the URI will not resolve or function properly. The optional **type** must be a normalized [NSID](nsid.md). The optional **key** must be a valid key (see Key Syntax below). A URI with a key must also have a type.

A `zu://` URI with a handle as its authority and no path is the form in which a DID claims a handle in its `alsoKnownAs` (see [Map](map.md)).

A path is not a *strong* reference, in that it is not content-addressed. The record at a path may change or be removed over time, or the DID itself may be deleted or unavailable. For `did:web`, control of the DID may change over time. For URIs with a handle in the authority, the handle-to-DID mapping can also change.

The authority of a `zu://` URI stands for the individual, not for a network location. Software looks for an individual's records at the domains of the handles that the individual's DID claims (see [Map](map.md)): it resolves the DID document, reads the claimed handles, and asks the hosts at those domains. A handle written in a URI is only used to look up the DID; the DID may claim other handles, and so have records at other domains. How software asks a host for records is specified in [Queries](query.md).

## Handle-based URIs

These two URIs refer to the same record, if the handle belongs to the DID:

- `zu://did:zu:ewvi7nxzyoun6zhxrhs64oiz/com.example.note/self`
- `zu://alice.4seas.example/com.example.note/self`

URIs referencing handles are not durable. If the individual changes their handle, any URI using that handle will become invalid, and could point to a record of another individual if the handle is reused.

## Generic URI Compliance

`zu://` URIs *align* with the generic Uniform Resource Identifier (URI) syntax described in [IETF RFC-3986](https://www.rfc-editor.org/rfc/rfc3986), but are *not* fully compliant:

- Authority part, preceded by double slash: supported
- Empty authority part: not supported
- Userinfo: not supported. A lone `@` character preceding a handle is not valid (eg, `zu://@alice.4seas.example` is not valid)
- Host and port separation: not supported. The syntax conflicts with DIDs in the authority part
- Path part: supported, optional
- Query and fragment: not supported
- Relative references: not supported

Un-encoded colon characters in DIDs in the authority part of the URI are disallowed by both RFC-3986 and the WHATWG URL Standard ([https://url.spec.whatwg.org/](https://url.spec.whatwg.org/)). It is possible to differentiate a DID in the authority section from a `host:port` pair without ambiguity: DIDs always have at least two colons, and always begin with `did:`.

## `zu://` URI Syntax

- The overall URI is restricted to a subset of ASCII characters
- For reference below, the set of unreserved characters, as defined in [RFC-3986](https://www.rfc-editor.org/rfc/rfc3986), includes alphanumeric (`A-Za-z0-9`), period, hyphen, underscore, and tilde (`.-_~`)
- Maximum overall length is 8 kilobytes (which may be shortened in the future)
- Hex-encoding of characters is permitted (but in practice not necessary and should be avoided to keep the URI normalized and human-readable)
- The URI scheme is `zu`, and an authority part preceded with double slashes is always required. `zu://` URIs always start with `zu://`.
- The authority must be non-empty, and is either a handle or a DID. Colons and unreserved characters should not be escaped in DIDs, but other reserved characters (including `#`, `/`, `$`, `&`, `@`) must be escaped. None of the DID methods Zu accepts allow these characters in DID identifiers.
- The path is either empty, a type, or a type and a key, separated by a single slash (`/`). Trailing slashes are not allowed, including a trailing slash after the authority with no other path.

## Normalization

Particularly when included in records, strict normalization should be followed to ensure that the representation is reproducible and can be used with simple string equality checks.

- No unnecessary hex-encoding in any part of the URI
- Any hex-encoding hex characters must be upper-case
- URI scheme is lowercase
- Authority as handle: lowercased
- Authority as DID: in normalized form, and no duplicate hex-encoding. For example, if the DID is already hex-encoded, don't re-encode the percent signs.
- No trailing slashes in path part
- No duplicate slashes or "dot" sections in path part (`/./` or `/abc/../` for example)
- Type: domain authority part lowercased
- Key is case-sensitive and not normalized

## Key Syntax

A key must fulfill these constraints:

- restricted to a subset of ASCII characters — the allowed characters are alphanumeric (`A-Za-z0-9`), period, dash, underscore, colon, or tilde (`.-_:~`)
- must have at least 1 and at most 512 characters
- the specific key values `.` and `..` are not allowed
- must be permissible to include in a path component of a URI (following RFC-3986, section 3.3). The above constraints satisfy this condition, by matching the "unreserved" characters allowed in generic URI paths.

Keys are case-sensitive.

The same key may be used under several types of the same individual. The pair of individual and key is not unique; the triple of individual, type and key is.

## Examples

Valid `zu://` URIs:

```
zu://did:zu:ewvi7nxzyoun6zhxrhs64oiz
zu://did:zu:ewvi7nxzyoun6zhxrhs64oiz/com.example.note
zu://did:zu:ewvi7nxzyoun6zhxrhs64oiz/com.example.note/self
zu://did:web:pol.4seas.example/com.example.forum.post/3jui7kd54zh2y
zu://alice.4seas.example/com.example.note/self
```

Invalid `zu://` URIs:

```
zu://alice.4seas.example/                    // trailing slash
zu://user:pass@alice.4seas.example           // userinfo not supported
zu://alice.4seas.example:3000                // not a valid DID or handle
zu://did:zu:ewvi7nxzyoun6zhxrhs64oiz/note/1  // type is not a valid NSID
zu://did:zu:ewvi7nxzyoun6zhxrhs64oiz/com.example.note/self?x=1  // query not supported
```

Valid keys:

```
3jui7kd54zh2y
self
example.com
~1.2-3_
dHJ1ZQ
pre:fix
_
```

Invalid keys:

```
alpha/beta
.
..
#extra
@handle
any space
any+space
number[3]
number(3)
"quote"
dHJ1ZQ==
```

## Usage and Implementation Guidelines

Generic URI and URL parsing libraries can sometimes be used with `zu://` URIs, but not always. A key requirement is the ability to work with the authority part of the URI as a simple string, without being parsed in to userinfo, host, and port sub-parts. Specifically: the Python 3 `urllib` module (from the standard library) works; the Javascript `url-parse` package works; the Golang `net/url` package does not work; and most of the popular Rust URL parsing crates do not work.

When referencing records, especially records of other individuals, best practice is to use a DID in the authority part, not a handle. For display, a handle can be used as a more human-readable alternative. In HTML, it is permissible to *display* the handle version of a URI and *link* (`href`) to the DID version.

When a *strong* reference to a record is required, best practice is to give the CID of the record (see [Records](record.md)) in addition to its path.

Most software processing records should treat keys as simple strings, and should not rely on a particular key format.

As a best practice, try to keep keys to under 80 characters in virtually all situations. While keys are case-sensitive, it is a recommended practice to use all-lower-case keys to avoid confusion and maximize possible re-use in case-insensitive contexts.

Test vectors for this specification will be published in [`vectors/`](../vectors/).

## Security and Privacy Considerations

Keys are user-controlled data, and may be chosen arbitrarily by hostile individuals. Software should not infer anything from a key that the record itself does not support.

A path names an individual and a type. Publishing a path publishes that the individual holds a record of that type.

## Possible Future Changes

The maximum length constraint may change.

Query and fragment parts, and relative references between records of the same individual (for example `../<type>/<key>`), may be supported.

The constraints on key syntax may be relaxed in the future to allow non-ASCII Unicode characters. Keys will always be valid Unicode, never relaxed to allow arbitrary byte-strings. The `%` character is reserved for possible use with URL encoding, but such encoding is not currently supported.

Direction for a later version: everything is a plugin. The data is an event log, so an application is in fact derived data, one view among others. A path names an individual, a type and a key, and no application. Different applications can then be created freely over the same records, while the data is more durable. For the purpose of coordination, the many kinds of data that have weak or coordination properties should be organized as causal logs. [Records](record.md) lets a record list the earlier records it follows, which is a first step in this direction. Which data is organized as causal logs is not specified yet.
