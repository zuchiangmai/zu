# Zu

Zu is the protocol behind polis, written down as specifications that other software can implement without reading the polis code.

**Version:** this is Zu v0.3.1. The earlier designs were v0.1 and v0.2. What each version of v0.3 changed is listed in [CHANGELOG.md](CHANGELOG.md).

## Scope

v0.3 specifies the relationship between one city and the data of its people: their DIDs and handles, the directory and its map, and how records are named, addressed and fetched.

Being designed now, not specified yet:

- Data exchanged between cities.

Not specified yet, to be designed later:

- Peer-to-peer connections between people of different cities. The specifications leave room for a host to run an [iroh](https://www.iroh.computer) relay for such connections: a DID document may later carry a service entry for it (see [DIDs](specs/did.md)), and content may later be fetched over such connections besides HTTPS (see [Retrieval](specs/retrieval.md)).

## Modules

| Module | Covers | Where |
|---|---|---|
| did | The DID methods Zu accepts, `did:zu` and `did:web`, and what Zu reads from a DID document | [`specs/did.md`](specs/did.md) (draft). The `did:zu` method itself: [`specs/did-zu.md`](specs/did-zu.md), the authoritative copy; reference implementation [4seas-community/did-method-zu](https://github.com/4seas-community/did-method-zu); directory `https://plc.zuzalu.ink` |
| map | Handles such as `alice.4seas.example`, checked by the directory and looked up in its map | [`specs/map.md`](specs/map.md) (draft) |
| data model | How records and messages are encoded, hashed and linked: DRISL, DASL CIDs, blob references | [`specs/data-model.md`](specs/data-model.md) (draft) |
| nsid | Names of record types, a reversed domain followed by a name, such as `com.example.note` | [`specs/nsid.md`](specs/nsid.md) (draft) |
| path | A record's address, individual > type > key, written as a `zu://` URI | [`specs/path.md`](specs/path.md) (draft) |
| retrieval | Fetching content by its CID with DASL RASL, and checking what was fetched | [`specs/retrieval.md`](specs/retrieval.md) (draft) |

The other modules of v0.3 have not been designed yet.

## Layout

| Folder | Holds |
|---|---|
| [`specs/`](specs/) | The specifications |
| [`packages/`](packages/) | The reference implementation |
| [`vectors/`](vectors/) | Test data shared by every implementation |

How a specification is written: [CONVENTIONS.md](CONVENTIONS.md).
