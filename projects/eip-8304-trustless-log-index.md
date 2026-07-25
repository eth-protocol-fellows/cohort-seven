# EIP-8304: Trustless log and transaction index

Make log retrieval verifiable without trusting an RPC provider.

## Motivation

If you're building a wallet and you want to show someone their token balance or transaction history, you have a few options and none of them are great. You run an archive node, which is expensive and not something a mobile wallet can do. You pay an indexing service like a subgraph, and now you're trusting them to not lie or omit things. Or you try to verify against the block header's logs bloom, which gives you false positives, and downloading every block header to check a single address is wildly impractical anyway.

But the real problem is deeper than convenience. Even when every returned log is genuine, the client cannot tell whether any matching results have been omitted. The `receiptsRoot` in a block header can prove a receipt belongs to a specific block, but it cannot efficiently prove that no other matching logs exist across a range of blocks. The bloom filter can only identify blocks that *may* contain logs and cannot provide verifiable completeness. Users still need to trust the RPC provider to execute the query correctly and return every matching result.

State has storage proofs. If I want to prove some account balance to you, I can give you a Merkle proof against the state root and you can check it. We don't have anything like this for logs.

EIP-8304 fixes this. During block execution the client builds hierarchical transaction and log index tables and writes their table roots into a system contract. Same trick EIP-4788 uses to get beacon roots into the EVM. Since those roots end up in state, anyone who already trusts the state root (which a light client gets from the consensus layer) can pull a log together with a proof and check that it's genuine and that no matching logs were left out. No header format change, no trusted middleman.

This approach doesn't touch the block header at all, and that's the key difference from earlier attempts. EIP-7745 and 7745b replaced `logsBloom` with a `LogIndex` struct in the header. 8304 takes a different path: keep the bloom for backwards compat and add the index on top via the system contract pattern that's already been proven out by 4788 and 2935. Less scary from a consensus perspective. It also means the index is additive — if a client doesn't care about verifiability, nothing changes for them.

There's also some recent work on native UTXOs on Ethereum and proposals like EIP-7811 (Prague EOF) and EIP-7708 (cross-chain state proofs) that would benefit directly from having verifiable logs on-chain.

## Project description

The project implements a full verification path from an execution block header to a verified log object, without following the header-replacement approach of EIP-7745:

```
execution block header
  → state root and system contract account/storage proof
    → EIP-8304 table root
      → index inclusion/exclusion/completeness proof
        → block/transaction entries
          → target block header and receipt proof
            → verified log (address, topics, log.data)
```

EIP-8304 commits to the address, topics, and position of a log, but not directly to `log.data`. Verifying a complete log therefore also requires a block entry to authenticate the target block header, a transaction entry to authenticate the transaction hash and index, and a receipt proof against the target `receiptsRoot`. The project covers this full chain.

There are roughly five work streams. Some already have code, some are greenfield.

**0. Reference spec (execution-specs / EELS)**

Python implementation in the EELS repo so client teams have a common reference to test against. This becomes the ground truth: EELS generates reference fixtures (entries, leaf hashes, table roots) covering empty blocks, fork activation, table merging, invalid roots, and boundary values. If the EIP author ships an executable spec, we integrate it. If not, we write it. Either way this is the artefact that makes sure geth and any other EL produce the same table roots.

**1. Polish the geth PoC**

Vansh already built a working PoC on the `8304` branch. It does index building for levels 0 and 1, posts table roots to the system contract during `PostExecution`, serves `eth_getLogProof`, and has a basic verifier. But it's a proof of concept: fixed 64-byte entries, flat Keccak256 over concatenated bytes instead of SSZ merkleization, everything in memory, and only two table levels active.

The goal is to bring it up to production quality:

- Variable-length entries (38/40/52 bytes) instead of fixed 64
- SSZ merkleization: SHA-256 per-entry hashes, SSZ `List[Hash32]` for table roots
- Proper k-way merge for multi-block tables. Level 1 currently copies the level 0 root as a placeholder instead of merging four sorted tables
- Activate all five table levels (1, 4, 16, 64, 256 blocks)
- File-backed table storage with the 1024-slot ring buffer from the spec
- Fork activation, system-contract deployment, background-merge deadlines, sync recovery, restarts, and reorg rollback

