# Retrieval

A specification for fetching content by its CID.

Content in Zu, both DRISL blocks and blobs, is linked by CID (see [Data Model](data-model.md)). A CID says what the content is, not where it is. Zu fetches content by CID with [RASL](https://dasl.ing/rasl.html), the retrieval specification of DASL, unchanged. This document states how Zu uses RASL: how fetched content is checked, where hints come from, and how hosts serve content. The security guidance for serving files and the advice on limits are adapted, with changes, from the [AT Protocol Blob specification](https://atproto.com/specs/blob) (bluesky-social/atproto-website, commit `e079bd4`), which is licensed under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/). It is a draft; the questions still open are listed under Possible Future Changes.

A link is self-certifying: whatever is returned can be checked against the hash in the link. Content can therefore be fetched from anyone, including an untrusted host, and still be trusted once it is checked.

This document covers the `rasl` URL, fetching over HTTPS, checking what was fetched, and serving content. How a file is given to a host and how long the host keeps it is specified in [Blobs](blob.md), and how software asks a host for records in [Queries](query.md). How content that is not public is fetched, and how data is moved in bulk, are left to other documents.

## RASL in Brief

This section is informative; the rules are those of RASL.

A `rasl` URL has a CID as its authority, and may list hosts that the CID can be fetched from as `hint` entries in its query:

```
rasl://bafkreifn5yxi7nkftsn46b6x26grda57ict7md2xuvfbsgkiahe2e7vnq4/?hint=pol.4seas.example
```

To fetch a CID, software sends a request for `https://<host>/.well-known/rasl/<cid>` to one or more hosts. The request uses `GET` or `HEAD`, carries no cookies or credentials of any kind, and uses no content negotiation. A host may answer with the content, or with a redirect to it. Software may also obtain a CID in any other way; RASL is the agreed way, not the only one.

## Checking Content

Software must compute the CID of the content it received, in the CID format of the [Data Model](data-model.md), with the codec of the requested CID. If it does not match the requested CID, the content must be discarded; software may then try another host. Content that matches can be trusted no matter which host served it.

The codec of the CID says how to read the bytes. Content behind a `raw` CID is a blob. Content behind a `dag-cbor` CID is a DRISL block, and should be decoded strictly according to DRISL.

RASL treats every response as `application/octet-stream`. The media type of a blob is the `mime` of the blob reference that links to it (see [Data Model](data-model.md)), and is applied by the software reading that record, after the check.

## Hints

RASL leaves it to the software where hints come from. Software may use the hints in a `rasl` URL, and any host it already knows to hold the content, such as the host it fetched the linking record from.

For content linked from an individual's records, software may use as hints the domains of the handles that the individual's DID claims (see [Map](map.md)). These domains are where an individual's data is looked for (see [Paths](path.md)).

## Serving Content

A host that serves content over RASL:

- answers at `/.well-known/rasl/<cid>` with exactly the bytes of that CID, or with a redirect to them;
- sets `Content-Type: application/octet-stream`;
- sets content security headers, for example:

```
Content-Security-Policy: default-src 'none'; sandbox
X-Content-Type-Options: nosniff
```

RASL requests carry no credentials, so a host serves at this path only content it would give to anyone.

## Usage and Implementation Guidelines

Software may send requests to several hosts in parallel and use the first response that passes the check.

A blob is hashed whole, so it can only be checked once it has been received completely. When the blob reference gives a `size`, software should stop reading a response that grows past it.

Hosts may set their own limits on how much content they store and serve. To maximize interoperability, hosts are recommended to prefer limits on an individual's overall use, such as a total size quota, over limits on the size of each blob.

Test vectors for this specification will be published in [`vectors/`](../vectors/).

## Security and Privacy Considerations

A link is only self-certifying if the content is actually checked against it. Software must not use content it has not checked.

Serving arbitrary user-uploaded files from a web server raises many content security issues, for example cross-site scripting (XSS) through scripts or SVG content served from the same origin as other web pages. The headers in Serving Content are effectively mandatory. The RASL path should not be used to serve pages or assets directly to browsers.

Some media types may contain sensitive metadata. For example, EXIF metadata in JPEG image files may contain GPS coordinates. Once content is served over RASL, anyone who knows its CID can fetch it.

Parsing of media files is a notorious source of memory safety bugs and security vulnerabilities. Even content type detection (or "sniffing") can be a source of exploits. Hosts are strongly recommended against parsing media files directly, without the use of strong sandboxing mechanisms.

Hosts may need to take measures to prevent malicious resource consumption, for example intentional exhaustion of disk space, network congestion, or bandwidth. Rate-limits, size limits, and quotas are recommended.

A host sees which CIDs are requested, and from which network addresses. A hint in a shared `rasl` URL tells everyone who receives it where the content is held. Anyone can ask a host whether it holds a given piece of content, by computing the CID of a known file and requesting it.

## Possible Future Changes

RASL may come to use paths in its URLs; Zu software should ignore them until then, as RASL requires.

Content may also be fetched over peer-to-peer connections between devices, for example through an [iroh](https://www.iroh.computer) relay run by a host, besides HTTPS. Such connections are not specified yet (see the [README](../README.md)); the check in Checking Content applies to content however it is fetched.

Hash types that allow large blobs to be checked while they stream, such as BLAKE3 in [BDASL](https://dasl.ing/bdasl.html), may be added (see [Data Model](data-model.md)).

Questions still open in this draft:

- How content that is not public is fetched.
- Whether every host must serve its content over RASL.
