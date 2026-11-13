# Cryptography

A specification for the keys and signatures used in Zu: which signature system signs Zu data, how a signature is made, and how it is checked.

This document is adapted, with changes, from the [AT Protocol Cryptography specification](https://atproto.com/specs/cryptography) (bluesky-social/atproto-website, commit `b389b84`), which is licensed under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/). It is a draft; the questions still open are listed under Possible Future Changes.

Zu uses public key cryptography to authenticate data. For simplicity and consistency, one signature system and one signing technique are used for everything that is signed with a Zu signing key. Data encryption is not specified in this version of Zu.

Two kinds of keys appear in Zu:

- **Zu signing keys** sign [records](record.md) and any other data that stands for a DID. They are `ed25519` keys, listed in the DID document (see [DIDs](did.md)). This document specifies them.
- **Rotation keys** sign the operations that change a `did:zu` DID document. They are `secp256k1` or NIST P-256 keys. Their signatures are specified by the [`did:zu` method specification](did-zu.md), unchanged from `did:plc`, and are not covered here.

This document covers the signature system, how data is signed, how signatures are verified, and how public keys are written as strings. How private keys are stored and handed over is covered in [DIDs](did.md).

## Signature System

Zu signing keys use Ed25519, as specified in [RFC 8032](https://www.rfc-editor.org/rfc/rfc8032). The variant is plain Ed25519: the message is signed as it is, with no separate hash step and no context string. The variants Ed25519ph and Ed25519ctx are not used.

Ed25519 *is* included in the WebCrypto API of current browsers. It is also the signature system of several designs Zu draws on, among them Radicle, iroh and p2panda.

No other signature system is accepted for Zu signing keys.

A public key is 32 bytes, and a signature is 64 bytes.

## Signing Data

Everything that is signed with a Zu signing key is signed in the same way:

- the data is a map in the [Data Model](data-model.md) that states what it is, with a `$type` field naming the kind of signed data and a `v` field
- encode the map, without its `sig` field, in a strict CBOR format ([DRISL](https://dasl.ing/drisl.html))
- sign the resulting bytes with Ed25519
- put the 64 signature bytes into the map as the `sig` field, of type `bytes`

The bytes are signed directly. They are not hashed with SHA-256 first, because Ed25519 hashes its input itself.

Plain Ed25519 has no way to say what a signature is for. The `$type` field does that instead: it is part of the signed bytes, so a signature made over one kind of data cannot be presented as a signature over another kind. Software must check `$type` and `v` before it accepts a signature, and must not sign a map that lacks them.

The kinds of signed data defined so far:

| `$type` | `v` | Defined in |
|---|---|---|
| `record` | `1` | [Records](record.md) |

## Verifying Signatures

Libraries disagree on a small number of unusual Ed25519 keys and signatures, which an honest signer never produces: for the same message, key and signature, one library answers "valid" and another "invalid". In Zu, everyone must reach the same answer on whether a piece of signed data counts. Zu therefore fixes one set of rules, the strict one.

A signature is valid only if all of the following hold:

- the public key is 32 bytes and the signature is 64 bytes
- the public key, and the first 32 bytes of the signature (`R`), are canonical encodings of points on the curve, as in RFC 8032 section 5.1.3: an encoding whose `y` value is not below the field prime is rejected
- neither the public key nor `R` is a point of small order, that is, one of the 8 points whose order divides 8
- the last 32 bytes of the signature (`S`), read as an integer, are below the group order `L`, as in RFC 8032 section 5.1.7
- the equation of RFC 8032 section 5.1.7 holds in its form without the cofactor: `[S]B = R + [k]A`

Software must apply all of these rules, whatever library or platform it uses. Where a library applies fewer of them, software adds the missing checks itself. The checks on the encodings of the public key and of `R`, and on small order, can be made on the bytes before the library is called.

Signatures should always be verified with the verification routine of a cryptographic library, never by comparing signature values as raw bytes. Two valid signatures over the same data, made with the same key, are not necessarily the same bytes.

## Public Key Encoding

When encoding public keys as strings, Zu uses the "multikey" encoding used by `did:key`. Because this representation includes a "codec" indicating the key type, it can be parsed unambiguously.

The process for encoding an Ed25519 public key in this format is:

- Take the 32 bytes of the public key
- Prepend the codec value for `ed25519-pub`, code 0xED, as varint-encoded bytes: [0xED, 0x01]
- Encode the combined 34 bytes with `base58btc`, and prefix with a `z` character, yielding a string

The decoding process is the same in reverse. Ed25519 public keys in this encoding begin with `z6Mk`.

To encode a key as a `did:key` identifier, use the above multikey encoding, and add the ASCII prefix `did:key:`. This form is used in the operations of the `did:zu` method.

The multikey string is the value of the `publicKeyMultibase` field of a DID document (see [DIDs](did.md)) and of the `pub` field of a record (see [Records](record.md)). A canonical public key has exactly one such string, so software can compare keys as strings.

### Encoded Example

An Ed25519 public key, encoded as a multikey, and as `did:key`:

```
z6MkhaXgBZDvotDkL5257faiztiGiC2QtKLGpbnnEGta2doK
did:key:z6MkhaXgBZDvotDkL5257faiztiGiC2QtKLGpbnnEGta2doK
```

## Usage and Implementation Guidelines

Ed25519 is available in the WebCrypto API of Chrome from version 137, Firefox from version 129 and Safari from version 17. The Web Cryptography API rejects small-order keys and uses the equation without the cofactor, as this document does, and notes that not every implementation performs the small-order check.

Libraries differ in which of the verification rules they apply. Some offer a stricter routine beside the default one, for example `verify_strict` in `ed25519-dalek`, which Radicle, iroh and p2panda use. Implementers should check each rule against the library they use, with the test vectors.

There is no specific recommended byte or string encoding for private keys. How a private key is stored, and how it is handed to another origin or device, is covered in [DIDs](did.md).

Test vectors for this specification, including keys and signatures that must be rejected, will be published in [`vectors/`](../vectors/).

## Security Considerations

The verification rules are strict so that all software agrees. If two implementations disagreed on one signature, a record could count in one city and not in another, and anyone able to craft such a signature could split what people see.

A signature shows that a holder of the private key signed the data. It does not show when: any time stated in signed data is the signer's own statement.

A signature does not show which holder signed. A person's Zu signing key may be held in several web origins and devices (see [DIDs](did.md)), and all of them sign with the same key.

Software that signs data for other software, for example a page asked to sign by another page, must sign only maps that state their `$type`, and should show the person what kind of data is being signed.

Ed25519 is not resistant to attacks by large quantum computers. Neither are the curves used for rotation keys.

## Possible Future Changes

The set of supported cryptographic systems is expected to evolve slowly. There are significant interoperability and implementation advantages to having as few systems as possible at any point in time.

More kinds of signed data will be added to the table under Signing Data as they are specified, for example the statement with which software proves to a host which DID it acts for.

Data encryption may be specified in a later version.

Questions still open in this draft:

- Whether the verification rules should be those of [ZIP-215](https://zips.z.cash/zip-0215) instead, which accept more encodings, use the equation with the cofactor, and allow many signatures to be verified in one batch.
