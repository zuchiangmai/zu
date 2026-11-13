# Queries

A specification for asking a host for records: one record by its path, an individual's records of a type, or the records of a type across everyone the host holds records for.

This document is not adapted from an AT Protocol specification. atproto lists the records of one repository and leaves questions across repositories to application servers; Zu asks one question of a host, filtered by fields of the record header, in the manner of a Nostr filter. It is a draft; the questions still open are listed under Possible Future Changes.

A [record](record.md) is found by its [path](path.md) when the path is known. Often it is not: software wants every note of a person, or every event written in a city. For that it sends a **query** to a host, and the host answers from the records it holds.

A query filters only on what the record header states: whose record it is, its type, its key and its time. The header is readable by anyone who holds it, so a host can answer without knowing any type. Questions about what a body says, such as which notes contain a word or which events start this week, are not asked of a host: software fetches the records concerned and answers them itself. An application is a view over records, and it computes its view where the records are.

This document covers the query, its parameters, the answer, and what a host must do to answer. How records reach a host, and how two parties bring their records into agreement, are left to the specification of synchronization. How records that are not public are asked for is not specified yet.

## The Query

A query is an HTTP `GET` request to a host:

```
GET https://<host>/.well-known/zu-recs?<parameters>
```

For an individual's records, the host is one of the domains of the handles the individual's DID claims (see [Paths](path.md)). For the records of a city, it is the city's own domain.

All parameters are optional:

| Parameter | Selects records |
|---|---|
| `by` | whose header `by` is this DID |
| `type` | whose header `type` is this NSID, or matches it when it ends in `.*` (see [NSIDs](nsid.md)) |
| `key` | whose header `key` is this key |
| `since` | whose header `at` is this time or later, in milliseconds |
| `until` | whose header `at` is earlier than this time, in milliseconds |

The parameters together are a **filter**. A record matches the filter when it matches every parameter given. Three filters cover the common cases:

- `by`, `type` and `key`: the record at one path.
- `by` and `type`: an individual's records of one type.
- `type` alone: the records of one type from everyone the host holds records for.

Three more parameters shape the answer:

| Parameter | Meaning |
|---|---|
| `n` | the most records to return in this answer; a host may return fewer |
| `from` | the `next` value of an earlier answer, to continue after it |
| `all` | when present, also return records that have been replaced or removed |

Without `all`, the host returns for each path only the record the path refers to (see [Records](record.md)), and nothing for a path whose record was removed. The filter is applied to that record: a record that was replaced is not returned, even when it matches and the record that replaced it does not. With `all`, it returns every valid record it holds that matches the filter.

## The Answer

A successful answer has status 200 and a body that is a map, encoded in DRISL, with these fields:

- `recs` (list, required): the headers of the matching records, each a map with every field as it was signed.
- `blocks` (map, optional): bodies the host chose to send along, each a map, under the string form of its CID.
- `next` (string, optional): present when more records match than were returned. It is opaque to the software, which sends it back as `from`.

The headers are ordered by `at`, latest first; headers with the same `at` are ordered by the binary form of their CIDs, greatest first. This is the order in which [Records](record.md) chooses between concurrent records.

Software checks every header it receives as described in [Records](record.md), and every body against the `body` CID of its header, whether the body came in `blocks` or was fetched afterwards (see [Retrieval](retrieval.md)). A header or a body that fails its check is discarded. Nothing in an answer is trusted because the host said it.

The answer carries the header `Access-Control-Allow-Origin: *`, so that a page of another origin, such as the site of another city, can read it.

A host answers a query it cannot parse with status 400.

## What a Host Does

A host answers from the records it holds. It answers the same for a given filter however it has arranged those records: indexes are the host's own matter and are not specified here.

A host holds the headers of the records it answers for, and so can tell which records are replaced. It does not need the definition of a type to answer for it (see [Types](type.md)).

A host may limit `n`, and may refuse a query whose filter would have it read more than it is willing to, with status 400. A query that gives neither `by` nor `type` is one a host may refuse.

## Usage and Implementation Guidelines

A filter names a set of records. Listing that set is a query; bringing two parties' copies of it into agreement is synchronization, which will name its sets in the same way.

A host that sends `blocks` saves software one request for each record. Software that receives none fetches the bodies by CID.

Software that shows the records of a type computes what it shows from the records, on the device. It does not expect the host to sort or search by anything in a body.

Test vectors for this specification will be published in [`vectors/`](../vectors/).

## Security and Privacy Considerations

A host can leave records out of an answer, and nothing in the answer shows it. What it does return can be checked; that it returned everything cannot. Software that needs to know asks more than one host.

A host sees every query: which DIDs, which types and which times someone asked about, and from which network address. A query for one person's records tells the host that someone is interested in that person.

With `all`, a host returns records that were replaced or removed. Removing a record does not take it out of the hands of those who already hold it (see [Records](record.md)).

A query across everyone a host holds records for lists the people who wrote records of a type. For some types that list is itself sensitive, and a host may decline to answer such a query.

## Possible Future Changes

Answers may also be offered in the JSON form of the data model.

Queries for records that are not public will need the asker to prove who is asking (see [Request Authentication](auth.md)).

Questions still open in this draft:

- Whether a query can ask for the records that list a given record in `prev`, so that replies to a record can be found.
- Whether the sets that synchronization reconciles are exactly these filters.
