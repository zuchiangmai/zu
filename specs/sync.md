# Synchronization

A specification for how two parties find out which records each of them lacks: which records they compare, in which order, and how they narrow down the differences.

This document is not adapted from an AT Protocol specification. Its form follows Willow's [region-based set reconciliation](https://willowprotocol.org/specs/rbsr/index.html), which applies [range-based set reconciliation](https://arxiv.org/abs/2212.13567) to sets that are chosen first. For now, what stands for a record in a comparison, the order of the records, the fingerprint and the messages are those of [Negentropy](https://github.com/hoytech/negentropy/blob/master/docs/negentropy-protocol-v1.md), protocol version 1. It is a draft; the questions still open are listed under Possible Future Changes.

This document covers the set of records two parties compare, what stands for a record, the order of the records in the set, comparing by fingerprints, the request that carries a comparison, and the stream of events that tells software of new records. What a record is, and which records a host keeps, is specified in [Records](record.md). How content is fetched once a party knows what it lacks is specified in [Retrieval](retrieval.md), and how files follow records in [Blobs](blob.md). How records are delivered to a host is specified in [Records](record.md#delivering).

## The Set

Two parties, such as a device and a host, or two hosts, first agree on the set of records they compare. A set is named by what record headers state, in the way a [query](query.md) names it:

- by `by`: the records of one individual, or of every individual the party holds records for;
- by `to`: the records meant for one city;
- by `type`: the records of one type, or of every type of one class.

The party that starts a comparison writes the set in its request as the filter of a query: the parameters `by`, `to` and `type`, with the meaning they have in [Queries](query.md). The other party compares the records it holds that match the same filter. It may decline to compare a set, as a host may decline a query.

Each party takes the records it holds that match. That includes records that have been replaced or removed: every header is kept (see [Records](record.md)), and a removal is a record, so it reaches the other party like any other. It also includes a record that is held back because its `at` is ahead of the party's clock: such a record is stored, and so it is held.

A party need not compare everything it holds. A device compares the sets it has a use for.

## The Items

In a comparison a record is a Negentropy item:

- its timestamp is the record's `at`, in milliseconds;
- its ID is the SHA-256 hash that the CID of the record carries, 32 bytes (see [Records](record.md) and the [Data Model](data-model.md)).

## The Order

Within a set, records are ordered as Negentropy orders items:

1. by `at`, the earlier first;
2. records with the same `at`, by their IDs, compared byte by byte.

## Comparing

A **range** is a run of consecutive records in this order. A **fingerprint** is a short value computed from the records of a range. Two parties that hold the same records in a range compute the same fingerprint.

- One party sends the fingerprint of a range.
- The other party computes the fingerprint of the records it holds in the same range. If the two are equal, the parties hold the same records there, and the range is done.
- If they differ, the range is split into smaller ranges, and each of them is compared in the same way.
- When a range holds few records, a party lists the records it holds in it instead of sending a fingerprint. The other party then sees which records each of them lacks.

A party splits a range by the number of records it holds in it, into ranges that hold roughly the same number each. It does not split by the length of time a range spans. Into how many ranges it splits, and below how many records it lists them, is each party's own choice; the two parties need not choose the same numbers.

The fingerprint and the messages are those of Negentropy protocol version 1. The fingerprint of a range is computed from the IDs of the records in it and from their number. A message is a list of ranges, each of them skipped, given as a fingerprint, or given as a list of IDs.

Comparing ends when no range is left whose fingerprints differ.

## The Request

A comparison is carried by HTTP requests. The party that starts it sends each of its messages in a request of its own, to a host of the other party, which is found as for a [query](query.md):

```
POST https://<host>/.well-known/zu-diff?<parameters>
Content-Type: application/octet-stream

<one Negentropy message>
```

The parameters are the filter that names the set: `by`, `to` and `type`, written as in a query. Every request of a comparison carries the same filter.

A successful answer has status 200, and its body is one Negentropy message: the host's answer to the message in the request. The party that started sends requests until it has no further message to send.

The host keeps nothing between two requests. It answers each request from the records it holds at that moment that match the filter.

A host that declines to compare a set, or cannot parse a request, answers with status 400.

The answer carries the header `Access-Control-Allow-Origin: *`, so that a page of another origin can read it.

## After Comparing

Comparing transfers no records. It tells the party that sent the first message which records each of the two lacks; the other party does not learn it from the comparison.

That party fetches the headers and bodies it lacks (see [Retrieval](retrieval.md)) and checks each of them itself (see [Records](record.md)). It delivers the records the other party lacks (see [Records](record.md#delivering)), and then the files those records refer to (see [Blobs](blob.md)).

## New Records

Software learns that records have reached a host from a stream of events, and then finds the records it lacks by comparing. An event carries no record.

Software opens the stream with an HTTP `GET` request to the host:

```
GET https://<host>/.well-known/zu-live?<parameters>
Accept: text/event-stream
```

The parameters are a filter: `by`, `to` and `type`, written as in a [query](query.md). The answer has status 200 and the media type `text/event-stream`. It does not end: the host adds events to it, as [server-sent events](https://html.spec.whatwg.org/multipage/server-sent-events.html).

When a record that matches the filter reaches the host, the host sends an event. The data of the event is a JSON object with strings taken from the record: `cid`, the CID of the record, its `by`, its `type` and, when the record states a city, its `to`:

```
data: {"cid":"<CID>","by":"<DID>","to":"<DID>","type":"ev/event"}
```

Software that does not hold that record compares the sets the record belongs to, or fetches the record by its CID (see [Retrieval](retrieval.md)).

Events are hints. A host may leave events out, for example when many records reach it at once, and it does not send again the events that software missed while it was not connected. Software compares the sets it has a use for when it opens the stream, and again after the stream has been interrupted. An event only makes software learn sooner what a comparison would find.

While it has no event to send, the host sends a comment line, one that starts with a colon, every 15 seconds or so, so that the connection is not closed for being idle.

A host that declines to watch a filter answers with status 400, as it may decline a query. The answer carries the header `Access-Control-Allow-Origin: *`, so that a page of another origin can read it.

## Usage and Implementation Guidelines

Headers are never discarded, so the older part of a set rarely changes. Most ranges match at the first comparison, and the work goes to the newest records.

A device that holds nothing of a set should fetch it with queries (see [Queries](query.md)) instead of comparing.

Negentropy takes as a timestamp any number that fits an unsigned 64-bit integer. Software written for Nostr may assume timestamps in seconds; an implementer who reuses it should check that it handles `at`, which is in milliseconds.

Test vectors for this specification will be published in [`vectors/`](../vectors/).

## Security and Privacy Considerations

`at` is the signer's own statement (see [Records](record.md)). A signer who states odd times makes a comparison take more messages. It does not make the result wrong.

A party can leave records out of what it compares, and the comparison does not show it.

The other party learns which set a party asks about, and so which individuals, cities and types it has a use for. A host learns the same from the filter of a stream of events, and sees for how long software stays connected.

Someone able to write records can try to choose them so that two ranges holding different records have the same fingerprint. A record in such a range then does not reach the other party, and nothing reports it. Negentropy's fingerprint adds up the IDs and includes the number of records. Its author [reports](https://logperiodic.com/rbsr.html) that such sums can be made to collide with about a day of work on a large machine, and describes what makes the attack hard to carry out. A fingerprint that a writer of records cannot defeat is left to a later design.

## Possible Future Changes

Negentropy, which Nostr uses, is used for now. A later design may replace what stands for a record, the order of records with the same `at`, the fingerprint, or the messages. The other design looked at is Willow's Transfer Protocol.

Slow or offline links may use a different way of comparing sets.

Software that cannot keep a connection open, such as an application in the background on a phone, may later be told of new records through a push service.

Questions still open in this draft:

- Whether a set can also be narrowed by key.
