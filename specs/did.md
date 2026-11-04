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

### `did:zu` in Zu

The following limits of `did:plc` apply to `did:zu` unchanged, and are enforced by the directory:

- Rotation keys are `secp256k1` ("k256") or NIST P-256 ("p256") keys.
- `verificationMethods` keys may be any syntactically-valid `did:key`, including `ed25519` keys. This was allowed in `did:plc` v0.2 (2025-06), following a request from the Radicle project.
- At most 10 entries each in `verificationMethods`, `services` and `alsoKnownAs`, and an anti-abuse limit on the size of each operation and on how many operations a DID may submit in a period.

### `did:web` in Zu

The [DID Web method](https://w3c-ccg.github.io/did-method-web/) is inherently tied to the domain name used, and does not provide a mechanism for migration or recovering from loss of control of the domain name.

In the context of Zu, only hostname-level `did:web` DIDs are supported: path-based DIDs are not supported. The same restrictions on top-level domains that apply to handles (see [Map](map.md)) also apply to `did:web` domains. The special `localhost` hostname is allowed, but only in testing and development environments. Port numbers (with separating colon hex-encoded) are only allowed for `localhost`, and only in testing and development.

The directory does not host `did:web` documents. When it needs one, for example to check a handle, it fetches the document from the domain over HTTPS.

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

A claimed handle is only valid once the directory has confirmed it bidirectionally and recorded it in the map. See [Map](map.md) for details.

The DID is the primary identifier in Zu, and a DID whose document does not contain a valid and confirmed handle can still participate. Software should be careful to either not display any handle for such a DID, or obviously indicate that any handle associated with it is invalid.

The public **Zu signing key** for the DID is found under the `verificationMethod` array, in an object with `id` ending `#zu`, the `controller` matching the DID itself, and `type` matching the fixed string `Multikey`. The key is an `ed25519` key. The first valid Zu signing key in the array should be used, and any others ignored. The `publicKeyMultibase` field will be the public key in multibase encoding. A DID uses the same signing key in every city.

A valid signing key is required for most Zu functionality. A DID with no valid key in its document is likely broken.

Zu does not yet define `service` entries for people. How the DID document of a city is marked is specified in a separate document.

A host is identified by a `did:zu` of its own, which stays the same when the host changes its domain names. The host claims its domain names as handles in `alsoKnownAs`, the same way a person claims handles, and they are confirmed through the map. For example, a host may claim `zu://pol.4seas.example`.

Other features may make use of further `verificationMethod` keys and `service` entries.

When parsing the `id` field in object elements of the `service` and `verificationMethod` arrays, implementations should support both relative fragment syntax (eg, `"id": "#zu"`) and fully-qualified syntax (eg, `"id": "did:web:example.com#zu"`).

## Representation of Public Keys

Public keys in DID documents under `verificationMethod`, including Zu signing keys, are represented as an object with the following fields:

- `id` (string, required): the DID followed by an identifying fragment. Use `#zu` as the fragment for Zu signing keys
- `type` (string, required): the fixed string `Multikey`
- `controller` (string, required): DID controlling the key, which in the current version of Zu must match the DID itself
- `publicKeyMultibase` (string, required): the public key itself, encoded in "multikey" format

The `publicKeyMultibase` format for `Multikey` is the same encoding scheme as used with `did:key`, but without the `did:key:` prefix: the multicodec for the key type (`ed25519-pub` for Ed25519) followed by the key bytes, encoded with `base58btc` and prefixed with `z`. Ed25519 keys in this encoding begin with `z6Mk`.

For example, a Zu signing key in a `did:zu` document:

```json
{
  "id": "did:zu:ewvi7nxzyoun6zhxrhs64oiz#zu",
  "type": "Multikey",
  "controller": "did:zu:ewvi7nxzyoun6zhxrhs64oiz",
  "publicKeyMultibase": "z6Mk…"
}
```

Zu does not yet have a cryptography specification of its own.

## Usage and Implementation Guidelines

Protocol implementations should be flexible to processing content containing DIDs based on unsupported DID methods. This is important to allow gradual evolution of the protocol ecosystem over time. In other words, implementations should distinguish between at least the distinct cases "invalid DID syntax", "unsupported DID method" and "supported DID method, but specific DID resolution failed".

While longer DIDs are supported in the protocol, a good best practice is to use relatively short DIDs, and to avoid DIDs longer than 64 characters.

DIDs are case-sensitive. While the currently-supported methods are *not* case sensitive, and could be safely lowercased, protocol implementations should reject DIDs with invalid casing. It is permissible to attempt case normalization when receiving user-controlled input, such as when parsing public URL path components, or text input fields.

Test vectors for this specification will be published in [`vectors/`](../vectors/).

## Security and Privacy Considerations

For `did:zu`, the directory is trusted to a limited degree: it can reject valid operations, refuse to serve information, or serve the wrong fork of a DID's history, but it cannot forge operations. See the method specification's Security Considerations.

For `did:web`, whoever controls the domain controls the DID. If the domain is lost, the DID is lost with it and cannot be recovered. The document can change without leaving any record, so software cannot check what the document said at an earlier time.

DID documents are public, and so are the handles and keys they list. The directory can see which DIDs are resolved, and by whom.

## Possible Future Changes

The hard maximum DID length limit may be reduced, at the protocol syntax level. We are not aware of any DID methods that we would consider supporting which have identifiers longer than, say, 256 characters.

There is a chance that the set of accepted DID methods will slowly expand over time.

Questions still open in this draft:

- Whether signing keys of other types (P-256, `secp256k1`) are also accepted.
- Which `service` entries people's DID documents carry.
- How a city's DID document is marked, to be specified in a separate document.
- How a host's DID document is marked.
- Whether `did:webvh` is accepted as a method of its own. A `did:webvh` DID can already publish a parallel `did:web` document, which this specification accepts as `did:web`.
