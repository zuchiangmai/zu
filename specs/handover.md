# Key Handover

A specification for how a Zu signing key is handed from the page that holds it to another page: on the same device, and between devices.

This document is not adapted from an AT Protocol specification; in atproto the hosting server holds the signing key and nothing is handed over. It is a draft; the questions still open are listed under Possible Future Changes.

A person's Zu signing key is held in web origins, and a person signs in to another city, or moves to another device, by handing that key over (see [DIDs](did.md#signing-in-to-another-city)). The rules of a handover are stated there: the key is handed over only after the person chose to in the page that holds it, that page shows who receives it, the receiving page checks the key before it uses it, and rotation keys are never handed over. This document specifies the messages and the steps.

Two pages take part:

- the **receiving page**, in the origin that wants the key, for example the site of city A;
- the **handover page**, in an origin that holds the key, for example a page of the site of city B.

This document covers how the receiving page finds the handover page, the messages, the exchange on one device, and the exchange between two devices. What the handover page shows the person, beyond the rules in [DIDs](did.md), is left to the software.

## Messages

The messages are maps in the [Data Model](data-model.md), encoded in DRISL. They are not signed.

A **request**, sent by the receiving page on the same device:

- `$type` (string, required): the fixed value `handover-request`.
- `v` (integer, required): the fixed value `1`.
- `did` (string, optional): the DID the person wants to use, when the receiving page knows it, for example from a handle the person entered.

A **key message**, sent by the handover page:

