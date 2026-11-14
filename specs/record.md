# Records

A specification for the signed record: its header and body, how it is signed and checked, how records refer to earlier records, and which record a path refers to.

This document is not adapted from an AT Protocol specification. atproto signs a whole repository with one signature, made by the server that hosts it, and proves a single record with a path through a Merkle tree. Zu signs each record by itself, so that a record can be checked wherever it is found and by whoever holds it. The designs this document draws on are named under Relationship to Other Designs. It is a draft; the questions still open are listed under Possible Future Changes.

A **record** is one piece of an individual's data, such as a note or a profile. It has a [path](path.md) of individual, type and key, and it is signed with one of the individual's Zu signing keys (see [DIDs](did.md)).

A record is made of two parts:

- a **header**, a small map that states the record's path, its time, the earlier records it follows and the CID of its body, and that carries the signature;
- a **body**, the content itself, in the shape that the record's type defines.

The header is what is signed. The body is tied to it by its CID. A record can therefore be checked with its header alone, and stays checkable when its body is fetched later, held back or discarded.

This document covers the header, the body, signing, verification, links to earlier records, time, and which record a path refers to. How the body of a type is defined is specified in [Types](type.md), and how software asks a host for records in [Queries](query.md). How records are synchronized between devices and hosts, and how records that are not public are handled, are left to other documents.

## Header

A header is a map in the [Data Model](data-model.md) with these fields, all of them required:

- `$type` (string): the fixed value `record`.
- `v` (integer): the fixed value `1`.
- `by` (string): the DID of the individual the record belongs to (see [DIDs](did.md)).
- `type` (string): the type of the record, an [NSID](nsid.md). What the body of that type holds is stated in its definition (see [Types](type.md)).
- `key` (string): the key of the record (see [Paths](path.md)).
- `at` (integer): when the record was signed, in milliseconds since 1970-01-01T00:00:00Z, as reported by the signer (see Time).
- `prev` (list of links): the CIDs of the earlier records that this record follows (see Earlier Records). The list may be empty. It has no duplicates, and is sorted by the binary form of the CIDs in ascending byte order.
- `body` (link, or null): the CID of the body. `null` means that the record at this path is removed (see Updating and Removing).
- `pub` (string): the public key of the Zu signing key that signed the header, in multikey encoding (see [Cryptography](cryptography.md)).
- `sig` (bytes): the signature (see Signing).

`by`, `type` and `key` together are the record's path. They are inside the signed header, so a record cannot be presented under a path other than the one it was signed for.

Software should ignore header fields it does not know. It must keep them when it stores a header or passes it on, because they are part of the signed bytes.

The **CID of a record** is the CID of its header: the header, including `sig`, encoded in DRISL and hashed as described in the Data Model, with the DRISL codec. It names exactly one signed record. A strong reference to a record is its path together with this CID.

In JSON, a header looks like this:

```json
{
  "$type": "record",
  "v": 1,
  "by": "did:zu:ewvi7nxzyoun6zhxrhs64oiz",
  "type": "com.example.note",
  "key": "self",
  "at": 1794528000000,
  "prev": [
    { "$link": "bafyrei…" }
  ],
  "body": { "$link": "bafyrei…" },
  "pub": "z6Mk…",
  "sig": { "$bytes": "…" }
}
```

## Body

The body of a record is a map in the Data Model, encoded in DRISL as a block of its own. Its fields are those that the record's type defines. A body refers to files with blob references (see [Blobs](blob.md)).

The body does not state the record's path or type. The type is stated once, in the header.

The header links to the body by its CID and does not contain it. Software that receives a body must check it against the `body` CID of the header before using it (see [Retrieval](retrieval.md)).

## Signing

To sign a record:

- build the header without the `sig` field, with `pub` set to the public key of the Zu signing key that will sign
- encode it in DRISL and sign the bytes, as described in [Cryptography](cryptography.md)
- add the signature as the `sig` field

## Verifying a Record

A record is valid only if all of the following hold:

- the header is a DRISL map with every required field, each of the right type; `$type` is `record` and `v` is `1`
- `by` is a syntactically valid DID of a method Zu accepts, `type` is a valid NSID, and `key` is a valid key
- `prev` has no duplicates and is sorted
- the signature in `sig` is valid for the header without `sig`, under the public key in `pub`, by the rules in [Cryptography](cryptography.md)
- `pub` is a Zu signing key of `by` (see Keys and the DID Document)
- `at` is greater than the `at` of every record listed in `prev` (see Time)

Software must not treat a record as the individual's until these checks have passed. A record that passes them can be trusted no matter who supplied it.

Whether the body follows the definition of the record's type is a separate check (see [Types](type.md)). It does not change whether a record is valid, or which record a path refers to.

### Keys and the DID Document

Software resolves the DID document of `by` and compares `pub`, as a string, with the `publicKeyMultibase` of each Zu signing key in it.

A key may have been removed from the DID document after the record was signed, for example when a lost key was replaced. So that an individual's earlier records stay valid:

- For `did:zu`, a record is also valid if `pub` was a Zu signing key of the DID at an earlier point of its operation log, and the record's `at` is not later than the time at which the directory recorded the operation that removed the key.
- For `did:web`, which keeps no history, only the keys in the current document count.

## Earlier Records

`prev` lists records that the signer had seen and that this record follows. It gives records an order that depends on no clock and on no server: if record B lists record A in `prev`, directly or through other records, then B was made after A, and B **follows** A.

