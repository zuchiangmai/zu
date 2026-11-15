# Services

A specification for what a host can do for applications besides keeping records: how a service and its calls are described, how a host states which services it has loaded, and how software calls one.

This document is adapted, with changes, from the [AT Protocol Lexicon specification](https://atproto.com/specs/lexicon), and follows the [AT Protocol HTTP API specification](https://atproto.com/specs/xrpc) (bluesky-social/atproto-website, commit `59daa1f`); both are licensed under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/). It is a draft; the questions still open are listed under Possible Future Changes.

Every host keeps records and files and answers for them (see [Records](record.md), [Queries](query.md), [Synchronization](sync.md) and [Blobs](blob.md)). Some applications need more from the host of a city: a recording turned into text, a question put to a language model, a picture drawn. Work of this kind is a **service**. A host does not have every service. It **loads** the ones its city has a use for. An application, written by anyone, asks for a service by its name, and works in every city whose host has loaded it.

This document covers the names of services and calls, the description of a service, the list of what a host has loaded, and the request that makes a call. Who does the work of a service, the host itself or a provider elsewhere, is the host's own matter. How a city records which services it loads, and how an application states which ones it needs, are not specified yet.

## Services and Calls

A **service** is one kind of work a host does for applications. It has one or more **calls**. A call takes an input, and either gives back an output or fails with a word that says why.

Zu keeps the word capability for something else: a signed pass that states what its holder may do, which the holder can hand on only with less (see the [README](../README.md)). A service is work a host does when asked.

A service is for one use, and its description states every call it has, what each call takes and what it gives back. Software can then tell what it may call without trying. A service that took any work as its input could not be described in this way, and each host would read such an input in its own way.

## Names

A service and a call each have a name. The two are written as the two segments of a type are (see [Paths](path.md#type-syntax)), the service first:

```
stt/start
```

- The name of a **service** has at least 1 and at most 32 characters; the allowed characters are lowercase ASCII letters (`a-z`), digits (`0-9`) and hyphens (`-`); the first character is a letter.
- The name of a **call** has at least 1 and at most 32 characters; the allowed characters are lowercase ASCII letters, digits and hyphens.

A reference regex for a service and a call written together is:

```
/^[a-z][a-z0-9-]{0,31}\/[a-z0-9-]{1,32}$/
```

An application names the service it needs, not the host or the provider that does the work. Several providers may offer the same service, and a city chooses among them.

## The Description

A **description** says what a service is and which calls it has. It is a map in the [Data Model](data-model.md), with these fields:

- `$type` (string, required): the fixed value `svc`.
- `v` (integer, required): the version of this format, currently `1`.
- `name` (string, required): the name of the service.
- `doc` (string, optional): what the service is for, in one or two sentences.
- `calls` (map, required): the calls, each under its name. It has at least one entry.

Each entry of `calls` is a map with these fields, all optional:

- `doc` (string): what the call does, in one or two sentences.
- `args` (map): the shape of the input, a field description of type `map`. Without it, the call takes an empty map.
- `ret` (map): the shape of the output, a field description of type `map`. Without it, the call gives back an empty map.
- `errs` (map): the words with which this call may fail, besides the ones every call may fail with (see A Call). Each word is the name of an entry, and the entry is a sentence that says when the call fails with it. A word has the syntax of a call name.

The names of these fields are not keywords of common programming languages, so that software generated from a description can use them as they are.

A description is encoded in DRISL and named by its CID. Two hosts that state the same CID follow the same description.

For example, a service that turns a recording into text, in the JSON form of the data model. The example is informative: this document defines no service.

```json
{
  "$type": "svc",
  "v": 1,
  "name": "stt",
  "doc": "Turns a recording of speech into text.",
  "calls": {
    "start": {
      "doc": "Starts work on one recording.",
      "args": {
        "type": "map",
        "req": ["ref"],
        "fields": {
          "ref": { "type": "link", "doc": "The CID of the recording, a blob the host holds." },
          "lang": { "type": "string", "doc": "The language spoken." }
        }
      },
      "ret": {
        "type": "map",
        "req": ["job"],
        "fields": {
          "job": { "type": "string", "doc": "Stands for this piece of work in later calls." }
        }
      },
      "errs": { "gone": "The host does not hold the recording." }
    },
    "get": {
      "doc": "Asks how far a piece of work has come.",
      "args": {
        "type": "map",
        "req": ["job"],
        "fields": { "job": { "type": "string" } }
      },
      "ret": {
        "type": "map",
        "req": ["done"],
        "fields": {
          "done": { "type": "boolean" },
          "text": { "type": "string", "doc": "The text, once the work is done." }
        }
      },
      "errs": { "gone": "The host knows no such piece of work." }
    }
  }
}
```

### Field Descriptions

A **field description** is a map whose `type` says what kind of value a field holds. Every field description may carry `doc` (string), a sentence about the field.

The types are those of the [Data Model](data-model.md) other than null, with `blob` for a blob reference and `any` for a value that is not described:

| `type` | Holds | May also carry |
|---|---|---|
| `boolean` | true or false | |
| `integer` | a signed 64-bit integer | |
| `string` | Unicode text | |
| `bytes` | raw bytes | |
| `link` | a CID | |
| `blob` | a file reference | |
| `list` | several values | `items` (a field description) |
| `map` | named fields | `fields` (map of name to field description), `req` (list of the names that must be present) |
| `any` | any value the data model allows | |

`float` is not a type, because the data model does not allow floats.

A field whose name is not in `fields` is not described. A host ignores such a field in an input, and software ignores such a field in an output.

## What a Host Has Loaded

Software asks a host which services it has loaded with an HTTP `GET` request:

```
GET https://<host>/.well-known/zu-svc?<parameters>
```

One parameter is defined:

| Parameter | Meaning |
|---|---|
| `to` | the DID of the city whose services are asked for |

A host that runs several cities loads services for each city separately, and answers a request without `to` with status 400. A host that runs one city answers for that city when `to` is absent.

A successful answer has status 200 and a body that is a map, encoded in DRISL, with these fields:

- `svcs` (map, required): the services the asker may call, each under its name. An entry is a map with one field, `spec` (link, required): the CID of the description the host follows for that service.
- `blocks` (map, optional): descriptions the host chose to send along, each a map, under the string form of its CID.

The list is the asker's. A request may carry an authentication statement (see [Request Authentication](auth.md)), and the host then lists what that DID may call. Without a statement, the host lists what anyone may call.

Software that does not hold a description fetches it by its CID (see [Retrieval](retrieval.md)) and checks it against that CID. A service that is not in `svcs` is not loaded for the asker, and software does not call it.

The answer carries the header `Access-Control-Allow-Origin: *`, so that a page of another origin can read it.

## A Call

Software makes a call with an HTTP `POST` request:

```
POST https://<host>/.well-known/zu-svc/<service>/<call>?<parameters>

<a map, encoded in DRISL>
```

`<service>` and `<call>` are the two names. The parameter `to` has the meaning it has above. The body is the input: a map with the fields that the `args` of the call describes.

A successful answer has status 200 and a body that is the output: a map, encoded in DRISL, with the fields that the `ret` of the call describes.

An answer that reports a failure has another status, and a body that is a map, encoded in DRISL, with these fields:

- `err` (string, required): a word that says why the call failed.
- `text` (string, optional): a sentence for a person to read.

Every call may fail with these words:

| Word | Status | Meaning |
|---|---|---|
| `bad` | 400 | the input does not follow the description |
| `deny` | 403 | the host does not make this call for the asker |
| `none` | 404 | the host has not loaded the service, or the service has no such call |
| `big` | 413 | the input is larger than the host accepts |
| `full` | 429 | the allowance of the asker, or of the city, is used up |
| `busy` | 503 | the host cannot do the work now; software may call again later |

A call that fails with a word from its own `errs` has status 400. Software treats a word it does not know as `deny`.

A host that has to know who is asking takes it from an authentication statement sent with the request (see [Request Authentication](auth.md)), and answers a request without a valid one with status 401.

The answer carries the header `Access-Control-Allow-Origin: *`, so that a page of another origin can read it.

## Changing a Description

A description is named by its CID, so a changed description is another description. A host states one description for each service it has loaded.

A description that replaces another under the same name changes it only within bounds, so that software written for either one works with a host that follows the other:

- a new call may be added;
- a new field must be optional, that is, not in `req`;
- a call or a field is not removed and not renamed; one no longer used is marked so in its `doc`;
- a name is not added to or removed from `req`;
- the `type` of a field does not change;
- a word may be added to `errs`, and is not removed.

A change beyond these needs a new name for the service.

## Usage and Implementation Guidelines

Describe one use in one service. A service with a few calls that say exactly what they take can be offered by several providers and used by many applications; a service that tries to cover every use cannot.

A call answers when its work is done. A service whose work takes long has one call that starts the work and another that asks for its result, as in the example above.

A call carries no file. Software gives the file to the host as a blob (see [Blobs](blob.md)) and puts its CID in the input.

An application should work without a service its city has not loaded, or say plainly which service is missing.

A host that passes a call on to a provider elsewhere answers as if it did the work itself: it checks who is asking and what the city allows before it passes the call on, and software does not learn where the work was done.

Test vectors for this specification will be published in [`vectors/`](../vectors/).

## Security and Privacy Considerations

Whoever does the work of a service sees the input. When a host passes a call on to a provider elsewhere, that provider sees it as well. A city that loads a service decides whom it lets see what its people send.

A call can cost the city money or space. A host should know who is asking before it makes such a call, and should limit how much each asker may use.

An authentication statement names the host and not the call (see [Request Authentication](auth.md)). A statement sent with one call can be sent again with another call to the same host for as long as it is accepted.

A name does not say which description a host follows; the CID in `spec` does. Software that depends on the exact shape of a call compares that CID with the one it was written for.

A description is data. A host that loads a service does not run anything that the description contains.

The list of what a host has loaded is public, and shows what a city spends its means on.

## Possible Future Changes

Field descriptions may take the constraints, references and unions that the definitions of record types will have, once those are specified (see [Paths](path.md)).

A description may become the body of a record, so that it has an author and a path.

An application may later also call a provider directly, with credentials that the person has set up, without depending on the city.

Who may call a service may later be stated by a capability that the asker holds, instead of by a list that the host keeps.

Questions still open in this draft:

- How a city records which services it loads, and which provider does the work of each.
- How an application states the services it needs.
- How the names of services are kept apart when the same name is chosen twice.
- Whether a description states the limits a host may set, such as how long a recording may be, and how a host lists the values it has set.
- Whether calls that change nothing are told apart from the others, so that their answers can be cached.
- How a call takes a file that no record refers to.
