# HTTP Header UCAN Invocation

![reliable](https://img.shields.io/badge/status-reliable-green.svg?style=flat-square)

## Authors

- [Alan Shaw](https://github.com/alanshaw), [Storacha](https://storacha.network/)

## Language

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in [RFC 2119](https://datatracker.ietf.org/doc/html/rfc2119).

## Abstract

This is a specification for sending [UCAN container][UCAN container]s in HTTP headers, leaving the HTTP request/response body usable for alternative purposes. Certain aspects are inspired by the [UCAN as Bearer Token Specification][UCAN bearer token].

## Introduction

UCAN RPC involves sending an [invocation][UCAN invocation] to a service, where it is executed, a [receipt] generated and sent back. Invocations, the delegations proving their authority, and receipts are bundled in [container][UCAN container]s. The standard HTTP transport encodes a container as DAG-CBOR and sends/receives it in the HTTP request/response body.

This poses some challenges when authorizing access to resources and returning them in the _same_ request. Returning resource bytes in the same response implies embedding them in the response container, which is problematic because:

- Content-addressed archives require the content hash to precede the data, so it has to be calculated by the service before any data can be sent. This is a performance problem for large data.
- The recipient must decode the entire container and extract the specific entry containing the data.
- Implementations typically buffer entire containers in memory before dispatching invocations or decoding responses.
- Bookkeeping becomes more difficult, since protocol implementations can no longer store transcript containers verbatim without also storing resource data.

Moving the UCAN tokens into an HTTP header allows the response body to be used to serve resources. The primary use case is UCAN authorized resource retrieval, as used by the [retrieval protocol].

## HTTP Header

The following HTTP header MUST be included when issuing a UCAN invocation via an HTTP `GET` _request_:

```yaml
X-UCAN-Container: <encoded-container>
```

In the header, `<encoded-container>` is a [container][UCAN container] holding the invocation, the [delegation][UCAN delegation]s proving its authority, and optionally other related tokens. The request body is empty.

### Container Encoding

The header value is a single byte codec identifier followed by the encoded container. The value MUST use one of the following codecs, as only base64 encoded values are valid in HTTP headers:

| Codec  | Byte | Encoding                     |
| ------ | ---- | ----------------------------- |
| `0x42` | `B`  | base64 (std padding)          |
| `0x43` | `C`  | base64url (no padding)        |
| `0x45` | `E`  | base64 (std padding), gzip    |
| `0x46` | `F`  | base64url (no padding), gzip  |

Senders SHOULD encode with `0x45` (base64, gzip). Receivers MUST accept all of the above codecs.

### Request

The request container MUST contain exactly one invocation addressed to the server — the invocation whose audience (or subject, when no audience is set) is the server's DID. A request whose container holds multiple invocations addressed to the server MUST fail, since the response body can only be used by one invocation at a time. Additional invocations not addressed to the server MUST be ignored — they MAY be present as context for the addressed invocation.

### Response

The _response_ headers MUST include an `X-UCAN-Container` header: a [container][UCAN container] holding the [receipt] for the executed task, along with any related invocations, delegations and receipts. The response MUST also include the standard [HTTP `Vary` header][Vary] naming `X-UCAN-Container`.

The response body is available for application use — for example, serving the resource bytes authorized by the invocation.

On failure the server MUST return an appropriate HTTP status code and MUST include a failure [receipt] in the response header container. The response body SHOULD contain a human readable message. Clients MUST inspect the receipt to determine the outcome of the invocation — an HTTP success status alone does not indicate a successful invocation.

### Replay

UCAN invocations in HTTP headers SHOULD be considered single use and SHOULD NOT be replayed without first altering the UCAN `nonce` field. A service receiving an invocation that has previously been executed MAY decline to perform its effects again — for example, a replayed retrieval is not guaranteed to be sent the resource bytes.

## Oversize Headers

When sending UCAN invocations via HTTP headers it is important to ensure the total header size does not exceed 8 KiB, in order to adhere to limits imposed by popular HTTP server software.

It is RECOMMENDED that the `X-UCAN-Container` _value_ does not exceed 4 KiB in size.

## Appendix A: Proof Caching

> This appendix is non-normative. It describes a possible future extension for reducing header size that is not currently implemented.

The [delegation][UCAN delegation]s proving an invocation's authority typically dominate the size of the request container. To save space and bandwidth, an agent could omit delegations from the request container — especially when making multiple requests to the same service using the same proofs — relying on the service having cached them from earlier requests.

To support this, response headers would include a `X-UCAN-Cache-Expiry` header, set to a cache expiry time in [Unix time] for the proofs referenced by the invocation. This would be the shortest expiry of the proofs the service has already received, or a maximum time the service is willing to cache the proofs for.

If proofs are omitted from a request and are not present in the server cache, the service would respond with [HTTP 510 (Not Extended)][HTTP 510]. The response body would be a [DAG-JSON] encoded error object listing the missing proofs required in order to execute the invocation, complying with the following schema:

```ipldsch
type MissingProofs struct {
  name    optional String # Typically "MissingProofs"
  message optional String # Instructions to resubmit the invocation
  proofs  [Link]          # CIDs of the delegations that were missing
}
```

e.g.

```json
{
  "name": "MissingProofs",
  "message": "proofs were missing, resubmit the invocation with the requested proofs",
  "proofs": [
    { "/": "bafyreibd7iyy74awztb3chw73f6yenasubabghjb7jjgu3wjfxrysm7qv4" }
  ]
}
```

The HTTP `Content-Type` header would be set to `application/json`, and a `X-UCAN-Cache-Expiry` header would also be set, allowing the request to be repeated with the required proofs whilst continuing to omit proofs the service already holds.

A client would not send unreachable proofs, even if they are part of the proof chain: the server needs to be able to walk the proof chain from the invocation to any missing delegation, either via cached delegations or via delegations provided in the request. A subsequent invocation could also rely on the original invocation being cached by the server.

[UCAN invocation]: https://github.com/ucan-wg/invocation
[UCAN delegation]: https://github.com/ucan-wg/delegation
[UCAN container]: https://github.com/ucan-wg/container
[UCAN bearer token]: https://github.com/ucan-wg/ucan-http-bearer-token
[receipt]: ./ucan.md#receipt
[retrieval protocol]: ./retrieval.md
[Vary]: https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Headers/Vary
[Unix time]: https://en.wikipedia.org/wiki/Unix_time
[HTTP 510]: https://www.rfc-editor.org/rfc/rfc2774#section-7
[DAG-JSON]: https://ipld.io/docs/codecs/known/dag-json/