A hard line is drawn between the consensus-critical path and the proof-serving database. The table root calculation during block processing determines the commitment written into the system contract and must match EELS exactly. The proof-index database stores the intermediate data needed to generate query proofs (sorted entries, entry-to-block mappings) and lives outside the consensus path — it can be rebuilt from canonical chain data if needed.

**2. Proof index, RPC, and proof format**

Build a separate proof-index database that supports canonical hash tracking, pruning, rebuilding, and reorg rollback. Implement an experimental RPC, `pureth_getLogsWithProofV1` (versioned because neither the API nor the proof schema are part of the EIP-8304 spec yet).

The RPC must support bounded-range queries, address and topic filters with a limited number of OR conditions, topic wildcards, pagination, and empty-result proofs. Each page is bound to the query, execution block hash, table roots, target blocks, receipt proofs, and coverage range. The query is complete only when contiguous pages reach the end of the requested range.

The proof format covers inclusion, exclusion, and completeness. Inclusion works via contiguous match ranges over sorted entries (the lexicographic sort on `[entry_type, value, block, tx, log]` means all matching entries for a given address or topic naturally cluster together). Exclusion is the degenerate case: matchStart is -1, and the verifier walks the full sorted list to confirm nothing matches. Completeness proofs for multi-condition queries, empty results, and cross-table ranges additionally require predecessor/successor boundary proofs, cross-table range coverage proofs, and the SSZ list entry count.

The MVP scope: bounded block ranges, limited OR conditions, topic wildcards, pagination, empty results. Out of scope: `pending` blocks, subscriptions, `removed` logs, queries with no filter conditions, and blocks beyond the ring buffer window.

**3. Client-side verification (TypeScript + viem)**

This is what makes it real. The plan has two parts:

A standalone `@pureth/log-proof` package that verifies the complete proof chain: storage proofs, index proofs, range completeness, block and transaction entries, receipt proofs, and full log contents including `log.data`. The verifier separately reports whether the proof is valid and whether the query is complete. The caller must independently verify that the block header belongs to the canonical chain and whether it is safe or finalized — a storage proof alone cannot perform those checks.

On top of that, a lightweight viem integration that calls `pureth_getLogsWithProofV1`, runs the verifier, and surfaces the results. We'll add experimental viem actions so the API feels native. Paul Miller's `micro-eth-signer` keeps the dependency footprint manageable.

The stretch is a minimal browser wallet that queries logs from the devnet, verifies them locally, and shows an ERC-20 balance computed from verified `Transfer` events.

**4. Devnet and testing**

Spin up a geth + nimbus devnet with kurtosis. Run it through assertoor: send transactions that emit logs, query proofs, verify them. Wire up a Hive simulator. Testing covers differential root comparison (EELS vs geth vs an independent root calculator), reorgs, restarts, malicious proofs, fuzzing, performance, and resource limits.

The acid test is two independent EL clients producing identical table roots for the same chain history. If geth and another client agree on every root at every level, the spec is correct and the implementation is solid. An independent root calculator (a minimal program that takes a block range and receipts and computes the expected roots) serves as a lightweight tiebreaker.

**The filtermaps overlap**

Geth's `core/filtermaps` package already exists on master. It was written by Zsolt Felföldi, the same person who authored EIP-8304. It builds filter maps, a two-layer structure with SHA256-hashed rows and epoch-based tail pruning. `eth_getLogs` uses it today for fast queries. The `MatcherBackend` interface in that package has a comment that says: "once EIP-7745 is implemented and active, these functions can also be trustlessly served by a remote prover." So the long term plan is probably not two separate index systems. It's figuring out how 8304's verifiability plugs into the filtermaps data structure. For this project we stay focused on 8304 as specified, but we document the integration point.

