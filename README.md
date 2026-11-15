# Zu

Zu is the protocol behind polis, written down as specifications that other software can implement without reading the polis code.

**Version:** this is Zu v0.3.6. The earlier designs were v0.1 and v0.2. What each version of v0.3 changed is listed in [CHANGELOG.md](CHANGELOG.md).

## Scope

v0.3 specifies the relationship between one city and the data of its people: their DIDs and handles, the directory, how records are signed, named, addressed, asked for and fetched, how files are given to a host and kept, and how two parties find out which records each of them lacks.

It also specifies, as a first version, how a person who signed up in one city signs in to another city, or moves to another device, by handing over the Zu signing key (see [DIDs](specs/did.md#signing-in-to-another-city)). How cities relate to signing keys is left to a later design. Future work: introduce a decentralized group key mechanism.

Being designed now, not specified yet:

- Data exchanged between cities.

Direction set, not specified yet:

- Everything is a plugin. The data is an event log, so an application is in fact derived data, one view among others. Different applications can then be created freely, while the data is more durable (see [Path](specs/path.md#possible-future-changes)).
- For the purpose of coordination, the many kinds of data that have weak or coordination properties should be organized as causal logs. [Records](specs/record.md) lets a record list the earlier records it follows; which data is organized this way is not specified yet.

Not specified yet, to be designed within v0.3:

- What surrounds the comparison in synchronization. [Synchronization](specs/sync.md) states the form: two parties first agree on a set of records, order it by time, compare fingerprints of ranges, and split a range whose fingerprints differ by the number of records in it, until the differences are found (see Meyer, [Range-Based Set Reconciliation](https://arxiv.org/abs/2212.13567), and Willow's [region-based set reconciliation](https://willowprotocol.org/specs/rbsr/index.html)). For now what stands for a record, the order, the fingerprint and the messages are those of [Negentropy](https://github.com/hoytech/negentropy/blob/master/docs/negentropy-protocol-v1.md), a set is written in a request as the filter of a query, and each message of a comparison is carried by a request to `/.well-known/zu-diff`. How a record is delivered to a host, and how software learns of new records as they arrive, are not designed yet.

Not specified yet, to be designed later:

- Data that is not public.
- How the body of a type is defined, and how classes are kept apart when the same name is chosen twice. For now a type is written as a class and a kind (see [Paths](specs/path.md)).
- Packing, exporting and moving an individual's data.
- Control of a city's keys by several signers. For now the host that runs a city holds its keys (see [DIDs](specs/did.md)).
- Peer-to-peer connections between people of different cities. The specifications leave room for a host to run an [iroh](https://www.iroh.computer) relay for such connections: a DID document may later carry a service entry for it (see [DIDs](specs/did.md)), and content may later be fetched over such connections besides HTTPS (see [Retrieval](specs/retrieval.md)).

## Modules

| Module | Covers | Where |
|---|---|---|
| did | The DID methods Zu accepts, `did:zu` and `did:web`; what Zu reads from a DID document, including its Zu signing keys; and how a person signs in to another city | [`specs/did.md`](specs/did.md) (draft). The `did:zu` method itself: [`specs/did-zu.md`](specs/did-zu.md), the authoritative copy; reference implementation [4seas-community/did-method-zu](https://github.com/4seas-community/did-method-zu); directory `https://plc.zuzalu.ink` |
| map | Handles such as `alice.4seas.example`: how a DID claims them, how a domain confirms them, and how software resolves them | [`specs/map.md`](specs/map.md) (draft) |
| data model | How records and messages are encoded, hashed and linked: DRISL, DASL CIDs, blob references | [`specs/data-model.md`](specs/data-model.md) (draft) |
| path | A record's address, individual > type > key, written as a `zu://` URI; how a type is written, a class and a kind, such as `ev/event` | [`specs/path.md`](specs/path.md) (draft) |
| retrieval | Fetching content by its CID with DASL RASL, and checking what was fetched | [`specs/retrieval.md`](specs/retrieval.md) (draft) |
| query | Asking a host for records: one record, an individual's records of a type, or a type across everyone the host holds records for | [`specs/query.md`](specs/query.md) (draft) |
| blob | Files: how a file is given to a host, how long the host keeps it, and whom it counts against | [`specs/blob.md`](specs/blob.md) (draft) |
| cryptography | Zu signing keys: Ed25519, how data is signed, and the strict rules by which signatures are verified | [`specs/cryptography.md`](specs/cryptography.md) (draft) |
| login | How a person logs in to a page with a Zu signing key that another page holds: how the key is handed to the other page, on the same device and between devices | [`specs/login.md`](specs/login.md) (draft) |
| auth | How software proves to a host which DID it acts for: a short-lived signed statement bound to that host | [`specs/auth.md`](specs/auth.md) (draft) |
| record | The signed record: a signed header and a body linked by CID, the city a record is meant for, links to earlier records, time, and which record a path refers to | [`specs/record.md`](specs/record.md) (draft) |
| sync | How two parties find out which records each of them lacks: the set they compare, its order, comparing by fingerprints, and the request that carries a comparison | [`specs/sync.md`](specs/sync.md) (draft) |

The other modules of v0.3 have not been designed yet.

## Layout

| Folder | Holds |
|---|---|
| [`specs/`](specs/) | The specifications |
| [`packages/`](packages/) | The reference implementation |
| [`vectors/`](vectors/) | Test data shared by every implementation |

How a specification is written: [CONVENTIONS.md](CONVENTIONS.md).
