# Revocation Checking

![draft](https://img.shields.io/badge/status-draft-yellow.svg?style=flat-square)

## Authors

- [Alan Shaw](https://github.com/alanshaw)

## Language

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in [RFC 2119](https://datatracker.ietf.org/doc/html/rfc2119).

## Abstract

This specification defines an HTTP API for checking [UCAN] delegations for revocation. It allows a verifier to look up the revocation status of an individual delegation, and to subscribe to a stream of revocations in order to maintain a local revocation set.

# Introduction

[UCAN revocation] defines how a delegation is revoked: an issuer in the delegation chain invokes `/ucan/revoke`, identifying the revoked delegation and providing a [path witness][UCAN revocation path witness] — the delegation chain proving the revoker's place in it. What that specification leaves open is how verifiers _discover_ revocations.

This specification defines the read API of a revocation service: a service [principal] that accepts `/ucan/revoke` invocations and serves the resulting revocation records over HTTP, both individually and as a stream.

## Concepts

### Roles

| Name               | Description                                                                                   |
| ------------------ | ------------------------------------------------------------------------------------------------ |
| Revocation Service | A service [principal] that accepts revocations and serves revocation records.                  |
| Revoker            | A [principal] revoking a delegation — an issuer in the revoked delegation's chain.             |
| Verifier           | A [principal] validating UCANs, which checks the delegations in proof chains for revocation.  |

### Revocation Record

A revocation record associates a revoked delegation with the `/ucan/revoke` invocation that revoked it (its _cause_) and the [path witness][UCAN revocation path witness] delegations proving the revoker was an issuer in the delegation's chain.

A record's `recorded_at` field is the time the record was recorded by the revocation service — not necessarily the time the revocation was issued.

### Schema Notation

Records in this document are described using [IPLD Schema] notation, accompanied by equivalent Go types whose `dagjsongen` tags define the wire keys. Records are encoded as [DAG-JSON].

# Publishing Revocations

Revocations are published by sending a `/ucan/revoke` invocation to the revocation service's UCAN RPC endpoint, as described by the [UCAN revocation] specification. The invocation arguments identify the revoked delegation and its [path witness][UCAN revocation path witness]:

```ipldsch
type RevokeArguments struct {
  revoke Link   # the delegation to revoke
  path   [Link] # path witness: delegation chain from the root to the revoked delegation
}
```

<details>
<summary>Go syntax</summary>

```go
type RevokeArguments struct {
	Revoke cid.Cid   `cborgen:"revoke"`
	Path   []cid.Cid `cborgen:"path"`
}
```

</details>

The delegations linked from `path` MUST be transmitted in the request [container][UCAN container]. The revocation service MUST verify that:

1. Every delegation linked from `path` is present in the request container.
2. The final element of `path` is the revoked delegation (`revoke`).
3. The root delegation of `path` is self-issued (its subject equals its issuer), each subsequent delegation's issuer matches the previous delegation's audience, and every delegation shares the root's subject (or has no subject).
4. Every delegation in `path` is valid.
5. The invocation issuer is an issuer of one of the delegations in `path`.

If verification succeeds the service MUST persist a [revocation record] and make it available via the API defined below.

# HTTP API

## Get Revocation

```
GET /revocation/{cid}
```

Retrieves the most recent [revocation record] for a delegation. `{cid}` is the string encoded CID of the delegation to check.

If a revocation exists for the delegation, the service MUST respond with HTTP status `200` and a [DAG-JSON] encoded [revocation record] body with `Content-Type: application/vnd.ipld.dag-json`. Since revocations are permanent, successful responses MAY be served with a long-lived, immutable cache policy (e.g. `Cache-Control: public, max-age=31536000, immutable`).

If no revocation exists for the delegation, the service MUST respond with HTTP status `404`. Not found responses MUST NOT be cached as immutable — the delegation may be revoked later.

If `{cid}` is not a valid CID, the service MUST respond with HTTP status `400`.

### Get Revocation Record Schema

```ipldsch
type Revocation struct {
  revoke      Link    # CID of the revoked delegation
  path        [Bytes] # encoded path witness delegations, root first
  cause       Bytes   # encoded /ucan/revoke invocation that revoked the delegation
  recorded_at String  # RFC3339 time the record was recorded
}
```

<details>
<summary>Go syntax</summary>

```go
type Revocation struct {
	Revoke     cid.Cid         `dagjsongen:"revoke"`
	Path       [][]byte        `dagjsongen:"path"`
	Cause      []byte          `dagjsongen:"cause"`
	RecordedAt jsg.DagJsonTime `dagjsongen:"recorded_at"`
}
```

</details>

The `path` and `cause` fields carry the complete DAG-CBOR encoded [envelope][UCAN envelope]s of the witness delegations and the revocation invocation respectively — the record is self-contained and verifiable. A [verifier] SHOULD verify that the `cause` is a valid `/ucan/revoke` invocation whose `revoke` argument matches the record, and that the path witness proves the revocation issuer was an issuer in the delegation's chain.

### Get Revocation Example

```json
{
  "revoke": { "/": "bafyreiehytyi4q3t2amvf2abdlt5xnnqtaqkknf6yxhre4klpjnejlnsc4" },
  "cause": { "/": { "bytes": "omF2AWNjYXBnL3VjYW4vcmV2b2tl" } },
  "path": [
    { "/": { "bytes": "omF2AWNjYXBsL3Rlc3QvaW52b2tl" } }
  ],
  "recorded_at": "2026-07-17T09:00:00Z"
}
```

## Revocation Firehose

```
GET /revocations/{since}
```

A [Server-Sent Events][SSE] stream of compact [revocation record]s, allowing a [verifier] to maintain a local revocation set without polling. `{since}` is either `0`, to stream all stored records, or an [RFC3339]/RFC3339Nano timestamp cursor, to stream records created after that time.

The service MUST respond with `Content-Type: text/event-stream` and `Cache-Control: no-cache`. The stream MUST first deliver stored records matching the cursor, then remain open and deliver new records as they arrive.

If `{since}` is neither `0` nor a valid RFC3339 timestamp, the service MUST respond with HTTP status `400`.

Each revocation is delivered as an event with:

- `id` — the CID of the revocation invocation (the record's `cause`).
- `event` — the literal string `revocation`.
- `data` — a compact [DAG-JSON] encoded record, per the schema below.

Consumers SHOULD use the `recorded_at` of the last received record as the cursor when reconnecting.

If the service encounters an error while streaming it SHOULD emit a final event with `event` set to the literal string `error` and `data` set to a JSON object with an `error` message field, then end the stream.

### Firehose Record Schema

```ipldsch
type FirehoseRevocation struct {
  revoke      Link   # CID of the revoked delegation
  path        [Link] # CIDs of the path witness delegations, root first
  cause       Link   # CID of the /ucan/revoke invocation
  recorded_at String # RFC3339 time the record was recorded
}
```

<details>
<summary>Go syntax</summary>

```go
type FirehoseRevocation struct {
	Revoke     cid.Cid         `dagjsongen:"revoke"`
	Path       []cid.Cid       `dagjsongen:"path"`
	Cause      cid.Cid         `dagjsongen:"cause"`
	RecordedAt jsg.DagJsonTime `dagjsongen:"recorded_at"`
}
```

</details>

The compact record carries links only. The full witness and cause blocks for a streamed revocation MAY be obtained from [Get Revocation].

### Firehose Example

```txt
id: bafyreif5fzax7oygfafacvxq2ndhtkshz2av5m42hqeixea7giirdxe5dm
event: revocation
data: {"revoke":{"/":"bafyreiehytyi4q3t2amvf2abdlt5xnnqtaqkknf6yxhre4klpjnejlnsc4"},"path":[{"/":"bafyreiehytyi4q3t2amvf2abdlt5xnnqtaqkknf6yxhre4klpjnejlnsc4"}],"cause":{"/":"bafyreif5fzax7oygfafacvxq2ndhtkshz2av5m42hqeixea7giirdxe5dm"},"recorded_at":"2026-07-17T09:00:00Z"}
```

# Verifier Guidance

A [verifier] validating an invocation SHOULD check every delegation in its proof chain against a revocation service, either by looking up each delegation with [Get Revocation] or by maintaining a local revocation set fed by the [Revocation Firehose].

Revocations are permanent: a revocation cannot be un-revoked, and a verifier MAY treat a revocation record as valid indefinitely. A revocation record for a delegation MAY be evicted from a local set once the revoked delegation's own expiry (plus a clock skew allowance) has passed, since the delegation can no longer be used regardless.

[UCAN]: https://github.com/ucan-wg/spec
[UCAN revocation]: https://github.com/ucan-wg/revocation
[UCAN revocation path witness]: https://github.com/ucan-wg/revocation#path-witness
[UCAN container]: https://github.com/ucan-wg/container
[UCAN envelope]: https://github.com/ucan-wg/spec#envelope
[principal]: https://github.com/ucan-wg/spec#principals
[IPLD Schema]: https://ipld.io/docs/schemas/
[DAG-JSON]: https://ipld.io/docs/codecs/known/dag-json/
[SSE]: https://developer.mozilla.org/en-US/docs/Web/API/Server-sent_events
[RFC3339]: https://www.rfc-editor.org/rfc/rfc3339
[revocation record]: #revocation-record
[verifier]: #roles
[Get Revocation]: #get-revocation
[Revocation Firehose]: #revocation-firehose
