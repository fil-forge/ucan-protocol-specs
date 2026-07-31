# Retrieval Protocol

![reliable](https://img.shields.io/badge/status-reliable-green.svg?style=flat-square)

## Authors

- [Alan Shaw](https://github.com/alanshaw)

## Language

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in [RFC 2119](https://datatracker.ietf.org/doc/html/rfc2119).

## Abstract

This specification defines UCAN capabilities for authorizing retrieval of resources, and the transport that allows resources to be served in the same HTTP request they are authorized in.

# Introduction

In this network, data retrievals are authorized by UCAN to allow for strict egress accounting. UCAN authorized retrievals do not prove delivery of data but they can be used to prove that data was requested by an authorized agent.

An agent discovers where content can be retrieved from via [location commitment]s — obtained from the indexer, or directly from the [blob protocol]'s add flow — and issues a retrieval invocation to the committed location using the [retrieval transport].

## Concepts

### Roles

| Name         | Description                                                                             |
| ------------ | ----------------------------------------------------------------------------------------- |
| Agent        | A [principal] identified by [`did:key`] identifier, representing a user in an application. |
| Storage Node | A [principal] that stores blob content and serves retrievals for it.                    |

### Common Types

Schemas in this document are described using [IPLD Schema] notation, accompanied by equivalent Go types whose `cborgen` tags define the wire keys. Failure values follow the `{name, message}` convention of the [receipt] spec.

# Retrieval Transport

Retrieval invocations are issued using the [HTTP header UCAN invocation] transport, which allows a UCAN invocation to authorize an HTTP request whose response carries the requested resource bytes — one round trip for authorization and delivery.

The client sends an HTTP `GET` request to a URL from the [location commitment] committing to the content's location, with the invocation and its proof [delegation][UCAN delegation]s transmitted in the `X-UCAN-Container` request header. The [receipt] is returned in the response header, and the response body carries the raw resource bytes.

> ℹ️ The URL path serves only to locate the resource endpoint — the content retrieved is identified by the invocation arguments, not the URL.

The response status is `200` when the response carries the entire blob, or `206` with a `Content-Range` header when it carries a partial range. On failure the server MUST return an appropriate HTTP status code (e.g. `404`, `416`, `500`) with the failure receipt in the response header container, as described in the [transport specification][HTTP header UCAN invocation].

# Capabilities

## Content Retrieve

An authorized agent MAY invoke the `/content/retrieve` capability on the [space] subject to retrieve blob bytes from a storage node. The audience of the invocation is the storage node that issued the [location commitment] for the blob.

### Content Retrieve Invocation

#### Content Retrieve Arguments Schema

```ipldsch
type RetrieveArguments struct {
  blob  Blob
  range Range
}

type Blob struct {
  digest Bytes # multihash digest of the blob to retrieve data from
}

# Byte range to extract from the blob.
# Encoded as the tuple [start, end].
type Range struct {
  start Int # offset of the first byte (inclusive)
  end   Int # offset of the last byte (inclusive)
} representation tuple
```

<details>
<summary>Go syntax</summary>

```go
type RetrieveArguments struct {
	Blob  Blob  `cborgen:"blob"`
	Range Range `cborgen:"range"`
}

type Blob struct {
	Digest multihash.Multihash `cborgen:"digest"`
}

// Encoded as the CBOR tuple [start, end].
type Range struct {
	Start uint64
	End   uint64
}
```

</details>

The `args.blob.digest` field MUST be a [multihash] digest of the blob payload bytes. Implementations SHOULD support the SHA2-256 algorithm. Implementations MAY support other hashing algorithms.

The `args.range` field MUST be a tuple of two unsigned integers. The first integer is the offset of the first byte to extract. The second integer is the offset of the last byte to extract. Both offsets are _inclusive_. The end offset MUST be greater than or equal to the start offset. Both offsets MUST be less than the total byte size of the blob. There is no "whole blob" form — a range is always required.

> ℹ️ Note this `Range` is a tuple with a required end offset — unlike the map-encoded `Range` with optional `end` used by [location commitment]s.

#### Content Retrieve Invocation Example

The following invocation example illustrates Alice requesting to retrieve 1,637 bytes from a blob stored in her space.

> ℹ️ Note: examples show the invocation payload; the enclosing signed envelope is elided. We use `// "/": "bafy.."` comments to denote the [task][UCAN task] link of the invocation.

```jsonc
{ // "/": "bafy..retrieve"
  "iss": "did:key:zAlice",
  "aud": "did:key:zStorageNode",
  "sub": "did:key:zAliceSpace",
  "cmd": "/content/retrieve",
  "args": {
    "blob": {
      // multihash of the blob to retrieve data from as byte array
      "digest": { "/": { "bytes": "mEi...sfKg" } }
    },
    // byte range to extract from the blob - first and last byte (both inclusive)
    "range": [2097152, 2098788]
  },
  "prf": [{ "/": "bafy..dlgAliceSpace" }],
  "nonce": { "/": { "bytes": "cmV0cmlldmU" } },
  "exp": 1735689600
}
```

### Content Retrieve Receipt

The invocation MUST fail if any of the following is true:

1. The subject space has no allocation for the blob on this storage node _(error name `NotAllocated`)_.
1. The blob content is not present on the storage node _(error name `NotFound`)_.
1. Provided `blob.digest` is not a valid [multihash].
1. Provided `blob.digest` [multihash] hashing algorithm is not supported.
1. Provided `range` references bytes outside of the total size of the blob or is otherwise invalid _(error name `RangeNotSatisfiable`)_.

Invocation MUST succeed if none of the above is true.

Successful execution of a `/content/retrieve` invocation MUST result in the requested data being sent to the agent that issued the request, in the body of the [retrieval transport] response. Replayed invocations MAY send the requested data, but it is not required. It is RECOMMENDED that `/content/retrieve` invocations specify a [nonce][UCAN nonce] or a non-static expiry to ensure that data is always sent.

The success value is an empty object — the data travels in the response body, not the receipt.

#### Content Retrieve Receipt Schema

```ipldsch
type RetrieveResult union {
  | RetrieveOK "ok"
  | Error      "error"
} representation keyed

type RetrieveOK struct {}
```

<details>
<summary>Go syntax</summary>

```go
type RetrieveOK struct{}
```

</details>

## Blob Retrieve

An authorized agent MAY invoke the `/blob/retrieve` capability on a storage node subject to retrieve a whole blob by digest.

Unlike [Content Retrieve], this capability is service-level, not space-scoped: the subject is the storage node itself, and any holder of a valid delegation may fetch the blob by digest, regardless of which space it was originally stored under. It is used, for example, by the indexer to fetch content claims from a storage node. Storage nodes SHOULD constrain the delegations they issue for this capability by [policy][UCAN delegation] to specific blob digests.

### Blob Retrieve Invocation

#### Blob Retrieve Arguments Schema

```ipldsch
type RetrieveArguments struct {
  blob RetrieveBlob
}

type RetrieveBlob struct {
  digest Bytes # multihash digest of the blob to retrieve
}
```

<details>
<summary>Go syntax</summary>

```go
type RetrieveArguments struct {
	Blob RetrieveBlob `cborgen:"blob"`
}

type RetrieveBlob struct {
	Digest multihash.Multihash `cborgen:"digest"`
}
```

</details>

The `args.blob.digest` field MUST be a [multihash] digest of the blob payload bytes. There is no range — the whole blob is retrieved.

#### Blob Retrieve Invocation Example

```jsonc
{ // "/": "bafy..blobRetrieve"
  "iss": "did:web:indexer.example.com",
  "aud": "did:key:zStorageNode",
  "sub": "did:key:zStorageNode",
  "cmd": "/blob/retrieve",
  "args": {
    "blob": {
      // multihash of the blob to retrieve as byte array
      "digest": { "/": { "bytes": "mEi...sfKg" } }
    }
  },
  "prf": [{ "/": "bafy..dlgStorageNode" }],
  "nonce": { "/": { "bytes": "YmxvYg" } },
  "exp": 1735689600
}
```

### Blob Retrieve Receipt

The invocation MUST fail if the blob content is not present on the storage node _(error name `NotFound`)_.

Successful execution MUST result in the blob bytes being sent in the body of the [retrieval transport] response. The success value is an empty object.

#### Blob Retrieve Receipt Schema

```ipldsch
type BlobRetrieveResult union {
  | RetrieveOK "ok"
  | Error      "error"
} representation keyed

type RetrieveOK struct {}
```

<details>
<summary>Go syntax</summary>

```go
type RetrieveOK struct{}
```

</details>

[blob protocol]:./blob.md
[space]:./blob.md#space
[location commitment]:./blob.md#location-commitment
[HTTP header UCAN invocation]:./http-header-ucan-invocation.md
[receipt]:./ucan.md#receipt
[retrieval transport]:#retrieval-transport
[Content Retrieve]:#content-retrieve
[multihash]:https://github.com/multiformats/multihash
[IPLD Schema]:https://ipld.io/docs/schemas/
[`did:key`]:https://w3c-ccg.github.io/did-key-spec/
[principal]:https://github.com/ucan-wg/spec#principals
[UCAN delegation]:https://github.com/ucan-wg/delegation
[UCAN task]:https://github.com/ucan-wg/invocation#task
[UCAN nonce]:https://github.com/ucan-wg/spec#nonce
