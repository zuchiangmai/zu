# How a Zu specification is written

Zu's specifications follow the form of the AT Protocol specifications at [atproto.com/specs](https://atproto.com/specs/atp).

## One specification, one document

Each specification is one document in `specs/`. A specification refers to another by link and never copies its rules.

## Shape of a document

1. A title and a one-line description.
2. A lead paragraph: what the document covers, and what it leaves to other documents.
3. Sections for the topic itself: structure, syntax, processing, examples. Syntax is given as rules followed by valid and invalid examples.
4. At the end, always these three, in this order:
   - **Usage and Implementation Guidelines**
   - **Security Considerations**, or **Security and Privacy Considerations** when the document makes something visible to others
   - **Possible Future Changes**

## Wording

Requirements are plain sentences. "must", "should" and "may" are written in lowercase and carry their ordinary meaning, as in the atproto specifications. Examples and notes are informative.

## Names in data

Field names in signed data and in record bodies follow the naming habits of Unix: all lowercase, short, and abbreviated where the short form is plain, as in `at`, `by`, `size`, `mime`, `tz`. Compound names are not written in camel case and not joined with underscores; where one word will not do, a specification picks a shorter word.

Field names are shared across types instead of invented per type, the way the fields of `struct stat` are the same for every file. A type uses these names where they fit:

| Meaning | Field |
|---|---|
| A name, a heading, prose | `name`, `title`, `text` |
| The sort of thing it is | `kind` |
| A file or other content: a link to it, its media type, its length | `ref`, `mime`, `size` |
| Time: a moment, a span, a zone | `at`, `start`, `end`, `tz` |
| Width and height | `w`, `h` |
| The person concerned, the state something changes to, a yes or no | `who`, `to`, `ok` |

The list is for the bodies of records: a type that needs a word for one of these meanings uses the word given here. It grows when a meaning that several types share gets a name.

Fields defined by another standard keep the names that standard gives them, even when those names are in camel case. Examples are `alsoKnownAs`, `verificationMethod` and `publicKeyMultibase` in DID documents.

Names beginning with `$` are reserved for the data model and the protocol (see [Data Model](specs/data-model.md)).

## Versions

The protocol as a whole has one version, stated in the [README](README.md); it is now v0.3. As in atproto, a single specification has no version number of its own: it is edited in place, and its git history is its record. A changelog is not kept separately. A data format that may change incompatibly carries its own version inside the data, the way atproto's repository format states "version 3" and its event stream "v0". A change that is expected but not yet made is listed under Possible Future Changes first.

## Test vectors

Every rule that two implementations could apply differently has test data in `vectors/`: valid inputs, invalid inputs, and expected results. An implementation conforms when it passes them.

## References

Other specifications are cited by stable URL.
