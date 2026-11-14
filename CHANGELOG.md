# Changelog

What each version of Zu v0.3 changed from the version before. Each version is marked by a git tag of the same name. Changes not yet released are listed under Unreleased.

## Unreleased

Added:

- [Queries](specs/query.md): software asks a host for records with `GET /.well-known/zu-recs`. A filter of `by`, `type`, `key`, `since` and `until` selects records by what their headers state; `n`, `from` and `all` shape the answer. The answer is a DRISL map of record headers, latest first, with bodies the host chose to send along and a value for continuing. A host returns for each path the record the path refers to unless asked for all, with the filter applied to that record, and answers the same however it has indexed its records. Questions about what a body says, such as which events start this week, are answered by the software, not the host.
- [Blobs](specs/blob.md): a record is delivered first, and the files it refers to are then uploaded with `PUT /.well-known/zu-blob/<cid>`. A host accepts a blob only if a current record it holds refers to it, and checks the bytes against the CID; the upload carries no authentication. A host keeps a blob while a current record it holds refers to it, may discard it afterwards, and discards it when the records that referred to it have been removed; there is no request for deleting a blob. A blob counts against the individual whose records refer to it.
- [Types](specs/type.md): a type of record is defined by a record of its own, written by whoever controls the domain in the type's name, and found by resolving that domain as a handle. A definition states `v`, `doc`, `key`, `body` and `defs`; a field description names one of the data model's types, or `ref`, `union` or `any`, and may carry `req`, `fields`, `items`, `format`, `min`, `max`, `enum`, `const`, `accept` and `nullable`. The `max` of a string counts Unicode code points. A time in a body is an integer in milliseconds, as in a record header, and there is no string format for it. A `ref` names a shape in this definition, in another definition, or the body of another type; the value of a `union` states which shape it follows in `$type`, and a shape its definition does not list is kept unchecked. Keys are `self` or `any`. Software stores a record and passes it on without knowing its type. Whether software checks a record against its definition, and what it does with a record that does not follow it, is left to the software; it does not change whether the record is valid or which record a path refers to. A definition may change only in ways that keep records written for the old definition following the new one, and the other way round; the constraints on a field are not changed. The document is adapted from the atproto Lexicon specification, with Lexicon's service endpoints left out and the names in Zu's own style. The types the protocol itself defines are named under the domain authority `ink.zuzalu`, and the one defined here is `ink.zuzalu.type`. The open questions on how the records of a type are defined, and on which kinds of keys a type uses, are dropped.

Changed:

- [Paths](specs/path.md), [Retrieval](specs/retrieval.md), [Data Model](specs/data-model.md) and [Records](specs/record.md) now point to Queries and Blobs.
- [Records](specs/record.md) states that whether a body follows the definition of its type is a separate check, which does not change whether a record is valid or which record a path refers to.
- [Records](specs/record.md): a host does not discard the header of a record because the record has been replaced or removed. When a record is removed, a host discards the bodies of the records that the removing record follows at that path, and the files those bodies referred to, and no longer serves them; a body or a file that a current record refers to is kept. The body of a record that has been replaced but not removed may be discarded, and whether a host keeps it is a new open question. Before, software could keep or discard earlier records as it chose.
- [Data Model](specs/data-model.md) lists the values of a union among the uses of `$type`.

## v0.3.4 (2026-11-13)

Added:

- [Key Handover](specs/handover.md): the messages and steps by which a Zu signing key is handed to another page. The receiving page finds the handover page at `/.well-known/zu-handover` of the handle's domain. On the same device the two pages exchange a request and the key with `postMessage`, and the handover page takes the receiving origin from the browser. Between devices the receiving page shows a code with a room URL and a one-time P-256 public key; the handover page seals the key with ECDH, HKDF-SHA-256 and AES-256-GCM and posts it to the room, which the receiving origin's host relays once, within 5 minutes, without being able to read it. The open question in [DIDs](specs/did.md) on how the pages find each other and what they exchange is dropped.

Changed:

- [Data Model](specs/data-model.md): DASL is stated as the set of specifications Zu follows for hash-linked data, with a table of the parts it uses — CID for links, DRISL for data that is signed or hashed, RASL for fetching content, and CAR for moving data in bulk once that is specified. Where DASL specifies something Zu needs, Zu uses it rather than defining its own. The open question on CAR archives is now only which document specifies bulk movement.

Still open, listed under Possible Future Changes in each specification: what was carried over from v0.3.3 and is not answered above, and, new in this version, whether the code for a handover may be shown by the device that holds the key, whether a person's records are sent along with the key, and which document specifies moving data in bulk.

