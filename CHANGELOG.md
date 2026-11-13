# Changelog

What each version of Zu v0.3 changed from the version before. Each version is marked by a git tag of the same name. Changes not yet released are listed under Unreleased.

## Unreleased

Added:

- [README](README.md) and [Path](specs/path.md): two directions are stated, without a format. Everything is a plugin: the data is an event log, so an application is derived data, one view among others, and different applications can be created freely while the data is more durable. And for the purpose of coordination, the many kinds of data that have weak or coordination properties should be organized as causal logs.

Changed:

- [DIDs](specs/did.md) and [README](README.md): a person now signs in to another city by handing the Zu signing key held in the first city's web origin over to the other city's page, instead of having the first city's page sign for it. The same handover brings the key to another device. A key is handed over only after the person chose to do so in the page that holds it, that page shows who receives the key, the receiving page checks the key against the DID document before using it, and rotation keys are never handed over. Signing keys are stored in a form the page can export. Signing in this way does not change the DID document and needs no rotation key; the person keeps the handles they already have. How cities relate to signing keys is left to a later design. The four open questions asked for the earlier version are replaced by two: how the pages find each other and what they exchange, and how a rotation key is used after it has been saved.
- [DIDs](specs/did.md): when a DID is created, software should require the person to save the rotation key in a password manager before going on.
- [Paths](specs/path.md), [Retrieval](specs/retrieval.md), [DIDs](specs/did.md) and [Map](specs/map.md): an individual's data is looked for at the domains of the handles its DID claims. A person's DID document carries no `service` entry for it, and the handle domains may be used as hints when fetching content. The open questions on where an individual's records are stored and on how software learns the host from a DID are dropped; the one on `service` entries now asks which entries hosts carry and whether people carry any. How software asks a host for records is not specified yet.
- [DIDs](specs/did.md): `did:webvh` is not accepted as a DID method. The open question on it is dropped. A `did:web` document that a `did:webvh` DID publishes in parallel is still accepted as `did:web`.
- [DIDs](specs/did.md): a host may run several cities. Each city has a DID and keys of its own, not shared with any other city on the same host, and held for now by the host. Control of a city's keys by several signers is left to a future version.
- [README](README.md): the Scope section now lists what is to be designed within v0.3 (how data is synchronized, how a request to a host is authenticated) and what is left for later (data that is not public; packing, exporting and moving an individual's data; control of a city's keys by several signers).

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