## Specification

### Index tables

Five levels, exponentially growing block ranges. Each level is a sorted table of index entries. A single-block table is written after processing block N, but its block entry refers to N−1. Higher-level tables are written at `first_block + table_size - 1 + table_size / 4`.

| Level | Table size | Publication delay |
|-------|-----------|-------------------|
| 0 | 1 block | 0 blocks |
| 1 | 4 blocks | 1 block |
| 2 | 16 blocks | 4 blocks |
| 3 | 64 blocks | 16 blocks |
| 4 | 256 blocks | 64 blocks |

### Entry format

Each entry is 64 bytes right now. The spec calls for variable-length encoding (38/40/52 bytes depending on type). We start fixed and migrate.

```
[0]       entry type (1 byte)
[1:33]    index value (32 bytes)
[33:41]   block number (8 bytes)
[41:45]   tx index (4 bytes)
[45:49]   log index (4 bytes)
[49:64]   reserved (15 bytes)
```

Entries sort lexicographically over all bytes. Entries of the same type cluster together, and within a type they sort by value then by block position. Consecutive matching entries land in a contiguous range inside the sorted slice. That contiguous property is what makes the matchStart/matchEnd proof model work and eliminates the need for scattered index lookups.

Seven entry types: block hash, transaction hash, log address, and log topic[0] through topic[3].

Both EELS and geth use big-endian encoding, lexicographic ordering, SHA-256 entry hashing, and SSZ table-root calculation per the pinned EIP-8304 commit. The proof schema carries and verifies the SSZ list entry count.

### System contract

Deployed at `0x0000000000000000000000000000000000008304` (address TBD pending spec finalization). A system call from `SystemAddress` fires during `PostExecution` and writes the table roots into the contract's storage. The slot formula:

```
slot = 1024 * table_size + ((first_block / table_size) % 1024)
```

The contract exposes `get(blockNumber, tableSize)` to read a stored root back. A light client calls `eth_call` or `eth_getProof` against the contract to fetch the trusted root for the block range it cares about. The ring buffer only covers recent blocks (1024 slots × 256 blocks = ~36 days at the largest table level). Long-term history beyond this window needs a separate ZKP-proven external contract and is outside this project's scope.

### Proof chain

The full verification path:

1. **State root to contract storage.** `eth_getProof` against the system contract at the reference head block proves the table root is genuine state.
2. **Table root to index entries.** The proof ships the sorted entries for the queried range. Recomputing the SSZ root and checking against the trusted contract value proves the entries are correct and complete.
3. **Index entries to block/transaction pointers.** Matching entries carry `(blockNumber, txIndex, logIndex)`. A block entry authenticates the target block header hash. A transaction entry authenticates the transaction hash and index.
4. **Block header to receipt.** A receipt proof against `receiptsRoot` in the target block header verifies the full log object: address, topics, and `log.data`. EIP-8304 commits to address, topics, and position but not to `log.data`, so the receipt proof is required for full verification.

Chain-tip blocks that cannot be authenticated through a later block entry or a block header supplied by the caller are excluded from the verified range.

### Inclusion, exclusion, and completeness

**Inclusion proof.** Every entry inside `[matchStart, matchEnd)` matches the filter. No entry outside that range matches. The contiguous sorting property makes this a single range instead of a scattered set of indices.

**Exclusion proof.** The degenerate case: matchStart is -1. The verifier walks the sorted list, confirms nothing matches, and the root ensures the server couldn't have dropped entries to fake an empty result.

**Completeness proof.** For each query branch, the proof selects one non-wildcard condition, proves the full candidate set together with contiguous leaf indexes, predecessor and successor boundaries, the entry count, and cross-table range coverage, and then uses receipts to check the remaining conditions. Multi-condition queries, pagination, and empty results require boundary and range proofs beyond simple inclusion.

### Verification

The verifier:

