# NSIDs

A specification for Namespaced Identifiers, the names of record types in Zu.

This document is adapted, with changes, from the [AT Protocol NSID specification](https://atproto.com/specs/nsid) (bluesky-social/atproto-website, commit `e079bd4`), which is licensed under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/). It is a draft; the changes that may still come are listed under Possible Future Changes.

Namespaced Identifiers (NSIDs) name the types of records in Zu. The type of a record is the second segment of its [path](path.md). For example, `com.example.note`.

The basic structure and semantics of an NSID are a fully-qualified hostname in reverse domain-name order, followed by an additional name segment. The hostname part is the **domain authority,** and the final segment is the **name**.

This document covers NSID syntax. A record states its type in its signed header (see [Records](record.md)), and how the records of a type are defined and checked is specified in [Types](type.md).

## NSID Syntax

The domain authority part of an NSID must be a valid [handle](map.md) (hostname) with the order of segments reversed. That is followed by a name segment of ASCII letters and digits, by convention a short lowercase word such as `note` or `event`.

For example, `com.example.note` is a valid NSID, where `com.example` is the domain authority, and `note` is the name. `com.example.fooBar` is also syntactically valid, but the name is not written that way in Zu.

The comprehensive list of syntax rules is:

- Overall NSID:
    - must contain only ASCII characters
    - separate the domain authority and the name by an ASCII period character (`.`)
    - must have at least 3 segments
    - can have a maximum total length of 317 characters
- Domain authority:
    - made of segments separated by periods (`.`)
    - at most 253 characters (including periods), and must contain at least two segments
    - each segment must have at least 1 and at most 63 characters (not including any periods)
    - the allowed characters are ASCII letters (`a-z`), digits (`0-9`), and hyphens (`-`)
    - segments can not start or end with a hyphen
    - the first segment (the top-level domain) can not start with a numeric digit
    - the domain authority is not case-sensitive, and should be normalized to lowercase (that is, normalize ASCII `A-Z` to `a-z`)
- Name:
    - must have at least 1 and at most 63 characters
    - the allowed characters are ASCII letters and digits only (`A-Z`, `a-z`, `0-9`)
    - hyphens are not allowed
    - the first character can not be a digit
    - case-sensitive and should not be normalized

A reference regex for NSID is:

```
/^[a-zA-Z]([a-zA-Z0-9-]{0,61}[a-zA-Z0-9])?(\.[a-zA-Z0-9]([a-zA-Z0-9-]{0,61}[a-zA-Z0-9])?)+(\.[a-zA-Z]([a-zA-Z0-9]{0,62})?)$/
```

## NSID Syntax Variations

When referring to a group or pattern of NSIDs, a trailing ASCII star character (`*`) can be used as a "glob" character. For example, `com.example.*` would refer to any NSIDs under the `example.com` domain authority, including nested sub-domains (sub-authorities). A free-standing `*` would match all NSIDs from all authorities. Currently, there may be only a single star character; it must be the last character; and it must be at a segment boundary (no partial matching of segment names). This means the star character must be preceded by a period, or be a bare star matching all NSIDs.

## Examples

Syntactically valid NSIDs:

```
com.example.fooBar
net.users.bob.ping
a-0.b-1.c
a.b.c
com.example.fooBarV2
cn.8.lex.stuff
```

Invalid NSIDs:

```
com.exa💩ple.thing
com.example
com.example.3
```

## Usage and Implementation Guidelines

The name segment is a single short lowercase word. The syntax allows neither hyphens nor underscores there, so where one word will not do, a shorter word is chosen rather than several joined together. atproto writes compound names in camel case, as in `getBlogPost`; Zu does not (see [CONVENTIONS](../CONVENTIONS.md)).

A **strongly-encouraged** best practice is to use authority domains with only ASCII alphabetic characters (that is, no digits or hyphens). This makes it significantly easier to generate client libraries in most programming languages.

The overall NSID is case-sensitive for display, storage, and validation. However, having multiple NSIDs that differ only by casing is not allowed. Namespace authorities are responsible for preventing duplication and confusion. Implementations should not force-lowercase NSIDs.

It is common to use "subdomains" as part of the "domain authority" to organize related NSIDs. For example, the NSID `com.example.forum.post` uses the `forum` segment. Note that this requires control of the full domain `forum.example.com`, in addition to the domain `example.com`.

Test vectors for this specification will be published in [`vectors/`](../vectors/).

## Security Considerations

An NSID records which domain named a type. It does not prove that the current holder of that domain stands behind the type, or anything about the data in a record of that type. If a domain expires or changes hands, the new holder can name types under it.

NSIDs which look similar to a well-known domain present the same impersonation challenges as handles (see [Map](map.md)).

## Possible Future Changes

It is conceivable that NSID syntax would be relaxed to allow Unicode characters in the final segment.

The "glob" syntax variation may be modified to make the distinction between single-level and nested matching more explicit.
