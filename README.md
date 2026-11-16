# Zu

Zu is the protocol behind pol, written down as specifications that other software can implement without reading the pol code. This is Zu v0.3.8; the earlier designs were v0.1 and v0.2. What each version of v0.3 changed is listed in [CHANGELOG.md](CHANGELOG.md).

```
What a host answers, under https://<host>/.well-known/

  GET  zu-did                     the DID that a handle's domain confirms
  GET  zu-recs?by=&type=...       records, as signed headers
  POST zu-recs                    records delivered to the host
  PUT  zu-blob/<cid>              a file that a record refers to
  GET  rasl/<cid>                 content, by its CID
  POST zu-diff                    one message of a comparison
  GET  zu-live                    events that tell of new records
  GET  zu-login                   the page that hands over a signing key
  POST zu-relay/<room>            leave a sealed key for another device
  GET  zu-relay/<room>            pick the sealed key up
  GET  zu-svc                     the services the host has loaded
  POST zu-svc/<service>/<call>    a call to one of them
```

## How it fits together

**A person is known by a DID.** Zu accepts `did:zu` and `did:web` ([DIDs](specs/did.md)). A DID document lists the person's handles, such as `alice.4seas.example`, and their Zu signing keys, for example one for each device and web origin that holds a key. A handle holds when the DID document claims it and the handle's domain confirms the DID ([Map](specs/map.md)). Cities, and the hosts that run them, have DIDs too.

**A record is signed by itself.** Each piece of a person's data is a record: a small signed header, and a body tied to it by its CID ([Records](specs/record.md)). The header states whose record it is, its type and key, when it was signed, and the earlier records it follows. Its individual, type and key make its path, such as `zu://alice.4seas.example/ev/event/kx7q2m5v4tbn3hpw` ([Paths](specs/path.md)). Data is encoded in DRISL and linked by DASL CIDs ([Data Model](specs/data-model.md)), and signed with Ed25519 ([Cryptography](specs/cryptography.md)). Whoever holds a record can check it.