1. Fetches the trusted `tableRoot` from the system contract (eth_call or eth_getProof)
2. Recomputes the SSZ root from `sortedEntries` and checks it matches, verifying the entry count
3. Checks the match range: everything inside matches, nothing outside does
4. For completeness: verifies predecessor/successor boundary proofs and cross-table range coverage
5. If matchStart is -1, scans to confirm no entry matches
6. Verifies receipt proofs against `receiptsRoot` for each returned log to authenticate `log.data`

The verifier separately reports proof validity and query completeness. The caller must verify that the block header belongs to the canonical chain and whether it is safe or finalized — a storage proof cannot perform those checks.

### What the verifier can and cannot prove

The verifier can prove: the table root is genuine state, the index entries are correct and complete, the matching range boundaries are accurate, and each returned log's address, topics, and data match the on-chain receipt.

The verifier cannot prove: that the RPC provider is available, that the block header is on the canonical chain, that the block is safe or finalized (the caller must check this separately), or that the business meaning of a log (e.g., "this Transfer event means tokens were sent") is what the caller assumes.

## Roadmap

About 14 weeks total. The PoC already exists so this isn't a from-scratch estimate.

### Milestone 0: Ramp-up and scoping (Week 1–2)

Close the open questions before writing any real code.

- Pin the EIP-8304 commit used by the project and record remaining open questions.
- Define the full proof chain from an execution block header and its state root to a verified log, including the relationship between the system-contract proof, table root, index proof, block/transaction entries, and receipt proof.
- Confirm the contract address, storage layout, and deployment signature with the EIP author.
- Define fork activation, system-contract deployment, entry-count semantics, query and pagination semantics, safe/finalized requirements, and the responsibilities of each component.
- Decide the RPC method name and parameter format. Settle on `pureth_getLogsWithProofV1` with a versioned schema.
- Set performance baselines: maximum query ranges, result counts, proof bytes, proof-generation and verification latency, consensus overhead, disk growth, and rebuild time.
- Get a devnet running. Vansh already has a geth binary that deploys the contract at genesis and posts roots.

Risks: EIP-8304 is still a Draft and parameters may change. A storage proof cannot prove a block header is canonical, so the verifier's scope boundary must be clearly stated. Bringing in a second EL client or production-grade RPC optimization too early could blow up the scope.

### Milestone 1: EELS reference implementation (Week 3–5)

Python implementation of index building in execution-specs.

- Implement EIP-8304 entries, the five table levels, table-root calculation, and system-contract update logic.
- Generate reference test data covering empty blocks, fork activation, table merging, invalid roots, and boundary values.
- Use EELS as the comparison baseline for the Geth implementation and the independent root calculator.

Risks: differences in encoding, sorting, hashing, SSZ length handling, or table merging may cause different implementations to generate different roots. Insufficient fixture coverage may miss fork-activation, merging, and boundary issues. If the EIP draft changes, existing fixtures need to be regenerated.

### Milestone 2: Polish geth implementation (Week 6–8)

This is where most of the work lives.

- Variable-length entries (38/40/52 bytes) instead of fixed 64.
- SSZ merkleization: SHA-256 per-entry hashes, SSZ `List[Hash32]` for table roots instead of flat Keccak256.
- Proper k-way merge for multi-block tables. Level 1 currently cheats by copying the level 0 root.
- Activate levels 2, 3, and 4.
- File-backed table storage with the 1024-slot ring buffer from the spec.
- Fork activation, system-contract deployment, background-merge deadlines, sync recovery, restarts, and reorg rollback.
- Keep the consensus-critical root calculation separate from the proof-serving database.
- Compare geth-generated entries, leaf hashes, and roots with EELS fixtures and the independent root calculator at each stage.

Risks: consensus-related logic must match the specification exactly. Background table merging introduces timing, persistence, and reorg-handling complexity. The consensus path must produce a deterministic root on time and must not depend on the proof-serving database.

### Milestone 3: Proof index and RPC (Week 9–10)

