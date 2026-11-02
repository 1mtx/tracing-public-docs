# Webhooks

A webhook tells a client when an anchor of theirs has ended: it was mined and
succeeded, or it failed. The backend POSTs a signed JSON body to an endpoint an
operator configured for that client, retries until the receiver acknowledges, and
keeps every delivery and every attempt in the database. The client proves it got
the request by signing its answer.

Both directions are signed with Ed25519, so neither side has to trust the network
or a shared secret:

- the **backend signs the request** with its own key, which clients read from
  `GET /api/webhooks/public-key`;
- the **client signs the response** with a key whose public half an operator
  registered with the endpoint.

Webhooks are off until the backend has a `WEBHOOK_SIGNING_KEY`. Without it the
routes answer `503 WEBHOOKS_DISABLED` and nothing is sent.

## What is announced

Every kind of anchor, once it has ended:

| Anchor | Announced when | `data` carries |
|---|---|---|
| A direct event type (one log per hash, e.g. the default `anchor`) | The transaction is mined and succeeded, or it failed | `anchorId`, `keys`, `count`, `txHash`, `blockNumber` |
| A request-batched Merkle tree (an event type whose request is the batch) | The tree is anchored, or has failed | `anchorId`, `batchId`, `root`, `treeRoot`, `treeSize`, `meta`, `txHash`, `blockNumber` |
| A deferred Merkle tree | Same | Same |

A tree webhook states facts about the tree and never lists its leaves, which can
be a hundred thousand: find them through `GET /api/anchors`. An anchor that is
still in flight is not announced; there is no "pending" webhook.

A client without an endpoint gets nothing and nothing is stored for it.

## The body

```json
{
  "id": "6f0c1e4e-5d0a-4a52-9a53-7a2d3c0f8f11",
  "type": "anchor.completed",
  "createdAt": "2026-10-19T10:00:00Z",
  "data": {
    "eventType": "anchor",
    "contractAddress": "0x5FbDB2315678afecb367f032d93F642f64180aa3",
    "status": "CONFIRMED",
    "anchorId": "9b2f4c1e-7d3a-4e8b-a0c5-2f6d1e9b7a40",
    "keys": ["0x9f1c..."],
    "count": 1,
    "txHash": "0xab12...",
    "blockNumber": 4181
  }
}
```

A tree:

```json
{
  "id": "...", "type": "anchor.completed", "createdAt": "...",
  "data": {
    "eventType": "<merkle type>", "contractAddress": "0x...", "status": "CONFIRMED",
    "anchorId": "...", "batchId": 42, "root": "0x...", "treeRoot": "0x...", "treeSize": 3,
    "meta": {"...": "the fields the event type declares"},
    "txHash": "0x...", "blockNumber": 4182
  }
}
```

`root` is the value anchored on chain; `treeRoot` is the root of the Merkle tree
(they differ when a plugin wraps the tree root with more data).

### Matching a webhook to the request

Every answer that names an anchor carries `anchorId`, a UUID, and the webhook
announcing that anchor's end repeats it as `data.anchorId`, whatever the event
type and its plugin. Store it when you send, and match the webhook on it.

- A direct anchor (the `202`) gets one per call, so one for a whole
  `POST /api/anchors/batch`.
- A request-batched tree (the `200`) gets the tree's. A
  retry of the same ordered list returns the same tree, so the same `anchorId`.
- Hashes queued for a deferred tree get none in their `202`: the tree does not
  exist yet. Its webhook still carries the tree's `anchorId` and `batchId`.

A direct anchor submitted before anchor ids existed has a webhook without one.

A failure:

```json
{
  "id": "...", "type": "anchor.failed", "createdAt": "...",
  "data": {
    "eventType": "anchor", "contractAddress": "0x...", "status": "FAILED",
    "anchorId": "...", "keys": ["0x9f1c..."], "count": 1, "reason": "reverted",
    "txHash": "0x...", "blockNumber": 4183
  }
}
```

`reason` is one of:

| `reason` | Meaning |
|---|---|
| `reverted` | The transaction was mined and reverted |
| `cancelled` | The transaction was cancelled before it mined |
| `rejected` | The node never took it |
| `reorged_out` | Another transaction took its place |
| `not_confirmed` | A direct anchor was never seen on chain, and the chain could be read, for longer than `WEBHOOK_CONFIRM_TIMEOUT` |
| `failed` | A Merkle tree failed for some other reason; an operator has been alerted |

A failure is only announced when it is **established**: the pool said so, or the
chain could be read and the anchor was absent past the timeout. If the backend
cannot tell (no index to ask, the index is down), it announces nothing rather than
guess, and the anchor stays unannounced until it can be settled.

## The request

```
POST /your/endpoint
Content-Type: application/json
X-Webhook-Id: 6f0c1e4e-5d0a-4a52-9a53-7a2d3c0f8f11
X-Webhook-Timestamp: 1792404000
X-Webhook-Attempt: 1
X-Webhook-Key-Id: 3f9a1c0b2d4e5f60
X-Webhook-Signature: <base64>
```

