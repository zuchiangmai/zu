# Synchronization

A specification for how two parties find out which records each of them lacks: which records they compare, in which order, and how they narrow down the differences.

This document is not adapted from an AT Protocol specification. Its form follows Willow's [region-based set reconciliation](https://willowprotocol.org/specs/rbsr/index.html), which applies [range-based set reconciliation](https://arxiv.org/abs/2212.13567) to sets that are chosen first. It is a draft, and a coarse one: it states what is compared and how a difference is narrowed down, and leaves the encoding of the messages to a later design. The questions still open are listed under Possible Future Changes.

This document covers the set of records two parties compare, the order of the records in it, and comparing by fingerprints. What a record is, and which records a host keeps, is specified in [Records](record.md). How content is fetched once a party knows what it lacks is specified in [Retrieval](retrieval.md), and how files follow records in [Blobs](blob.md). How a record is delivered to a host, and how software learns of new records as they arrive, are not specified yet.

## The Set

Two parties, such as a device and a host, or two hosts, first agree on the set of records they compare. A set is named by what record headers state, in the way a [query](query.md) names it:

- by `by`: the records of one individual, or of every individual the party holds records for;
- by `type`: the records of one type, or of every type under one domain authority.

Each party takes the records it holds that match. That includes records that have been replaced or removed: every header is kept (see [Records](record.md)), and a removal is a record, so it reaches the other party like any other.

A party need not compare everything it holds. A device compares the sets it has a use for.

## The Order

Within a set, records are ordered:

1. by `at`, the earlier first;
2. records with the same `at`, by their path (see [Paths](path.md)).

How records with the same `at` and the same path are ordered is not specified yet.

## Comparing

A **range** is a run of consecutive records in this order. A **fingerprint** is a short value computed from the records of a range. Two parties that hold the same records in a range compute the same fingerprint.

- One party sends the fingerprint of a range.
- The other party computes the fingerprint of the records it holds in the same range. If the two are equal, the parties hold the same records there, and the range is done.
- If they differ, the range is split into smaller ranges, and each of them is compared in the same way.
- When a range holds few records, a party lists the records it holds in it instead of sending a fingerprint. The other party then sees which records each of them lacks.

A party splits a range by the number of records it holds in it, into ranges that hold roughly the same number each. It does not split by the length of time a range spans.

Comparing ends when no range is left whose fingerprints differ.

## After Comparing

Comparing transfers no records. It tells which records each party lacks.

A party fetches the headers and bodies it lacks (see [Retrieval](retrieval.md)) and checks each of them itself (see [Records](record.md)). It delivers the records the other party lacks, and then the files those records refer to (see [Blobs](blob.md)).

## Usage and Implementation Guidelines

Headers are never discarded, so the older part of a set rarely changes. Most ranges match at the first comparison, and the work goes to the newest records.

A device that holds nothing of a set should fetch it with queries (see [Queries](query.md)) instead of comparing.

Test vectors for this specification will be published in [`vectors/`](../vectors/) once the encoding is designed.

## Security and Privacy Considerations

`at` is the signer's own statement (see [Records](record.md)). A signer who states odd times makes a comparison take more messages. It does not make the result wrong.

A party can leave records out of what it compares, and the comparison does not show it.

The other party learns which set a party asks about, and so which individuals and types it has a use for.

A fingerprint must be hard to forge: someone able to write records must not be able to choose them so that two ranges holding different records have the same fingerprint. This is a requirement on the encoding to be designed.

## Possible Future Changes

The encoding is left to a later design: how a set and a range are written, how a fingerprint is computed, how a record is named in a list, and how the messages are carried. The designs looked at are [Negentropy](https://github.com/hoytech/negentropy), which Nostr uses, and Willow's Transfer Protocol.

Slow or offline links may use a different way of comparing sets.

Questions still open in this draft:

- How two paths are compared, and how records with the same `at` and the same path are ordered.
- What stands for a record in a comparison: the CID of its header, or more.
- Whether a set can also be narrowed by key.
- Into how many ranges a range is split, and below how many records a party lists them.
- How a record is delivered to a host.
- How software learns of new records as they arrive.
