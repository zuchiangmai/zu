# Data Model

A specification for how records and messages in Zu are encoded, hashed and linked, using DASL.

This document is adapted, with changes, from the [AT Protocol Data Model specification](https://atproto.com/specs/data-model) (bluesky-social/atproto-website, commit `6daf9ea`), which is licensed under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/). It is a draft; the questions still open are listed under Possible Future Changes.

Records and messages in Zu are stored, transmitted, encoded and authenticated in one consistent way. The data model has a binary representation (CBOR) and a textual one (JSON).

When data needs to be signed, referenced by content hash, or stored compactly, it is encoded in CBOR, an IETF format roughly based on JSON. The normalized subset of CBOR that Zu uses is [DRISL](https://dasl.ing/drisl.html), the successor of [DAG-CBOR](https://ipld.io/specs/codecs/dag-cbor/spec/). All DRISL data is valid CBOR and can be read by any CBOR library. Writing DRISL, or verifying it strictly, needs a library configured for its normalization rules.

A distinct piece of data is a **node**. A node encoded in DRISL is a **block**. A node may contain nested maps and lists. Nodes may refer to each other by URL or URI, as JSON on the web does, or strongly by hash, which is called a **link**. Linked nodes can form larger structures such as [Merkle trees](https://en.wikipedia.org/wiki/Merkle_tree) and [directed acyclic graphs](https://en.wikipedia.org/wiki/Directed_acyclic_graph). A link can also point to arbitrary binary data, a **blob**.

A link does not say where on the network its content can be found; how content is fetched by its CID is specified in [Retrieval](retrieval.md). A link is "self-certifying": whatever is returned can be checked against the hash in the link. Content can therefore be passed on by anyone and still be trusted, even when it comes from an untrusted party.

Links are written as [Content Identifiers](https://dasl.ing/cid.html) (CIDs), which have a binary and a string form. A CID carries a code that says whether it points to a node (DRISL) or to raw bytes. The CIDs allowed in Zu are listed below.

This document covers the data types, blob references, the JSON representation and the CID formats. Record types are named by [NSIDs](nsid.md), records are addressed by [paths](path.md), and content is fetched by CID as described in [Retrieval](retrieval.md). How records are grouped into an individual's data, how data is moved between servers, and how data is packed into archives are left to other documents.

## Relationship With DASL and IPLD

The Zu data model aligns with [DASL](https://dasl.ing/), a set of specifications for hash-linked data. DASL libraries and tooling can be used to implement Zu. DASL is itself a subset of [InterPlanetary Linked Data (IPLD)](https://ipld.io/docs/data-model/).

IPLD specifies a normalized JSON encoding called **DAG-JSON**, but Zu uses different conventions when writing JSON (see below). The JSON encoding is not byte-deterministic; the DRISL representation is the one used whenever data is signed or hashed.

The `did:zu` operation log ([`did-zu.md`](did-zu.md)) encodes its operations in DAG-CBOR and writes the `prev` link as a CID string of the same format as below. Its own rules apply to it unchanged.

## Data Types

| DASL Type | JSON            | CBOR                    | Note                    |
| ---       | ---             | ---                     | ---                     |
| null      | Null            | Special Value (major 7) |                         |
| boolean   | Boolean         | Special Value (major 7) |                         |
| integer   | Number          | Integer (majors 0, 1)   | signed, 64-bit          |
| string    | String          | UTF-8 String (major 3)  | Unicode, UTF-8          |
| float     | Number          | Special (major 7)       | not allowed in Zu       |
| bytes     | `$bytes` Object | Byte String (major 2)   |                         |
| link      | `$link` Object  | CID (tag 42)            | CID                     |
| list      | Array           | Array (major 4)         |                         |
| map       | Object          | Map (major 5)           | keys are always strings |

A reference to a file, such as an image, is a map of a fixed shape, the `blob` reference described below.

To stay compatible with the default number type of JavaScript, integers should be limited to 53 bits of precision. JSON numbers can have any number of digits, but a Zu integer is limited to 64 bits in any case.

Data can be checked against the data model without knowing what kind of record it is. For example, a JSON object containing a `$bytes` object whose value is a boolean instead of a base64 string is valid JSON, but never valid in the Zu data model.

Field names starting with `$` are reserved for the data model and the protocol, in both JSON and CBOR. Examples are `$bytes` and `$link` (used in JSON) and `$type` (used by blob references). Implementations should ignore `$` fields they do not know, so that the protocol can evolve. Applications should not use or invent `$` fields of their own.

### Nullable and False-y

Setting a map field to `null` is different from leaving the field out. JSON and CBOR both keep this distinction.

A null or missing field is also different from a "false-y" value such as `false`, `0`, an empty list or an empty map.

### Why No Floats?

CBOR and JSON both support floating-point numbers natively, but Zu does not allow them.

The IPLD specification describes [the difficulties](https://ipld.io/docs/data-model/kinds/#float-kind) of floats in content-addressed data. Decoding a float into a machine-native number and encoding it again does not always give the same bytes. This is certainly true for special values and corner cases, and can be true for ordinary values on less common architectures.

Rules for reliable round-trips of floats may be found in the future; for now floats are not allowed.

Where a value cannot be an integer, encode it as a string, or as bytes. Both round-trip safely.

## `blob` Type

References to blobs (arbitrary files) have one fixed shape in Zu, so they can be found and processed without knowing what kind of record holds them.

A blob reference is a map with these fields:

- `$type` (string, required): the fixed value `blob`.
- `ref` (link, required): the CID of the blob, with the codec `raw`. In JSON it is a `$link` object, as usual.
- `mime` (string, required, not empty): the content type of the blob; `application/octet-stream` if not known.
- `size` (integer, required, non-negative): the length of the blob in bytes.

## JSON Representation

Zu uses its own conventions for JSON instead of DAG-JSON, so that links and bytes read naturally in HTTP APIs. The DAG-JSON specification itself says it is meant mainly for debugging and development, and its use of `/` as a field name is confusing.

Normalizations such as key sorting are not required in JSON. Only DRISL is a byte-reproducible representation.

Most types encode in JSON directly; only `link` and `bytes` need special treatment.

### `link`

A link is encoded in JSON as an object with the single key `$link`, whose value is the CID as a string.

For example, a node with one field `src` of type `link`:

```
{
  "src": {
    "$link": "bafyreidfayvfuwqa7qlnopdjiqrxzs6blmoeu4rujcjtnci5beludirz2a"
  }
}
```

This is close to the DAG-JSON encoding, with `$link` as the key name instead of `/`.

### `bytes`

Bytes are encoded in JSON as an object with the single key `$bytes`, whose value is the base64 encoding of the bytes. The base64 scheme is the one in [RFC 4648, section 4](https://datatracker.ietf.org/doc/html/rfc4648#section-4), plain "base64". It is not URL-safe, and `=` padding is optional.

For example, a node with one field `salt` of type `bytes`:

```
{
  "salt": {
    "$bytes": "nFERjvLLiw9qm45JrqH9QTzyC2Lu1Xb4ne6+sBrCzI0"
  }
}
```

DAG-JSON instead uses two nested objects, an outer key `/` and an inner key `bytes`, with the same base64 encoding.

## Link and CID Formats

The [original IPFS CID specification](https://github.com/multiformats/cid) is very flexible: it supports many hash types, a field for the type of the linked content, and several string encodings. To make implementations interoperate, Zu allows only one "blessed" set of CID types, which aligns with the [DASL CID](https://dasl.ing/cid.html) specification:

- CID version: 1 (0x01)
- Codec: DRISL (0x71, also known as `dag-cbor`) for links to DRISL data, and `raw` (0x55) for links to blobs
- Hash type: `sha-256` (0x12) with a size of 256 bits (0x20 bytes)
- Encoding: binary inside DRISL `link` fields, and `base32` (with the `b` prefix) as a string everywhere else

As in DASL, a blob is hashed whole. It is not split into chunks or a Merkle tree.

A CID can appear in a Zu object in two ways:

- As a `link`. In DRISL, the binary CID is prefixed with `0x00` and carried in CBOR tag 42. In JSON it is a `$link` object (see above).
- As a `string` holding the CID's string form, in both DRISL and JSON.

## Usage and Implementation Guidelines

Existing DASL libraries can be used to read and write DRISL and CIDs. When writing new data, implementations should strictly follow the blessed CID format.

## Security and Privacy Considerations

Parsing untrusted CBOR opens the way to resource-consumption attacks. Use a library that guards against huge allocations, deep nesting, invalid references and similar problems. This matters most for libraries written in languages without strong memory safety, such as C and C++; high-level languages often wrap parsers written in such languages.

A link is only self-certifying if the data is actually checked against it. Data fetched by CID should be hashed and compared before it is trusted.

## Possible Future Changes

Floats may be supported in some form.

More hash types are likely to be added to the blessed set of CIDs. [BDASL](https://dasl.ing/bdasl.html) adds BLAKE3 to DASL CIDs for verifying large data while it streams.

Questions still open in this draft:

- Whether a record also states its type inside its own data, besides its path.
- Which documents use DASL [CAR](https://dasl.ing/car.html) archives to move or export data in bulk.
