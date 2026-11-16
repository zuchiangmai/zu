# Cities

A specification for a city: how its DID document is marked, who its residents are, the roles that run it, how people take and leave them, and the chain in which a host orders these records.

This document is not adapted from an AT Protocol specification. The roles and their records follow the relay-based groups of Nostr (NIP-29), the identity documents of Radicle and the power levels of Matrix; the chain follows the operation log that the `did:plc` directory keeps. It is a draft; the questions still open are listed under Possible Future Changes.

A city is an individual with a DID of its own (see [DIDs](did.md)). It is run by its **operator**: whoever holds the city's keys, which for now is the host that runs it. People join a city as **residents**. Some residents also hold a **role**: coordinator, curator or inviter.

Who is a resident, and who holds which role, is not kept in a list of the host's own. It is stated in signed [records](record.md) written to the city, and anyone who holds those records works it out the same way. To keep that answer the same everywhere, the host that answers for the city orders these records in one chain, and accepts a record into it only when the record follows the latest one and its time agrees with the host's clock.

This document covers how cities and hosts are marked, residents and invitations, roles, the records and the rules that give them effect, the chain, and how software works out the state of a city. How a person lets another key act for them is specified in [Capabilities](cap.md). What each role may do in a city's applications beyond what is said here is left to those applications.

## Marking a City and a Host

The DID document of a city carries a `service` entry with the id `#pol` and the type `city`. Its `serviceEndpoint` is the origin of the city's own domain, where the city's host answers requests for the city, such as [queries](query.md):

```json
{
  "id": "#pol",
  "type": "city",
  "serviceEndpoint": "https://zuchiangmai.com"
}
```

For `did:zu`, the entry is written in the `services` of the city's operations (see the [`did:zu` method specification](did-zu.md)), and is signed with the first operation when the city is made:

```
services: { pol: { type: "city", endpoint: "https://zuchiangmai.com" } }
```

The DID document of a host carries the same entry with the type `host`, and as its `serviceEndpoint` the origin of one of the domains the host claims as handles (see [DIDs](did.md)).

Software treats a DID as a city only when its document carries this entry with the type `city`. A person's DID document carries no `#pol` entry.

## Residents

A person is a resident of a city once both of these are on the city's chain:

- a **join**: a record of the person, written to the city, that says the person joins;
- an **admission**: either a record of the same content, signed by a coordinator in office or by the city itself, that lists the join in `prev`; or an invitation that the join lists in `prev` and carries the secret of (see Invitations).

The person is a resident from the later of the two. A resident leaves with a record of their own that says they leave. Leaving takes effect at once and ends every role the person holds, except that the last coordinator in office cannot leave (see Rules).

## Invitations

An **invitation** is a record of type `city/invite`, written to the city by an inviter or a coordinator. Its body has one field:

- `hash` (bytes, required): the SHA-256 hash of a secret of at least 16 random bytes.

The secret is given to the person invited, for example inside a link. How it travels is up to the software. The join of the invited person lists the invitation in `prev` and carries the secret in its `code` field.

An invitation admits one person: the first join on the chain that carries its secret. It counts only if it is current at its path when the join goes on the chain (see [Records](record.md)), and its signer is an inviter or a coordinator in office at that place in the chain. The signer withdraws it by removing it. Invitations are not on the chain.

## Roles

There are three roles:

| Kind | Who appoints and dismisses | What it is for |
|---|---|---|
| `coordinator` | the city at once, or the coordinators by vote | admitting residents, appointing curators and inviters, opening invitations, and what the city's applications give coordinators |
| `curator` | the city, or any one coordinator | what the city's applications give curators, such as reviewing events |
| `inviter` | the city, or any one coordinator | opening invitations and withdrawing them |

Only a resident can hold a role. Nobody appoints themselves. Anyone may give up a role they hold, except the last coordinator in office.

The operator is not a role. The operator acts by signing with the city's keys, and such records have the city's DID as `by`.

## The Records

Residents and roles are stated in records of type `city/role`, written to the city: `to` is the city's DID. Their bodies have these fields:

- `who` (string, required): the DID of the person concerned.
- `kind` (string, required): `resident`, `coordinator`, `curator` or `inviter`.
- `ok` (boolean, required): `true` to join or take the role, `false` to leave or give it up.
- `code` (bytes, optional): in a join only, the secret of the invitation it uses.

A join is a `city/role` record whose `by` and `who` are the same person, with `kind` `resident` and `ok` `true`. An admission by a coordinator or the city has the same body, with the person in `who`.

A `city/role` record is written once. It is not updated or removed; a host refuses a record that would update or remove one, with the word `deny`.

Software ignores a `city/role` record with a `kind` it does not know.

## Rules

The records on a city's chain take effect one after another, in the order of the chain (see The Chain). Each record is judged by the state that the records before it have produced: who is a resident, who holds which role, and which proposals are open. A record to which no rule below gives effect stays on the chain and has no effect.

