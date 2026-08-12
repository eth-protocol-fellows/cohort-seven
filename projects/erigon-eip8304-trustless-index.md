# Erigon - EIP-8304: Trustless log and transaction index

Enable wallets and dApps with a block header to verify the chain origin, field consistency, and query completeness of logs without trust in RPC filtering.

## Motivation

Wallets and dApps use centralized RPC services or third-party indexers to obtain transaction history and contract logs. An RPC response may contain genuine logs, yet the client cannot detect missing matches.

A receipt proof can confirm that a log comes from the chain, but it cannot show whether the requested range contains another log absent from the response. Without a way to verify this property, the wallet must trust the RPC service to execute the query as requested.

Existing mechanisms do not solve this problem:

- `receiptsRoot` can prove the presence of a receipt, but it cannot prove the absence of other matching logs in a block range;
- A Bloom filter can show that a block may contain a log, and growth in usage has raised its false-positive rate;
- Client log indexes can speed up queries, but a remote user cannot verify their contents against a block header.

EIP-8304 proposes ordered transaction and log index tables built after block execution, with each table root written to a system contract. A wallet or light client with an execution block header can verify:

1. the presence of the table root in the Ethereum state of that block;
2. the membership of each log returned by the RPC service in the corresponding table;
3. the absence of omitted logs in the query range.

As part of Pureth, this project will implement a verifiable access layer that lets light clients, wallets, and applications treat RPC services and indexers as untrusted providers of data and proofs, not as trusted databases.

## Project description

This project will implement EIP-8304 in Erigon, so the client can produce a common transaction and log index from block execution and write deterministic table roots to a system contract. The implementation will become part of Erigon block processing, data recovery, and testing flows.

The work has four parts:

1. Implement self-contained protocol primitives that generate entries from ordered transactions and receipts, with sorting, hashing, an SSZ root, five table levels, and a merge schedule;
2. Integrate table computation into Erigon post-transaction processing and execute the system contract call in the corresponding block, so block import and block building produce the same `stateRoot`;
3. Implement a minimal hot table store for recent table data, with support for restart and bounded reorg/unwind;
4. After the preceding implementation reaches stability, add a proof PoC for one bounded query condition to verify entry membership and the absence of omitted matches in the requested range.

The design separates consensus computation from query proofs. Consensus computation uses protocol-defined transactions, receipts, and block data. It cannot depend on `LogAddrIdx`, `LogTopicIdx`, the receipt cache, Commitment History, or pruning flags. Nodes must produce the same table roots and `stateRoot` with or without local indexes or historical proof data.

Query proofs build on table roots stored in the system contract. Erigon log indexes can help find candidate results, and Commitment History can provide historical state proofs when the data exists. These features serve query and proof generation; they do not take part in table root computation or change block validity.

### Relationship to other projects

