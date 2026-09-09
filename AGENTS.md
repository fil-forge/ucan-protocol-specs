# Forge Network UCAN Protocol Specifications

Public specifications for the UCAN protocols the Forge network runs on. Each
spec is one Markdown file at the repo root describing a protocol as it is
implemented: the capabilities, their argument and receipt schemas, the failure
names, and the flow between principals. `README.md` is the index; every spec
has one entry there.

There is no build, test, or lint step. The checks are editorial: the spec
matches the implementation, links resolve, and the document follows the
conventions below.

## Code over specs

The implementation is the source of truth. When a spec and the code disagree,
change the spec and flag the divergence to the user; never describe behaviour
the code does not have. Verify against these repos before writing:

- `fil-forge/libforge` `commands/*` — the bound commands and Go wire types for
  every capability. A spec's Go syntax block mirrors the struct and `cborgen`
  tags found here; the IPLD schema is derived from them.
- `fil-forge/sprue` — the upload service: blob, upload, provider, access,
  index, replication, routing.
- `fil-forge/piri` — the storage node: blob allocate/accept, PDP, retrieval,
  location commitments.
- `fil-forge/hilt` and `fil-forge/ingot` — S3 tenant management and the S3
  gateway.
- `fil-forge/swarf` — revocation checking.
- The egress tracking service (`etracker` in the smelt stack) — egress
  tracking. Its repo is not cloned in this workspace; ask before asserting
  behaviour the spec does not already state.
- `fil-forge/ucantone` — UCAN primitives, receipts, containers, the `/ucan`
  namespace.

The repos live as siblings under `../` in the Forge workspace. Read the
relevant handler and command package, then write.

## Relationship to RFCs

Design proposals live in `fil-one/RFC/rfcs/`. An RFC argues for a change
(motivation, trade-offs, alternatives); a spec here describes the resulting
protocol. When turning an RFC into a spec, carry over the normative rules,
schemas, error names, and examples. Leave out the motivation, trade-offs, and
alternatives, and do not cite the RFC in the body. Write the spec as if it is
the only description of the protocol that has ever existed.

## Document structure

Copy `provider.md` for a short spec and `blob.md` for a long one. In order:

1. `# <Name> Protocol` title, then a status badge on its own line (see
   `README.md` for the badge markup and meaning).
2. `## Authors` and, when the current maintainer differs, `## Editors`, as
   GitHub-linked names.
3. `## Abstract` — one paragraph saying what the protocol lets whom do.
4. `## Language` — the RFC 2119 boilerplate, copied verbatim.
5. `# Introduction` — the problem and the shape of the solution, in plain
   prose.
6. `## Concepts` — `### Roles` table (Name, Description), one `###` per
   domain term, and `### Common Types` with the shared schemas.
7. An optional flow section with a `mermaid` sequence diagram.
8. `# Capabilities` — one `## <Verb> <Noun>` section per command, each with
   `### … Invocation` (`#### … Arguments Schema`, `#### … Invocation Example`)
   and `### … Receipt` (failure list, success rule, `#### … Receipt Schema`,
   `#### … Receipt Example`).
9. Protocol-level behaviour sections (`# Routing`, `# Retrieval`, …) after
   the capabilities when the protocol has rules beyond individual commands.
10. Reference-style link definitions at the very end, one per line, local
    specs first (`./blob.md#add-blob`), external URLs after.

## Conventions

- **Schemas** are IPLD Schema in an `ipldsch` block, immediately followed by
  a collapsed `<details><summary>Go syntax</summary>` block holding the Go
  struct with its `cborgen` tags. DIDs and CIDs are `String` in IPLD schema;
  the Go type says `did.DID` or `cid.Cid`. Comment schema fields with `#`.
- **Failures** are named inline where the rule is stated, e.g. _(error name `Foo`)_.
  Receipt sections list them as a numbered
  "Invocation MUST fail if any of the following is true" list, then
  "Invocation MUST succeed otherwise" and what `out.ok` carries. Error names
  match the `*ErrorName` constants in libforge or the service.
- **Examples** are `jsonc` showing the invocation payload with the signed
  envelope elided. Mark the task link with a `// "/": "bafy.."` comment on
  the opening brace, comment each argument, and use placeholder identifiers
  (`did:key:zAlice`, `did:web:upload.example.com`, `bafy..dlgAlice`). Receipt
  examples are `/ucan/assert/receipt` invocations whose `ran` refers back to
  the example task. Include a receipt example only when the success value has
  content or the shape is not obvious.
- **Cross-references** use reference-style links defined at the end of the
  file. Link to a heading anchor in another spec (`./provider.md`,
  `./blob.md#space`) and check that the heading exists.
- **Normative language** uses RFC 2119 keywords in capitals. Prose is plain
  and declarative; no history ("previously", "now", "was renamed"), no
  version-relative language, no editorial asides about the drafting.
- **Naming** says Forge and the service role ("upload service", "storage
  node", "S3 gateway"), and uses concrete product names (Sprue, Piri, Hilt,
  Ingot) only where the spec is about that deployment, as `s3.md` does.
- **Status** follows the README ladder. A new spec is `draft` until the
  protocol is implemented end to end, then `reliable`. Update the badge and
  nothing else when status changes.

## Adding a spec

1. Write `<name>.md` following the structure above.
2. Add a bullet to `README.md` under Overview, alphabetical by title, with
   the same one-line description style as its neighbours.
3. Verify every internal anchor and every claim against the implementing
   repo.

Commit messages follow `spec: <name>` for a new spec and `fix: …` for
corrections; the user commits and pushes.