## v0.3.3 (2026-11-13)

Added:

- [Cryptography](specs/cryptography.md): Zu signing keys are Ed25519 keys and no other type is accepted. Data is signed as a DRISL-encoded map that states its `$type` and `v`, and signatures are verified by one strict set of rules, so that all software agrees: canonical encodings, no small-order points, `S` below the group order, and the equation without the cofactor. Public keys are written as multikeys. The document is adapted from the atproto Cryptography specification.
- [Request Authentication](specs/auth.md): where a host has to know who is asking, software sends a signed statement in the `Authorization` header, with the scheme `Zu`. The statement names the DID, the origin of the host it is for, a time and a nonce; a host accepts it once, within 5 minutes of its own clock, and only under a key in the current DID document. Delivering signed records and fetching public content need no statement.
- [Map](specs/map.md): the `/.well-known/zu-did` response should carry `Access-Control-Allow-Origin: *`, so that a page of another origin can check a handle by itself.
- [Records](specs/record.md): every record is signed by itself. A record is a signed header and a body linked by CID. The header states the record's path, a time reported by the signer, the earlier records it follows (`prev`), and the public key that signed it; the CID of the header is the CID of the record. A record that updates or removes another lists it in `prev`. When a path has been written more than once it refers to the record that no other record of the path follows, and between concurrent records to the one with the greatest time. Time is filled in by the rules of a hybrid logical clock and only orders concurrent records. A record signed with a key since removed from a `did:zu` document stays valid if its time is before the removal.
- [README](README.md) and [Path](specs/path.md): two directions are stated, without a format. Everything is a plugin: the data is an event log, so an application is derived data, one view among others, and different applications can be created freely while the data is more durable. And for the purpose of coordination, the many kinds of data that have weak or coordination properties should be organized as causal logs.
- [README](README.md): a direction is stated for how data is synchronized, without a format. It is RBSR: two parties compare fingerprints of the records they hold in a range, and split a range whose fingerprints differ into smaller ranges. Which form of RBSR, what is reconciled, and the format are not designed yet.

Changed:

- [Paths](specs/path.md), [Data Model](specs/data-model.md), [NSIDs](specs/nsid.md) and [DIDs](specs/did.md) now point to Records and Cryptography. Five open questions are answered by them and dropped: which record a path refers to when a key has been written more than once, how records are signed, whether a record states its type inside its own data (asked in two documents), and whether signing keys of other types are accepted.
- [DIDs](specs/did.md) and [README](README.md): a person now signs in to another city by handing the Zu signing key held in the first city's web origin over to the other city's page, instead of having the first city's page sign for it. The same handover brings the key to another device. A key is handed over only after the person chose to do so in the page that holds it, that page shows who receives the key, the receiving page checks the key against the DID document before using it, and rotation keys are never handed over. Signing keys are stored in a form the page can export. Signing in this way does not change the DID document and needs no rotation key; the person keeps the handles they already have. How cities relate to signing keys is left to a later design. The four open questions asked for the earlier version are replaced by two: how the pages find each other and what they exchange, and how a rotation key is used after it has been saved.
- [DIDs](specs/did.md): when a DID is created, software should require the person to save the rotation key in a password manager before going on.
- [Paths](specs/path.md), [Retrieval](specs/retrieval.md), [DIDs](specs/did.md) and [Map](specs/map.md): an individual's data is looked for at the domains of the handles its DID claims. A person's DID document carries no `service` entry for it, and the handle domains may be used as hints when fetching content. The open questions on where an individual's records are stored and on how software learns the host from a DID are dropped; the one on `service` entries now asks which entries hosts carry and whether people carry any. How software asks a host for records is not specified yet.
- [DIDs](specs/did.md): `did:webvh` is not accepted as a DID method. The open question on it is dropped. A `did:web` document that a `did:webvh` DID publishes in parallel is still accepted as `did:web`.
- [DIDs](specs/did.md): a host may run several cities. Each city has a DID and keys of its own, not shared with any other city on the same host, and held for now by the host. Control of a city's keys by several signers is left to a future version.
- [README](README.md): the Scope section now lists what is to be designed within v0.3 (how data is synchronized) and what is left for later (data that is not public; packing, exporting and moving an individual's data; control of a city's keys by several signers).

Still open, listed under Possible Future Changes in each specification: what was carried over from v0.3.2 and is not answered above, and, new in this version, how the two pages of a key handover find each other and what they exchange, how a rotation key is used after it has been saved, whether the verification rules should be those of ZIP-215, the five questions on records (the size of the body, stolen keys, naming the cities a record is meant for, numbering records, and limits), and whether an authentication statement names its request.