**Records of the city.** A record whose `by` is the city's DID takes effect at once. It may admit a resident, appoint anyone who is a resident to a role, and dismiss anyone from a role.

**Residents.** A join takes effect as described under Residents. A record with `kind` `resident`, `ok` `false` and the signer's own DID in `who` makes the signer leave.

**Curators and inviters.** A record by a coordinator in office that appoints a resident other than the signer, or dismisses someone from the role, takes effect at once. A record in which the holder gives up their own role takes effect at once.

**Coordinators.** A coordinator may give up the role by a record of their own, which takes effect at once unless they are the last coordinator in office. Any other record by a coordinator in office about a coordinator is a vote:

- The first such record about a person and a direction (`who`, with `ok` `true` or `false`) opens a **proposal**, and counts as its signer's vote.
- A later record by another coordinator in office with the same `who`, `kind` and `ok`, while the proposal is open, is a vote for it.
- The number of coordinators, `n`, is the number in office just before the proposal on the chain. The proposal passes with `n / 2 + 1` votes, the division rounded down: 1 of 1, 2 of 2, 2 of 3, 3 of 4, 3 of 5.
- It takes effect with the vote that reaches that number.
- A vote counts if its signer is in office at the vote's place on the chain. A vote cast before the signer left office keeps counting.
- A proposal closes when 14 days have passed since its `at`. A vote whose `at` is later than that does not count.
- A proposal that would dismiss the last coordinator in office has no effect.

## The Chain

A city's **governance records** are its `city/role` records: the joins, admissions, appointments, dismissals and votes written to it. The host that answers for the city keeps them in one chain, in which each record lists the one before it in `prev`.

Besides the checks of [Records](record.md#verifying-a-record), a host accepts a governance record only if both hold:

- its `prev` lists the latest governance record of the city that the host holds, or, when the host holds none, the record is the first;
- its `at` is within 60 seconds of the host's own clock, before or after.

A record that fails either check is answered with the word `stale` (see [Records](record.md#delivering)). The host never accepts two governance records that list the same latest record, so the chain does not fork.

Software that signs a governance record:

1. fetches the city's latest governance record, with the query `to=<city>&type=city/role&n=1` (see [Queries](query.md)), and lists it in `prev`, along with the records the new one acts on, such as the proposal it votes for or the join it admits;
2. sets `at` to the later of its own clock and the host's, as read from the `Date` header of the host's latest answer;
3. when the host answers `stale`, fetches the latest record again and checks whether the record still has effect: the signer still holds what the record needs, the person concerned is still a resident, the proposal is still open, and the signer is not the last coordinator. If so, it signs the same body again and delivers it. If not, it tells the person why the record was not delivered.

A record signed while offline waits on the device until it can be delivered, and then goes through the same steps.

## Working Out the State

To work out who is a resident and who holds which role, software fetches every governance record of the city, with `to=<city>&type=city/role&all` (see [Queries](query.md)), orders them by `prev` into the chain, and applies the Rules from the first record to the last.

Software that holds the same chain reaches the same state. Two governance records that list the same record as the one before them show that the host has erred.

Applications that depend on a role, such as a review that only a curator may make, look up the signer's role in this state.

## Usage and Implementation Guidelines

The time a proposal stays open, 14 days, and the window of 60 seconds are counted with the `at` of records. The host's clock check keeps those times close to the moment the records reach the host.

Software should show records that are waiting to be delivered, and say why a record was not delivered.

A host may refuse a governance record to which no rule gives effect, with the word `deny`.

Test vectors for this specification will be published in [`vectors/`](../vectors/).

## Security and Privacy Considerations

The operator holds the city's keys and can admit, appoint and dismiss anyone at any time. Every such record has the city's DID as `by`, so anyone who works out the state can see which changes the operator made alone and which the coordinators voted for.

The host decides which governance records it accepts, and in which order. It can refuse records or delay them, which is a denial of service, and it can give different parties different chains, which they can detect by comparing them. It cannot change a record or put one in another's place.

The time of a governance record cannot be set more than 60 seconds back. A vote signed before a proposal closes but delivered after it does not count.

Someone who has a person's key can act as the person until the key is removed from the person's DID document. Because of the clock check, the key cannot be used afterwards with a time before its removal. The operator can undo what was done with it.

Governance records are public. Anyone who holds them sees who lives in the city, who holds which role, who proposed what and who voted for it.

The secret of an invitation is carried by the join that uses it, and so is public once the invitation is spent. Whoever holds an invitation's secret before that can use it.

## Possible Future Changes

A city may later define roles of its own, with permissions it chooses. Software already ignores kinds it does not know.

Control of a city's keys by several signers is left to a future version.

Records of applications that depend on a role, such as a curator's review, may later list the city's latest governance record in `prev`, so that a review cannot be signed with a time before the curator was dismissed.

Questions still open in this draft:

- Whether a coordinator may make a resident leave.
- Which of a host's domains its `#pol` entry names, when it claims several.
- Whether an invitation may admit more than one person.