- Build a separate proof-index database supporting canonical hashes, pruning, rebuilding, and reorg rollback.
- Implement `pureth_getLogsWithProofV1`, supporting bounded-range queries, filters, pagination, and empty-result proofs.
- Bind RPC proofs to the query, execution block hash, table roots, target blocks, receipt proofs, and coverage range.
- Handle the full proof format: inclusion, exclusion, completeness proofs with predecessor/successor boundaries and cross-table range coverage.

Risks: completeness proofs for cross-block ranges, OR conditions, wildcards, empty results, and pagination are more complex than single-range inclusion proofs. The RPC must limit query ranges, result counts, proof bytes, concurrency, and computation budgets. The proof-serving database must track canonical chain changes without entering the consensus-critical path.

### Milestone 4: TypeScript and viem client (Week 11–12)

- Implement the standalone `@pureth/log-proof` package to verify storage proofs, index proofs, range completeness, block/transaction entries, and receipt proofs.
- Verify the complete log contents, including block and transaction hashes, transaction and receipt log indexes, address, topics, and `log.data`.
- Add lightweight viem actions so the API surfaces naturally in wallet code.
- A controlled ERC-20 demo: mint, burn, and transfer operations all emit `Transfer` logs. Starting from a known balance, the client verifies each page of proofs, calculates the balance change, and cross-checks it against the ending state. (This rule is a demo assumption, not something EIP-8304 guarantees.)

Risks: EIP-8304 does not directly commit to `log.data`, so receipt-proof verification must be implemented correctly. SSZ tooling in JavaScript is minimal — we'll likely need to implement a minimal SSZ library for proof types. If the verifier API is too tightly coupled to the first RPC format, future evolution becomes harder. Edge cases involving malicious proofs are easy to miss and need dedicated negative tests.

### Milestone 5: End-to-end testing and hardening (Week 13–14)

- Use a Kurtosis devnet to demonstrate end-to-end verification of controlled ERC-20 log queries.
- Add differential, reorg, restart, malicious-proof, fuzz, performance, and resource-abuse tests.
- Performance numbers: gas overhead per block for the system call, storage growth rate, proof sizes at each table level, proof-generation and verification latency.
- Prepare technical documentation, test results, known limitations, and a performance report.
- Prepare upstream or draft PRs to geth.

Risks: devnet and end-to-end testing may expose cross-component integration problems late in the project. If the performance targets defined in Milestone 0 are not met, the MVP scope will need to be reduced.

## Possible challenges

**The spec is early.** EIP-8304 was merged into the EIPs repository but remains a Draft. `SYSTEM_ADDRESS` has been defined, while `INDEX_CONTRACT_ADDRESS` and the deployment signature remain TBD. Details like variable-length encoding and SSZ merkleization are not finalized. Mitigation: we pin to a specific spec commit, version the RPC and proof schemas, and sync with the mentors regularly. Any ambiguity that affects the state root must be resolved first.

**PoC performance.** The multi-block merge is O(n) per level. The flat Keccak256 over all entries gets expensive for level 4 tables covering 256 blocks. Everything sits in memory. SSZ merkleization helps with hashing since you can incrementally update the tree instead of rehashing the whole thing. File-backed storage and the ring buffer fix the memory problem. But there are probably surprises we haven't hit yet.

**Implementation consistency.** Differences in encoding, sorting, hashing, SSZ length handling, table merging, or write timing will cause root mismatches across implementations. We retain intermediate entries, leaf hashes, and roots and use EELS fixtures plus an independent root calculator to identify the first point of divergence. `simpleserialize.com` helps debug SSZ serialization differences.

**Query completeness.** This is harder than inclusion. Multi-condition queries, OR conditions, wildcards, empty results, pagination, and cross-table coverage all need boundary and range proofs, counterexamples, and fuzzing. The completeness proof must cover every query branch.

**Ring buffer and history.** The protocol ring buffer only covers recent blocks (~36 days). Long-term history needs a separate ZKP-proven external contract and is outside this project's scope. Block entries are delayed by one block, so chain-tip blocks that cannot be authenticated are excluded from the verified range.

