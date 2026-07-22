# Erigon SSZ Execution Blocks and EIP-7807

Building the Erigon foundations for proof-friendly, SSZ-native execution blocks.

## Motivation

Ethereum currently uses different serialization and commitment schemes across its two main protocol layers. The Consensus Layer uses SSZ and `hash_tree_root`, while execution blocks are still represented through RLP and Merkle-Patricia trie commitments. This separation adds encoding complexity, makes some execution block data difficult to prove independently, and limits reuse of the proof-oriented tooling already available in the Consensus Layer.

[EIP-7807](https://eips.ethereum.org/EIPS/eip-7807) proposes migrating execution blocks to an SSZ representation. It introduces an SSZ-native execution block root and forward-compatible field and list layouts based on `ProgressiveContainer` and `ProgressiveList`. This would make individual execution block fields easier to prove, bring the Execution Layer and Consensus Layer data models closer together, and provide foundations for more efficient Engine API and verifiable RPC designs.

## Project description

I propose to implement the Erigon-side foundations for EIP-7807 SSZ execution blocks.

The project focuses on Erigon because it already contains substantial SSZ infrastructure in Caplin and its Engine API code, but its canonical execution block model remains RLP/MPT-oriented. The missing piece is an explicit, tested mapping from Erigon execution data to the EIP-7807 layout, together with the progressive Merkleization, root calculation, generalized-index mapping, and proof interfaces required to inspect that layout.

The affected protocol area is the Execution Layer block representation and its boundary with SSZ-based proof and API infrastructure. The initial project will not replace Erigon's current consensus-critical block hash or block import path. Instead, it will build and validate a parallel EIP-7807 root and proof surface that can be reviewed before deeper protocol integration.

The project will first map Erigon's existing `Header`, `Block`, transaction, receipt, withdrawal, execution request, and block access list data to the EIP-7807 `ExecutionPayload` structure. It will then add the progressive SSZ Merkleization support required by EIP-7807, compute an SSZ execution block root from existing Erigon block data, and validate the result with deterministic fixtures and an independent reference implementation.

After root computation is stable, the project will define generalized-index mappings and generate Merkle proofs for selected fields and list roots. A small experimental JSON-RPC inspection interface will demonstrate how local tools and future query layers can retrieve an EIP-7807 root or proof without understanding Erigon's internal data model.

The main deliverables are:

- An EIP-7807-to-Erigon field mapping and implementation boundary.
- Progressive SSZ root and proof helpers required by the EIP-7807 layout.
- An explicit EIP-7807 execution payload view over existing Erigon block data.
- Deterministic EIP-7807 block-root computation with unit and fixture tests.
- Generalized-index mappings and proof generation for selected fields and list roots.
- An experimental root/proof inspection API and a reproducible demonstration.
- Reviewable upstream PRs, technical documentation, and a final EPF report.

## Specification

### 1. Codebase mapping and adapter boundary

The first implementation step is to define an explicit adapter from existing Erigon execution data into the EIP-7807 view. `types.BlockWithReceipts` is a useful candidate input boundary because it can aggregate the block, receipts, execution requests, and block access list data required for root calculation.

The mapping will cover the 18 top-level fields of the EIP-7807 `ExecutionPayload`, including transactions, `receipts_root`, gas and fee containers, withdrawals, `requests_hash`, the block access list, and fork-specific fields. The current RLP block hash and MPT roots will remain available for lookup and comparison, but they will not be reinterpreted as EIP-7807 roots.

Canonical element encodings will be preserved where EIP-7807 requires them. Transactions and receipts will retain their EIP-2718 typed-envelope bytes, withdrawals and the block access list will retain their specified RLP representation, and `requests_hash` will be aligned with `ExecutionRequests.hash_tree_root()` rather than assumed equivalent to Erigon's current header/check paths.

The current Erigon baseline and the intended reuse boundary are:

| Area | Existing Erigon code | Intended role in this project |
| --- | --- | --- |
| SSZ interfaces and Merkleization | `common/ssz`, `cl/ssz`, `cl/merkle_tree` | Reuse existing hashing and tree utilities; add focused progressive helpers without changing existing SSZ behavior. |
| Existing SSZ containers and lists | `cl/cltypes/solid` | Reuse list/container patterns where suitable and add only the progressive structures required by EIP-7807. |
| Canonical execution data | `execution/types/block.go`, `receipt.go`, `hashing.go`, and `eip7685_requests.go` | Provide the source `Header`, `Block`, receipt, withdrawal, request, and block access list data for the adapter. |
| Engine API SSZ transport | `execution/engineapi/engine_types/ssz.go`, `sszrest_wire.go`, and `sszrest_handler.go` | Reuse conversion helpers where semantics match; keep transport encoding separate from EIP-7807 root computation. |

The initial 18-field mapping is grouped into four reviewable areas:

| Mapping group | Representative EIP-7807 fields | Primary Erigon sources | Main Phase 1 confirmations |
| --- | --- | --- | --- |
| Core header fields | `parent_hash`, `miner`, `state_root`, `number`, `timestamp`, `mix_hash` | `Header` and `Block` accessors | Naming aliases and integer/default handling. |
| Progressive list and byte fields | `transactions`, `receipts_root`, `extra_data`, `withdrawals`, `block_access_list` | `Block.Transactions()`, canonical typed receipt bytes, `Header.Extra`, withdrawals, and `BlockWithReceipts` | Canonical raw-byte sources, pre-fork empty behavior, and historical block access list retrieval. |
| Gas and fee containers | `gas_limits`, `gas_used`, `base_fees_per_gas`, `excess_gas` | `Header.GasLimit`, `GasUsed`, `BaseFee`, `BlobGasUsed`, `ExcessBlobGas`, and chain configuration | Exact regular/blob semantics across Amsterdam and Gloas. |
| Fork-specific roots and requests | `parent_beacon_block_root`, `requests_hash`, `slot_number` | `Header.ParentBeaconBlockRoot`, execution requests, and `Header.SlotNumber` | Pre-fork defaults and the canonical `ExecutionRequests.hash_tree_root()` conversion. |

The full field-by-field table, preliminary generalized indices, code-path notes, and unresolved questions are maintained in the [detailed technical design](https://hackmd.io/@jackcc/epf7-project-draft). Values marked preliminary there will not become implementation constants until the current EIP revision and proof convention are confirmed.

### 2. Progressive SSZ support

EIP-7807 depends on the forward-compatible SSZ types defined by [EIP-7495](https://eips.ethereum.org/EIPS/eip-7495) and [EIP-7916](https://eips.ethereum.org/EIPS/eip-7916). I will implement or extend the minimal Erigon helpers needed for:

- Progressive Merkle tree construction.
- `ProgressiveContainer` roots and `active_fields` mix-in handling.
- `ProgressiveList` and `ProgressiveByteList` roots.
- Stable generalized indices as fields or list lengths evolve.
- Merkle branch generation for progressive trees.

These additions should reuse Erigon's existing SSZ and Merkle tree packages where appropriate, but package placement will be confirmed with maintainers before the first implementation PR. Existing normal SSZ behavior used by Caplin must remain unchanged.

### 3. EIP-7807 execution block root

The adapter will construct a typed EIP-7807 execution payload view and expose a clear root-computation path conceptually similar to:

```go
func ExecutionPayloadSSZFromBlock(
    blockWithReceipts *types.BlockWithReceipts,
) (*ExecutionPayloadSSZ, error)

func CalcEIP7807BlockRoot(
    blockWithReceipts *types.BlockWithReceipts,
) (common.Hash, error)
```

The final names and package locations will follow Erigon conventions. The important design property is that conversion and root calculation remain explicit and independently testable. The new root will initially be computed alongside the current RLP block hash rather than replacing consensus-critical behavior.

### 4. Generalized indices and proof format

I will document and test the mapping from EIP-7807 field paths to generalized indices, including the effect of the `active_fields` mix-in on proofs anchored at the final `ProgressiveContainer` root.

The design note currently records both field-tree and final mixed-root generalized indices. For example, `parent_hash`, `transactions`, and `receipts_root` have preliminary field-tree indices `2`, `26`, and `27`, and corresponding mixed-root proof indices `4`, `42`, and `43`. These values are review inputs, not frozen API constants; tests will derive or validate them against the exact EIP-7495/EIP-7807 revision used by the implementation.

A proof result will contain, at minimum:

- The block identifier and current RLP block hash for lookup/debug context.
- The EIP-7807 execution block root.
- The requested field path and generalized index.
- The leaf or subroot value.
- The Merkle branch and the metadata required to verify it.
- The spec revision or fork context used to construct the view.

The first proof targets will be top-level fields and list roots. Proofs for arbitrary transaction or receipt subfields can be follow-up work.

### 5. Experimental inspection API

Once roots and proofs are stable, I will prototype a small developer-facing JSON-RPC interface, for example:

```text
debug_getEIP7807BlockRoot(blockId)
debug_getEIP7807Proof(blockId, pathOrGindex)
```

The exact namespace and method names will be agreed with Erigon maintainers. The API is intended for local testing, fixtures, review, and future query-layer consumers. It is not proposed as a finalized production RPC standard.

If maintainers prefer not to expose JSON-RPC during the fellowship, the same root/proof result will be demonstrated through a local fixture command or test tool. This keeps the core success criterion independent of RPC integration approval.

### 6. Testing and validation

The implementation will include:

- Unit tests for progressive containers, lists, byte lists, active fields, and generalized-index stability.
- Boundary cases including empty transactions/receipts, pre-Capella blocks without withdrawals, post-Capella withdrawals, post-Prague execution requests, blob-gas fields, and Amsterdam/Gloas fields when present in Erigon types.
- Fixture tests across representative execution fork eras.
- Side-by-side output of the current RLP block hash and the EIP-7807 SSZ root.
- Proof verification tests for selected fields and list roots.
- Cross-implementation checks against EIP test cases, consensus-spec SSZ utilities, or a small independent Python reference implementation.
- Regression tests for existing Erigon SSZ behavior when shared packages are touched.

## Roadmap

The project is planned as an approximately 18-week effort from July to November 2026.

### Phase 1 — Spec alignment, field mapping, and maintainer review (Weeks 1–2)

- Finalize the EIP-7807 field-to-Erigon mapping.
- Confirm canonical transaction, receipt, withdrawal, request, and block access list inputs.
- Confirm gas-field semantics, package placement, generalized-index convention, and the first PR boundary with maintainers.
- Resolve the remaining Phase 1 questions: canonical test-vector source, spec-revision pinning, field-path naming, experimental RPC namespace, and the first upstream PR split.
- Produce a tracking issue or design note for review.

### Phase 2 — Progressive SSZ Merkleization (Weeks 3–6)

- Implement progressive Merkle tree, `ProgressiveContainer`, `ProgressiveList`, and `ProgressiveByteList` root helpers required by EIP-7807.
- Add unit tests and reference-vector comparisons.
- Submit a focused PR for helpers and tests.

### Phase 3 — EIP-7807 payload view and block-root calculation (Weeks 7–11)

- Implement the Erigon-to-EIP-7807 adapter.
- Compute transaction, receipt, withdrawal, request, and block access list roots according to the specification.
- Add representative cross-fork fixtures and deterministic root tests.
- Submit a reviewable root-computation PR.

### Phase 4 — Generalized indices, proofs, and experimental RPC (Weeks 12–15)

- Implement path/generalized-index resolution and proof generation for selected fields and list roots.
- Define the proof response shape.
- Add experimental root/proof inspection methods and a local demonstration.
- Document how to reproduce and verify the results.

### Phase 5 — Upstream review, cleanup, and final report (Weeks 16–18)

- Address review feedback and split remaining work into small PRs where necessary.
- Run targeted and regression test suites.
- Document unresolved spec or integration questions.
- Prepare the final EPF report and presentation with links to code, tests, fixtures, and demonstrations.

## Possible challenges

- **The specifications are still evolving.** EIP-7807 and its progressive SSZ dependencies may change during the fellowship. I will record the exact spec revisions used by tests, keep conversion and Merkleization modular, and avoid premature consensus-path integration.
- **Root computation is sensitive to small representation differences.** Endianness, active fields, list shape, fork defaults, and the exact transaction/receipt/request bytes can all change the result. The project will use an explicit mapping, independent reference calculations, and boundary-case fixtures from the beginning.
- **Erigon's existing SSZ code primarily serves CL and Engine API use cases.** Reusing it in an EL block-root path may expose package ownership or abstraction issues. I will confirm placement with maintainers and add new focused helpers instead of changing existing behavior unnecessarily.
- **Some fork-specific field mappings require maintainer or spec-author confirmation.** Gas containers, pre-fork default values, block access list retrieval, and generalized-index proof conventions need early review. Phase 1 is designed to resolve these questions before implementation expands.
- **The full idea can exceed a fellowship-sized review scope.** Work will be split into small layers: progressive helpers, payload mapping, root calculation, proof generation, and experimental RPC. Production activation, networking/storage migration, arbitrary query-language semantics, and a complete Pureth implementation remain out of scope.
- **Upstream review timing is outside the project author's control.** Success will be measured by the quality and reproducibility of the implementation and submitted PRs, while merged PRs remain the preferred outcome.

## Goal of the project

The project will be considered successful when Erigon can deterministically construct an EIP-7807 execution payload view from existing block data, compute its SSZ root, and verify representative results through independent fixtures and tests.

The finished project should provide:

- A reviewed and documented EIP-7807 field mapping for Erigon.
- Tested progressive SSZ Merkleization required by the execution block layout.
- Deterministic EIP-7807 root calculation over representative Erigon blocks.
- Stable generalized-index mappings and verifiable proofs for selected fields and list roots.
- A reproducible experimental interface or demonstration for inspecting roots and proofs.
- A set of reviewable upstream PRs and a final report describing results, remaining limitations, and the path toward deeper Erigon integration.

The wider impact is to give Erigon a concrete, testable starting point for SSZ-native execution blocks and proof-oriented execution data access. The project does not claim to activate EIP-7807 on mainnet or complete all Pureth components; it is finished when the core root/proof substrate is correct, tested, documented, and ready for continued upstream review.

## Collaborators

### Fellows

[Jack](https://github.com/JackCC703)

### Mentors

[RazorClient](https://github.com/RazorClient) and [Etan Kissling](https://github.com/etan-status)

## Resources

- [Detailed technical design and Phase 1 field mapping](https://hackmd.io/@jackcc/epf7-project-draft)
- [Erigon repository](https://github.com/erigontech/erigon)
- [EIP-7807: SSZ execution blocks](https://eips.ethereum.org/EIPS/eip-7807)
- [EIP-7919: Pureth Meta](https://eips.ethereum.org/EIPS/eip-7919)
- [EIP-7495: SSZ ProgressiveContainer](https://eips.ethereum.org/EIPS/eip-7495)
- [EIP-7916: SSZ ProgressiveList](https://eips.ethereum.org/EIPS/eip-7916)
- [EIP-7688: Forward compatible consensus data structures](https://eips.ethereum.org/EIPS/eip-7688)
- [Erigon PR #21729: Engine API SSZ alignment work](https://github.com/erigontech/erigon/pull/21729)
- [execution-apis PR #793](https://github.com/ethereum/execution-apis/pull/793)
- [Ethereum consensus-specs](https://github.com/ethereum/consensus-specs)
- [remerkleable SSZ reference implementation](https://github.com/ethereum/remerkleable)
