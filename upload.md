# Upload Protocol

![reliable](https://img.shields.io/badge/status-reliable-green.svg?style=flat-square)

## Authors

- [Irakli Gozalishvili](https://github.com/gozala)
- [Yusef Napora](https://github.com/yusefnapora), [DAG House](https://dag.house)

## Editors

- [Alan Shaw](https://github.com/alanshaw)

## Abstract

The upload protocol allows authorized agents to manage the list of top level content entries — "uploads" — in a space. An upload maps a content root to the shards that contain it.

## Language

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in [RFC2119](https://datatracker.ietf.org/doc/html/rfc2119).

# Introduction

Capabilities under the `/upload` namespace are used to manage the list of top level content entries in a space. While not required, it is generally assumed that user content like files is turned into a [UnixFS] DAG, packed into one or more [content archive][CAR]s and stored in the space using the [blob protocol]. The root of the DAG is then added to the upload list.

## Concepts

### Roles

| Name           | Description                                                                                |
| -------------- | --------------------------------------------------------------------------------------------- |
| Agent          | A [principal] identified by [`did:key`] identifier, representing a user in an application.  |
| Upload Service | A service [principal] that maintains the list of uploads for each space.                    |

### Space

A namespace, often referred as a "space", is an owned resource that can be shared. It corresponds to a unique asymmetric cryptographic keypair and is identified by a [`did:key`] URI.

### Upload

An upload is a content entry in a space: a root link, the list of shards — [content archive][CAR]s stored as blobs — that contain the content DAG, and optionally a link to the [index] describing the blocks within the shards.

### Common Types

Schemas in this document are described using [IPLD Schema] notation, accompanied by equivalent Go types whose `cborgen` tags define the wire keys. Failure values follow the `{name, message}` convention of the [receipt] spec.

# Capabilities

## Add Upload

An authorized agent MAY invoke the `/upload/add` capability on the [space] subject to include content identified by `args.root` in the list of content entries for the space.

It is expected that the [content archive][CAR]s containing the content are stored in the space using [Add Blob]. The upload service MAY enforce this invariant by failing the invocation, or MAY choose to succeed the invocation but fail to serve the content when requested.

Invoking `/upload/add` again with the same `root` updates the entry: the `shards` of all invocations for the root are merged into a union, and the `index` is replaced with the most recently provided value.

### Add Upload Invocation

#### Add Upload Arguments Schema

```ipldsch
type AddArguments struct {
  root   Link            # root of the content entry
  shards [Link]          # content archives (CARs) containing the content DAG
  index  optional Link   # content archive (CAR) containing the index of the upload
}
```

<details>
<summary>Go syntax</summary>

```go
type AddArguments struct {
	Root   cid.Cid   `cborgen:"root"`
	Shards []cid.Cid `cborgen:"shards"`
	Index  *cid.Cid  `cborgen:"index,omitempty"`
}
```

</details>

The `args.root` field MUST be set to the [IPLD link] of the desired content entry.

The `args.shards` field MUST be set to the list of [IPLD link]s of the [content archive][CAR]s containing the content DAG.

The optional `args.index` field MAY be set to the link of the [content archive][CAR] containing the [index] of the upload, registered with [Add Index].

#### Add Upload Invocation Example

> ℹ️ Note: examples show the invocation payload; the enclosing signed envelope is elided. We use `// "/": "bafy.."` comments to denote the [task][UCAN task] link of the invocation.

```jsonc
{ // "/": "bafy..uploadAdd"
  "iss": "did:key:zAlice",
  "aud": "did:web:upload.example.com",
  "sub": "did:key:zAliceSpace",
  "cmd": "/upload/add",
  "args": {
    // root of the content entry
    "root": { "/": "bafy..root" },
    // content archives (CARs) containing the content DAG
    "shards": [
      { "/": "bag..shard1" },
      { "/": "bag..shard2" }
    ],
    // content archive (CAR) containing the index of the upload
    "index": { "/": "bag..index" }
  },
  "prf": [{ "/": "bafy..dlgAliceSpace" }],
  "nonce": { "/": { "bytes": "dXBsb2Fk" } },
  "exp": 1735689600
}
```

### Add Upload Receipt

Invocation MUST fail if the subject space is not provisioned with a provider _(error name `InsufficientStorage`)_.

Invocation MUST succeed otherwise. The success value is an empty object.

#### Add Upload Receipt Schema

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

## List Uploads

An authorized agent MAY invoke the `/upload/list` capability on the [space] subject to list content entries in the space at the time of invocation.

Results contain the root and index links only. Since the shard list for an upload can be very large, shards are listed separately — and paginated — with [List Upload Shards].

### List Uploads Invocation

#### List Uploads Arguments Schema

```ipldsch
type ListArguments struct {
  cursor optional String # cursor to start listing from
  size   optional Int    # desired page size
}
```

<details>
<summary>Go syntax</summary>

```go
type ListArguments struct {
	Cursor *string `cborgen:"cursor,omitempty"`
	Size   *uint64 `cborgen:"size,omitempty"`
}
```

</details>

The optional `args.cursor` MAY be specified in order to paginate over the list of content entries.

The optional `args.size` MAY be specified to signal desired page size, that is number of items in the result.

#### List Uploads Invocation Example

```jsonc
{ // "/": "bafy..uploadList"
  "iss": "did:key:zAlice",
  "aud": "did:web:upload.example.com",
  "sub": "did:key:zAliceSpace",
  "cmd": "/upload/list",
  "args": {
    // cursor where to start listing from
    "cursor": "cursor-value-from-previous-invocation",
    // size of page
    "size": 40
  },
  "prf": [{ "/": "bafy..dlgAliceSpace" }],
  "nonce": { "/": { "bytes": "bGlzdA" } },
  "exp": 1735689600
}
```

### List Uploads Receipt

The `out.ok.results` field MUST be set to the page of listed content entries.

The optional `out.ok.cursor` field, if present, MAY be passed as the `cursor` argument of a subsequent invocation to fetch the next page.

#### List Uploads Receipt Schema

```ipldsch
type ListResult union {
  | ListOK "ok"
  | Error  "error"
} representation keyed

type ListOK struct {
  cursor  optional String # cursor to fetch the next page from
  results [ListUploadItem]
}

type ListUploadItem struct {
  root  Link          # root of the content entry
  index optional Link # content archive (CAR) containing the index of the upload
}
```

<details>
<summary>Go syntax</summary>

```go
type ListOK struct {
	Cursor  *string          `cborgen:"cursor,omitempty"`
	Results []ListUploadItem `cborgen:"results"`
}

type ListUploadItem struct {
	Root  cid.Cid  `cborgen:"root"`
	Index *cid.Cid `cborgen:"index,omitempty"`
}
```

</details>

#### List Uploads Receipt Example

```jsonc
{
  "iss": "did:web:upload.example.com",
  "aud": "did:web:upload.example.com",
  "sub": "did:web:upload.example.com",
  "cmd": "/ucan/assert/receipt",
  "args": {
    // refers to the invocation from the example
    "ran": { "/": "bafy..uploadList" },
    "out": {
      "ok": {
        // cursor where to start listing from on next call
        "cursor": "cursor-value-for-next-invocation",
        "results": [
          {
            "root": { "/": "bafy..root" },
            "index": { "/": "bag..index" }
          }
          // ...
        ]
      }
    }
  },
  "iat": 1735689500
}
```

## List Upload Shards

An authorized agent MAY invoke the `/upload/shard/list` capability on the [space] subject to list the shards of a content entry at the time of invocation.

The shard list of an upload can be very large, so it is not included in [List Uploads] results — this capability pages over the shards of a single upload.

### List Upload Shards Invocation

#### List Upload Shards Arguments Schema

```ipldsch
type ListArguments struct {
  root   Link            # root of the content entry to list shards for
  cursor optional String # cursor to start listing from
  size   optional Int    # desired page size
}
```

<details>
<summary>Go syntax</summary>

```go
type ListArguments struct {
	Root   cid.Cid `cborgen:"root"`
	Cursor *string `cborgen:"cursor,omitempty"`
	Size   *uint64 `cborgen:"size,omitempty"`
}
```

</details>

The `args.root` field MUST be set to the [IPLD link] of the content entry to list shards for.

The optional `args.cursor` MAY be specified in order to paginate over the list of shards.

The optional `args.size` MAY be specified to signal desired page size, that is number of items in the result.

#### List Upload Shards Invocation Example

```jsonc
{ // "/": "bafy..shardList"
  "iss": "did:key:zAlice",
  "aud": "did:web:upload.example.com",
  "sub": "did:key:zAliceSpace",
  "cmd": "/upload/shard/list",
  "args": {
    // root of the content entry to list shards for
    "root": { "/": "bafy..root" },
    // cursor where to start listing from
    "cursor": "cursor-value-from-previous-invocation",
    // size of page
    "size": 40
  },
  "prf": [{ "/": "bafy..dlgAliceSpace" }],
  "nonce": { "/": { "bytes": "c2hhcmRz" } },
  "exp": 1735689600
}
```

### List Upload Shards Receipt

The `out.ok.results` field MUST be set to the page of [IPLD link]s of the [content archive][CAR]s that contain the content entry's DAG.

The optional `out.ok.cursor` field, if present, MAY be passed as the `cursor` argument of a subsequent invocation to fetch the next page.

#### List Upload Shards Receipt Schema

```ipldsch
type ListResult union {
  | ListOK "ok"
  | Error  "error"
} representation keyed

type ListOK struct {
  cursor  optional String # cursor to fetch the next page from
  results [Link]          # content archives (CARs) containing the content DAG
}
```

<details>
<summary>Go syntax</summary>

```go
type ListOK struct {
	Cursor  *string   `cborgen:"cursor,omitempty"`
	Results []cid.Cid `cborgen:"results"`
}
```

</details>

#### List Upload Shards Receipt Example

```jsonc
{
  "iss": "did:web:upload.example.com",
  "aud": "did:web:upload.example.com",
  "sub": "did:web:upload.example.com",
  "cmd": "/ucan/assert/receipt",
  "args": {
    // refers to the invocation from the example
    "ran": { "/": "bafy..shardList" },
    "out": {
      "ok": {
        // cursor where to start listing from on next call
        "cursor": "cursor-value-for-next-invocation",
        "results": [
          { "/": "bag..shard1" },
          { "/": "bag..shard2" }
          // ...
        ]
      }
    }
  },
  "iat": 1735689500
}
```

## Remove Upload

An authorized agent MAY invoke the `/upload/remove` capability on the [space] subject to remove a content entry from the list in the space.

> ⚠️ Please note that removing an upload entry MUST NOT remove the [content archive][CAR]s containing the content. Blob removal is a separate, per-digest decision made with [Remove Blob], owned by the client's reference accounting — shards may be shared across uploads.

### Remove Upload Invocation

#### Remove Upload Arguments Schema

```ipldsch
type RemoveArguments struct {
  root Link # root of the content entry to remove
}
```

<details>
<summary>Go syntax</summary>

```go
type RemoveArguments struct {
	Root cid.Cid `cborgen:"root"`
}
```

</details>

The `args.root` field MUST be set to the [IPLD link] of the content entry to remove.

#### Remove Upload Invocation Example

```jsonc
{ // "/": "bafy..uploadRemove"
  "iss": "did:key:zAlice",
  "aud": "did:web:upload.example.com",
  "sub": "did:key:zAliceSpace",
  "cmd": "/upload/remove",
  "args": {
    // root of the content entry to remove
    "root": { "/": "bafy..root" }
  },
  "prf": [{ "/": "bafy..dlgAliceSpace" }],
  "nonce": { "/": { "bytes": "cmVtb3Zl" } },
  "exp": 1735689600
}
```

### Remove Upload Receipt

Removal MUST be idempotent: removing an unknown or already removed content entry MUST succeed. The success value is an empty object.

#### Remove Upload Receipt Schema

```ipldsch
type RemoveResult union {
  | RemoveOK "ok"
  | Error    "error"
} representation keyed

type RemoveOK struct {}
```

<details>
<summary>Go syntax</summary>

```go
type RemoveOK struct{}
```

</details>

[blob protocol]:./blob.md
[space]:./blob.md#space
[Add Blob]:./blob.md#add-blob
[Remove Blob]:./blob.md#remove-blob
[index]:./index.md#index-format
[Add Index]:./index.md#add-index
[receipt]:./ucan.md#receipt
[List Uploads]:#list-uploads
[List Upload Shards]:#list-upload-shards
[CAR]:https://ipld.io/specs/transport/car/
[UnixFS]:https://docs.ipfs.tech/concepts/file-systems/#unix-file-system-unixfs
[IPLD link]:https://ipld.io/docs/schemas/features/links/
[IPLD Schema]:https://ipld.io/docs/schemas/
[`did:key`]:https://w3c-ccg.github.io/did-key-spec/
[principal]:https://github.com/ucan-wg/spec#principals
[UCAN task]:https://github.com/ucan-wg/invocation#task
