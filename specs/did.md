# DIDs

A specification for the DID methods Zu accepts, and for what Zu reads from a DID document.

This document is adapted, with changes, from the [AT Protocol DID specification](https://atproto.com/specs/did) (bluesky-social/atproto-website, commit `f68a700`), which is licensed under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/). It is a draft; the questions still open are listed under Possible Future Changes.

Zu uses [Decentralized Identifiers](https://en.wikipedia.org/wiki/Decentralized_identifier) (DIDs) as persistent identifiers for people, for cities, and for hosts, the servers that run cities. The core DID standard was developed by the W3C, and describes a framework for compliant identifier systems ("DID methods"), of which several exist. To ensure broad interoperation, Zu only accepts a small number of DID methods.

An example DID is `did:zu:ewvi7nxzyoun6zhxrhs64oiz`.

## Accepted DID Methods

Currently, Zu accepts two DID methods:

- `did:zu`: the `did:plc` method (v0.3.0) under its own name and its own directory, `https://plc.zuzalu.ink`. See the [`did:zu` method specification](did-zu.md). Its operation formats, signing rules and limits are those of `did:plc`, unchanged.
- `did:web`: a [W3C community draft](https://w3c-ccg.github.io/did-method-web/) based on HTTPS (and DNS). The identifier section is a hostname. This method is accepted to provide an independent alternative to `did:zu`, the same way atproto accepts it beside `did:plc`. See usage details below.

It is possible that other methods will be accepted in the future, but the intention is to keep the accepted set as small as possible. It is certainly not the intention to support all or even a significant fraction of all DID methods.

`did:plc` is not accepted. `did:plc:<id>` and `did:zu:<id>` are different DIDs even when the identifier part is the same, and software must not treat them as equivalent.

`did:webvh` is not accepted. A `did:web` document that a `did:webvh` DID publishes in parallel is accepted as `did:web`, under its `did:web` identifier.

### `did:zu` in Zu

The following limits of `did:plc` apply to `did:zu` unchanged, and are enforced by the directory:

- Rotation keys are `secp256k1` ("k256") or NIST P-256 ("p256") keys.
- `verificationMethods` keys may be any syntactically-valid `did:key`, including `ed25519` keys. This was allowed in `did:plc` v0.2 (2025-06), following a request from the Radicle project.
- At most 10 entries each in `verificationMethods`, `services` and `alsoKnownAs`, and an anti-abuse limit on the size of each operation and on how many operations a DID may submit in a period.

### `did:web` in Zu

The [DID Web method](https://w3c-ccg.github.io/did-method-web/) is inherently tied to the domain name used, and does not provide a mechanism for migration or recovering from loss of control of the domain name.

In the context of Zu, only hostname-level `did:web` DIDs are supported: path-based DIDs are not supported. The same restrictions on top-level domains that apply to handles (see [Map](map.md)) also apply to `did:web` domains. The special `localhost` hostname is allowed, but only in testing and development environments. Port numbers (with separating colon hex-encoded) are only allowed for `localhost`, and only in testing and development.

The directory does not host `did:web` documents. Software fetches them from the domain over HTTPS.

## DID Identifier Syntax

The DID Core specification places syntax restrictions on all DID identifiers, and individual DID methods might have their own more specific syntax. While Zu supports only a small set of DID methods today, it still makes sense to consider the general DID syntax in many parsing and validation contexts.

The general syntax constraints used to validate generic DID identifiers in Zu are:

- The entire URI is made up of a subset of ASCII, containing letters (`A-Z`, `a-z`), digits (`0-9`), period, underscore, colon, percent sign, or hyphen (`._:%-`)
- The URI is case-sensitive
- The URI starts with lowercase `did:`
- The method segment is one or more lowercase letters (`a-z`), followed by `:`
- The remainder of the URI (the identifier) may contain any of the above-allowed ASCII characters, except for percent-sign (`%`)
- The URI (and thus the remaining identifier) may not end in `:`.
- Percent-sign (`%`) is used for "percent encoding" in the identifier section, and must always be followed by two hex characters
- Query (`?`) and fragment (`#`) sections are allowed in DID URIs, but not in DID identifiers. In the context of Zu, the query and fragment parts are not allowed.

DID identifiers do not generally have a maximum length restriction, but in the context of Zu, there is a current hard limit of 2048 characters.

In the context of Zu, implementations do not need to validate percent encoding. The percent symbol is allowed in DID identifier segments, but the identifier should not end in a percent symbol. A DID containing invalid percent encoding *should* fail any attempt at registration, resolution, etc.

A reasonable starting-point regex for DIDs in the context of Zu is:

```
// NOTE: does not constrain overall length
/^did:[a-z]+:[a-zA-Z0-9._:%-]*[a-zA-Z0-9._-]$/
```

### Examples

Valid DIDs for use in Zu (correct syntax, and accepted method):

```
did:zu:ewvi7nxzyoun6zhxrhs64oiz
did:web:user.example.com
```

Valid DID syntax, but method not accepted in Zu:

```
did:plc:ewvi7nxzyoun6zhxrhs64oiz
did:method:val:two
did:m:v
did:method::::val
did:method:-:_:.
did:key:zQ3shZc2QzApp2oymGvQbzP8eKheVshBHbU4ZYjeXqwSKEn6N
```

Invalid DID identifier syntax (regardless of DID method):

```
did:METHOD:val
did:m123:val
DID:method:val
did:method:
did:method:val/two
did:method:val?two
did:method:val#two
```

## DID Documents

After a DID document has been resolved, Zu-specific information needs to be extracted. This parsing process is agnostic to the DID method used to resolve the document. For `did:zu`, the directory renders the document from the latest operation, as described in the method specification; rotation keys are not part of the document.

The DID declared in the document (in the `id` field) should always be verified against what was expected (eg, used for resolution).

The **handles** claimed by the DID are found in the `alsoKnownAs` array. Each element of this array is a URI string. Handles have the URI prefix `zu://`, followed by the handle hostname, with no other URI parts or trailing characters. Every syntactically valid `zu://` entry is a claimed handle, and the first one is the primary handle. Other URIs, such as atproto's `at://`, should be ignored.

A claimed handle is only valid once the software using it has confirmed it bidirectionally. See [Map](map.md) for details.

The DID is the primary identifier in Zu, and a DID whose document does not contain a valid and confirmed handle can still participate. Software should be careful to either not display any handle for such a DID, or obviously indicate that any handle associated with it is invalid.

The public **Zu signing keys** for the DID are found under the `verificationMethod` array, in objects whose `id` fragment is exactly `zu`, or `zu-` followed by eight characters from `a-z2-7`, with the `controller` matching the DID itself and `type` matching the fixed string `Multikey`. Each key is an `ed25519` key, and the `publicKeyMultibase` field will be the public key in multibase encoding. Fragments are compared as whole strings, so an `id` such as `#a#zu` is not a Zu signing key.

A DID may have several Zu signing keys, for example one for each device and web origin that holds a private key. Every valid Zu signing key stands for the DID: none takes priority, and anything signed with one of them is signed by the DID. Software should derive the eight characters after `zu-` from the key itself, as the first eight characters of the unpadded lowercase base32 encoding (RFC 4648) of the SHA-256 hash of the raw public key, so that a fragment is never reused for a different key and says nothing about the device or city that holds it.

At least one valid Zu signing key is required for most Zu functionality. A DID with no valid key in its document is likely broken.

A person's DID document carries no `service` entry for where the person's data is. Software looks for it at the domains of the handles the DID claims (see [Map](map.md) and [Paths](path.md)). How the DID document of a city is marked is specified in a separate document.

A host is identified by a `did:zu` of its own, which stays the same when the host changes its domain names. The host claims its domain names as handles in `alsoKnownAs`, the same way a person claims handles, and they are confirmed in the same way (see [Map](map.md)). For example, a host may claim `zu://pol.4seas.example`.

A host may run several cities. Each city has a DID of its own, with keys of its own that it does not share with any other city on the same host. For now the host that runs a city holds the city's keys.

Other features may make use of further `verificationMethod` keys and `service` entries.

When parsing the `id` field in object elements of the `service` and `verificationMethod` arrays, implementations should support both relative fragment syntax (eg, `"id": "#zu"`) and fully-qualified syntax (eg, `"id": "did:web:example.com#zu"`).

## Representation of Public Keys

Public keys in DID documents under `verificationMethod`, including Zu signing keys, are represented as an object with the following fields:

- `id` (string, required): the DID followed by an identifying fragment. Use `#zu`, or `#zu-` followed by eight characters from `a-z2-7`, as the fragment for Zu signing keys
- `type` (string, required): the fixed string `Multikey`
- `controller` (string, required): DID controlling the key, which in the current version of Zu must match the DID itself
- `publicKeyMultibase` (string, required): the public key itself, encoded in "multikey" format

The `publicKeyMultibase` format for `Multikey` is the same encoding scheme as used with `did:key`, but without the `did:key:` prefix: the multicodec for the key type (`ed25519-pub` for Ed25519) followed by the key bytes, encoded with `base58btc` and prefixed with `z`. Ed25519 keys in this encoding begin with `z6Mk`.

For example, two Zu signing keys in a `did:zu` document:

```json
[
  {
    "id": "did:zu:ewvi7nxzyoun6zhxrhs64oiz#zu",
    "type": "Multikey",
    "controller": "did:zu:ewvi7nxzyoun6zhxrhs64oiz",
    "publicKeyMultibase": "z6Mk…"
  },
  {
    "id": "did:zu:ewvi7nxzyoun6zhxrhs64oiz#zu-7xq2k4mf",
    "type": "Multikey",
    "controller": "did:zu:ewvi7nxzyoun6zhxrhs64oiz",
    "publicKeyMultibase": "z6Mk…"
  }
]
```

Zu does not yet have a cryptography specification of its own.

## Signing In to Another City

A person's private Zu signing keys are held in the web origins that use them, such as the site of city B where the person signed up. A page from another origin, such as the site of city A, cannot read them.

A person signs in to city A by handing the Zu signing key held in city B's web origin over to city A's page. Afterwards both origins hold the same key, and city A's page signs with it by itself. No key is added to the DID for city A.

The same handover brings the key to another device: to the same city on another device, or to another city on another device.

Signing in this way does not change the DID document, and needs no rotation key. A person who is already signed in somewhere keeps the handles they already have, and uses them in city A as well.

A handover follows these rules:

- A key is handed over only after the person has chosen to do so in the page that holds it. Being opened by another page, by a link or by a scanned code must never by itself hand over a key.
- Before it hands the key over, the page that holds it shows who receives it: the address of the receiving site and the name of its city.
- The receiving page checks, before it uses the key, that its public key is one of the Zu signing keys in the person's DID document. A key that does not match is discarded.
- On the same device, the receiving page opens the page that holds the key in a window of its own, and the two pages exchange the request and the key directly. Between devices, the receiving page shows a code carrying a public key made for this one handover; the key is sent encrypted to that public key, and a host that relays it sees only ciphertext.
- Only Zu signing keys are handed over. A rotation key is never handed to another origin.

So that a key can be sent to another device, the page that holds a Zu signing key stores it in a form it can export.

How cities relate to signing keys, for example whether each city has a key of its own, is left to a later design. Future work: introduce a decentralized group key mechanism. What is not specified yet is listed under Possible Future Changes.

## Usage and Implementation Guidelines

Protocol implementations should be flexible to processing content containing DIDs based on unsupported DID methods. This is important to allow gradual evolution of the protocol ecosystem over time. In other words, implementations should distinguish between at least the distinct cases "invalid DID syntax", "unsupported DID method" and "supported DID method, but specific DID resolution failed".

While longer DIDs are supported in the protocol, a good best practice is to use relatively short DIDs, and to avoid DIDs longer than 64 characters.

DIDs are case-sensitive. While the currently-supported methods are *not* case sensitive, and could be safely lowercased, protocol implementations should reject DIDs with invalid casing. It is permissible to attempt case normalization when receiving user-controlled input, such as when parsing public URL path components, or text input fields.

When a DID is created, software should require the person to save the rotation key in a password manager before going on.

Test vectors for this specification will be published in [`vectors/`](../vectors/).

## Security and Privacy Considerations

For `did:zu`, the directory is trusted to a limited degree: it can reject valid operations, refuse to serve information, or serve the wrong fork of a DID's history, but it cannot forge operations. See the method specification's Security Considerations.

For `did:web`, whoever controls the domain controls the DID. If the domain is lost, the DID is lost with it and cannot be recovered. The document can change without leaving any record, so software cannot check what the document said at an earlier time.

DID documents are public, and so are the handles and keys they list. The directory can see which DIDs are resolved, and by whom.

The handles a DID claims are also where its data is looked for. A DID document therefore shows which cities a person holds a handle in, and for `did:zu` the audit log shows when each handle was added and removed.

Each Zu signing key can sign for the DID until a rotation key removes it from the document. Every key added or removed is public, and for `did:zu` stays in the audit log, so a DID with one key for each device or web origin reveals how many keys it has and when they changed.

The host that runs a city holds the city's keys, so whoever controls the host can sign for every city it runs.

Every origin that has received a person's Zu signing key holds the same key, in a form it can export. Whoever controls the script served from any of these origins can copy the key and sign for the person anywhere, until a rotation key removes that key from the DID document. A key that has been handed over cannot be taken back: removing it means replacing it in every place that uses it.

## Possible Future Changes

The hard maximum DID length limit may be reduced, at the protocol syntax level. We are not aware of any DID methods that we would consider supporting which have identifiers longer than, say, 256 characters.

There is a chance that the set of accepted DID methods will slowly expand over time.

The limit of 10 `verificationMethods` entries for `did:zu` is used for now and will be raised, so that each device and web origin can hold its own Zu signing key. On this point `did:zu` departs from `did:plc`.

Handing the same Zu signing key to every city a person signs in to is a first version. How cities relate to signing keys is left to a later design. Future work: introduce a decentralized group key mechanism.

A city's keys are held by its host for now. Control of a city's keys by several signers is left to a future version.

Questions still open in this draft:

- Whether signing keys of other types (P-256, `secp256k1`) are also accepted.
- Which `service` entries hosts' DID documents carry, and whether people's carry any. One entry left room for is the [iroh](https://www.iroh.computer) relay a host runs for peer-to-peer connections, which Zu does not specify yet (see the [README](../README.md)).
- How a city's DID document is marked, to be specified in a separate document.
- How a host's DID document is marked.
- How a Zu signing key relates to the handles the DID claims, for example whether a city accepts only some of a DID's signing keys for the handle it confirms.
- How far the limit of 10 `verificationMethods` entries is raised for `did:zu`, and whether the limit on the size of each operation is raised with it.
- How the receiving page finds the page that holds the person's key, and the formats of the request and of the key that is handed over, on the same device and between devices.
- How a rotation key is used after it has been saved, for example to replace a Zu signing key that was lost, and whether any origin keeps it.
