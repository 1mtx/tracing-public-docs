# Anchoring

The integrity service records a hash on chain, so that anyone can later check
that the same hash existed at that time and has not changed. The service never
sees your content: you hash it, you send the hash, and the service anchors the
hash in a registry contract that belongs to you alone.

## Contents

- [Before you start](#before-you-start)
- [Anchor one hash](#anchor-one-hash)
- [Anchor many hashes in one transaction](#anchor-many-hashes-in-one-transaction)
- [Look a hash up](#look-a-hash-up)
- [Event types](#event-types)
- [Your registry and its index](#your-registry-and-its-index)
- [Errors](#errors)
- [SDKs](#sdks)

## Before you start

- **Credentials.** VMatrix creates your client and gives you an API key. Send
  it as `X-API-Key: <key>` on every call. HTTP Basic Auth is also accepted when
  VMatrix set up a user for you.
- **Your own registry.** Each client anchors into its own registry contract,
  deployed when the client is created. Anchoring and lookups are always scoped
  to that contract: two clients that anchor the same hash never see each
  other's records.
- **Hashes.** A hash is 32 bytes, written as `0x` and 64 hex digits. You compute
  it with your own algorithm; it must not be all zeros.

## Anchor one hash

```
POST /api/anchors
X-API-Key: <key>
Content-Type: application/json

{ "hash": "0x1111111111111111111111111111111111111111111111111111111111111111", "signingTime": 1786332600 }
```

| Field | Type | Meaning |
|---|---|---|
| `hash` | string | The 32-byte content hash |
| `signingTime` | integer | When the content was signed, in Unix **seconds**. Required; `0` is a legal value, negative is refused. No upper bound is checked, so a value in milliseconds is accepted and means a date far in the future: check the unit. |

Answer `202`:

```json
{ "blockchainStatus": "SUBMITTED", "contractAddress": "0x5FbDB2315678afecb367f032d93F642f64180aa3", "anchorId": "9b2f4c1e-7d3a-4e8b-a0c5-2f6d1e9b7a40" }
```

`202` means the node accepted the transaction, not that it is mined. There is
no transaction hash yet on purpose: a slow transaction is re-signed at a higher
fee under a new hash. Learn the outcome in one of two ways:

- read the hash back with `GET /api/anchors` (below);
- receive the [webhook](../reference/webhooks.md) for this `anchorId`.

## Anchor many hashes in one transaction

`POST /api/anchors/batch` takes a JSON array of the same objects, from 1 to 200
items, and anchors them all in one transaction. Each item still emits its own
log, identical to a single anchor.

```json
[
  { "hash": "0x1111111111111111111111111111111111111111111111111111111111111111", "signingTime": 1786332600 },
  { "hash": "0x2222222222222222222222222222222222222222222222222222222222222222", "signingTime": 1786332601 }
]
```

The answer is the single answer plus `count`, with one `anchorId` for the
whole transaction. Validation is all or nothing: one bad item refuses the whole
array, and the error names its zero-based `index`, for example
`{ "error": "INVALID_HASH", "index": 2 }`.

## Look a hash up

```
GET /api/anchors?hash=0x1111...1111
X-API-Key: <key>
```

```json
{
  "hash": "0x1111...1111",
  "txHashes": ["0x0f97b6d9..."],
  "proofType": "transactionHash",
  "proof": ["0x0f97b6d9..."]
}
```

- An empty list means the hash was never anchored in your registry.
- Several transactions can be listed: anchoring the same hash again is allowed,
  and every proving transaction is returned.
- Every transaction is re-read from the chain and must carry a matching log
  before it is returned. A `502` means the check could not be made, never that
  the hash is missing.
- Read `proof` and `proofType` rather than `txHashes`. Today `proofType` is
  `transactionHash` and `proof` repeats `txHashes`; a later proof form will use
  the same pair. Treat an unknown `proofType` as unverifiable.
- The answer also carries `event`: the ABI, signature, `topic0` and contract a
  verifier needs to decode the log on chain without trusting the service.

To verify on your own, take each transaction from `proof`, read its receipt
from the chain, and check that it holds a log from your registry whose indexed
topic is your hash.

## Event types

The default event type, `anchor`, is everything above: one log per hash. Other
event types can be enabled for a client. They use the same paths with
`?eventType=<name>`; the request fields, the answer and the log layout are the
type's own, and you read them from:

```
GET /api/anchors/schema?eventType=<name>
```

The schema also gives the ABI and `topic0`, so a verifier can pin the event
layout before any proof exists.

An event type anchors in one of these modes:

| Mode | What happens | Answer |
|---|---|---|
| One log per hash | Same as `anchor` | `202` |
| Merkle tree, built later | Hashes are queued; the service builds a tree by the type's rules | `202` with `blockchainStatus: QUEUED`; read the tree back with `GET /api/anchors?eventType=<name>&hash=` |
| Merkle tree, one per request | The request is the batch: the service builds the tree, anchors its root and waits for confirmation | `200` with the tree's fields |

For the Merkle modes:

- You hash each leaf; leaves enter the tree as given.
- A hash is stored once. A list that repeats a hash, or names one that an
  earlier request already queued or anchored, is refused with
  `409 DUPLICATE_HASH` and nothing is stored. The hashes of a tree that failed
  are released.
- In the one-per-request mode, sending the identical ordered list again returns
  the same tree and anchors nothing twice. `504 CONFIRMATION_TIMEOUT` means the
  tree is stored and still anchoring: send the same request again.
- A lookup returns the audit path of the leaf and the `state` of its tree.
  `batchId` reads the leaf in one named tree.

## Your registry and its index

| Route | What it gives |
|---|---|
| `GET /api/contract` | Your registry contract and its state |
| `GET /api/index/events` | Logs of your registry in chain order, filtered by `topic0` (the event's schema id) to `topic3`, `txHash`, block range; paged with `cursor` |
| `GET /api/index/txs` | Transactions sent to your registry |
| `GET /api/index/txs/{hash}` | One transaction with its logs |
| `GET /api/index/abis` | The event ABIs the index knows |
| `GET /api/index/status` | How far your index has read the chain |

The index routes are always scoped to your own registry. Only indexed fields
(the topics) can be searched; decoded values are shown but not searchable. A
client without an index gets `503 INDEX_NOT_CONFIGURED` on these routes only.

## Errors

Errors are JSON: `{ "error": "<CODE>" }`, sometimes with `field`, `index` or
`message`.

| Status | Codes | What to do |
|---|---|---|
| 400 | `INVALID_HASH`, `INVALID_SIGNING_TIME`, `INVALID_REQUEST_BODY`, `EMPTY_BATCH`, `BATCH_TOO_LARGE`, `BATCH_NOT_SUPPORTED`, `MISSING_HASH`, `MULTIPLE_HASHES` | Fix the request |
| 401 | | Check the API key |
| 404 | `UNKNOWN_EVENT_TYPE` | Check `eventType` |
| 409 | `DUPLICATE_HASH` | The hash is already stored; do not resend it in a new request |
| 413 | `REQUEST_TOO_LARGE` | Send fewer items |
| 422 | `BATCH_REJECTED` | The event type refused the batch; nothing was stored |
| 501 | `QUERY_NOT_AVAILABLE` | This event type is looked up through the index only |
| 502 | `ANCHORING_FAILED`, `TX_BUMP_LIMIT_REACHED`, `ANCHOR_REVERTED`, `RPC_ERROR`, `INDEX_ERROR` | Retry later; for a lookup, the check could not be made |
| 503 | `POOL_EXHAUSTED`, `SERVER_UNAVAILABLE` | Retry later |
| 504 | `CONFIRMATION_TIMEOUT` | Send the identical request again |

## SDKs

The SDKs hash, check and send requests for you:

- .NET: [tracing-sdk-dotnet](https://github.com/OneMatrixL1/tracing-sdk-dotnet)
- PHP: [tracing-sdk-php](https://github.com/OneMatrixL1/tracing-sdk-php)