| Header | |
|---|---|
| `X-Webhook-Id` | The delivery id, equal to `id` in the body. **Constant across retries.** |
| `X-Webhook-Timestamp` | Unix seconds when this attempt was sent. |
| `X-Webhook-Attempt` | 1, 2, 3 ... Keeps rising across manual retries. |
| `X-Webhook-Key-Id` | First 8 bytes of the SHA-256 of the backend's public key, in hex. |
| `X-Webhook-Signature` | Base64 Ed25519 signature, below. |

### Verifying it

The signed message is the bytes

```
"v1." + id + "." + timestamp + "." + body
```

where `body` is the raw request body exactly as received (do not re-serialise it).
Verify with the key from `GET /api/webhooks/public-key`, and refuse a timestamp
that is more than a few minutes off, which stops a captured request being replayed.

```csharp
using System.Text;
using Org.BouncyCastle.Crypto.Parameters;
using Org.BouncyCastle.Crypto.Signers;

public sealed class WebhookVerifier
{
    private readonly Ed25519PublicKeyParameters backendKey;

    // publicKeyBase64: the publicKey field of GET /api/webhooks/public-key.
    public WebhookVerifier(string publicKeyBase64) =>
        backendKey = new Ed25519PublicKeyParameters(Convert.FromBase64String(publicKeyBase64), 0);

    // rawBody: the request body exactly as received, before any JSON parsing.
    public bool Verify(string id, string timestamp, string signatureBase64, byte[] rawBody)
    {
        if (!long.TryParse(timestamp, out var ts) ||
            Math.Abs(DateTimeOffset.UtcNow.ToUnixTimeSeconds() - ts) > 300)
            return false; // too old or too new: refuse a replay

        byte[] signature;
        try { signature = Convert.FromBase64String(signatureBase64); }
        catch (FormatException) { return false; }

        var prefix = Encoding.UTF8.GetBytes($"v1.{id}.{timestamp}.");
        var verifier = new Ed25519Signer();
        verifier.Init(false, backendKey);
        verifier.BlockUpdate(prefix, 0, prefix.Length);
        verifier.BlockUpdate(rawBody, 0, rawBody.Length);
        return verifier.VerifySignature(signature);
    }
}
```

Pass the headers `X-Webhook-Id`, `X-Webhook-Timestamp` and `X-Webhook-Signature`.
In ASP.NET Core, read the body yourself (copy `Request.Body` into a byte array)
rather than binding it to a model: the signature covers the exact bytes.

## The response

Answer **2xx** with the header `X-Webhook-Response-Signature`: a base64 Ed25519
signature, made with the private key matching the public key the operator
registered, over

```
"v1." + id + "." + hex(sha256(request body)) + "." + hex(sha256(response body))
```

Binding the answer to this request's body and to the answer's own body means a
signature cannot be replayed for another webhook or another answer. An empty
response body is the hash of nothing.

```csharp
using System.Security.Cryptography;
using System.Text;
using Org.BouncyCastle.Crypto.Parameters;
using Org.BouncyCastle.Crypto.Signers;

public sealed class WebhookResponseSigner
{
    private readonly Ed25519PrivateKeyParameters clientKey;

    // seedBase64: the 32-byte private seed of the key pair whose public half the
    // operator registered.
    public WebhookResponseSigner(string seedBase64) =>
        clientKey = new Ed25519PrivateKeyParameters(Convert.FromBase64String(seedBase64), 0);

    // The value of X-Webhook-Response-Signature for this answer.
    public string Sign(string id, byte[] rawRequestBody, byte[] responseBody)
    {
        static string Sha(byte[] b) => Convert.ToHexString(SHA256.HashData(b)).ToLowerInvariant();

        var message = Encoding.UTF8.GetBytes($"v1.{id}.{Sha(rawRequestBody)}.{Sha(responseBody)}");
        var signer = new Ed25519Signer();
        signer.Init(true, clientKey);
        signer.BlockUpdate(message, 0, message.Length);
        return Convert.ToBase64String(signer.GenerateSignature());
    }
}
```

Sign the response body you actually send, byte for byte, and send it with the
header.

A 2xx without a valid response signature **does not count** as delivered; it is
retried like a failure, and the attempt records why.

### Delivery is at least once

If the backend crashes after your answer and before it records it, the same
delivery is sent again. Keep the `X-Webhook-Id` you have processed and ignore a
repeat (still answering 2xx, signed).

## Retries

| Answer | |
|---|---|
| 2xx with a valid response signature | Delivered. |
| Network error, timeout, 5xx, 408, 429, or a 2xx with a bad or missing signature | Retried. |
| Any other 3xx or 4xx | The receiver gave an answer: the delivery ends as `failed` at once. |

