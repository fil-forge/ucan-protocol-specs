# PDP Protocol

![reliable](https://img.shields.io/badge/status-reliable-green.svg?style=flat-square)

## Authors

- [Hannah Howard](https://github.com/hannahhoward)

## Editors

- [Alan Shaw](https://github.com/alanshaw)

## Abstract

The proof of data possession (PDP) protocol allows a storage node to prove possession of stored blobs, and a verifier to confirm the inclusion of blobs within aggregated storage proofs, using cryptographic merkle tree commitments.

## Language

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in [RFC2119](https://datatracker.ietf.org/doc/html/rfc2119).

# Introduction

A storage node aggregates the blobs it accepts into larger composite pieces and maintains cryptographic merkle tree proofs demonstrating the inclusion of each blob within its aggregate. Blobs are identified by their content [multihash] to enable content-addressed verification. When a blob is accepted into an aggregate, the storage node generates an [inclusion proof] proving the blob's position within the aggregate.

This enables verification that a specific blob is stored and properly indexed, and provides the cryptographic proofs necessary for auditing the storage node's storage claims.

## Concepts

### Roles

There are several distinct roles that [principal]s may assume in this specification:

| Name         | Description                                                                                                          |
| ------------ | -------------------------------------------------------------------------------------------------------------------- |
| Principal    | The general class of entities that interact with a UCAN. Identified by a DID that can be used in the `iss`, `aud` or `sub` field of a UCAN. |
| Storage Node | A [principal] that accepts blobs for aggregation and computes merkle tree inclusion proofs for blobs within its aggregates. |
| Verifier     | A [principal] that queries a storage node to verify blob possession and retrieve [inclusion proof]s — for example the upload service, or an auditor. |

### Piece

A piece is the commitment representation of a blob: the root of a merkle tree computed over the (padded) blob bytes. A piece is identified by a piece link — a CIDv1 with the `raw` codec wrapping a [`fr32-sha256-trunc254-padded-binary-tree`][multicodec] multihash, which encodes the tree height and padding alongside the 32 byte root digest.

### Aggregate

An aggregate is a larger composite piece built from accepted blob pieces, as described by the [Filecoin data segment][data segment] specification. The storage node periodically proves possession of its aggregates; how those proofs are anchored (e.g. on-chain) is out of scope of this specification.

### Inclusion Proof

An inclusion proof is a merkle tree proof — a path of sibling digests and a leaf index — demonstrating that a piece is included at a specific position within an aggregate.

### UCAN Primitives

This specification builds on the [UCAN] suite of specifications, in particular [invocation][UCAN invocation]s, [task][UCAN task]s, [receipt]s, [promise][UCAN promise]s and [container][UCAN container]s. These are introduced in the [blob protocol][blob protocol concepts] and the wire format of receipts is defined in the [UCAN extensions spec][receipt].

### Common Types

Schemas in this document are described using [IPLD Schema] notation, accompanied by equivalent Go types whose `cborgen` tags define the wire keys. Multihashes are encoded as raw bytes. The following types are shared by several capabilities:

```ipldsch
# Merkle tree inclusion proof, encoded as the tuple [index, path].
type ProofData struct {
  index Int     # index of the proven leaf within its level (leftmost is 0)
  path  [Bytes] # sibling node digests (32 bytes each), leaf to root
} representation tuple

# Convention for failure values, see the receipt spec.
type Error struct {
  name    String # machine readable error name
  message String # human readable description
}
```

<details>
<summary>Go syntax</summary>

```go
// ProofData is merkletree.ProofData from
// github.com/filecoin-project/go-data-segment.
// It marshals as the CBOR tuple [index, path].
type ProofData struct {
	Path  []Node // sibling node digests, leaf to root
	Index uint64 // index of the proven leaf within its level
}

// Node is a 32 byte merkle tree node digest.
type Node [32]byte

type Error struct {
	Name    string `cborgen:"name"`
	Message string `cborgen:"message"`
}
```

</details>

# Capabilities

## PDP Accept

The `/pdp/accept` task represents the acceptance of a blob into an aggregate. Its receipt carries the [inclusion proof].

The storage node issues this invocation itself when it accepts a blob for storage: the issuer, audience and subject are all the storage node. It is a placeholder for work the storage node performs asynchronously — the invocation is created when the blob is accepted, and its receipt is issued later, once the blob has been aggregated.

In the [blob protocol], the [Accept Blob][accept blob] result's `pdp` field is an [await][UCAN promise] on this task, and the invocation is transmitted in the [container][UCAN container] alongside the accept blob receipt.

### PDP Accept Invocation

The invocation MUST be issued without a `nonce` _(zero length byte array)_, making the task idempotent: its [task][UCAN task] link is deterministic and can be independently derived by any party from the blob multihash. This allows the receipt to be issued and looked up without coordination once aggregation completes. The invocation SHOULD NOT expire, since the time to aggregation is unbounded.

#### PDP Accept Arguments Schema

```ipldsch
type AcceptArguments struct {
  blob Bytes # multihash digest of the accepted blob
}
```

<details>
<summary>Go syntax</summary>

```go
type AcceptArguments struct {
	Blob multihash.Multihash `cborgen:"blob"`
}
```

</details>

The `args.blob` field MUST be a [multihash] digest of the blob payload bytes.

#### PDP Accept Invocation Example

> ℹ️ Note: examples show the invocation payload; the enclosing signed envelope is elided. We use `// "/": "bafy.."` comments to denote the [task][UCAN task] link of the invocation.

```jsonc
{ // "/": "bafy..pdp"
  "iss": "did:key:zStorageNode",
  "aud": "did:key:zStorageNode",
  "sub": "did:key:zStorageNode",
  "cmd": "/pdp/accept",
  "args": {
    // multihash of the blob as byte array
    "blob": { "/": { "bytes": "mEi...sfKg" } }
  },
  "prf": [],
  "nonce": { "/": { "bytes": "" } },
  // does not expire
  "exp": null
}
```

### PDP Accept Receipt

The receipt MUST be issued once the blob has been included in an aggregate. Until then no receipt exists — a [Verifier] MAY query the status of a pending aggregation with [PDP Info].

#### PDP Accept Receipt Schema

```ipldsch
type AcceptResult union {
  | AcceptOK "ok"
  | Error    "error"
} representation keyed

type AcceptOK struct {
  piece          Link      # piece representation of the blob
  aggregate      Link      # aggregate piece containing the blob
  inclusionProof ProofData # proof of the piece's inclusion in the aggregate
}
```

<details>
<summary>Go syntax</summary>

```go
type AcceptOK struct {
	Aggregate      cid.Cid              `cborgen:"aggregate"`
	InclusionProof merkletree.ProofData `cborgen:"inclusionProof"`
	Piece          cid.Cid              `cborgen:"piece"`
}
```

</details>

The `out.ok.piece` field MUST be set to the [piece] link of the blob.

The `out.ok.aggregate` field MUST be set to the [piece] link of the [aggregate] containing the blob.

The `out.ok.inclusionProof` field MUST be set to the merkle tree proof of the piece's position within the aggregate.

#### PDP Accept Receipt Example

Shows an example receipt for the above `/pdp/accept` invocation. A [receipt] is itself an invocation of `/ucan/assert/receipt` issued by the executor. Receipt examples show only the salient payload fields.

```jsonc
{
  "iss": "did:key:zStorageNode",
  "aud": "did:key:zStorageNode",
  "sub": "did:key:zStorageNode",
  "cmd": "/ucan/assert/receipt",
  "args": {
    // the /pdp/accept task this receipt is for
    "ran": { "/": "bafy..pdp" },
    "out": {
      "ok": {
        // piece representation of the blob
        "piece": { "/": "bafk..piece" },
        // aggregate containing this blob
        "aggregate": { "/": "bafk..aggregate" },
        // merkle tree inclusion proof: [index, path]
        "inclusionProof": [
          3,
          [
            { "/": { "bytes": "mNo...de0" } },
            { "/": { "bytes": "mNo...de1" } },
            { "/": { "bytes": "mNo...de2" } }
          ]
        ]
      }
    }
  },
  "iat": 1735689500
}
```

## PDP Info

A [Verifier] MAY invoke the `/pdp/info` capability on a storage node subject to retrieve information about a blob's aggregates and [inclusion proof]s.

The storage node is the subject of this capability: a storage node delegates it to the upload service when it joins the network, and MAY delegate it to other verifiers. The information allows the [Verifier] to:

1. Confirm the blob is stored by the storage node.
2. Retrieve cryptographic proofs of inclusion for auditing purposes.
3. Track which aggregates contain the blob across time, as aggregates may change.

### PDP Info Invocation

#### PDP Info Arguments Schema

```ipldsch
type InfoArguments struct {
  blob Bytes # multihash digest of the blob to query
}
```

<details>
<summary>Go syntax</summary>

```go
type InfoArguments struct {
	Blob multihash.Multihash `cborgen:"blob"`
}
```

</details>

The `args.blob` field MUST be a [multihash] digest of the blob payload bytes.

#### PDP Info Invocation Example

```jsonc
{ // "/": "bafy..info"
  "iss": "did:web:upload.example.com",
  "aud": "did:key:zStorageNode",
  "sub": "did:key:zStorageNode",
  "cmd": "/pdp/info",
  "args": {
    // multihash of the blob as byte array
    "blob": { "/": { "bytes": "mEi...sfKg" } }
  },
  "prf": [{ "/": "bafy..dlgStorageNode" }],
  "nonce": { "/": { "bytes": "aW5mbw" } },
  "exp": 1735689600
}
```

### PDP Info Receipt

Invocation MUST fail if the blob is not known to the storage node. Invocation MUST fail if the [PDP Accept] task for the blob produced a failure receipt _(error name `PDPAcceptFailed`)_. Invocation MUST fail if the piece resolved for the blob does not match the piece recorded in the [PDP Accept] receipt — a hard invariant violation _(error name `PieceMismatch`)_.

If the blob is known to the storage node but is still pending aggregation, the invocation MUST succeed with the canonical [piece] link of the blob and an empty `aggregates` list.

Otherwise the success value MUST list every [aggregate] containing the blob, each with the [inclusion proof] for the blob within that aggregate.

#### PDP Info Receipt Schema

```ipldsch
type InfoResult union {
  | InfoOK "ok"
  | Error  "error"
} representation keyed

type InfoOK struct {
  piece      Link # piece representation of the blob
  aggregates [AcceptedAggregate]
}

type AcceptedAggregate struct {
  aggregate      Link      # aggregate piece containing the blob
  inclusionProof ProofData # proof of the piece's inclusion in the aggregate
}
```

<details>
<summary>Go syntax</summary>

```go
type InfoOK struct {
	Piece      cid.Cid                 `cborgen:"piece"`
	Aggregates []InfoAcceptedAggregate `cborgen:"aggregates"`
}

type InfoAcceptedAggregate struct {
	Aggregate      cid.Cid              `cborgen:"aggregate"`
	InclusionProof merkletree.ProofData `cborgen:"inclusionProof"`
}
```

</details>

#### PDP Info Receipt Example

An example receipt for the above `/pdp/info` invocation:

```jsonc
{
  "iss": "did:key:zStorageNode",
  "aud": "did:key:zStorageNode",
  "sub": "did:key:zStorageNode",
  "cmd": "/ucan/assert/receipt",
  "args": {
    // the /pdp/info task this receipt is for
    "ran": { "/": "bafy..info" },
    "out": {
      "ok": {
        // piece representation of the blob
        "piece": { "/": "bafk..piece" },
        // aggregates containing this blob
        "aggregates": [
          {
            // aggregate piece identifier
            "aggregate": { "/": "bafk..aggregate" },
            // merkle tree inclusion proof: [index, path]
            "inclusionProof": [
              3,
              [
                { "/": { "bytes": "mNo...de0" } },
                { "/": { "bytes": "mNo...de1" } },
                { "/": { "bytes": "mNo...de2" } }
              ]
            ]
          }
        ]
      }
    }
  },
  "iat": 1735689500
}
```

[multihash]:https://github.com/multiformats/multihash
[multicodec]:https://github.com/multiformats/multicodec/blob/master/table.csv
[IPLD Schema]:https://ipld.io/docs/schemas/
[data segment]:https://github.com/filecoin-project/FIPs/blob/master/FRCs/frc-0058.md
[piece]:#piece
[aggregate]:#aggregate
[inclusion proof]:#inclusion-proof
[Verifier]:#roles
[PDP Accept]:#pdp-accept
[PDP Info]:#pdp-info
[blob protocol]:./blob.md
[blob protocol concepts]:./blob.md#ucan-primitives
[accept blob]:./blob.md#accept-blob
[receipt]:./ucan.md#receipt
[principal]:https://github.com/ucan-wg/spec#principals
[UCAN]:https://github.com/ucan-wg/spec
[UCAN invocation]:https://github.com/ucan-wg/invocation
[UCAN task]:https://github.com/ucan-wg/invocation#task
[UCAN promise]:https://github.com/ucan-wg/promise
[UCAN container]:https://github.com/ucan-wg/container
