# Map

A specification for human-friendly handles of Zu identities, and for the map that links each handle to its DID.

This document is adapted, with changes, from the [AT Protocol Handle specification](https://atproto.com/specs/handle) (bluesky-social/atproto-website, commit `b0090eb`); its section Claiming a Handle is adapted from the [AT Protocol DID specification](https://atproto.com/specs/did) in the same repository. Both are licensed under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/). It is a draft; the questions still open are listed under Possible Future Changes.

DIDs (`did:zu` or `did:web`, see [DIDs](did.md)) are the long-term persistent identifiers for people and for cities in Zu, but they are opaque and unfriendly for human use. **Handles** are mutable and human-friendly names, in the form of a DNS hostname. For example, `alice.4seas.example`. Handles must be bi-directionally linked to a DID: the DID document must claim the handle, and the handle's domain must confirm the DID.

In atproto, every service checks this link for itself. In Zu, the check is made by the **directory**, the public server that also keeps the `did:zu` operation log (currently `https://plc.zuzalu.ink`). The directory records each result in the **map**, and the map is the authority on which DID a handle belongs to. Software looks handles up in the map and does not check domains itself.

Lookups go to the directory because keys stay with the people who own them. No hosting server holds a person's keys or always answers for them, the way an atproto PDS does, so the directory is the place every city can ask.

This document covers handle syntax, how a DID claims a handle, how a domain confirms one, how the directory checks and records them, and how software looks them up. Cities, and how a city's DID is marked, are left to other documents.

## Handle Identifier Syntax

To synthesize other standards, and define "handle" syntax specifically:

- The overall handle must contain only ASCII characters, and can be at most 253 characters long (in practice, handles may be restricted to a slightly shorter length)
- The overall handle is split in to multiple segments (referred to as "labels" in standards documents), separated by ASCII periods (`.`)
- No proceeding or trailing ASCII periods are allowed, and there must be at least two segments. That is, "bare" top-level domains are not allowed as handles, even if valid "hostnames" and "DNS names." "Trailing dot" syntax for DNS names is not allowed for handles.
- Each segment must have at least 1 and at most 63 characters (not including the periods). The allowed characters are ASCII letters (`a-z`), digits (`0-9`), and hyphens (`-`).
- Segments can not start or end with a hyphen
- The last segment (the "top level domain") can not start with a numeric digit
- Handles are not case-sensitive, and should be normalized to lowercase (that is, normalize ASCII `A-Z` to `a-z`)

To be explicit (the above rules already specify this), no whitespace, null bytes, joining characters, or other ASCII control characters are allowed in the handle, including as prefix/suffix.

Modern "hostnames" (and thus handles) allow ASCII digits in most positions, with the exception that the last segment (top-level domain, TLD) cannot start with a digit.

IP addresses are not valid syntax: IPv4 addresses have a final segment starting with a digit, and IPv6 addresses are separated by colons (`:`).

A reference regular expression (regex) for the handle syntax is:

```
/^([a-zA-Z0-9]([a-zA-Z0-9-]{0,61}[a-zA-Z0-9])?\.)+[a-zA-Z]([a-zA-Z0-9-]{0,61}[a-zA-Z0-9])?$/
```

## Additional Non-Syntax Restrictions

"Reserved" top-level domains should not fail syntax validation, but they must immediately fail any attempt at registration, confirmation, lookup, etc. See also: [https://en.wikipedia.org/wiki/Top-level_domain#Reserved_domains](https://en.wikipedia.org/wiki/Top-level_domain#Reserved_domains)

`.local` hostnames (for mDNS on local networks) should not be used in Zu.

The `.onion` TLD is a special case for Tor protocol hidden services. Confirmation of handles via Tor would require ecosystem-wide support, so they are currently disallowed.

To summarize the above, the initial list of disallowed TLDs includes:

- `.alt`
- `.arpa`
- `.example`
- `.internal`
- `.invalid`
- `.local`
- `.localhost`
- `.onion`

The `.test` TLD is intended for examples, testing, and development. It may be used in Zu development, but should fail in real-world environments.

The `.invalid` TLD should only be used for the special `handle.invalid` value (see below). This value is syntactically valid, but should not be accepted as a valid handle in most contexts.

The examples in this document use `.example`, so they are syntactically valid but would fail in a real-world environment.

## Identifier Examples

Syntactically valid handles (which may or may not have existing TLDs):

```
alice.4seas.example
jay.bsky.social
8.cn
name.t--t        // not a real TLD, but syntax ok
XX.LCS.MIT.EDU
a.co
xn--notarealidn.com
xn--fiqa61au8b7zsevnm8ak20mc4a87e.xn--fiqs8s
xn--ls8h.test
example.t        // not a real TLD, but syntax ok
```

Invalid syntax:

```
jo@hn.test
💩.test
john..test
xn--bcher-.tld
john.0
cn.8
www.masełkowski.pl.com
org
name.org.
```

Valid syntax, but must always fail confirmation due to other restrictions:

```
2gzyxa5ihm7nsggfxnu52rck2vv4rvmdlkiu3zzui5du4xyclen53wid.onion
laptop.local
blah.arpa
```

## Claiming a Handle

A DID claims handles in the `alsoKnownAs` array of its DID document. Each element of this array is a URI string. Handles have the URI prefix `zu://`, followed by the handle hostname, with no other URI parts or trailing characters:

```json
"alsoKnownAs": ["zu://alice.4seas.example", "zu://alice.zuzalu.example"]
```

A person may hold handles from several cities. Every syntactically valid `zu://` entry is a claimed handle, and the first one is the primary handle, shown by default. URIs with other prefixes, such as atproto's `at://`, are not handles in Zu and are ignored by this specification.

For `did:zu`, the directory accepts at most 10 `alsoKnownAs` entries per DID, each at most 258 characters: 253 for the hostname and 5 for the prefix. `zu://` has the same length as `at://`, so the per-entry limit is unchanged.

A claim is the DID owner's own statement. For `did:zu` it is signed with one of the DID's rotation keys as part of an operation; for `did:web` it is whatever the document served by the domain says. It does not by itself prove that the handle belongs to the DID.

## Domain Confirmation

The domain of a handle confirms the DID with one of two methods. Both demonstrate a reasonable degree of authority over the domain name at a point in time, and are relatively efficient to look up. A city that issues handles under its own domain confirms them there.

The DNS TXT method is the recommended method for individual handle configuration, but the directory should fully support both methods. The intended use-case for the HTTPS method is cities and other services that issue many handles under one domain and may not have the infrastructure to automate a DNS TXT record for each.

### DNS TXT Method

For this method, a DNS TXT record is registered for the `_zu` sub-domain under the handle hostname. The record value should have the prefix `did=`, followed by the full DID. This method aligns with [RFC-1464](https://www.rfc-editor.org/rfc/rfc1464.html), "Using the Domain Name System To Store Arbitrary String Attributes".

For example, the handle `alice.4seas.example` would have a TXT record on the name `_zu.alice.4seas.example`, and the value would look like `did=did:zu:ewvi7nxzyoun6zhxrhs64oiz`.

Any TXT records with values not starting with `did=` should be ignored. Only a single valid record should exist at any point in time. If multiple valid records with different DIDs are present, confirmation should fail. In this case confirmation can be re-tried after a delay, or using a recursive resolver.

Note that very long handles can not be confirmed using this method if the additional `_zu.` name segment pushes the overall name over the 253 character maximum for DNS queries. The HTTPS method will work for such handles.

DNSSEC is not required.

### HTTPS well-known Method

For this method, a web server at the handle domain implements a special well-known endpoint at the path `/.well-known/zu-did`. A valid HTTP response will have an HTTP success status (2xx), `Content-Type` header set to `text/plain`, and include the DID as the HTTP body with no prefix or wrapper formatting.

For example, the handle `alice.4seas.example` would be confirmed by a GET request to `https://alice.4seas.example/.well-known/zu-did`, and a valid response would look like:

```
HTTP/1.1 200 OK
Content-Length: 31
Content-Type: text/plain
Date: Sat, 31 Oct 2026 00:47:21 GMT

did:zu:ewvi7nxzyoun6zhxrhs64oiz
```

The response `Content-Type` header does not need to be strictly verified.

The web server's response body should not contain any prefix or suffix whitespace, but the directory should strip small amounts of prefix or suffix whitespace from the response body before attempting to parse it as a DID.

Secure HTTPS on the default port (443) is required for all real-world confirmations. Unsecured HTTP can only be used for local development and testing.

HTTP redirects (eg, 301, 302) are allowed, up to a reasonable number of redirect hops.

## Directory Verification

The directory checks a handle in both directions: the DID document claims it (see Claiming a Handle), and the domain confirms the same DID (see Domain Confirmation). Only when both agree is the handle valid for that DID. Handles should not be trusted or considered valid until both directions are confirmed, otherwise anybody could create handle aliases for third-party DIDs.

The directory checks a handle:

- as soon as it accepts a `did:zu` operation that adds the handle to a DID's `alsoKnownAs`;
- the first time the handle is looked up and is not yet in the map; and
- again when its previous result expires, so that every handle in the map is re-checked periodically.

The directory keeps the `did:zu` operation log itself. A `did:web` document is not hosted by the directory, so it is fetched from the domain over HTTPS at each check.

For each check, the directory records the handle, the DID, whether the check passed, and the time of the check. These records form the map.

The directory follows these practices when checking:

- It is ok to attempt both methods in parallel, and to use the first successful result available. If the two methods return conflicting results (aka, different DIDs), the DNS TXT result should be preferred, though it is also acceptable to record the result as ambiguous and check again later.
- DNS TTL values provide a possible cache lifetime, but are probably too aggressive (aka, too short a lifetime) for handles.
- Use of a recursive DNS resolver can help with propagation delays, which are important when a DID adds a handle and waits for confirmation.
- Checks should be made from a relatively trusted network environment and configuration. Running them from multiple regions and environments can help mitigate (though not fully resolve) concerns about traffic manipulation or intentionally segmented responses.

## Handle Lookup

Software that needs the DID for a handle asks the directory and uses the map's answer. The answer gives the DID, whether the handle is currently valid, and the time of the last check. Software should not check the domain itself in place of the map.

Software that starts from a DID reads the claimed handles from its DID document, and shows a handle only if the map currently records it as valid for that DID.

The lookup endpoint and its response format are not yet specified.

## Invalid Handles

If a check finds that a handle no longer confirms the DID that claims it, the handle is marked as invalid in the map. In responses, the special handle value `handle.invalid` can be used to indicate that there is no bi-directionally valid handle for a given DID. This handle can not be used in most situations (search queries, requests, etc).

A DID with an invalid handle is still a valid DID: its operation log, keys and data are unaffected. Software should keep displaying its content (to prevent breakage), either without a handle or with a clear indication that the handle is invalid. The DID owner can claim another handle at any time.

## Usage and Implementation Guidelines

Handles may be prefixed with the "at" symbol (like `@alice.4seas.example`) in user interfaces, but this is not a valid syntax for a handle in records, APIs, and other back-end contexts.

Internationalized Domain Names ("IDN", or "punycode") are not directly relevant to the low-level handle syntax. In their encoded form, IDNs are already valid hostnames, and thus valid handles. Such handles must be stored and transmitted in encoded ASCII form. Handles that "look like" IDNs, but do not parse as valid IDNs, are valid handles, just as they are valid hostnames. Applications may, optionally, parse and display IDN handles as Unicode.

Handles are not case-sensitive, which means they can be safely normalized from user input to lower-case (ASCII) form. Only normalized (lowercase) handles should be stored in records or used in outbound API calls. Applications should not preserve user-provided case information and attempt to display handles in anything other than lower-case.

Very long handles are known to present user interface challenges, but they are allowed, and application developers are expected to support them.

Handles should generally not be truncated to local context. For example, the handle `@alice.4seas.example` should not be displayed as `@alice`, even inside the city at `4seas.example`.

Providers of handle "namespaces", such as a city issuing subdomains of its own domain, may impose any additional limits on handles that they wish. It is recommended to constrain the allowed segment length to something reasonable, and to reserve a common set of segment strings like `www`, `admin`, `mail`, etc. There are multiple public lists of "commonly disallowed usernames" that can be used as a starting point.

From a practical standpoint, handles should be limited to at most 249 characters, fewer than the 253 allowed for DNS names. This is because DNS confirmation works with the prefix `_zu.`, which adds 4 characters, and that overall name needs to be valid.

Handle hostnames are expected to be mainstream DNS domain names, registered through the mainstream DNS name system. Handles with non-standard TLDs, or using non-standard naming systems, will fail to interoperate.

Test vectors for this specification will be published in [`vectors/`](../vectors/).

## Security and Privacy Considerations

The directory is trusted for lookups. It could answer with the wrong DID, or give different answers to different askers. A current answer can be checked against the DID document's claim and against the domain, but nothing in this draft lets anyone check the map's past records after the fact. How the map is witnessed is an open question.

Authority over a domain is authority to confirm handles under it. If a city's domain expires or changes hands, the new holder can confirm handles under it for other DIDs. Periodic checks then mark the earlier handles invalid; the DIDs that claimed them, and their data, are not affected.

Handles which look similar to a well-known domain present security and impersonation challenges. For example, handles like `paypa1.com` or `paypal.cc` being confused for `paypal.com`. Very long handles can result in similar issues when truncated at the start or end (`paypal.com…`).

DID documents, and therefore claimed handles, are public. The directory can see which handles are looked up, and by whom.

## Possible Future Changes

The handle syntax is relatively stable.

It is conceivable that `.onion` handles would be allowed at some point in the future.

Questions still open in this draft:

- How long a result is cached, and how often the directory re-checks.
- Whether a domain that is temporarily unreachable counts as a failed check.
- How invalid handles are marked in the map, and how long past results are kept.
- Whether a domain's confirmation should be signed with the city's `did:zu` key, so that a past result can be checked again after the city is gone, as in DIF's [Well-Known DID Configuration](https://identity.foundation/well-known-did-configuration/resources/did-configuration).
- Who witnesses the map's records, and whether several witnesses are required.
- Whether 10 `alsoKnownAs` entries, and so at most 10 handles per DID, is enough.
- The lookup endpoint and its response format.