- `$type` (string, required): the fixed value `handover`.
- `v` (integer, required): the fixed value `1`.
- `did` (string, required): the DID the key belongs to.
- `key` (bytes, required): the private Zu signing key, as the 32 bytes of an Ed25519 private key ([RFC 8032, section 5.1.5](https://www.rfc-editor.org/rfc/rfc8032#section-5.1.5)).

## Finding the Handover Page

On the same device, the receiving page learns from the person which handle to use, for example `alice.4seas.example`, and asks that handle's domain for the handover page:

```
GET https://alice.4seas.example/.well-known/zu-handover
```

A valid response has an HTTP success status (2xx), and its body is the `https` URL of the handover page, as plain text with no wrapper. The handover page may be in another origin than the handle's domain; a city that issues handles under its domain answers for all of them with the page of its own site. The response should carry the header `Access-Control-Allow-Origin: *`, so that the receiving page can read it.

Between devices no lookup is needed: the person opens the handover page on the device that holds the key.

## On the Same Device

1. On the person's action, the receiving page opens the handover page in a new window.
2. When it is ready, the handover page sends the string `zu-handover-ready` to the page that opened it, with `postMessage`.
3. The receiving page answers with a request, as DRISL bytes, sent with `postMessage` to the origin of the handover page only.
4. The handover page takes the origin that the browser reports for that message as the receiving origin. It does not take it from the content of any message. It shows the person who receives the key and waits for the person's choice.
5. If the person agrees, the handover page sends the key message, as DRISL bytes, with `postMessage` to the receiving origin only. It then closes its window.

If the window closes without a key message, the receiving page treats the handover as declined.

## Between Devices

The receiving page is on one device, and the handover page on another.

### The Code

The receiving page makes a P-256 key pair for this one handover, and a room number of 16 random bytes. It shows a code, usually a QR code, whose content is a URL:

```
https://<receiving origin>/.well-known/zu-relay/<room>#<public key>
```

- `<room>` is the room number as 32 lowercase hexadecimal characters.
- `<public key>` is the public key of the pair, as the 65 bytes of an uncompressed point, in base64url without padding. It is in the fragment, which browsers do not send to servers.

The URL without its fragment is the **room URL**.

The person scans the code from within the handover page on the device that holds the key. Opening the URL in a browser hands over nothing.

### Sealing the Key

The handover page reads the receiving origin from the URL, shows the person who receives the key, and waits for the person's choice. If the person agrees, it:

- makes a P-256 key pair of its own for this one handover
- computes the ECDH shared secret from its private key and the public key in the code, a 32-byte value
- derives a 32-byte key with HKDF-SHA-256: the shared secret as input, the two public keys as salt (the one from the code first, then its own, each as 65 bytes), and the ASCII string `zu-handover-1` as info
- encrypts the DRISL bytes of the key message with AES-256-GCM under that key, with 12 random bytes as the nonce, and the ASCII bytes of the room URL as additional data

It sends a **sealed message**, a DRISL map:

- `$type` (string, required): the fixed value `handover-sealed`.
- `v` (integer, required): the fixed value `1`.
- `epk` (bytes, required): its own public key, as 65 bytes.
- `iv` (bytes, required): the 12-byte nonce.
- `ct` (bytes, required): the ciphertext, with the 16-byte authentication tag at its end.

```
POST <room URL>
Content-Type: application/octet-stream

<DRISL bytes of the sealed message>
```

Both key pairs are discarded after the handover.

### The Relay

The host of the receiving origin relays the sealed message. It cannot read it.

- A room comes into being with the first request for it, and is forgotten 5 minutes later.
- A room takes one message. The host answers the first `POST` of at most 64 KiB with a success status, and any later `POST` to the same room with status 409. A code therefore works once.
- `GET <room URL>` answers with the message bytes once they are there, and with status 404 before that and after the room is forgotten. The receiving page asks again until the message arrives.
- Both responses carry `Access-Control-Allow-Origin: *`, because the handover page is usually in another origin.
- The host keeps messages in memory only, and may limit how many rooms it holds.

The receiving page fetches the sealed message, derives the same key from its private key and `epk`, and decrypts.

## Receiving the Key

However the key message arrived, the receiving page, before it uses the key:

- checks that `$type` is `handover` and `v` is `1`
- if the person named a DID, checks that `did` is that DID
- resolves the DID document of `did`, and checks that the public key of `key` is one of its Zu signing keys

A key message that fails a check is discarded. Otherwise the receiving page stores the key, in a form it can export (see [DIDs](did.md)).

## Usage and Implementation Guidelines

All the primitives used here are in the WebCrypto API: ECDH on P-256, HKDF with SHA-256, and AES-GCM.

The receiving page should tell the person which DID, and which name, it is about to sign in as, before it replaces a key it already holds.

Some applications open links in a browser of their own, whose storage is separate from the person's browser and is often cleared. A receiving page that finds itself in one should say so before it shows a code.

Test vectors for this specification will be published in [`vectors/`](../vectors/).

## Security and Privacy Considerations

A handover gives the receiving origin the key itself. Everything in [DIDs](did.md) about origins that hold a key applies to it from then on.

On the same device, the handover page knows the receiving origin from the browser, and no other page can read the messages. A page that opens the handover page can only ask; the person decides.

Between devices, nothing proves that the code on the screen belongs to the device in front of the person. Someone can start a handover on a device of their own, at a city the person trusts, and bring the person to scan that code: the handover page then truthfully shows the trusted city, and the key goes to the other device. This is how accounts of messaging applications have been taken over through their "link a device" codes. The only defences are the rules in [DIDs](did.md): the code is scanned from within the handover page, on the person's own initiative, and the page says what is about to happen.

The relaying host sees the room number, the size and time of the message, and the network addresses of both devices. It does not see the key. A host that alters the sealed message causes decryption to fail. It cannot seal a message of its own either, because the public key in the code is in the fragment of the URL and never reaches it.

Anyone who photographs the code can send a message to the room first. The receiving page then holds a key message it did not expect; the checks under Receiving the Key apply, and the receiving page shows which DID it would sign in as.

## Possible Future Changes

The sealing may move to HPKE ([RFC 9180](https://www.rfc-editor.org/rfc/rfc9180)) once browsers provide it.

Questions still open in this draft:

- Whether the code may also be shown by the device that holds the key and scanned by the receiving device, for a device with no camera, and what the person then compares on the two screens.
- Whether a person's records are sent along with the key, before records are synchronized in another way.