Before the shift to EIP-8304 and Erigon, I studied the [implementation of EIP-7745 in Geth](https://hackmd.io/@zBK5wwtLTrqYmlLNZd8CPA/Sk39wAi-Gl). This work provided insight into multilevel log indexes, background merges, and query acceleration. The FilterMap structure in EIP-7745 differs from the definition in EIP-8304, so this project will not copy its data format or consensus integration model.

This project does not implement EIP-7807, SSZ execution blocks, or a general SSZ Query Language. These designs may compose with EIP-8304 in the future, but they fall outside the scope of this project.

### Project scope

The project starts with reference fixtures, protocol primitives, five table levels, the system contract update, and a minimal Erigon hot table store with recovery tests. Once these components reach stability, work will continue on single-condition query proofs, a local verifier, parallel execution, and sync-path validation as the schedule permits. The goal is an implementation that runs, passes tests, and supports review. The project does not aim to cover every proof interface or storage optimization within the available time.

Items outside the scope include mainnet activation, RPC standardization, a ZK history index contract, complete snapshot/P2P distribution, long-term history serving, cross-chain applications, and a second execution client.

The verifier proves a result against an execution block header supplied by the caller. The caller remains responsible for confirming that the header belongs to the canonical chain and has the required safe or finalized status.

## Erigon engineering changes

The existing `eth_getLogs` path remains unchanged. Erigon uses `LogAddrIdx` and `LogTopicIdx` to find candidate transactions, reads or regenerates receipts from `RCacheDomain`, applies exact filtering, and returns logs. This path improves the speed of standard log queries and takes no part in EIP-8304 table computation.

EIP-8304 uses a separate path. After execution of all transactions in a block, Erigon builds entries defined by the specification from ordered transactions and receipts, creates the L0 table, and merges higher-level tables when the schedule requires a merge. A system contract call writes due table roots to Ethereum state. The hot table store keeps the related table data for use by the proof generator.

My article [Implementing EIP-8304 and SSZ in the Execution Layer](https://hackmd.io/@zBK5wwtLTrqYmlLNZd8CPA/rJpY1JOQMl) examines this path. SSZ root computation is not the main challenge. The engineering work lies in safe integration with the state transition and sound handling of background merges, reorgs, crash recovery, and disk I/O. These concerns form the focus of validation work in Erigon.

The following table presents the planned code changes. Their placement may change during implementation and requires confirmation from Erigon maintainers.

| Area | Candidate locations | Main work |
|---|---|---|
| Fork configuration | `execution/chain/chain_config.go`, `execution/chain/spec/config.go` | activation, deployment parameters, and chain config |
| Protocol primitives | a new package under `execution/protocol` | entry encoding, sorting, SHA2-256, SSZ root, merge, schedule, and fixtures |
| State transition | `execution/protocol/rules/merge/merge.go` | compute due roots and execute the system call in the shared post-transaction `Finalize` path |
| Table lifecycle | `db/kv`, `db/state/statecfg` | minimal hot storage, progress, restart, and bounded unwind/rebuild |
| Proof PoC | a new proof package | bounded single-condition table proof and local verifier |
| Testing | Erigon tests and EELS-compatible fixtures | differential, restart, reorg, and focused configuration tests |

## Specification

### 1. Deterministic table root

The project implements a pure computation module with no dependency on the Erigon database. Its inputs are ordered transactions, receipts, and required block metadata. Its outputs are encoded entries, leaf hashes, the entry count, and `table_root`.

The implementation follows the selected revision of EIP-8304:

- seven entry types;
- big-endian integer encoding;
- lexicographic ordering of encoded entries;
- SHA2-256 entry hashes, not Keccak-256;
- an SSZ list root that commits to the entry count;
- the table for block N uses the block entry for block N-1;
- each transaction entry contains the `cumulativeLogCount` before that transaction;
- each log index is relative to its transaction receipt, not the whole block.

The module must not read `txNum`, existing log indexes, the receipt cache, Commitment History, pruning flags, or snapshots. Erigon, the reference calculator, and shared fixtures must match for every intermediate result.

### 2. Block processing and multilevel tables

EIP-8304 requires complete receipts, so the system contract update must occur after execution of all transactions and before production of the final `stateRoot`.

The plan uses the shared `Finalize` path, so block import, block building, and serial execution call the same logic. Tests for consistency with parallel execution follow this integration.

After execution of all transactions in each block, Erigon builds the L0 table and checks whether the block requires a higher-level table root. For each due table, the client may use a verified precomputed result. If that result is missing, corrupt, or tied to a chain outside the canonical chain, the client must regenerate it. The client calls `INDEX_CONTRACT_ADDRESS` from `SYSTEM_ADDRESS` and writes the table root to Ethereum state.

The implementation covers five table levels spanning `1, 4, 16, 64, 256` blocks and submits each table root at the time defined by EIP-8304. Background merges reduce computation during block processing but have no bearing on the result.

### 3. Table data lifecycle

The project starts with a minimal hot table store that supports unwind/rebuild and stores:

- `first_block`, `table_size`, and the covered range;
- canonical block hashes;
- ordered entries or equivalent proof material;
- entry count, `table_root`, and merge progress.

After a reorg, all tables that cover an unwind block and all background results from the old chain must become invalid. The proof service must not read partial writes, data from an old chain, or data without confirmed canonical metadata.

After sync, Erigon may need data from recent blocks to compute the next due table root. This data must remain available without `--prune.include-receipts`. Nodes with different configurations need the same data source to obtain the same result. If required data is missing, the node must stop and report an error.

### 4. Query proofs and independent verification

EIP-8304 defines the table root but does not define the RPC response or proof serialization for this project. The first version uses a local proof format and test entry point, with no dependency on RPC design.

The proof PoC supports one bounded block range and one non-wildcard address or topic condition. It provides:

- an execution block header and a system-contract account/storage proof;
- table parameters, entry count, roots, and complete range coverage;
- inclusion or non-inclusion data;
- required consecutive leaf indexes and boundaries on both sides;
- authenticated block, transaction, and log positions.

The independent verifier returns two values:

```text
proofValid
queryComplete
```

The first value states whether the proof is valid against the given header. The second states whether the declared range contains no omitted result. Complex Boolean filters and pagination enter the work after the single-condition proof passes negative tests.

The table root does not cover the complete `log.data`. If time remains, the project will examine receipt proofs, an experimental RPC, and proof composition.

### 5. Deliverables

The project will produce EIP-8304 reference fixtures, Erigon primitives, state-transition integration, a minimal hot table store, tests, design notes, and reproduction steps. The code will use PRs or draft PRs sized for review. If the schedule and client integration leave time, work will continue on query proofs, a local verifier, parallel execution tests, and sync-path tests.

## Roadmap

The week numbers follow the Cohort 7 schedule. Week 6 starts on 2026-07-27, and Week 21 is the final project week.

### Milestone 0: Lock scope and specification (Weeks 6–7)

I will resolve specification and client-integration questions that can change the implementation direction, so consensus logic does not need a redesign after coding starts. The focus is the implementation basis, cross-client test data, and Erigon integration boundaries.

- Pin EIP-8304 revisions;
- Align shared fixtures with the Geth and Reth projects;
- Resolve open questions about the SSZ list, empty tables, the system contract, and the proof format;
- Confirm the boundaries for `Finalize`, minimal storage, and one sync path with Erigon maintainers.

Acceptance: no unresolved assumption can cause a silent change to `stateRoot`.

### Milestone 1: Fixtures and core module (Weeks 8–10)

This phase implements the table computation module without an Erigon database dependency and uses shared fixtures to establish the expected result of each step. It serves as the common basis for client integration and cross-client comparison.

- Implement the reference calculator and worked-example fixtures;
- Implement entry encoding, sorting, hashing, the SSZ root, the L0 builder, merge, and schedule;
- Compare encoded entries, leaf hashes, the entry count, and roots.

Acceptance: Erigon and the reference calculator produce the same results for all fixtures.

### Milestone 2: State-transition integration (Weeks 11–13)

This phase connects the validated table computation logic to Erigon block processing and makes table roots part of Ethereum state. The focus is consistent logic and output across block execution entry points.

- Add an experimental fork configuration and deployment parameters;
- Integrate L0 and the system contract call into the shared post-transaction path;
- Add all higher-level due-root calculations;
- Cover import/building and serial execution.

Acceptance: integrated execution paths and local configurations produce the same consensus result.

### Milestone 3: Table lifecycle and recovery (Weeks 14–16)

This milestone addresses storage and recovery of table data, so a node can continue block processing after a restart or reorg. The implementation must avoid data from an old chain or incomplete writes and support table regeneration when required.

- Implement the minimal hot table store and progress tracking;
- Implement the deterministic deadline fallback;
- Complete restart and bounded reorg/unwind/rebuild support;
- Record focused local performance baselines for L0 construction and four-way merge using fixed fixtures, report the observed cost of the bounded rebuild test without turning it into a comprehensive benchmark suite.

Acceptance: restart and bounded reorg tests confirm that the node does not read table data from an old chain or incomplete table data. At this point, the implementation should run as a standalone artifact and support review.

Correctness and recoverability are the required outcomes of this milestone. Performance work is limited to focused local baselines for L0 construction and four-way merge using fixed fixtures, a pinned Erigon commit, and documented hardware. The bounded rebuild test will record its input size and elapsed time. Background merging, full-node profiling, sustained-load testing, and cross-machine comparisons are optional and will move to the final integration buffer if recovery work takes longer than expected.

### Milestone 4: Query proofs and execution-path validation (Weeks 17–18)

This phase uses the preceding implementation to test the query capability of EIP-8304 and adds coverage for other Erigon execution paths. Its scope depends on progress in the preceding milestones and will not come at the cost of correctness in existing work.

- Based on progress in the preceding milestones, implement bounded single-condition inclusion and empty-result proofs, with a local verifier and negative tests;
- Test parity with parallel execution and select one sync path for a bootstrap test;
- If integration work from a preceding milestone remains open, use this phase to complete the implementation, recovery tests, and review comments.

Acceptance: the existing implementation remains complete and usable, with progress on the proof PoC and execution-path tests as the schedule permits.

### Milestone 5: Integration, review, and schedule buffer (Weeks 19–21)

The final phase turns the code into an artifact that runs, supports reproduction, and is ready for review, with attention to issues found through tests and review. This period absorbs engineering delays from preceding milestones.

- Avoid expansion of the feature scope;
- Address maintainer and mentor review;
- Fix integration issues and complete focused tests and required performance measurements;
- Publish code, fixtures, reproduction steps, measurements, and known limitations.

Acceptance: reviewable Erigon code and a reproducible demonstration exist. Open work has a stated status and path for continuation.

The project will proceed through these tasks within the available schedule without reducing time for tests, review, or the final three weeks of integration in exchange for more features. If a milestone slips, later work will move or shrink, and the final report will record the completed work and continuation plan.

## Possible challenges

- The specification has Draft status. `INDEX_CONTRACT_ADDRESS`, deployment data, SSZ details, and proof serialization may change. Pinned revisions and intermediate fixtures limit the impact.
- Integration points require maintainer confirmation. Code reading cannot serve as the sole basis for decisions about `Finalize`, parallel execution, the storage model, and sync bootstrap.
- The sync data source has open questions. The receipt cache is an optional feature, and recent table inputs need a data path that does not depend on this configuration.
- Derived data must track the canonical chain. Combinations of `txNum` progress, block ranges, restarts, reorgs, and background tasks can expose data from an old chain.
- Query completeness can expand the scope. A single-condition proof involves coverage and boundary semantics, so bounded queries have priority; work on OR, wildcard, pagination, and large-range queries depends on available time.
- Client integration carries the main schedule risk. If the boundaries around `Finalize`, storage, or sync exceed the complexity found during research, the project will give priority to serial execution and the minimal hot store and will not expand RPC, immutable storage, or sync-mode coverage.
- The upstream review schedule sits outside project control. Completion rests on reviewable PRs, reproducible fixtures, tests, and clear design notes; it does not require a merge during the Fellowship.
- The relationship between EIP-8304 and the broader Pureth roadmap may evolve. EIP-7919 currently references EIP-7745 and does not yet identify EIP-8304 as a component.

## Goal of the project

The project aims to enable Erigon to generate EIP-8304 tables with deterministic results without reliance on local indexes and to write the correct roots to the system contract during block processing. On this foundation, work during the Fellowship will cover query proofs and validation of more Erigon execution paths as time permits.

The project results will be assessed against these outcomes:

1. Erigon and the reference calculator produce matching entries, hashes, merge results, and roots for the same fixtures;
2. Block import/building and serial execution produce the same `stateRoot`;
3. The receipt cache, Commitment History, and pruning flags have no effect on the consensus result;
4. The minimal hot table store supports restart and bounded reorg/unwind without reads from old-chain data;
5. The code, tests, fixtures, measurements, known limitations, and open questions form a reviewable PR or draft PR and a final report.

After completion of state-transition integration and data recovery, the project will use the remaining time for single-condition inclusion/empty-result proofs, a local independent verifier, parallel-execution parity, and one sync bootstrap path. Historical state proofs, receipt proofs, an experimental RPC, TypeScript/Viem, and an ERC-20 end-to-end demonstration will not take priority over the work above.

## Collaborators

### Fellows

- [Ray](https://github.com/rayjun): proposal, Erigon implementation, proof PoC, and documentation
- [Skas](https://github.com/Skanislav): client-side verification, Viem integration

### Mentors

- [Tamaghna Choudhuri](https://github.com/RazorClient)
- [Alex Sharov](https://github.com/AskAlexSharov)

## Resources

- [EIP-8304](https://eips.ethereum.org/EIPS/eip-8304)
- [EIP-8304 specification PR](https://github.com/ethereum/EIPs/pull/11811)
- [EIP-8304 discussion](https://ethereum-magicians.org/t/eip-8304-trustless-log-and-transaction-index/28824)
- [Erigon](https://github.com/erigontech/erigon)
- [Execution specs / EELS](https://github.com/ethereum/execution-specs)
- [Implementation of EIP-7745 in Geth](https://hackmd.io/@zBK5wwtLTrqYmlLNZd8CPA/Sk39wAi-Gl)
- [Implementing EIP-8304 and SSZ in the Execution Layer](https://hackmd.io/@zBK5wwtLTrqYmlLNZd8CPA/rJpY1JOQMl)
- [Erigon log indexes and EIP-8304 research](https://hackmd.io/@zBK5wwtLTrqYmlLNZd8CPA/HJlN794rGl)
- [Erigon SSZ execution blocks proposal](https://github.com/eth-protocol-fellows/cohort-seven/blob/main/projects/erigon-ssz-execution-blocks-eip-7807.md)
- [Erigon federated history proposal](https://github.com/eth-protocol-fellows/cohort-seven/blob/main/projects/erigon-federated-history-network.md)
- [EIP-4788](https://eips.ethereum.org/EIPS/eip-4788)
- [EIP-2935](https://eips.ethereum.org/EIPS/eip-2935)
- [EIP-6466](https://eips.ethereum.org/EIPS/eip-6466)
- [Ethereum package / Kurtosis](https://github.com/ethpandaops/ethereum-package)
- [Hive](https://github.com/ethereum/hive)