**A host keeps records and files.** Software delivers records to a host, then uploads the files they refer to ([Records](specs/record.md#delivering), [Blobs](specs/blob.md)). Others ask the host for records ([Queries](specs/query.md)), or fetch content by its CID ([Retrieval](specs/retrieval.md)). Where a host has to know who is asking, software sends a short-lived signed statement ([Request Authentication](specs/auth.md)).

**Two copies catch up.** Two parties find out which records each of them lacks by comparing fingerprints, as Negentropy does, and software learns of new records from a stream of events ([Synchronization](specs/sync.md)).

**One key, more pages.** A person signs in to another city, or moves to another device, by handing the Zu signing key over to the new page ([Login](specs/login.md)).

**A host serves applications.** Besides keeping records, a host can do work for applications, such as turning speech into text. It describes each service and its calls, and lists the ones it has loaded ([Services](specs/svc.md)).

**A city is run in signed records.** The DID documents of cities and hosts carry a `#pol` entry. People join a city as residents, and coordinators, curators and inviters run it. Joins, appointments and votes are records that the city's host orders in one chain, so that everyone works out the same state ([Cities](specs/city.md)). A person can let another key, such as an assistant's, call services on their behalf with a pass ([Capabilities](specs/cap.md)).

## Modules

The modules are grouped in five lanes: Data, Sync, Net, Auth and App. A module is listed under the lane most of it belongs to; a section of it that belongs to another lane is listed under that lane as well. The lanes run side by side, not one on top of another. Every specification is a draft; the questions still open are listed at its end, under Possible Future Changes.

### Data

What can be stated and checked on one machine: how records are encoded, signed, addressed, linked and kept.

| Module | Covers | Where |
|---|---|---|
| data model | How records and messages are encoded, hashed and linked: DRISL, DASL CIDs, blob references | [`specs/data-model.md`](specs/data-model.md) |
| cryptography | Zu signing keys: Ed25519, how data is signed, and the strict rules by which signatures are verified | [`specs/cryptography.md`](specs/cryptography.md) |
| record | The signed record: a signed header and a body linked by CID, the city a record is meant for, links to earlier records, time, which record a path refers to, and how records are delivered to a host | [`specs/record.md`](specs/record.md) |
| path | A record's address, individual > type > key, written as a `zu://` URI; how a type is written, a class and a kind, such as `ev/event` | [`specs/path.md`](specs/path.md) |
| blob: Keeping and Discarding | How long a host keeps a file: while a record it holds refers to it | [`specs/blob.md`](specs/blob.md#keeping-and-discarding) |

### Sync

What arises only between two copies: which records to compare, finding which ones each lacks, and handing records over.

| Module | Covers | Where |
|---|---|---|
| sync | How two parties find out which records each of them lacks: the set they compare, its order, comparing by fingerprints, the request that carries a comparison, and the events that tell software of new records | [`specs/sync.md`](specs/sync.md) |
| query | Asking a host for records: one record, an individual's records of a type, or a type across everyone the host holds records for | [`specs/query.md`](specs/query.md) |
| record: Delivering | Giving records to a host, `POST /.well-known/zu-recs`, and what the host did with each | [`specs/record.md`](specs/record.md#delivering) |
| blob | Files: how a file is given to a host, how long the host keeps it, and whom it counts against | [`specs/blob.md`](specs/blob.md) |

### Net

How parties reach each other: fetching content, relays, and telling software that something new has arrived.

| Module | Covers | Where |
|---|---|---|
| retrieval | Fetching content by its CID with DASL RASL, and checking what was fetched | [`specs/retrieval.md`](specs/retrieval.md) |
| sync: New Records | The stream of events that tells software of new records, `GET /.well-known/zu-live` | [`specs/sync.md`](specs/sync.md#new-records) |
| login: The Relay | The host that passes a sealed key from one device to another and cannot read it | [`specs/login.md`](specs/login.md#the-relay) |

### Auth

What needs someone to decide: who is who, which name is whose, which software acts for whom, and what counts against whom.

| Module | Covers | Where |
|---|---|---|
| did | The DID methods Zu accepts, `did:zu` and `did:web`; what Zu reads from a DID document, including its Zu signing keys; and how a person signs in to another city | [`specs/did.md`](specs/did.md); the `did:zu` method itself: [`specs/did-zu.md`](specs/did-zu.md) |
| map | Handles such as `alice.4seas.example`: how a DID claims them, how a domain confirms them, and how software resolves them | [`specs/map.md`](specs/map.md) |
| login | How a person logs in to a page with a Zu signing key that another page holds: how the key is handed to the other page, on the same device and between devices | [`specs/login.md`](specs/login.md) |
| auth | How software proves to a host which DID it acts for: a short-lived signed statement bound to that host | [`specs/auth.md`](specs/auth.md) |
| city | A city: how its DID document and its host's are marked, residents and invitations, the roles of coordinator, curator and inviter, appointments and votes, and the chain in which the host orders them | [`specs/city.md`](specs/city.md) |
| cap | Passes: how a person lets another key, such as an assistant's, call services in a city on the person's behalf, and how a host checks the pass | [`specs/cap.md`](specs/cap.md) |
| blob: Limits | The limits a host may set, and whom a file counts against | [`specs/blob.md`](specs/blob.md#limits) |

The `did:zu` method specification in this repository is the authoritative copy. Its reference implementation is [4seas-community/did-method-zu](https://github.com/4seas-community/did-method-zu), and its directory runs at `https://plc.zuzalu.ink`.

### App

What different applications must agree on to read the same data.

| Module | Covers | Where |
|---|---|---|
| path: Type Syntax | How a type is written, a class and a kind, such as `ev/event` | [`specs/path.md`](specs/path.md#type-syntax) |
| svc | What a host can do for applications besides keeping records: how a service and its calls are described, how a host lists the services it has loaded, and how software calls one | [`specs/svc.md`](specs/svc.md) |

The other modules of v0.3 have not been designed yet.

## Status

**Scope.** v0.3 specifies the relationship between one city and the data of its people. As a first version, it also specifies how a person who signed up in one city signs in to another city, or moves to another device, by handing over the Zu signing key (see [DIDs](specs/did.md#signing-in-to-another-city)). How cities relate to signing keys is left to a later design. Future work: introduce a decentralized group key mechanism.

**Being designed now:**

- Data exchanged between cities.

**Direction set, not specified yet:**

- Permissions are capabilities. A capability is a signed pass that states what its holder may do; the holder can hand it on, only with less. A host checks the pass rather than a list of who may do what, as in [Meadowcap](https://willowprotocol.org/specs/meadowcap/), [UCAN](https://github.com/ucan-wg/spec) and [Keyhive](https://github.com/inkandswitch/keyhive). In Zu the word capability is kept for this. [Capabilities](specs/cap.md) specifies the first use: a person lets a key call services in a city. Passes for reading and writing records, and passes handed on, are not specified yet.
- Everything is a plugin. The data is an event log, so an application is in fact derived data, one view among others. Different applications can then be created freely, while the data is more durable (see [Paths](specs/path.md#possible-future-changes)).
- For the purpose of coordination, the many kinds of data that have weak or coordination properties should be organized as causal logs. [Records](specs/record.md) lets a record list the earlier records it follows; which data is organized this way is not specified yet.

**To be designed later:**

- Data that is not public.
- A fingerprint for synchronization that a writer of records cannot defeat. For now [Synchronization](specs/sync.md) compares records as [Negentropy](https://github.com/hoytech/negentropy/blob/master/docs/negentropy-protocol-v1.md) does.
- How the body of a type is defined, and how classes are kept apart when the same name is chosen twice. For now a type is written as a class and a kind (see [Paths](specs/path.md)).
- Packing, exporting and moving an individual's data.
- Control of a city's keys by several signers. For now the host that runs a city holds its keys (see [DIDs](specs/did.md)).
- Peer-to-peer connections between people of different cities. The specifications leave room for a host to run an [iroh](https://www.iroh.computer) relay for such connections: a DID document may later carry a service entry for it (see [DIDs](specs/did.md)), and content may later be fetched over such connections besides HTTPS (see [Retrieval](specs/retrieval.md)).
- App: every version of an application is scanned automatically, and the page of an application shows a security scorecard.
- App, to be introduced later: reproducible builds, by which anyone can check that an application was built from its source.

## Layout

| Folder | Holds |
|---|---|
| [`specs/`](specs/) | The specifications |
| [`packages/`](packages/) | The reference implementation, not started yet |
| [`vectors/`](vectors/) | Test data shared by every implementation, not started yet |

How a specification is written: [CONVENTIONS.md](CONVENTIONS.md).
