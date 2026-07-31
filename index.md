# Index Protocol

![reliable](https://img.shields.io/badge/status-reliable-green.svg?style=flat-square)

## Editors

- [Alan Shaw](https://github.com/alanshaw)

## Abstract

The indexing protocol allows authorized agents to submit verifiable claims about content-addressable data to be published on the [InterPlanetary Network Indexer (IPNI)][IPNI], making it publicly queryable.

## Language

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in [RFC2119](https://datatracker.ietf.org/doc/html/rfc2119).

# Introduction

Content stored with the [blob protocol] is opaque to the network: a blob is a byte array identified by its [multihash]. To make the IPLD blocks _within_ blobs discoverable and retrievable, an agent builds an [index] describing which block digests live at which byte ranges of which blobs, stores the index in their space, and registers it with the upload service. The upload service publishes a corresponding claim to an indexer, which fetches and verifies the index and publishes its records to [IPNI].

## Concepts

### Roles

| Name           | Description                                                                                         |
| -------------- | --------------------------------------------------------------------------------------------------- |
| Agent          | A [principal] identified by [`did:key`] identifier, representing a user in an application.          |
| Upload Service | A service [principal] that verifies index registrations and forwards index claims to the indexer.   |
| Indexer        | A service [principal] that caches content claims, verifies indexes and publishes records to [IPNI]. |

### Space

A namespace, often referred as a "space", is an owned resource that can be shared. It corresponds to a unique asymmetric cryptographic keypair and is identified by a [`did:key`] URI.

### Index

An [index] is a [content archive][CAR] describing the IPLD blocks contained within one or more blobs, in terms of blob slices — byte ranges of a blob addressed by the [multihash] of the bytes within the range. See [Index Format].

### Retrieval Authorization

Both capabilities in this protocol carry a _retrieval authorization_ in the invocation `meta` field: a `retrievalAuth` list of [delegation][UCAN delegation] links authorizing the audience to invoke [`/content/retrieve`][content retrieve] to fetch the index blob from the network.

The list MUST contain the delegation chain in invocation order: the root delegation — issued by the space, whose subject identifies the space — first, and the leaf delegation, whose audience is the invocation audience, last. The linked delegations themselves MUST be transmitted in the request [container][UCAN container]: the `meta` field carries only links.

It is RECOMMENDED that retrieval authorization delegations are constrained by [policy][UCAN delegation] to the digest of the index blob.

```ipldsch
# Transmitted in the invocation meta field.
type IndexMetadata struct {
  retrievalAuth [Link] # /content/retrieve delegation chain, root first
}
```

<details>
<summary>Go syntax</summary>

```go
type IndexMetadata struct {
	RetrievalAuth []cid.Cid `cborgen:"retrievalAuth"`
}
```

</details>

### Common Types

Schemas in this document are described using [IPLD Schema] notation, accompanied by equivalent Go types whose `cborgen` tags define the wire keys. Failure values follow the `{name, message}` convention of the [receipt] spec.

# Capabilities

## Add Index

An authorized agent MAY invoke the `/index/add` capability on the [space] subject to register an [index] with the upload service and have it published to the network.

The index MUST first be stored in the space as a blob using [Add Blob]. The `/index/add` invocation is then made with a link to the stored index.

### Add Index Invocation

#### Add Index Arguments Schema

```ipldsch
type AddArguments struct {
  index Link # content archive (CAR) containing the index
}
```

<details>
<summary>Go syntax</summary>

```go
type AddArguments struct {
	Index cid.Cid `cborgen:"index"`
}
```

</details>

The `args.index` field MUST be a link to the [content archive][CAR] containing the [index] — a CIDv1 with the `car` codec wrapping the [multihash] of the archive bytes (the digest under which the archive was stored as a blob).

The invocation `meta` field MUST contain a [retrieval authorization] allowing the upload service to retrieve the index blob, so that it may be fetched, verified and cached by the network.

#### Add Index Invocation Example

> ℹ️ Note: examples show the invocation payload; the enclosing signed envelope is elided. We use `// "/": "bafy.."` comments to denote the [task][UCAN task] link of the invocation.

```jsonc
{ // "/": "bafy..indexAdd"
  "iss": "did:key:zAlice",
  "aud": "did:web:upload.example.com",
  "sub": "did:key:zAliceSpace",
  "cmd": "/index/add",
  "args": {
    // content archive (CAR) containing the index
    "index": { "/": "bag..index" }
  },
  "meta": {
    // delegation chain authorizing retrieval of the index blob,
    // delegation blocks transmitted in the request container
    "retrievalAuth": [
      { "/": "bafy..dlgSpaceRetrieve" },
      { "/": "bafy..dlgServiceRetrieve" }
    ]
  },
  "prf": [{ "/": "bafy..dlgAliceSpace" }],
  "nonce": { "/": { "bytes": "aW5kZXg" } },
  "exp": 1735689600
}
```

### Add Index Receipt

Invocation MUST fail if any of the following is true:

1. Provided subject space is not provisioned with a provider _(error name `InsufficientStorage`)_.
1. Provided `index` is not stored in the subject space _(error name `IndexNotFound`)_.

Invocation MUST succeed if none of the above is true. Executing the invocation, the upload service MUST publish an [Assert Index] claim for the index to the indexer, re-delegating retrieval authority for the index blob to the indexer.

The success value is an empty object.

#### Add Index Receipt Schema

```ipldsch
type AddResult union {
  | AddOK "ok"
  | Error "error"
} representation keyed

type AddOK struct {}
```

<details>
<summary>Go syntax</summary>

```go
type AddOK struct{}
```

</details>

## Assert Index

The upload service MAY invoke the `/assert/index` capability to claim that a content graph can be found in the blob(s) that are identified and indexed by the linked [index]. The claim is issued by the upload service on its own subject and addressed to the indexer.

### Assert Index Invocation

#### Assert Index Arguments Schema

```ipldsch
type IndexArguments struct {
  index Link # content archive (CAR) containing the index
}
```

<details>
<summary>Go syntax</summary>

```go
type IndexArguments struct {
	Index cid.Cid `cborgen:"index"`
}
```

</details>

The `args.index` field MUST be a link to the [content archive][CAR] containing the [index].

The invocation `meta` field MUST contain a [retrieval authorization] allowing the indexer to retrieve the index blob.

#### Assert Index Invocation Example

```jsonc
{ // "/": "bafy..assertIndex"
  "iss": "did:web:upload.example.com",
  "aud": "did:web:indexer.example.com",
  "sub": "did:web:upload.example.com",
  "cmd": "/assert/index",
  "args": {
    // content archive (CAR) containing the index
    "index": { "/": "bag..index" }
  },
  "meta": {
    // delegation chain authorizing retrieval of the index blob,
    // delegation blocks transmitted in the request container
    "retrievalAuth": [
      { "/": "bafy..dlgSpaceRetrieve" },
      { "/": "bafy..dlgServiceRetrieve" },
      { "/": "bafy..dlgIndexerRetrieve" }
    ]
  },
  "prf": [],
  "nonce": { "/": { "bytes": "YXNzZXJ0" } },
  "exp": 1735689600
}
```

### Assert Index Receipt

Executing the invocation, the indexer MUST:

1. Cache the claim.
2. Locate the index blob via its [location commitment]s.
3. Retrieve the index blob using [`/content/retrieve`][content retrieve], authorized by the [retrieval authorization].
4. Decode and verify the [index].
5. Publish a record to [IPNI] for every slice digest in the index, associating it with the claim.

Invocation MUST fail if the index blob cannot be located, retrieved or decoded.

The success value is an empty object.

#### Assert Index Receipt Schema

```ipldsch
type IndexResult union {
  | IndexOK "ok"
  | Error   "error"
} representation keyed

type IndexOK struct {}
```

<details>
<summary>Go syntax</summary>

```go
type IndexOK struct{}
```

</details>

# Index Format

## Index Schema

The index schema is a variant type keyed by a format descriptor label, designed to allow format evolution through versioning and additional schema variants.

```ipldsch
type Index union {
  | ShardedDagIndex "index/sharded/dag@0.1"
} representation keyed
```

## Sharded DAG Index

A sharded DAG index SHOULD describe the complete set of blocks that make up a content DAG in terms of blob slices.

### Sharded DAG Index Schema

```ipldsch
type ShardedDagIndex struct {
  shards [Link] # links to BlobIndex blocks
}

# Index of the slices within a single blob.
# Encoded as the tuple [digest, slices].
type BlobIndex struct {
  digest Bytes # multihash digest of the blob
  slices [BlobSlice]
} representation tuple

# A byte range of a blob addressed by the multihash of the bytes within it.
# Encoded as the tuple [digest, range].
type BlobSlice struct {
  digest Bytes # multihash digest of the slice
  range  Range
} representation tuple

# Encoded as the tuple [start, end].
type Range struct {
  start Int # offset of the first byte (inclusive)
  end   Int # offset of the last byte (inclusive)
} representation tuple
```

<details>
<summary>Go syntax</summary>

```go
type ShardedDagIndexModel struct {
	DagO_1 *ShardedDagIndexModel_0_1 `cborgen:"index/sharded/dag@0.1,omitempty"`
}

type ShardedDagIndexModel_0_1 struct {
	Shards []cid.Cid `cborgen:"shards"`
}

// Encoded as the tuple [digest, slices].
type BlobIndexModel struct {
	Digest multihash.Multihash
	Slices []BlobSliceModel
}

// Encoded as the tuple [digest, range].
type BlobSliceModel struct {
	Digest multihash.Multihash
	Range  RangeModel
}

// Encoded as the tuple [start, end]. Both offsets are inclusive.
type RangeModel struct {
	Start int64
	End   int64
}
```

</details>

The `shards` field is a list of links to `BlobIndex` blocks, allowing blob indexes to be bundled in the same archive or externalized by linking to them. It is RECOMMENDED to bundle all the `BlobIndex` blocks inside the [content archive][CAR] of the index.

It is RECOMMENDED to include a `BlobSlice` in `slices` that spans the full range of the blob to make it available. On the flip side this creates a choice to share only a partial index of the blob when so desired.

### Sharded DAG Index Example

The root block of the index:

```jsonc
{
  "index/sharded/dag@0.1": {
    "shards": [
      { "/": "bafy..blobIndexLeft" },
      { "/": "bafy..blobIndexRight" }
    ]
  }
}
```

A linked `BlobIndex` block (`bafy..blobIndexLeft`):

```jsonc
[
  // blob multihash
  { "/": { "bytes": "blb...left" } },
  // slices within the blob
  [
    [{ "/": { "bytes": "block..1" } }, [0, 127]],
    [{ "/": { "bytes": "block..2" } }, [128, 255]],
    [{ "/": { "bytes": "block..3" } }, [256, 383]],
    [{ "/": { "bytes": "block..4" } }, [384, 511]]
  ]
]
```

## Index Archive

An index is serialized as a [content archive][CAR]:

- The archive MUST contain exactly one root: the block containing the `Index` variant.
- The root block MUST be encoded as DAG-CBOR.
- Blocks MUST be addressed by CIDv1 with the `dag-cbor` codec and SHA2-256 multihash.
- For deterministic encoding, slices within a `BlobIndex` and shards within the archive SHOULD be sorted by decoded digest bytes, and the root block SHOULD be written last.

The archive is stored as a blob in the space, and referenced from [Add Index] and [Assert Index] by a CIDv1 with the `car` codec wrapping the [multihash] of the archive bytes.

[blob protocol]:./blob.md
[space]:./blob.md#space
[Add Blob]:./blob.md#add-blob
[location commitment]:./blob.md#location-commitment
[content retrieve]:./retrieval.md#content-retrieve
[receipt]:./ucan.md#receipt
[index]:#index-format
[Index Format]:#index-format
[retrieval authorization]:#retrieval-authorization
[Add Index]:#add-index
[Assert Index]:#assert-index
[IPNI]:https://github.com/ipni/specs/blob/main/IPNI.md
[CAR]:https://ipld.io/specs/transport/car/
[multihash]:https://github.com/multiformats/multihash
[IPLD Schema]:https://ipld.io/docs/schemas/
[`did:key`]:https://w3c-ccg.github.io/did-key-spec/
[principal]:https://github.com/ucan-wg/spec#principals
[UCAN delegation]:https://github.com/ucan-wg/delegation
[UCAN task]:https://github.com/ucan-wg/invocation#task
[UCAN container]:https://github.com/ucan-wg/container
