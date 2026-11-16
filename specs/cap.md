# Capabilities

A specification for passes: how a person lets another key, such as an assistant's, call services in a city on the person's behalf.

This document is not adapted from an AT Protocol specification. It follows the designs in which an authorization is a signed record kept where the verifier can find it, and the caller proves only which key it is: the signers that Farcaster registers in its shared log, and the delegation certificates that Keyhive synchronizes. The fields follow UCAN and the permissions of atproto's OAuth. It is a draft; the questions still open are listed under Possible Future Changes.

A **pass** is a record in which a person states that a key may make some calls to services in a city, on the person's behalf. A host does not keep a list of who may act for whom: it reads the pass that the caller names, and checks it.

This document covers the pass, how a key uses it, and how a host checks it. How a call is made is specified in [Services](svc.md), and how software proves which key is asking in [Request Authentication](auth.md). Passes for reading and writing records are not specified in this version.

## A Pass

A pass is a [record](record.md) of type `cap/grant`. Its `by` is the person who gives it, the **giver**, and its `to` is the city in which it counts. Its body has these fields:

- `who` (string, required): the **holder**, the key that may use the pass, written as a `did:key` of an Ed25519 key (see [DIDs](did.md)).
- `can` (list of strings, required): the calls the holder may make, at least one. Each entry is a call written as `<service>/<call>`, or `<service>/*` for every call of a service (see [Services](svc.md#names)).
- `end` (integer, optional): the time after which the pass no longer counts, in milliseconds since 1970-01-01T00:00:00Z. A pass without `end` counts until it is removed.

In JSON, the body of a pass looks like this:

```json
{
  "who": "did:key:z6Mk…",
  "can": ["stt/start", "stt/get"],
  "end": 1797120000000
}
```

The giver changes a pass by writing a new record at its path, and **revokes** it by removing it (see [Records](record.md#updating-and-removing)). Passes are not governance records, and are not on the city's chain (see [Cities](city.md)).

## Using a Pass

The holder makes a call as described in [Services](svc.md#a-call), with an authentication statement whose `by` is the holder's `did:key`, whose `pub` is the holder's key, and whose `cap` is the CID of the pass (see [Request Authentication](auth.md)).

A host lets the call proceed only if, besides the statement being valid, all of the following hold:

- the host holds a record with the CID in `cap`, it is valid, and it is a `cap/grant` record;
- it is the record that its path refers to: it has not been replaced or removed (see [Records](record.md#which-record-a-path-refers-to));
- its `who` is the `by` of the statement;
- its `to` is the city the call is made in;
- its `can` lists the call, by its name or by its service with `*`;
- it has no `end`, or `end` is later than the host's own clock;
- the giver may make the call themselves, by what the city allows its people at the time of the call, for example as a resident or in a role (see [Cities](city.md)).

When the pass does not pass these checks, the host answers with the word `deny` (see [Services](svc.md#a-call)).

The host counts the call against the giver, as if the giver had made it.

When software asks a host which services it has loaded, with a statement that names a pass, the host lists the calls that the pass allows and that its giver may make (see [Services](svc.md#what-a-host-has-loaded)).

## Usage and Implementation Guidelines

Give a pass only the calls the holder needs. Set `end` when the holder is needed for a while only.

Software shows a person the passes they have given, with the query `by=<person>&type=cap/grant` (see [Queries](query.md)), and lets them revoke each.

A holder whose key is lost or exposed is replaced: the giver revokes its pass and gives a new one to a new key.

A `did:key` signs authentication statements only. It is never the `by` of a record, so a holder cannot give passes of its own.

Test vectors for this specification will be published in [`vectors/`](../vectors/).

## Security and Privacy Considerations

A pass is a record, and is public. Anyone who holds it sees that the giver let a key make these calls in this city, and until when.

Whoever holds the holder's private key can make the calls until the pass is revoked or `end` has passed. Revoking takes effect at a host once the host holds the record that removes the pass. Calls already made are not undone.

A pass does not give more than its giver has. When the giver leaves the city, or loses a role that a call needs, the calls that need it fail, without the pass being revoked.

A pass counts only in the city in `to`, and only at the host that answers for that city.

## Possible Future Changes

Passes may later cover reading and writing records, as Meadowcap's capabilities and atproto's `repo` permissions do. A record signed by a holder would then carry the pass, which changes [Records](record.md).

A pass may later limit the input of a call, as UCAN's policies do.

A holder may later hand a pass on to another key, with less than it has.

A pass may later lapse when it has not been used for some time, as Farcaster's signers do.

Questions still open in this draft:

- Where an assistant keeps its key: on the city's host, or on the person's device.
- Which of the calls a pass allows still need the person to agree at the time of the call.
- Whether a pass counts at hosts other than the one that answers for its city.