## v0.3.2 (2026-11-09)

Added:

- [DIDs](specs/did.md): for now, a person signs in to city A with the Zu signing key held in the web origin of city B, where the person signed up. City A's page asks city B's page to sign, gets back only the signature, and adds no key of its own to the DID. This is a first version. Future work: introduce a decentralized group key mechanism. How the two pages find each other and exchange messages, what a sign-in signature covers, which requests city B's page signs without asking, and what happens when city B's site is gone are new open questions.

Changed:

- [README](README.md): a Scope section states that v0.3 specifies the relationship between one city and the data of its people, and, as a first version, signing in to another city. Data exchanged between cities is being designed; peer-to-peer connections between people of different cities are left to be designed later.
- [DIDs](specs/did.md) and [Retrieval](specs/retrieval.md): their open questions and possible future changes leave room for an iroh relay run by a host, as a `service` entry and as a way to fetch content besides HTTPS.
- [Map](specs/map.md) and [DIDs](specs/did.md): the directory stores DID documents, with the handles written in them, but no longer checks those handles, and keeps no record of confirmed handles. As in atproto, software resolves each handle through DNS or HTTPS and checks it in both directions itself; caching results and re-resolving them periodically is recommended. Whether hosts look up handles is a new open question; the questions on how long past results are kept, who witnesses the map's records, and the lookup endpoint are dropped.
- [DIDs](specs/did.md): a DID may have several Zu signing keys, for example one for each device and web origin that holds a private key. Every `verificationMethod` whose fragment is `zu`, or `zu-` followed by eight characters derived from the key, is a Zu signing key, and all of them stand for the DID. Previously only the first `#zu` key was used, and a DID used the same signing key in every city. How a signing key relates to the handles a DID claims is a new open question. The limit of 10 `verificationMethods` entries is used for now and will be raised for `did:zu`, departing from `did:plc`; how far is a new open question.

Still open, listed under Possible Future Changes in each specification: everything carried over from v0.3.1, and, new in this version, how a Zu signing key relates to the handles a DID claims, how far the limit of 10 signing keys is raised, whether hosts look up handles, and the four questions on signing in to another city.

## v0.3.1 (2026-11-06)

Added:

- [NSIDs](specs/nsid.md): record types are named by a reversed domain followed by a name, such as `com.example.note`, following atproto's NSIDs. Whoever controls the domain names the types under it.
- [Paths](specs/path.md): every record is addressed by individual > type > key. The individual is a DID, and people, cities and hosts are all individuals. A path is written as a `zu://` URI, such as `zu://did:zu:ewvi7nxzyoun6zhxrhs64oiz/com.example.note/self`. The key syntax follows atproto's record keys.
- [Retrieval](specs/retrieval.md): content is fetched by its CID with DASL RASL, from `https://<host>/.well-known/rasl/<cid>`, and must be checked against the CID before it is used. A host serves this way only content it would give to anyone.

Changed:

- [Data Model](specs/data-model.md): record types are named by NSIDs, so the open question is now only whether a record also states its type inside its own data. Links point to Retrieval for how content is fetched.
- [Map](specs/map.md): the `zu://` handle in `alsoKnownAs` is described as a `zu://` URI with no path.
- [CONVENTIONS](CONVENTIONS.md): the protocol version is written as `major.minor.patch`, marked by a git tag, and its changes are listed in this file.

Still open, carried over from v0.3.0 and listed under Possible Future Changes in each specification: which kinds of keys a type uses, which record a key refers to when it has been written more than once, how an individual's records are grouped and signed, and how software finds from a DID the host that stores the individual's records.

## v0.3.0 (2026-11-04)

The first set of v0.3 specifications, a breaking redesign of v0.2:

- [DIDs](specs/did.md): Zu accepts `did:zu` and `did:web`. The signing key is an Ed25519 key named `#zu`. People, cities and hosts each have a DID; a host keeps its DID when its domain names change.
- [`did:zu`](specs/did-zu.md): the `did:plc` method under its own name, with its directory at `https://plc.zuzalu.ink`.
- [Map](specs/map.md): handles are claimed as `zu://` entries in `alsoKnownAs`, the first being the primary handle, and confirmed by their domain through a `_zu` DNS TXT record or `/.well-known/zu-did`. The directory checks a handle once and records only confirmed handles; each device records for itself which handles no longer work.
- [Data Model](specs/data-model.md): data is encoded in DRISL, linked by DASL CIDs, and refers to files by blob references.
