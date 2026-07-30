# UCAN Specification Extensions

![draft](https://img.shields.io/badge/status-draft-yellow.svg?style=flat-square)

## Authors

- [Irakli Gozalishvili]

## Editors

- [Irakli Gozalishvili]
- [Alan Shaw](https://github.com/alanshaw)

## Language

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in [RFC2119](https://datatracker.ietf.org/doc/html/rfc2119).

## Abstract

The [UCAN] specification reserves the `/ucan` command namespace for core functionality. Here we define protocol extensions to the core namespace that we hope to standardize in the core [UCAN] specifications.

# Terminology

## Roles

There are several roles that agents in the authorization flow may assume:

| Name      | Description |
| --------- | ----------- |
| Principal | The general class of entities that interact with a UCAN. Listed in the `iss`, `aud` or `sub` field |
| Issuer    | Principal issuing a UCAN. It is the signer of the [UCAN]. Listed in the `iss` field |
| Audience  | Principal a UCAN is addressed to. Listed in the `aud` field |
| Executor  | Principal that carries out an [invocation][UCAN invocation] and attests to its result |

# Capabilities

## Receipt

### Motivation

The [UCAN invocation] specification describes a [receipt][UCAN receipt] record — a cryptographically signed attestation of an invocation's output. We represent receipts on the wire as [UCAN invocation]s of a special `/ucan/assert/receipt` capability, per the [UCAN receipt draft][UCAN receipt]. Reusing the invocation format means receipts inherit its envelope, signature and encoding rules, can be transmitted in [container][UCAN container]s alongside other UCANs, and can be attested by principals other than the executor by supplying proofs.

### Receipt Schema

Schemas are described using [IPLD Schema] notation, accompanied by equivalent Go types whose `cborgen` tags define the wire keys. The arguments of the `/ucan/assert/receipt` invocation are:

```ipldsch
type ReceiptArguments struct {
  ran Link # task link of the executed task
  out Result
}

type Result union {
  | OK  "ok"
  | Err "error"
} representation keyed

type OK any  # success value, shape defined by the executed task's capability
type Err any # failure value, shape defined by the executed task's capability
```

<details>
<summary>Go syntax</summary>

```go
type ReceiptArguments struct {
	Ran cid.Cid `cborgen:"ran"`
	Out Result  `cborgen:"out"`
}

// Exactly one of OK or Err is set. The value is the raw encoded success or
// failure value of the executed task.
type Result struct {
	OK  *datamodel.Raw `cborgen:"ok,omitempty"`
	Err *datamodel.Raw `cborgen:"error,omitempty"`
}
```

</details>

### Receipt Issuer

The receipt MUST be signed by the [executor] of the task it attests to. The issuer, subject and audience of the receipt invocation MUST all be set to the executor DID.

### Receipt Ran

The `args.ran` field MUST be set to the [task][UCAN task] link of the task the receipt is for. Note this is the link of the task _(the `{sub, cmd, args, nonce}` record)_, not of the enclosing signed invocation.

### Receipt Output

The `args.out` field MUST be set to the result of the task execution. The result MUST have either an `ok` field, set to the success value defined by the task's capability, or an `error` field, set to the failure value.

It is RECOMMENDED that failure values contain a `name` field with a machine readable error name and a `message` field with a human readable description:

```ipldsch
type Error struct {
  name    String # machine readable error name
  message String # human readable description
}
```

<details>
<summary>Go syntax</summary>

```go
type Error struct {
	Name    string `cborgen:"name"`
	Message string `cborgen:"message"`
}
```

</details>

### Receipt Expiration

Expiration semantics for receipts are not yet settled. The `exp` field of the receipt invocation MUST NOT be interpreted as bounding the validity of the attested result — consumers SHOULD NOT reject a receipt because its enclosing invocation has expired.

### Receipt Example

An example receipt attesting to the successful execution of an `/http/put` task _(only the salient payload fields are shown)_:

```jsonc
{
  "iss": "did:key:zMh...der",
  "aud": "did:key:zMh...der",
  "sub": "did:key:zMh...der",
  "cmd": "/ucan/assert/receipt",
  "args": {
    // the task this receipt is for
    "ran": { "/": "bafy..put" },
    "out": {
      "ok": {}
    }
  },
  // Unix timestamp at which the receipt was issued
  "iat": 1735689500
}
```

## Conclusion

### Motivation

Some tasks are executed by principals that have no channel of their own to the service awaiting the result. For example, the [blob protocol]'s `/http/put` task is performed by the uploading client on behalf of a derived principal, while the upload service awaits its [receipt] before proceeding. We define a `/ucan/conclude` capability that allows any agent to deliver a receipt to an audience that awaits it.

### Conclusion Schema

```ipldsch
type ConcludeArguments struct {
  receipt Link # the delivered receipt invocation
}

type ConcludeResult union {
  | ConcludeOK "ok"
  | Error      "error"
} representation keyed

type ConcludeOK struct {}
```

<details>
<summary>Go syntax</summary>

```go
type ConcludeArguments struct {
	Receipt cid.Cid `cborgen:"receipt"`
}

type ConcludeOK struct{}
```

</details>

### Conclusion Receipt

The `args.receipt` field MUST be set to the link of the delivered [receipt] invocation. Note this is the link of the signed receipt invocation, not a [task][UCAN task] link.

The receipt itself MUST be transmitted in the request [container][UCAN container] alongside the `/ucan/conclude` invocation. Invocation MUST fail with error name `ConclusionReceiptNotFound` if the linked receipt is not present in the request container.

### Conclusion Result

Invocation MUST succeed if the linked receipt is present in the request container and is a valid [receipt]. The success value is an empty object.

### Conclusion Example

An example invocation delivering an `/http/put` receipt to the upload service:

```jsonc
{
  "iss": "did:key:zAlice",
  "aud": "did:web:upload.example.com",
  "sub": "did:web:upload.example.com",
  "cmd": "/ucan/conclude",
  "args": {
    // link to the receipt invocation, transmitted in the same container
    "receipt": { "/": "bafy..putReceipt" }
  },
  "prf": [],
  "nonce": { "/": { "bytes": "Y29uY2x1ZGU" } },
  "exp": 1735689600
}
```

[UCAN]:https://github.com/ucan-wg/spec
[UCAN invocation]:https://github.com/ucan-wg/invocation
[UCAN task]:https://github.com/ucan-wg/invocation#task
[UCAN receipt]:https://github.com/ucan-wg/receipt
[UCAN container]:https://github.com/ucan-wg/container
[IPLD Schema]:https://ipld.io/docs/schemas/
[executor]:#roles
[receipt]:#receipt
[blob protocol]:./blob.md
[Irakli Gozalishvili]:https://github.com/Gozala
