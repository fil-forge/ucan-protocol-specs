# Attested Authority

![draft](https://img.shields.io/badge/status-draft-yellow.svg?style=flat-square)

## Authors

- [Petra Jaros](https://github.com/Peeja)

## Editors

- [Alan Shaw](https://github.com/alanshaw)

## Abstract

This specification describes a generic scheme for using externally verified identities as [UCAN] issuers. Such identities — including email addresses, OAuth-based identities, and others — have no associated keypair under user control, and therefore cannot sign UCANs directly. A trusted authority performs an out-of-band verification (email loop, OAuth exchange, etc.) and produces a cryptographic attestation on behalf of the subject identity. A generic [Varsig] signature type (`authority-attestation`) is defined that encodes the authority's attestation in place of a conventional asymmetric signature. This allows attested identities to appear as `iss` in root UCAN delegations while remaining structurally honest about the nature of the verification performed.

Concrete DID methods for specific identity types (e.g. [`did:mailto`][did:mailto]) are defined as extensions of this scheme.

## Language

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in [RFC2119](https://datatracker.ietf.org/doc/html/rfc2119).

# Introduction

UCAN delegation chains require every issuer to sign with a private key. Many real-world identities (email addresses, OAuth-based social identities, phone numbers, and others) have no keypair under user control. This specification defines a generic attested authority scheme such that:

- An externally verified identity can appear as `iss` in a UCAN delegation.
- The "signature" bytes encode a real cryptographic signature from a trusted [authority], together with attestation metadata.
- Verifiers can determine the authority's identity from the signature itself, resolve its DID, and verify the signature.
- New verification methods (email loop, OAuth, etc.) can be added without defining new [Varsig] types or DID verification method types.

## Concepts

### Roles

| Name      | Description                                                                                                       |
| --------- | ------------------------------------------------------------------------------------------------------------------ |
| Principal | The general class of entities that interact with a UCAN. Identified by a DID that can be used in the `iss`, `aud` or `sub` field of a UCAN. |
| Subject   | The externally verified identity a UCAN is issued by — a [principal] with no keypair of its own, e.g. a [`did:mailto`][did:mailto] identifier. |
| Authority | A trusted [principal] — for example the upload service — that performs out-of-band verification of a subject identity and issues [attestation]s on its behalf. |
| Verifier  | A [principal] validating a UCAN issued by an attested identity.                                                     |

### Attestation

An _attestation_ is a [signature invocation] — a `/ucan/attest/proof` invocation signed by the [authority] — carried in place of a conventional signature in the [envelope][UCAN envelope] of a UCAN issued by an attested identity.

### Schema Notation

Schemas in this document are described using [IPLD Schema] notation, accompanied by equivalent Go types whose `cborgen` tags define the wire keys.

# DID Documents for Attested Identities

All DID methods defined under this scheme share the same [DID document] structure. A DID document for an attested identity MUST contain at least one verification method of type `AuthorityAttestation` and MUST NOT contain conventional key material, as the subject has no keypair:

- `id` MUST be the subject DID with the [authority] DID as the fragment.
- `controller` MUST be the [authority] DID.
- `authority` MUST be the [authority] DID.

The verification method MUST be referenced from the `capabilityInvocation` and `capabilityDelegation` verification relationships.

```json
{
  "@context": "https://www.w3.org/ns/did/v1",
  "id": "did:mailto:example.com:jsmith",
  "verificationMethod": [
    {
      "id": "did:mailto:example.com:jsmith#did:web:upload.example.com",
      "type": "AuthorityAttestation",
      "controller": "did:web:upload.example.com",
      "authority": "did:web:upload.example.com"
    }
  ],
  "capabilityInvocation": [
    "did:mailto:example.com:jsmith#did:web:upload.example.com"
  ],
  "capabilityDelegation": [
    "did:mailto:example.com:jsmith#did:web:upload.example.com"
  ]
}
```

The `AuthorityAttestation` verification method type indicates that authentication for this DID is performed by a trusted authority, named in the `authority` field.

In theory, this verification method is available to any kind of DID which can put one in its DID document. In practice, this verification method is likely only useful for the narrow cases described in this document.

## Resolution

DID documents for attested identities are generally not hosted or published by the subject (who has no infrastructure). They are constructed synthetically by verifiers from the DID string alone, with the [authority] supplied by resolver configuration. The [`did:mailto`][did:mailto] method defines the resolution rules for email-based identities. Further DID methods for other externally verified identity types (e.g. OAuth-based identities) MAY be defined as extensions of this scheme.

# The Verification Protocol

Before issuing an [attestation], the [authority] MUST perform out-of-band verification appropriate to the DID method of the subject. The specific verification approach is the authority's internal concern; the [attestation] does not encode which method was used. The verifier trusts the authority to have performed appropriate verification for the subject DID method it attests.

## Email Loop (`did:mailto`)

1. The client authors a delegation payload in which the [`did:mailto`][did:mailto] subject issues some capability to the client's agent identity, and issues an invocation to the [authority] asking for it to be signed. The exact invocation is not specified here — in this protocol suite it is part of the access protocol.
2. The authority sends an email to the address encoded in the `did:mailto` DID, containing a verification link pointing back to the authority's web server. The link MUST encode the delegation payload (or a digest of it) such that the authority can recover it, MUST be protected against tampering (for example with an HMAC over the payload and expiry, or by encoding the request as an invocation signed by the authority), and MUST expire. A verification link expiry of 15–60 minutes is RECOMMENDED.
3. The authority SHOULD include a full description of the delegation payload in the email it sends — for instance the DAG-JSON representation, or a more human readable layout. This ensures that the controller of the email inbox has an opportunity to understand what they are authorizing.
4. When the user clicks the link, the authority validates it. If the link is not valid — because it has been tampered with or has expired — the authority MUST NOT create an attestation. The authority's web server MAY display a useful message explaining the failure, and MAY offer to send a new email with a new link for the same delegation payload.
5. The authority produces an [attestation] and builds a signed delegation from the payload, using the attestation bytes as the signature. It stores that delegation for later retrieval.

The attestation operation MUST be idempotent: clicking the same link a second time produces a byte-identical delegation.

The authority MAY define a policy governing what delegations it will verify and attest to, and reject requests which are not permitted by that policy. For instance, the authority may reject delegations without a recent `nbf` ("not before"), or with an `exp` ("expiration") that is `null` or too far in the future. To reject an attestation request, the authority MUST return a failure for the original attestation request invocation. If it does not reject the request, the authority MUST use the delegation payload as given, with no changes.

# The `authority-attestation` Varsig Type

## Algorithm Code

This scheme uses a signature algorithm code from the [multicodec] private use area, which is guaranteed never to be assigned a conflicting meaning:

```txt
0x300001  (within private use range 0x300000–0x3FFFFF)
```

This code SHOULD be replaced with a registered code if the scheme is standardized.

## Varsig Header Structure

A [Varsig] header for this type has the following structure:

```txt
0x34      Varsig prefix
0x01      Varsig version 1
0x300001  authority-attestation algorithm discriminant (varint)
0x71      Payload encoding: DAG-CBOR
```

The header is a constant value which signals that the signature should be interpreted as an authority-attestation signature.

## Signature Invocation

The signature bytes (the `.0` field of the [UCAN envelope]) for this type are the DAG-CBOR encoding of a complete, signed [UCAN invocation] with a specific shape:

```jsonc
[
  { "/": { "bytes": "..." } }, // the authority's signature
  {
    "h": { "/": { "bytes": "..." } },
    "ucan/inv@1.0.0-rc.1": {
      "iss": "did:web:upload.example.com",
      "sub": "did:web:upload.example.com",
      "cmd": "/ucan/attest/proof",
      "args": {
        // CID of the attested payload
        "proof": { "/": "bafk..sigPayload" }
      },
      "prf": [],
      "nonce": { "/": { "bytes": "" } },
      "exp": null
    }
  }
]
```

### Signature Invocation Arguments Schema

```ipldsch
type ProofArguments struct {
  proof Link # CID of the attested payload
}
```

<details>
<summary>Go syntax</summary>

```go
type ProofArguments struct {
	Proof cid.Cid `cborgen:"proof"`
}
```

</details>

### Signature Invocation Fields

- `iss` and `sub` MUST both be the [authority]'s DID. As an issuer, the DID must be resolvable to a verification method which can sign the invocation.
- `aud` MUST be absent — the invocation is not addressed to anyone in particular.
- `cmd` MUST be `/ucan/attest/proof`.
- `args` MUST contain a single field, `proof`, whose value is a CIDv1 with the `raw` codec wrapping a [multihash] digest of the outer delegation's `SigPayload` (the token payload plus the Varsig header). The digest SHOULD be SHA2-256, [the only required algorithm in the UCAN cryptosuite][UCAN cryptosuite]. Using another algorithm may limit interoperability.
- `prf` MUST be empty. This invocation is issued by its subject, with inherent authority and no proofs required.
- `meta` is OPTIONAL, and can be used by the authority to track extra facts about the verification process, for informational purposes. Information stored in `meta` MUST NOT be considered to affect the validity of the signature.
- `nonce` MUST be empty. As an assertion of fact, the invocation is inherently idempotent.
- `exp` MUST be `null`. As an assertion of fact, the invocation cannot expire: the signature cannot have not happened because time has passed. The delegation's `exp` controls the expiration of the delegation.
- `iat` is OPTIONAL, and for informational purposes only. If it is provided, it MUST be the time that the attestation request was received, not the time that the signature was created. This ensures that verifying the same request twice produces an identical signature and delegation.
- `cause` is OPTIONAL, and for informational purposes only. If present, it MUST be a link to the receipt of the attestation request invocation.

This invocation is itself signed in the normal way, by its issuer, the attesting [authority]. It is never dispatched to an executor and produces no receipt of its own.

# Verification

When a [verifier] encounters a UCAN with a [Varsig] header with the algorithm discriminant `0x300001`, it MUST:

1. **Decode the signature invocation** — interpret the signature bytes as a canonically encoded [UCAN invocation]. Fail if it does not decode, or if its `cmd` is not `/ucan/attest/proof`.
2. **Resolve the issuer DID** — perform the "Resolve" operation on the UCAN issuer's DID. For a [`did:mailto`][did:mailto], for instance, this means expanding the DID algorithmically into its DID document. Then find a `capabilityDelegation` (or, for invocations, `capabilityInvocation`) verification method with the type `AuthorityAttestation` and an `authority` matching the signature invocation's issuer and subject. If none is found, fail.
3. **Validate the signature invocation** — resolve the [authority]'s DID and validate the authority's signature on the signature invocation, using any of the authority's own verification methods. If the invocation is invalid, fail.
4. **Verify the digest** — take the digest of the UCAN's `SigPayload` using the same algorithm as the multihash within the `args.proof` CID. If the algorithm is not supported, fail. If the computed digest does not match `args.proof`, fail.
5. **Succeed** — if the process has not failed yet, the UCAN's signature is valid.

If the process fails at any point, the verifier MUST consider the UCAN to have an invalid signature.

# Security Considerations

## The Trust Gap

This scheme does not eliminate the need for out-of-band trust configuration. The verifier must decide whether to trust a given [authority] for a given DID method and domain or provider. This is unavoidable: no cryptographic scheme can bootstrap trust from an identity that has no keypair. The scheme makes the trust relationship explicit and self-describing rather than implicit.

The trusted authority is designated in the verification method. Therefore, the trust gap is manifested in the algorithm which expands a DID document from a [`did:mailto`][did:mailto] or similar DID. DIDs of methods with stored, non-algorithmic DID documents can select their own trusted authority.

## Authority Compromise

If the authority's keypair is compromised, an attacker can issue attestations for arbitrary subjects. In this case, the authority SHOULD rotate keys, and reflect that in the authority's own DID document. Verifiers SHOULD notice the updated DID document and no longer validate attestations signed by the compromised key.

## Subject Identity Compromise

The attestation is no stronger than the underlying identity. For [`did:mailto`][did:mailto]: if the email account is compromised, an attacker can complete the email loop and obtain a valid attestation. This is an inherent property of email-based identity.

## Replay

Attestation is an idempotent operation: attesting the same payload always produces the same invocation, so replaying the verification has no meaningful effect. An attestation for one payload cannot be used for a different payload, since the authority's signature covers the full `SigPayload`.

## Canonicalization

The payload digest in the [signature invocation] MUST be computed over the canonical DAG-CBOR encoding of the UCAN payload, consistent with UCAN's canonicalization requirements. This is the same encoding that is signed over in the `SigPayload`, preventing canonicalization attacks.

[UCAN]: https://github.com/ucan-wg/spec
[UCAN invocation]: https://github.com/ucan-wg/invocation
[UCAN envelope]: https://github.com/ucan-wg/spec#envelope
[UCAN cryptosuite]: https://github.com/ucan-wg/spec#cryptosuite
[Varsig]: https://github.com/ChainAgnostic/varsig
[multicodec]: https://github.com/multiformats/multicodec
[multihash]: https://github.com/multiformats/multihash
[IPLD Schema]: https://ipld.io/docs/schemas/
[DID document]: https://www.w3.org/TR/did-core/#dfn-did-documents
[did:mailto]: ./did-mailto.md
[principal]: https://github.com/ucan-wg/spec#principals
[attestation]: #attestation
[authority]: #roles
[verifier]: #roles
[signature invocation]: #signature-invocation