- A record that updates or removes the record at a path lists the records it replaces (see Updating and Removing).
- A record may also list records at other paths, including records of other individuals, to state that it was made knowing them. Whether a type uses this, and how, is up to the type.

Records of which neither follows the other are **concurrent**: they were made without knowledge of each other, for example on two devices that were offline.

Software may hold a record whose `prev` lists records it does not have.

## Time

`at` is the signer's own statement. It is filled in by the rules of a hybrid logical clock, with the counter folded into the milliseconds. When software signs a record, `at` is:

- not lower than the current time of its own clock,
- greater than the `at` of every record listed in `prev`, and
- greater than the `at` of every record it has signed before with the same key.

When software receives a record:

- A record whose `at` is not greater than the `at` of a record it lists in `prev` is not valid. Software checks this for the listed records it holds.
- A record whose `at` is more than 5 minutes ahead of the receiving software's own clock is held back. Software stores it and may pass it on, but does not act on it until its own clock has reached that time. It does not reject it.

Time never decides between two records of which one follows the other. It only orders records that are concurrent.

## Updating and Removing

A path may be written more than once.

- To **update** a record, sign a new record with the same `by`, `type` and `key` and the new body. Its `prev` contains the CID of every record of that path that is current, as far as the signer knows (see Which Record a Path Refers To).
- To **remove** a record, do the same with `body` set to `null`.

The earlier records remain valid signed records. A host does not discard the header of a record because the record has been replaced or removed.

When a record is removed, a host discards the body of every record that the removing record follows at that path, and no longer serves it. The files those bodies referred to are discarded with them (see [Blobs](blob.md)). A body or a file that a current record refers to is kept. Because the removing record is kept, a host that is handed one of those bodies again does not keep it.

A host may discard the body of a record that has been replaced but not removed.

## Which Record a Path Refers To

Among the valid records with a given path:

- A record is **replaced** if another valid record with the same path follows it.
- The records that are not replaced are **current**.
- If one record is current, the path refers to it.
- If several records are current, they were written concurrently. The path then refers to the one with the greatest `at`; if these are equal, to the one whose CID is the greatest when the binary CIDs are compared byte by byte. The others remain available: software may show them, and the next record written at the path lists all of them in `prev`.
- If the record the path refers to has `body` set to `null`, the path refers to no record.

Software that holds the same valid records reaches the same answer. Software that holds different records may reach different answers until it has exchanged them.

## Relationship to Other Designs

This section is informative.

- **atproto** signs a commit over a Merkle tree of all the records of a repository, with a key held by the hosting server. A single record carries no signature of its own. Zu keeps atproto's path of individual, type and key, and signs each record instead, because a person's records are written on several devices and held by several hosts.
- **p2panda**, **Willow** and **Subduction** sign a small header or entry that carries the hash of the content, which is stored apart from it. Zu's header and body follow that shape. Like a Willow entry, the header states its own path.
- **Automerge**, **Subduction** and **Radicle** let a change list the hashes of the changes it was made after, across authors. `prev` is that list.
- **Nostr** signs every event by itself, as Zu does, and chooses between two versions by the author's timestamp. Zu looks at `prev` first, and uses time only between concurrent records.
- **Evolu** stamps every change with a hybrid logical clock and holds back changes whose time is too far ahead. The rules for `at` follow it.

## Usage and Implementation Guidelines

Keep `prev` short: list the records being replaced, and whatever else the type calls for.

Do not use `at` for anything that must be right against a dishonest signer. It is suited to ordering concurrent records and to display.

When referring to a record of another individual, give its path together with its CID.

Hosts may set limits on the size of the headers and bodies they accept.

Test vectors for this specification will be published in [`vectors/`](../vectors/).

## Security and Privacy Considerations

Whoever holds a Zu signing key of a DID can sign records for it, at any path. A person's key may be held in several web origins (see [DIDs](did.md)), and a record does not show which of them signed it.

`at` is reported by the signer and may be false. A signer can state a time up to 5 minutes ahead and so win over concurrent records for that long. It can also state a time in the past.

Someone who has obtained a key that was since removed from the DID document can still sign records with an `at` before the removal. Such records cannot follow any record made after the removal, because their `at` would not be greater, so they cannot replace it; at a path that was written after the removal they are concurrent records with a lower `at`, and the path does not refer to them. At a path that was not written since, they are accepted. This specification does not prevent that.

A record is valid wherever it is checked. A record signed in one city counts in every other city, and anyone who has a record can pass it on.

Nothing in a record shows that other records exist. A host that holds back a record of an individual cannot be detected from the records it does serve.

The header is not hidden. Whoever holds a header sees the DID, the type, the key, the time and the signing key, and through `prev` which records the signer had seen, even when the body is held back. Keys are chosen by the signer and may themselves carry information.

Parsing untrusted headers and bodies carries the risks described in the [Data Model](data-model.md).

## Possible Future Changes

Bodies may be encrypted, with the header left readable so that records can still be exchanged.

Hosts, or other cities, may attest to when they first saw a record, which would give a time that does not depend on the signer.

Questions still open in this draft:

- Whether the header also states the size of the body, so that a host can refuse a body before fetching it.
- How records signed with a key that was stolen are told apart from the individual's own.
- Whether a record can state which cities it is meant for, so that a record signed in one city does not count in another.
- Whether records are numbered for each signing key, so that a missing record can be noticed.
- What the limits on the size of a header and on the number of entries in `prev` are.
- Whether a host keeps the body of a record that has been replaced but not removed.
