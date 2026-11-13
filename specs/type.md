# Types

A specification for defining a type of record: what its body holds, which keys it uses, where its definition lives and how software finds it.

This document is adapted, with changes, from the [AT Protocol Lexicon specification](https://atproto.com/specs/lexicon) (bluesky-social/atproto-website, commit `63312bd`), which is licensed under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/). It is a draft; the questions still open are listed under Possible Future Changes.

Every [record](record.md) states its type in its signed header, as an [NSID](nsid.md) such as `com.example.note`. A **type definition** says what the body of such a record holds.

A definition is itself a record, written by whoever controls the domain in the type's name. There is no registry: a person or a city with a handle can define types under that handle's domain, and nobody approves them.

Software that does not know a type still stores a record of it and passes it on. A definition is for checking and for writing software, not for permission.

This document covers the definition, the field types, the string formats, the kinds of keys, how a definition is found, and how a definition may change. How records are signed is specified in [Records](record.md), and how data is encoded in the [Data Model](data-model.md).

## A Definition

The types the protocol itself defines have the domain authority `ink.zuzalu`, from the domain name `zuzalu.ink`. The one defined here is `ink.zuzalu.type`, the type of a type definition. Anything else the protocol comes to define is named under the same authority.

A definition is a record of type `ink.zuzalu.type` whose key is the NSID it defines. Its body is a map with these fields:

- `v` (integer, required): the version of this definition format, currently `1`.
- `doc` (string, optional): what records of this type are for, in one or two sentences.
- `key` (string, required): which keys records of this type use (see Keys).
- `body` (map, required): the shape of the body, a field description of type `map` (see Field Types).
- `defs` (map, optional): named shapes used more than once inside `body`, each a field description. A `ref` of the form `#name` points to one of them.

For example, the definition of `com.example.note`, written as a record at `zu://did:zu:ewvi7nxzyoun6zhxrhs64oiz/ink.zuzalu.type/com.example.note`:

```json
{
  "v": 1,
  "doc": "Something a person wrote, with an optional picture.",
  "key": "any",
  "body": {
    "type": "map",
    "req": ["text"],
    "fields": {
      "text": { "type": "string", "max": 10000 },
      "title": { "type": "string", "max": 120 },
      "pic": { "type": "blob", "accept": ["image/png", "image/jpeg"], "max": 5000000 }
    }
  }
}
```

The body of a record of this type is checked against the `body` of its definition.

## Field Types

A **field description** is a map whose `type` says what kind of value the field holds. Every field description may carry `doc` (string) and `nullable` (boolean, whether the value may be `null`).

The types are those of the [Data Model](data-model.md) other than null, with `blob` for a blob reference, plus three that exist only in definitions:

| `type` | Holds | May also carry |
|---|---|---|
| `boolean` | true or false | `const` |
| `integer` | a signed 64-bit integer | `min`, `max`, `enum`, `const` |
| `string` | Unicode text | `min`, `max` (Unicode code points), `format`, `enum`, `const` |
| `bytes` | raw bytes | `min`, `max` (bytes) |
| `link` | a CID | |
| `blob` | a file reference | `accept` (list of media types), `max` (bytes) |
| `list` | several values | `items` (a field description), `min`, `max` (number of values) |
| `map` | named fields | `req` (list of field names that must be present), `fields` (map of name to field description) |
| `ref` | a shape defined elsewhere | `ref` (see References) |
| `union` | one of several shapes | `refs` (a list, see References) |
| `any` | any value the data model allows | |

`float` is not a type, because the data model does not allow floats.

A field whose name is not in `fields` is not described. Software keeps such fields and passes them on (see Checking a Record).

`accept` entries may end in `/*`, as in `image/*`.

The `max` and `min` of a string count Unicode code points, not bytes and not what a reader sees as one character, so that every implementation counts the same.

### References

A `ref` names a shape in one of three ways:

- `#name`: the entry `name` in the `defs` of this definition;
- `<NSID>#name`: the entry `name` in the `defs` of the definition of that NSID;
- `<NSID>`: the `body` of the definition of that NSID.

A `union` holds a value that follows one of the shapes named in `refs`. The value is a map, and it states which shape it follows in a `$type` field, whose value is that entry of `refs` written in full, with the NSID in front of `#name`. A value that is not a map, or has no `$type`, fails the check. A value whose `$type` is not among `refs` is kept and not checked, like a field that is not described, so that a definition can come to allow more shapes.

## String Formats

A `string` field may be constrained to one of these formats:

| `format` | The string is |
|---|---|
| `did` | a DID (see [DIDs](did.md)) |
| `handle` | a handle (see [Map](map.md)) |
| `nsid` | an NSID (see [NSIDs](nsid.md)) |
| `key` | a record key (see [Paths](path.md)) |
| `path` | a `zu://` URI (see [Paths](path.md)) |
| `cid` | a CID in string form (see [Data Model](data-model.md)) |
| `uri` | a URI with a scheme |
| `lang` | a language tag ([BCP 47](https://www.rfc-editor.org/info/bcp47)) |

There is no format for a time. A time in a body is an `integer`, in milliseconds since 1970-01-01T00:00:00Z, as `at` is in a record header (see [Records](record.md)). Where the zone matters, as for the start of an event, it is given beside the time as a `string` holding the name of the zone in the IANA time zone database, such as `Asia/Bangkok`.

## Keys

`key` says which keys records of a type use:

- `self`: the key is the string `self`. An individual has at most one record of this type.
- `any`: the writer chooses the key, within the key syntax of [Paths](path.md).

A record whose key is not one that its type's `key` allows fails the check (see Checking a Record).

## Finding a Definition

The domain authority of an NSID, read in forward order, is a hostname: `com.example.note` has the hostname `example.com`. Software finds the definition by:

- resolving that hostname as a handle, which gives a DID (see [Map](map.md));
- fetching the record at the path `<that DID>/ink.zuzalu.type/<the NSID>`.

Nothing else is needed: a definition is found through the same handle resolution as everything else, and whoever holds the domain as a handle holds the definitions under it.

A definition that cannot be found is not an error. Software that wanted to check a record of that type treats the type as unknown.

## Checking a Record

Software checks a record against the definition of its type in one of three ways:

- **check**: the record must be checked against its definition. If the definition cannot be found, the record is refused.
- **do not check**: the record is not checked against any definition, even one that is known.
- **check if known** (the default): the record is checked when the definition is known or can be fetched, and accepted when it cannot.

A host that stores records for people uses **check if known**, so that a type nobody has heard of still works.

Checking a record means: its key is one that the definition's `key` allows; its body is a map valid under the data model; every name in `req` is present; and every field that `fields` describes holds a value of the type it describes, within its constraints. Fields that `fields` does not describe are kept as they are. A record whose `body` is `null` removes the record at its path and has no body to check.

A record that fails the check is not used: software does not repair it and does not use part of it, and a host may refuse to store it. It is still a validly signed record. Where software holds it, it replaces the records it follows like any other record (see [Records](record.md)), so that which record a path refers to does not depend on which definitions software knows.

## Changing a Definition

A definition may change within bounds: a record that passed the check of the old definition must pass the check of the new one, and a record written for the new one must pass the check of the old one.

- a new field must be optional, that is, not in `req`;
- a field is not removed from `fields` and not renamed; one no longer used is marked so in its `doc`;
- a name is not added to or removed from `req`;
- the `type` and `format` of a field do not change;
- the constraints on a field (`min`, `max`, `enum`, `const`, `accept`, `nullable`) are not added, removed or changed;
- `key` does not change;
- entries may be added to the `refs` of a `union`, and are not removed.

A change beyond these needs a new NSID. A type still being worked out should say so in its name, as in `com.example.test.note`.

A definition belongs to whoever controls its domain. That authority can also be lost: if the domain changes hands, the new holder defines the types under it. Where an ecosystem has to leave such an authority behind, it forks the types into a namespace of its own.

## Usage and Implementation Guidelines

Field names follow the naming rules in [CONVENTIONS](../CONVENTIONS.md): lowercase, short, and drawn from the shared vocabulary where it fits. A definition does not repeat what the record header already carries: not the type, not the key, and not the time the record was signed.

A definition is a record like any other, so it is signed, it can be updated, and its history is kept by whoever keeps the earlier records.

Keep `any` fields out of bodies: a field nobody describes cannot be checked by anyone.

Test vectors for this specification will be published in [`vectors/`](../vectors/).

## Security and Privacy Considerations

A definition says nothing about whether the data in a record is true. It constrains the shape, not the content.

An NSID records which domain named a type (see [NSIDs](nsid.md)). Whoever holds that domain can change the definition, and can publish one that refuses records other software accepted. Software that has fetched a definition should keep working from it when the domain stops answering, rather than refusing records it accepted before.

Fetching a definition tells the domain's host which type the fetching software is interested in, and resolving the handle tells the DNS resolvers and the directory the same.

A record of an unknown type is stored and passed on unchecked. A host that accepts such records accepts whatever the data model allows, so the limits a host sets on size and nesting apply to them.

## Possible Future Changes

More string formats may be added, for example for a calendar date without a time of day.

A `default` for a field, and open sets of suggested values beside the closed `enum`, may be added.

Questions still open in this draft:

- Which DID holds the definitions of the protocol's own types, and so claims `zuzalu.ink` as a handle.
- Whether a definition may describe a type under a domain other than the one its writer's handle confirms, as atproto allows through its own `_lexicon` DNS record.
- Whether the key of a definition record being the NSID is enough, or the body should state the NSID as well.
