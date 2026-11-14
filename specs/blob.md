# Blobs

A specification for files: how a file is given to a host, how long the host keeps it, and whom it counts against.

This document is not adapted from an AT Protocol specification, but it follows the life of a blob in atproto: a file is uploaded first, kept once a record refers to it, and discarded when no record does. The guidance on serving files safely is in [Retrieval](retrieval.md). It is a draft; the questions still open are listed under Possible Future Changes.

A **blob** is a file: a run of bytes, named by a CID with the `raw` codec (see [Data Model](data-model.md)). A blob has no path of its own. It belongs to an individual through a [record](record.md) of that individual whose body refers to it with a blob reference. Whose file it is, and what it is for, is whatever that record says.

This document covers uploading a blob, when a host keeps one and when it may discard it, and limits. Fetching a blob is specified in [Retrieval](retrieval.md). Blobs that are not public are not specified yet.

## Uploading

Software gives a blob to a host with an HTTP `PUT` request:

```
PUT https://<host>/.well-known/zu-blob/<cid>
Authorization: Zu <statement>
Content-Type: application/octet-stream

<the bytes of the file>
```

- `<cid>` is the CID of the bytes, computed by the software before it uploads, in the format of the [Data Model](data-model.md) with the `raw` codec.
- The `Authorization` header carries an authentication statement (see [Request Authentication](auth.md)). A blob is not signed, so the statement is how the host knows which DID uploads it.

The host computes the CID of the bytes it received. If it is not the CID in the URL, the host discards the bytes and answers with status 400.

The host answers a successful upload with a success status. Uploading a blob the host already holds succeeds and changes nothing.

Other answers:

| Status | Meaning |
|---|---|
| 401 | the statement is missing or does not pass |
| 403 | the host does not keep files for this DID |
| 413 | the blob is larger than the host accepts, or the individual's allowance is used up |

## Keeping and Discarding

A host keeps a blob for as long as a record gives it a reason to:

- A blob that no record refers to yet is kept for at least one hour after it was uploaded, so that the software has time to sign and deliver the record that refers to it.
- A blob is kept while a record the host holds refers to it and is the record its path refers to (see [Records](record.md)).
- When no such record remains, the host may discard the blob. When the records that referred to it have been removed, the host discards it, as it discards their bodies (see [Records](record.md)).

A host finds the blob references in a body by their shape, at any depth, without knowing the record's type (see [Data Model](data-model.md)).

There is no request for deleting a blob. A person removes a file by removing or replacing the records that refer to it.

A host need not serve a blob that no record refers to.

## Limits

A host may limit the size of a single blob and the total an individual may have. It is recommended to limit the total rather than each blob (see [Retrieval](retrieval.md)).

A blob counts against the individual whose records refer to it, and before any record refers to it, against the DID that uploaded it. How a host counts a blob that the records of several individuals refer to is its own matter.

A type may narrow what a blob field accepts, by media type and by size (see [Types](type.md)). A host that checks a record against its definition applies those bounds to the blob reference.

## Usage and Implementation Guidelines

Upload the blob first, then deliver the record that refers to it. A host that receives a record referring to a blob it does not hold can keep the record; the blob can be uploaded, or fetched from another host, afterwards.

The media type of a blob is the `mime` of the blob reference, written by whoever wrote the record. A host stores bytes and does not decide what they are.

The same file has the same CID wherever it is, so a host stores one copy however many records refer to it.

Test vectors for this specification will be published in [`vectors/`](../vectors/).

## Security and Privacy Considerations

A host that accepted files from anyone would be used to store anything. Requiring an authentication statement, holding an unreferenced blob only for a short time, and limits on each individual keep that in bounds.

Once a record refers to a blob and the host serves it, anyone who knows the CID can fetch it, and may keep it. Removing the record stops the host from serving the blob; it does not recall copies.

Because a file has one CID, a host can tell that two people hold the same file, and so can anyone who has the file and asks a host for its CID.

A host learns which DID uploads which file, and when.

Files can carry hidden data, such as the place a photograph was taken. A host should not parse files to remove it (see [Retrieval](retrieval.md)); software removes it before it uploads.

## Possible Future Changes

Large files may be uploaded in parts and resumed, and checked while they are received, with the hash types of [BDASL](https://dasl.ing/bdasl.html) (see [Data Model](data-model.md)).

Questions still open in this draft:

- How long a host keeps a blob after the last record that referred to it was replaced or removed.
- Whether a host tells software its limits before an upload.
