# Forge Network UCAN Protocol Specifications

This repository contains the specs for the Forge Network UCAN protocol and associated subsystems.

## Overview

* [Access Protocol](./access.md) — governs how agents request capabilities from accounts, verified via email, and how delegations are delivered to their audience.
* [Attested Authority](./attestation.md) — a scheme for using externally verified identities (email, OAuth, etc.) as UCAN issuers via attestations from a trusted authority.
* [Blob Protocol](./blob.md) — allows authorized agents to store arbitrary content blobs with a storage node.
* [did:mailto DID Method](./did-mailto.md) — a DID method for identifying principals by email address, with no central source of truth.
* [Egress Tracking Protocol](./egress-tracking.md) — allows storage nodes to record the data they serve, so egress fees can be calculated.
* [HTTP Header UCAN Invocation](./http-header-ucan-invocation.md) — sends UCAN containers in HTTP headers, leaving the request/response body free for other purposes.
* [Index Protocol](./index.md) — allows authorized agents to publish verifiable claims about content-addressable data to IPNI, making it publicly queryable.
* [PDP Protocol](./pdp.md) — allows a storage node to prove possession of stored blobs using cryptographic merkle tree commitments.
* [Provider Protocol](./provider.md) — defines how accounts provision spaces with a provider, making storage capabilities invocable.
* [Replication Protocol](./replication.md) — enables distributed storage of blobs across multiple nodes in the network after initial upload.
* [Retrieval Protocol](./retrieval.md) — defines UCAN capabilities for authorizing retrieval of resources, served in the same HTTP request they are authorized in.
* [S3 Tenant Management](./s3.md) — how tenants of an S3 compatible gateway are managed, and how S3 style credentials map to UCAN authorizations.
* [UCAN Extensions](./ucan.md) — protocol extensions to the core `/ucan` namespace that we hope to standardize in the core UCAN specifications.
* [Upload Protocol](./upload.md) — allows authorized agents to manage the list of top level content entries — "uploads" — in a space.

## Status Indicators

We use the following label system to identify the status of each spec:

- ![wip](https://img.shields.io/badge/status-wip-orange.svg?style=flat-square) A work-in-progress to describe an idea before committing to a full draft.
- ![draft](https://img.shields.io/badge/status-draft-yellow.svg?style=flat-square) A draft ready for review. It should be implementable.
- ![reliable](https://img.shields.io/badge/status-reliable-green.svg?style=flat-square) A spec that has been implemented. It will change as we learn how it works in practice.
- ![stable](https://img.shields.io/badge/status-stable-brightgreen.svg?style=flat-square) It may be improved but should not change fundamentally.
- ![permanent](https://img.shields.io/badge/status-permanent-blue.svg?style=flat-square) This spec will not change.
- ![deprecated](https://img.shields.io/badge/status-deprecated-red.svg?style=flat-square) This spec is no longer in use.

Nothing in this spec repository is `permanent`. While some of the subsystems are `stable`, some are still in a `draft` or `reliable` status.

## Contribute

Suggestions, contributions, criticisms are welcome. Though please make sure to familiarize yourself deeply with IPFS, the models it adopts, and the principles it follows.

This repository falls under the IPFS [Code of Conduct](https://github.com/ipfs/community/blob/master/code-of-conduct.md).