Redirects are never followed: a receiver must not be able to aim the backend at an
address of its choosing.

Retries wait 10 s, 1 min, 5 min, 30 min, 2 h, 6 h, 12 h (a little jitter is added,
never subtracted), for `WEBHOOK_MAX_ATTEMPTS` attempts in all (default 8). After
the last one the delivery is `failed`. Nothing is dropped: it stays in the
database, with each attempt.

### Retrying by hand

A `failed` delivery can be retried by the client or an operator:

```
POST /api/webhooks/deliveries/{id}/retry                       (client)
POST /api/admin/users/{username}/webhook/deliveries/{id}/retry (operator)
```

It goes back to `pending`, due now, with a fresh set of attempts (the attempt
numbers go on from where they stopped), sent to the client's endpoint **as it is
now**: if you fixed or moved the receiver, a retry reaches the new one. A delivery
that is still pending is being retried already (`409 DELIVERY_NOT_FAILED`), and a
client with no endpoint has nowhere to send it (`409 WEBHOOK_NOT_CONFIGURED`).
Deliveries of another client are reported as not found.

## Setting it up

An operator, with the admin key, registers the endpoint. A client cannot set its
own: the URL is where the backend sends requests from inside its network.

```bash
curl -X PUT "$BACKEND/api/admin/users/acme/webhook" \
  -H "X-Admin-Key: $ADMIN_KEY" -H 'Content-Type: application/json' \
  -d '{"url":"https://hooks.acme.example/anchors","clientPublicKey":"<base64 of the 32-byte Ed25519 public key>"}'
```

- `url` must be `https`, carry no credentials, and must not point at a loopback,
  private, link-local or carrier-grade-NAT address. The check runs again when the
  backend connects, on the address the name resolves to, so a name that later
  resolves somewhere private is refused too.
- Only the delivery in flight when you change the endpoint keeps its old one for
  its automatic retries; everything that ends afterwards goes to the new one.
- `GET` reads it, `DELETE` removes it. After a delete, deliveries already made
  keep retrying to the endpoint they were made with.

Generate a client key pair with, for example:

```bash
openssl genpkey -algorithm ed25519 -out client.key
openssl pkey -in client.key -pubout -outform DER | tail -c 32 | base64   # the public key to register
openssl pkey -in client.key -outform DER | tail -c 32 | base64           # the seed WebhookResponseSigner takes; keep it secret
```

### Reading what happened

```bash
curl "$BACKEND/api/webhooks/deliveries?state=failed&history=true" -H "X-API-Key: $KEY"
```

Newest first; `state` is `pending`, `succeeded` or `failed`; `limit` 1 to 200;
`history=true` adds every attempt (status, error, duration). Each delivery carries
the exact body that was sent. An operator reads a client's with
`GET /api/admin/users/{username}/webhook/deliveries`.

## Configuration

| Variable | Default | |
|---|---|---|
| `WEBHOOK_SIGNING_KEY` | none | Base64 of a 32-byte Ed25519 seed (`openssl rand -base64 32`). **Unset turns webhooks off.** Keep it: a new key means every client must learn a new public key. |
| `WEBHOOK_MAX_ATTEMPTS` | `8` | Attempts before a delivery fails, 1 to 50, and again after each manual retry. |
| `WEBHOOK_TIMEOUT` | `10s` | One request to a receiver, at most `1m`. |
| `WEBHOOK_CONFIRM_TIMEOUT` | `1h` | How long a direct anchor may go unseen on chain before it is announced as `not_confirmed`. |
| `WEBHOOK_ALLOW_INSECURE` | `false` | Allow `http` and private addresses. For development and tests only. |

## How it stays correct

- **One announcement per end.** A delivery is created with a key naming what it is
  about (`tree:42:anchored`, `watch:17`), so an end seen twice (a callback and then
  the sweep) makes one delivery.
- **Trees** are announced from the same at-least-once hook that runs a plugin's
  after-confirmation code, which survives restarts. A failed tree has that hook
  too; trees that failed before webhooks existed were marked done by the
  migration, so they are not announced late.
- **Direct anchors** have no row of their own, so a *watch* is written before the
  pool is called and carries the pool's handle (sender and nonce) once it accepts.
  The pool's callbacks end it. A restart loses callbacks, so a periodic sweep looks
  up watches older than `WEBHOOK_CONFIRM_TIMEOUT` in the client's index by sender
  and nonce: found and successful is `anchor.completed`, found and reverted is
  `anchor.failed`, and not found on a readable chain is `not_confirmed`. The pool
  also reports failures it discovers by itself after a restart; those end the watch
  by its handle. A client with no index gets no sweep, and a watch that never
  learned its handle is dropped without a word.
- **A client with no endpoint when an anchor is submitted is not watched**, so an
  endpoint set while a direct anchor is already in flight does not get that one.
  Trees are decided when they end, so for them the endpoint in force then is used.
