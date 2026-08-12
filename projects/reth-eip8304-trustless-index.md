# EIP-8304: Trustless Log and Transaction Index in reth

Making log and transaction lookups trustlessly provable by implementing EIP-8304 in reth.

- Author: mzf11125 ([hackmd.io/@mzf11125](https://hackmd.io/@mzf11125), [github.com/mzf11125](https://github.com/mzf11125))
- Target client: reth
- EIP: [EIP-8304](https://eips.ethereum.org/EIPS/eip-8304) (Draft, Core, Standards Track), successor to [EIP-7745](https://eips.ethereum.org/EIPS/eip-7745), requires [EIP-4788](https://eips.ethereum.org/EIPS/eip-4788)
- EPF project area: Pureth: Trustless Log Index (mentors Etan Kissling and Tamaghna Choudhuri)
- Implementation branch: [paradigmxyz/reth pull request 26528](https://github.com/paradigmxyz/reth/pull/26528)

## Motivation

Emitting logs is cheap relative to writing state, but the protocol offers no efficient way to look their contents back up. The original per-block bloom filters are now oversaturated and effectively useless, so applications fall back on centralized indexers whose answers cannot be verified. Relying on trusted sources for the output of smart contracts undercuts the purpose of keeping those contracts on a trustless ledger, and the problem grows as throughput grows.

EIP-8304 proposes an indexing mechanism cheap enough to update every block while still allowing reasonably sized query proofs, restoring trustless provability across the stack, from an application signing a transaction to the same or another application verifying the events that transaction produced. Prototyping it in reth tests the design against a production-grade codebase and gives the community an independent implementation alongside the author's reference lineage in geth.

A reth-specific consideration this project takes on directly: in reth the system-call machinery (EIP-4788 beacon root, EIP-2935 blockhash history) is orchestrated in-tree via `apply_pre_execution_changes` but implemented in the external `alloy-evm` crate. A core question is therefore whether EIP-8304's system contract and index-generation hook belong in reth's tree or upstream in `alloy-evm`. This boundary is treated as a first-class open question.

## Project description

reth, like most execution clients, has no protocol-level, trustless mechanism for proving log and transaction lookups. Nodes keep private indexes for their own queries, and end users depend on centralized providers that cannot prove the correctness of their results. EIP-8304 addresses this by storing the root hashes of fixed-depth binary structures called index tables in a system contract as part of block processing, which makes log and transaction lookups efficiently and trustlessly provable.

This project implements EIP-8304 in reth. The foundational, execution-independent layer is already built and tested as a self-contained crate. The remaining work is the client integration (fork gating, the block-processing hook, and the system contract) and the query-proof layer. Because EIP-8304 is an early-stage Draft with unresolved fields, the design is refined against implementation findings and against coordination with the mentors, the reth maintainers, and the EIP author.

### Relationship to the Pureth project

The cohort-seven project idea "Pureth: Trustless Log Index," proposed by Etan Kissling and Tamaghna Choudhuri, targets the trustless log index described by EIP-7745 and the 7745b proposal, with an end goal of a Geth and Nimbus interoperability devnet. This project takes the same problem into reth and onto EIP-8304, the author's successor to EIP-7745, which reframes the index as a simpler design that commits table roots through a system contract.

Two deliberate differences from the parent project idea, both to be confirmed with the mentors before scope is locked:

1. EIP-8304 rather than 7745 or 7745b. EIP-8304 is the newer proposal by the same author and is expected to supersede 7745. Building on it aims for longer-term relevance if it becomes the adopted approach.
2. reth rather than Geth or Nimbus. The Pureth work already runs a reth track ("[Pureth in Reth](./pureth-in-reth.md)," the SSZ execution blocks and SSZ query language effort in a shared reth fork). An EIP-8304 implementation in reth is intended to complement that track rather than stand apart from it.

This project therefore sits inside the Pureth: Trustless Log Index area, next to the sibling SSZ tracks. Pureth is two components by design. The SSZ execution blocks and query language work belongs to Arsh and Parth Singh, and the Trustless Log Index is the component this project fills.

#### Overlap and coordination

There is no overlap in what the two reth projects build, and the distinction holds even though both use SSZ, read logs, and target reth.

The question each answers is different. The SSZ execution blocks work authenticates a single field inside a receipt the verifier has already identified, for example proving that a given log carries a given address. This project is the discovery layer. It finds which blocks and transactions a topic or address appears in across a range, and proves the returned set is complete. Field authentication against a known object versus discovery across many blocks.

The layer each touches is different, which is the strongest point. The SSZ execution blocks work is explicit that existing canonical MPT roots stay unchanged. It is an RPC and provider layer serving proofs over data that already exists, with no consensus change. This project is a Core EIP that commits new state, the index table roots, into a system contract during block processing. One side touches RPC, providers, and primitives, the other touches block execution and a system contract.

Even the SSZ structures differ. The SSZ execution blocks work builds a tree of the receipt object so a verifier can walk into a field. This project builds a list of index-entry hashes so a verifier can prove an entry exists in a sorted table. The same hashing tools over different shapes.

The only shared surface is small and unavoidable. Both read the same reth receipt and log primitives, and both use the same SSZ hashing libraries. This project does not reuse the receipt schema from the SSZ execution blocks work, because its entries are their own encoding hashed into their own tables. The coordination point is therefore to share receipt-primitive access and SSZ tooling in the common reth fork rather than duplicate them.

The two layers also compose. Discovery finds the receipt, and the SSZ query layer proves a field inside it, which makes them complementary halves of the same Pureth goal. A separate FilterMaps effort, coordinated earlier with Chronos, is deconflicted by agreement and is not a cohort-seven committed project, so it creates no intra-cohort overlap.

## Specification

### Current status

A self-contained foundation crate is implemented on branch `feat/eip8304-trustless-index` in [github.com/mzf11125/reth](https://github.com/mzf11125/reth/tree/feat/eip8304-trustless-index), at `crates/ethereum/eip8304/`. It deliberately covers everything that does not touch the execution pipeline, which keeps the harder integration work cleanly separated and independently testable.

Built:

- `constants`: table sizes [1, 4, 16, 64, 256], entry type ids, the system-call gas limit, and the index-contract bytecode and address placeholders.
- `entry`: `IndexEntry` for all seven entry types, big-endian binary encoding, lexicographic ordering, and per-block entry construction from receipts. The three subtle spec rules are handled: the delayed parent block entry, the cumulative log count on transaction entries, and per-transaction log indexing.
- `table`: `IndexTable` with an SSZ `hash_tree_root` over the per-entry hashes plus mix-in-length, and a k-way merge of adjacent sorted tables.
- `store`: `IndexTableStore` with per-level generation scheduling and delays, four-into-one merging, and pruning.
- `builder`: entry construction and set/get calldata encoding for the system contract.
- `tests`: 26 passing (18 unit, 8 integration), up to date with `paradigmxyz/reth` main.

Known issues being addressed:

- The embedded index-contract bytecode needs correcting. It will be derived by stripping the constructor prefix from the EIP's deployment input rather than hand-transcribed, with a test that pins the constant to the spec runtime.
- The SSZ table-root construction currently assumes the list limit equals the entry count. This is the literal reading of the spec, but it changes the root and needs confirmation against a reference or from the EIP author.
- A worked-example conformance test (the spec's blocks 40 to 43) should pin the exact binary encodings and the resulting root.

Deferred until fork gating and `alloy-evm` coordination:

- The execution-pipeline hook that generates tables during block processing and issues the set system call.
- The finalized `INDEX_CONTRACT_ADDRESS`, still TBD in the EIP.
- The `alloy-hardforks` fork variant and a `ChainSpec` `eip8304_activated` method.

### Implementation approach

The project follows an implementation-first and least-invasive approach, and the foundation-first sequencing above is a direct result of it. By building the pure data-structure layer first and deferring everything that couples to the execution pipeline, the `alloy-evm` boundary becomes a single well-defined next step rather than a tangle to fight from day one. Remaining changes are divided into small, reviewable pull requests so the mentors and reth maintainers can give feedback as the work develops.

### Remaining work and areas of investigation

- reth's pre-execution hook (`apply_pre_execution_changes`) and the system-call path, including the `alloy-evm` boundary.
- The existing EIP-4788 and EIP-2935 system contracts as the closest structural analogs, and whether the 8304 contract belongs in reth or upstream in `alloy-evm`.
- Fork gating behind an experimental flag, since no fork is allocated yet.
- Query-proof construction, including inclusion and non-inclusion (range) proofs, and a verification path both on-chain via the contract get interface and off-chain against a state witness.
- reth's storage layer (MDBX) for the node's own maintained tables and per-level ring buffers.
- Coordination with the `eth_getLogs` and FilterMaps indexing effort ([reth issue 16999](https://github.com/paradigmxyz/reth/issues/16999)) and the Pureth-in-reth track to avoid duplication.

### Scope

In scope:

- Single-block index-table generation from receipts, with canonical encoding and ordering (built).
- SSZ table-root hashing and multi-level merging (built, pending the SSZ-limit confirmation).
- A system contract exposing the get and set interface and storing roots in per-level ring buffers, integrated into block processing in the style of EIP-4788 and EIP-2935.
- Construction and verification of at least one query proof, inclusion and non-inclusion, against a stored root.
- Focused unit and integration tests per stage.
- Documentation of the integration points, constraints, and any deviations discovered during implementation.
- Small, reviewable pull requests coordinated with the mentors and reth maintainers.

Out of scope:

- The ZKP-backed history index contract for million-block tables, a separate and technically independent effort.
- Reprocessing full historical chain data or building alternative index contracts.
- Cross-chain messaging applications built on top of the proofs.
- Getting EIP-8304 adopted into a network upgrade or any consensus-level finalization.
- Production-grade performance optimization.
- The fully stateless, witness-based table-merging path.

### Deliverables

- A foundational crate generating index tables, hashing roots, and merging tables (delivered).
- A corrected, spec-pinned index-contract bytecode and a worked-example conformance test.
- A system contract (get and set plus ring-buffer root storage) integrated into reth's block processing, with a documented decision on whether it lives in reth or `alloy-evm`.
- A demonstrated, verifiable query proof (inclusion and non-inclusion) against a stored root.
- Tests covering each stage.
- Documentation of reth's integration points, trust assumptions, and known limitations.
- A design note recording where the EIP is under-specified and how the implementation resolved it, fed back to the author.

### Development and review strategy

Work is submitted through small pull requests rather than one large implementation. A possible sequence for the remaining work:

1. Corrected, spec-pinned contract bytecode and worked-example conformance tests.
2. Fork gating behind an experimental flag.
3. System contract get and set with ring-buffer storage, placed per the `alloy-evm` decision.
4. Block-processing integration at the single-block level.
5. Multi-level merging driven from block processing.
6. Query-proof construction and verification.
7. Documentation and integration tests.

This sequence is not fixed and will be adjusted based on findings and mentor and maintainer feedback.

## Roadmap

Week numbers are cohort weeks. Week 6 is the week of 27 July 2026 and week 21 is the final week in mid November, which leaves 16 weeks of runway. The ranges are estimates rather than fixed dates, because milestone 2 depends on the `alloy-evm` placement decision and on mentor and maintainer feedback.

| Milestone | Target weeks | Main deliverable |
| --------- | ------------ | ---------------- |
| 1. Foundation crate | 6 to 7 | Spec-pinned contract bytecode and worked-example conformance vectors |
| 2. System contract and block processing | 8 to 12 | `alloy-evm` placement decision, fork gating, single-block table generation, set system call, first integration PRs |
| 3. Merging and query proofs | 13 to 17 | Multi-level merging driven from block processing, inclusion and non-inclusion query proofs, verification path |
| 4. Documentation and integration | 18 to 21 | Design note, trust assumptions and limitations, feedback to the EIP author, final report |

### Milestone 1, weeks 6 to 7 (largely complete): foundation crate

Entry extraction and encoding, table-root hashing, merging, the table store with scheduling, and set/get calldata helpers, with unit and integration tests. Remaining within this milestone: fix the contract bytecode, pin it to the spec, and add the worked-example conformance vectors.

### Milestone 2, weeks 8 to 12: system contract and block-processing integration

Resolve the reth-versus-`alloy-evm` placement question, add fork gating behind an experimental flag, wire single-block table generation into the block-processing path, and issue the set system call. Submit the first integration PRs.

### Milestone 3, weeks 13 to 17: merging and query proofs

Drive multi-level merging from block processing with the delayed scheduling, then construct inclusion and non-inclusion query proofs and a verification path. Add integration and negative tests.

### Milestone 4, weeks 18 to 21: documentation and integration

Document the final approach, trust assumptions, and limitations. Update the design note. Feed the resolved open questions back to the author and the Ethereum Magicians thread. Address review feedback and prepare for integration where feasible. Weeks 20 and 21 are held for project wrap up, the final development update, and the project showcase.

## Possible challenges

- EIP-8304 is an early Draft with TBD fields (system-contract address, deployment transaction), so the specification may shift under implementation.
- The SSZ list-limit ambiguity in the table-root construction. Only a reference implementation or the author can settle it, and it changes every root.
- Whether the pivot from 7745 to 8304 and from Geth/Nimbus to reth aligns with how the mentors intend to run the Pureth: Trustless Log Index project. This needs their sign-off early.
- The `alloy-evm` boundary: whether the system contract and system-call plumbing belong upstream in `alloy-evm` or in reth.
- Overlap with the FilterMaps and issue 16999 indexing effort and with the Pureth-in-reth track.
- Storage cost and layout for node-maintained tables and per-level ring buffers in MDBX.
- Realistic query-proof sizes. The specification expects roughly 30 to 50 individual table proofs per query.

## Goal of the project

A working reth prototype that generates EIP-8304 index tables during block processing, commits their roots through a system contract, and can produce and verify a trustless query proof for a log or transaction lookup, together with a clear account of reth's integration points and the open issues discovered along the way.

## Collaborators

### Fellows

Worked on solo, coordinated with the fellows on the [Pureth in Reth](./pureth-in-reth.md) track to avoid duplication.

### Mentors

Etan Kissling and Tamaghna Choudhuri, the proposers of the Pureth: Trustless Log Index project idea. Their sign-off on the pivot to EIP-8304 and to reth is being sought early.

## Resources

- [EIP-8304](https://eips.ethereum.org/EIPS/eip-8304) specification and the Ethereum Magicians discussion thread.
- [EIP-7745](https://eips.ethereum.org/EIPS/eip-7745) and the 7745b gist (the basis of the parent Pureth project idea).
- [Pureth: Trustless Log Index project idea](./project-ideas.md) (Etan Kissling and Tamaghna Choudhuri).
- [Pureth in Reth](./pureth-in-reth.md) project (the shared reth fork for the SSZ tracks).
- [EIP-4788](https://eips.ethereum.org/EIPS/eip-4788) (beacon root) and [EIP-2935](https://eips.ethereum.org/EIPS/eip-2935) (blockhash history) as system-contract references.
- geth filtermaps, the author's reference lineage for the EIP-7745 data structure.
- [reth issue 16999](https://github.com/paradigmxyz/reth/issues/16999) (`eth_getLogs` indexing) for coordination.
- Implementation branch: [paradigmxyz/reth pull request 26528](https://github.com/paradigmxyz/reth/pull/26528).
- Development updates: [hackmd.io/@mzf11125](https://hackmd.io/@mzf11125).
