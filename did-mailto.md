# did:mailto DID Method

![reliable](https://img.shields.io/badge/status-reliable-green.svg?style=flat-square)

## Authors

- [Irakli Gozalishvili](https://github.com/Gozala)

## Editors

- [Alan Shaw](https://github.com/alanshaw)

## Abstract

This specification describes the "mailto" [DID Method], which conforms to the core [DID-CORE] specification. The method can be used independent of any central source of truth, and is intended for bootstrapping secure interaction between two parties that can span across an arbitrary number of devices. It is suitable for long sessions that need to operate under network partitions.

## Language

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in [RFC 2119](https://datatracker.ietf.org/doc/html/rfc2119).

# Introduction

## Overview

> This section is non-normative.

Most documentation about decentralized identifiers (DIDs) describes them as identifiers that are rooted in a public source of truth like a blockchain, a database, or similar. This publicness lets arbitrary parties resolve the DIDs to an endpoint and keys. It is an important feature for many use cases.

However, the vast majority of interactions between people, organizations, and things have simpler requirements. When Alice(Corp|Device) and Bob want to interact, there are exactly and only 2 parties in the world who should care: Alice and Bob. Instead of arbitrary parties needing to resolve their DIDs, only Alice and Bob do.

One to one interactions are an excellent fit for [did:key] identifiers, however they suffer from a key discovery problem and introduce additional challenges when interaction sessions span more than two devices.

Mailto DIDs are designed to be used in conjunction with [did:key] and facilitate bootstrapping sessions between two parties that span multiple devices.

Mailto DIDs are a more accessible alternative to [did:web] and [did:dns] because a lot more people have an email address than there are people with a [did:web] or [did:dns] identifier or the skills to acquire one.

A `did:mailto` identifier has no key material of its own. In this protocol suite, UCANs issued by a `did:mailto` principal are verified through [attestation]s issued by a trusted [authority].

## Terminology

### Authority

An _authority_ is a trusted [principal] — for example the upload service — that verifies a user's control of an email address out of band (for example, by sending a confirmation email) and issues [attestation]s on behalf of the corresponding `did:mailto` identifier.

### Attestation

An _attestation_ is a signed statement from an [authority] verifying a signature payload on behalf of a `did:mailto` principal. It takes the place of a signature on UCANs issued by the principal. The attestation mechanism is defined in the [attested authority] specification.

## The `did:mailto` Format

The format for the `did:mailto` method conforms to the [DID-CORE] specification and is an encoding of the [email address]. It consists of the `did:mailto:` prefix, followed by the `domain` part of an email address, a `:` character and the percent encoded `local-part` of the email address.

The ABNF definition can be found below. The formal rules describing valid `domain-name` syntax are described in [RFC1035], [RFC1123], [RFC2181]. The `domain-name` and `user-name` correspond to the `domain` and `local-part` respectively of the email address described in [RFC2822]. All "mailto" DIDs MUST conform to the DID Syntax ABNF Rules.

```abnf
did       = "did:mailto:" domain-name ":" user-name
user-name = 1*idchar
idchar    = ALPHA / DIGIT / "." / "-" / "_" / pct_enc
pct_enc   = "%" HEXDIG HEXDIG
```

### EXAMPLE 1. <jsmith@example.com>

```txt
did:mailto:example.com:jsmith
```

### EXAMPLE 2. <tag+alice@web.mail>

```txt
did:mailto:web.mail:tag%2Balice
```

## Operations

The following section outlines the DID operations for the `did:mailto` method.

### Create (Register)

A `did:mailto` identifier is derived purely from an email address — there is nothing to register and no document to publish, as a single source of truth is (intentionally) not implied.

The same `did:mailto` identifier MAY (intentionally) be exercised with different key material in different sessions: which keys currently act for the identifier is established by [attestation]s from an [authority], not by the identifier itself.

### Read (Resolve)

Resolution does not involve the email system. A resolver MUST be configured with the [DID] of a trusted [authority], and MUST resolve a `did:mailto` identifier to a [DID document] containing a single verification method of type `AuthorityAttestation`:

- `id` MUST be the resolved DID with the authority DID as the fragment.
- `controller` MUST be the [authority] DID.
- `authority` MUST be the [authority] DID.

The verification method MUST be referenced from the `capabilityInvocation` and `capabilityDelegation` verification relationships, permitting the [DID Subject] to invoke and delegate capabilities.

An `AuthorityAttestation` verification method declares that signatures attributed to the [DID Subject] are verified as [attestation]s issued by the named [authority].

#### EXAMPLE 3. Resolved DID document

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

### Deactivate (Revoke)

This DID method defines no deactivation operation. Authority exercised by a `did:mailto` principal is withdrawn at the authorization layer: an [authority] ceases to attest for the identifier, and previously issued UCANs are revoked using [UCAN revocation].

### Update

This DID method does not support updating the DID document.

[DID]: https://www.w3.org/TR/did-core/
[did subject]: https://www.w3.org/TR/did-core/#did-subject
[did method]: https://w3c-ccg.github.io/did-spec/#specific-did-method-schemes
[did-core]: https://w3c-ccg.github.io/did-spec/
[did document]: https://www.w3.org/TR/did-core/#dfn-did-documents
[did:key]: https://w3c-ccg.github.io/did-key-spec/
[did:web]: https://w3c-ccg.github.io/did-method-web/
[did:dns]: https://danubetech.github.io/did-method-dns/
[email address]: https://www.rfc-editor.org/rfc/rfc2822.html#section-3.4.1
[rfc2822]: https://www.rfc-editor.org/rfc/rfc2822.html#section-3.4.1
[rfc1035]: https://www.rfc-editor.org/rfc/rfc1035
[rfc1123]: https://www.rfc-editor.org/rfc/rfc1123
[rfc2181]: https://www.rfc-editor.org/rfc/rfc2181
[attestation]: #attestation
[authority]: #authority
[attested authority]: ./attestation.md
[principal]: https://github.com/ucan-wg/spec#principals
[UCAN revocation]: https://github.com/ucan-wg/revocation