**Background merging.** Multi-block table construction must handle deadlines, disk usage, restarts, and reorgs without affecting the consensus-critical path. The merge cannot block block production.

**The filtermaps overlap.** Geth already has a log index in `core/filtermaps`. 8304 builds a different one. Long term these probably converge: the filtermaps data structure plus 8304-style consensus commitment. For now we implement 8304 as specified and document the integration point.

**Gas cost.** Every block does at least one SSTORE for the level 0 table root. We need hard numbers on the per-block overhead to make the case to validators.

## Goal of the project

By the end of the project, EELS, geth, and the independent root calculator are expected to generate matching encoded entries, leaf hashes, and table roots for the same fixtures. Geth is expected to update the system contract correctly during execution, sync recovery, restarts, and reorgs, while keeping the consensus root independent of the proof-index database.

The RPC is expected to return paginated proofs bound to the query and execution block hash for supported bounded queries and empty results. The TypeScript verifier is expected to verify the complete proof chain and reject omissions, forgeries, duplicates, incorrect ranges, block headers, or receipt roots, expired table slots, incorrect cursors, and modified `log.data`.

The final demo uses a controlled ERC-20 contract where mint, burn, and transfer all emit `Transfer` logs. Starting from a known balance, the client verifies each page of proofs, calculates the balance change, and cross-checks it against the ending state. This rule is a demo assumption and is not guaranteed by EIP-8304.

A devnet with geth committing the correct table roots and serving verifiable log proofs. A client (viem, ideally with a minimal wallet UI) queries logs over RPC, verifies the proof chain against the contract, and displays the result. All without trusting the RPC provider.

## Collaborators

### Fellows

- Vansh Sahay ([@vanshsahay](https://github.com/vanshsahay))

### Mentors

- Tamaghna Choudhuri ([@RazorClient](https://github.com/RazorClient))

## Resources

- [EIP-8304 specification PR](https://github.com/ethereum/EIPs/pull/11811)
- [EIP-8304 discussion on Ethereum Magicians](https://ethereum-magicians.org/t/eip-8304-trustless-log-and-transaction-index/28824)
- [Vansh's PoC on the 8304 branch](https://github.com/VanshSahay/go-ethereum/tree/8304)
- [Implementing EIP-8304 and SSZ in the Execution Layer](https://hackmd.io/@zBK5wwtLTrqYmlLNZd8CPA/rJpY1JOQMl)
- [8304 performance notes](https://gist.github.com/zsfelfoldi/3d22525b5732a9540bb9d9961ed5d238)
- [EIP-7919 (Pureth Meta)](https://eips.ethereum.org/EIPS/eip-7919)
- [Pureth explainer from the Nimbus team](https://blog.nimbus.team/pureth-efficient-complete-data-verification-for-ethereum/)
- [EIP-4788](https://eips.ethereum.org/EIPS/eip-4788): the system contract pattern 8304 follows
- [EIP-2935](https://eips.ethereum.org/EIPS/eip-2935): same deploy pattern
- [EIP-7745 reference implementation](https://github.com/zsfelfoldi/go-ethereum/tree/eip-7745b): Zsolt's prior art, header-based LogIndex
- [EIP-7792](https://eips.ethereum.org/EIPS/eip-7792): Etan's verifiable logs, different approach, good design-space context
- [EIP-6466](https://eips.ethereum.org/EIPS/eip-6466): SSZ receipts
- [execution-specs / EELS](https://github.com/ethereum/execution-specs)
- [go-ethereum](https://github.com/ethereum/go-ethereum)
- [ethereum-package (Kurtosis devnet)](https://github.com/ethpandaops/ethereum-package)
- [Hive](https://github.com/ethereum/hive)
- [assertoor](https://github.com/ethpandaops/assertoor)
- [viem](https://viem.sh)
- [micro-eth-signer](https://github.com/paulmillr/micro-eth-signer)
- [ACDE 239: EIP-8304 segment](https://www.youtube.com/live/Y1r-O9Vdl7I?si=RiuG8FOOxcNJ3LJZ&t=1729)
