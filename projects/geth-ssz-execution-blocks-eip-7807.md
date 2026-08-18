# SSZ Execution Blocks (EIP-7807) for Geth

Implementing the progressive SSZ groundwork and the EIP-7807 block container and block hash in go-ethereum.

## Motivation

Ethereum users today read chain data through a small number of trusted RPC providers. A block's identity is keccak256 over its RLP-encoded header: it can prove the whole block is untampered, but nothing about a single transaction, receipt, or field inside it. Verifying a part means downloading and re-hashing everything, so wallets and dApps do not verify at all; they trust the provider, and cannot tell if it is compromised, buggy, or omitting results.

The Pureth effort ([EIP-7919](https://eips.ethereum.org/EIPS/eip-7919)) addresses this by moving execution-layer data structures to SSZ, where the hash of an object is the root of a Merkle tree over its contents, and any single field becomes provable against the block hash with a few hundred bytes. [EIP-7807](https://eips.ethereum.org/EIPS/eip-7807) is the block-level step: the execution block becomes one SSZ container and the block hash becomes its hash_tree_root. After the June 2026 rescope, 7807 is deliberately small: transactions, receipts, and withdrawals keep their existing encodings, so the block can migrate before any object conversion work.

Why Geth, and why now: Geth is the most used execution client, and it has no SSZ execution-block implementation. The current SSZ activity in Geth ([PR #35171](https://github.com/ethereum/go-ethereum/pull/35171), a draft of the REST/SSZ engine-api transport from [execution-apis #793](https://github.com/ethereum/execution-apis/pull/793), static types only) adds an alternative encoding for engine-api messages alongside JSON-RPC; it does not change what the block hash is. Meanwhile [EIP-7688](https://eips.ethereum.org/EIPS/eip-7688) is CFI for [Glamsterdam](https://eips.ethereum.org/EIPS/eip-7773) and was merged into the Gloas consensus specs in July 2026 ([consensus-specs #4630](https://github.com/ethereum/consensus-specs/pull/4630)), with implementation PRs open across the CL clients. If it ends up in the fork, Geth's embedded light client has to verify progressive SSZ structures; its SSZ handling today is split between third-party libraries with no progressive-type support and hand-rolled merkleization in beacon/types that carries TODO comments asking for an SSZ encoder. So this project's groundwork has a plausible consumer inside Geth independent of EIP-7807's scheduling.

## Project description

I am taking the SSZ execution-blocks track of Pureth, in Geth.

| # | Deliverable | Scope |
| --- | --- | --- |
| 1 | A progressive-SSZ engine inside Geth (new package, e.g. core/ssz): serialization codec, static and progressive merkleization ([EIP-7916](https://eips.ethereum.org/EIPS/eip-7916)), ProgressiveContainer support ([EIP-7495](https://eips.ethereum.org/EIPS/eip-7495)), hash_tree_root, and basic proof generation (generalized indices and single-field proofs, round-tripped through the existing beacon/merkle verifier). | Committed |
| 2 | The EIP-7807 ExecutionPayload container: mapping Geth's existing block onto the 18-field container at hashing time and computing the SSZ block roots, with the header-summary root proven equal to the full-payload root. | Committed |
| 3 | Deterministic side-by-side root computation: for representative blocks across fork eras, the EIP-7807 root computed alongside the existing keccak block hash and validated against generated fixtures, without touching any consensus-critical path. | Committed |
| 4 | The keccak/SSZ dual-hash transition behind a devnet-only eip7807Time flag: the fork gate, the main hash and root sites made fork-aware, and a boundary-chain import test. | Stretch |
| 5 | Engine API wiring: the fork-aware block-hash check in newPayload and the new versioned methods. | Stretch |
| 6 | A single-client Kurtosis devnet where the modified Geth produces and accepts SSZ-hashed blocks. | Stretch |
| 7 | [EIP-6465](https://eips.ethereum.org/EIPS/eip-6465) SSZ withdrawals as a first small object-conversion pilot (its container depends only on the progressive types). | Stretch |

Stretch goals are not promised: 4 to 6 form a dependency chain and are attempted in that order; 7 depends only on the committed scope, so it can be attempted at any point after it. Testing is deliberately not listed as a deliverable because it is the working method: the conformance harness is built before the engine code and runs on every change from the first week. The Testing section below describes it; none of it ships in the geth binary.

## Specification

### Current state of Geth

From going through the codebase:

- The block hash is hard-coded keccak-over-RLP in one canonical function (Header.Hash() in core/types/block.go), and the three MPT roots (transactions, receipts, withdrawals) are computed at about twenty DeriveSha call sites.
- Geth has three disconnected SSZ mechanisms, none progressive-capable: ferranbt/fastssz in exactly one file for the era accumulator, the protolambda zrnt/ztyp libraries powering the beacon light client, and hand-rolled sha256 merkleization in beacon/types with TODO comments to replace it "when an SSZ encoder lands".
- SSZ proof verification exists (beacon/merkle.VerifyProof, single branch), but there is no SSZ proof generation and no multiproofs (the trie package's Prove targets the state MPT, a different tree).
- The batched SHA-256 primitive (prysmaticlabs/gohashtree) is already in the module graph as a transitive dependency. So the fast hashing primitive exists, but the merkleization engine, the progressive types, and proof generation have to be built.

### The engine (core/ssz)

Four capabilities: encoding and strict decoding of SSZ values; classic and progressive merkleization with the standard mix-ins (length, selector, active fields, using the post-January-2026 tree orientation); hash_tree_root over static and progressive types; and generalized-index resolution with basic proof generation. The concrete API surface is deliberately not fixed here: it is the subject of an M0 design document reviewed with the mentor, following Geth conventions for consensus-critical code (explicit per-type methods, no reflection, no code generation). Hashing goes through a backend interface with stdlib sha256 as default and gohashtree as the batched option, with an equivalence test guaranteeing both backends always agree. The progressive subtree boundaries get dedicated boundary tests.

### The block mapping

The EIP-7807 container (18 fields) is computed from Geth's existing structures at hashing time. The RLP Header remains the single in-memory and database representation; no parallel header struct is introduced. Transactions map as their existing typed-envelope bytes, withdrawals and the block access list as their RLP bytes, into ProgressiveList[ProgressiveByteList] / ProgressiveByteList fields. The receipts_root field is recomputed as the progressive SSZ root over the raw receipt envelopes per the EIP; it is not the legacy MPT receipts root carried in today's header. Derived values (the blob gas limit and blob base fee) are computed from Geth's fork-aware blob schedule rather than constants. Both computation paths are implemented and must agree: the full-body path hashing all lists, and the header-summary path substituting the stored roots. Their equality is a standing test.

### Testing

The harness is built before the engine code, so every function is checked against the official answers the day it is written; unimplemented types show as skipped cases, and the pass/skip counts double as a progress meter. Five layers:

- The official ssz_generic vectors (release assets on ethereum/consensus-specs), pinned to v1.7.0-alpha.2 or later because earlier releases predate the January 2026 orientation change. A canary test asserting one known post-orientation-flip root guards the pin, so a stale vector set fails with one clear error instead of hundreds of confusing ones.
- Round-trip serialization tests, and differential root comparison against two independent implementations: ethereum/remerkleable (the Python reference the official vectors are generated from) and pk910/dynamic-ssz (the only Go library with progressive support today, used strictly as a test oracle, never as a runtime dependency).
- Fuzzers for the decoder and the progressive merkleizer, following Geth's existing FuzzRLP pattern: random and malformed inputs must be decoded correctly or rejected cleanly, never crash.
- For the 7807 container itself, generated vectors from a small remerkleable-based tool, since no official test suite for execution-layer container types exists anywhere yet.
- A standalone Python reference script (built on remerkleable) that recomputes the EIP-7807 block root from the same block data Geth consumes and diffs it against Geth's output. This checks the Go implementation against an independent reference on real blocks rather than only synthetic vectors, and is delivered as an explicit validation tool for the block phase.

## Roadmap

Weeks are program weeks; the split follows the dependency order (engine before block before transition). Each milestone ends with a checkable exit criterion.

| Weeks | Milestone | Work | Exit criterion |
| --- | --- | --- | --- |
| through 6 | M0, ramp and design | Geth architecture ramp-up; the engine design document (API surface, per-type method conventions, the hashing backend interface, error handling for strict decoding) written and reviewed with the mentor; the conformance harness scaffold (pinned vector download, canary test, pass/skip reporting). Partly complete: the EIP set, the client survey, and the codebase mapping above are done. | Design document agreed; harness runs end to end with all cases skipped. |
| 6-10 | M1, static SSZ core | The static codec and classic merkleization per the engine section; both hashing backends with their equivalence test; the decoder fuzzer runs from here on. | The full classic ssz_generic suite passes, valid and invalid cases, on both backends. |
| 10-13 | M2, progressive types | Progressive merkleization, ProgressiveList/ProgressiveByteList, ProgressiveContainer; generalized-index resolution and single-field proof generation; the merkleizer fuzzer. | Progressive vectors green; differential comparison against remerkleable and dynamic-ssz agrees on randomized structures. |
| 13-16 | M3, the EIP-7807 block | The container mapping and both computation paths per the block-mapping section; the remerkleable-based generator and the generated container vectors; the standalone Python reference script that recomputes the block root from the same block data and diffs it against Geth. | Generated vectors reproduce; summary-equals-full holds under fuzzing; the Python reference reproduces Geth's EIP-7807 root on the same block data. |
| 16-19 | M4, validation, hardening, upstream | Side-by-side 7807 roots across blocks from every fork era; extended fuzzing; benchmarks on realistic payloads against the existing Go SSZ libraries; the upstream issue proposing core/ssz with those numbers, and working the review feedback. | Fixtures reproduce deterministically across eras; the upstream issue is open with benchmarks. |
| 19-22 | M5, stretch and wrap-up | If no upstream review work remains, pick up stretch-goal items in order (or the withdrawals pilot if the remaining time fits it better). Submit the final project report, including the documented next phase beyond the fellowship. | Final report submitted. |

## Possible challenges

- I am new to the Geth codebase, which is large and has strong conventions of its own, so there is a real risk of losing early weeks to orientation. Mitigated by the dedicated M0 milestone, which budgets for exactly this before any engine code is due; part of that ramp is already done (the EIP study and the codebase mapping above), and mentor access covers the rest.
- Spec churn: EIP-7916/7495 are Review-stage, so their text can still change, and it has happened before: the merkleization orientation flipped in January 2026, silently invalidating every earlier implementation and vector set. A repeat mid-project could invalidate written code. Mitigated by pinning the exact vector release the engine is validated against (with the canary test failing loudly on a stale pin) and watching the spec repos so a change is caught the day it lands. The same applies to EIP-7807 itself, rescoped as recently as June 2026; there the exposure is smaller, since the container mapping is a thin layer over the engine and a field change is a local edit.
- Correctness bar: a wrong hash_tree_root is a silent consensus fault. Nothing crashes; the code just computes a different root than every other client, and the divergence only surfaces when a node rejects blocks its peers accept. This is why testing is the working method rather than a phase: conformance vectors from week one, plus differential comparison against two independent implementations, so a shared bug is very unlikely.
- No official test suite exists for the EIP-7807 container itself, so the block-mapping work is validated against self-generated vectors. If the generator misreads the spec, the implementation and the fixtures would share the bug. Mitigated by generating vectors through remerkleable, the independent Python reference the official suites themselves are built from, rather than through this project's own engine.

## Goal of the project

Success is solid groundwork validated against the reference vectors and a working demonstration, with merged upstream PRs as a bonus rather than the goal. Concretely, the project is successful when: the engine passes the full ssz_generic suite (valid and invalid cases) and agrees bit-for-bit with the reference implementations; the EIP-7807 container reproduces generated vectors and the summary/full equality holds under fuzzing; and the 7807 root is computed deterministically alongside the existing block hash across representative fork-era blocks. The continuation beyond the fellowship (the withdrawals/transactions/receipts conversions and cross-client test assets) is documented as the project's next phase.

## Collaborators

Fellows: solo on the Geth side. Mentors: [Tamaghna Choudhuri](https://github.com/RazorClient) and [Etan Kissling](https://github.com/etan-status).

## Resources

| Resource | Link |
| --- | --- |
| EIP-7919 Pureth Meta | https://eips.ethereum.org/EIPS/eip-7919 |
| EIP-7807 SSZ execution blocks | https://eips.ethereum.org/EIPS/eip-7807 |
| EIP-7916 ProgressiveList / EIP-7495 ProgressiveContainer | https://eips.ethereum.org/EIPS/eip-7916, https://eips.ethereum.org/EIPS/eip-7495 |
| EIP-6465 SSZ withdrawals root | https://eips.ethereum.org/EIPS/eip-6465 |
| SSZ spec / merkle proofs spec | https://github.com/ethereum/consensus-specs/blob/master/ssz/simple-serialize.md, https://github.com/ethereum/consensus-specs/blob/master/ssz/merkle-proofs.md |
| Official spec test vectors (release assets) | https://github.com/ethereum/consensus-specs/releases |
| Reference implementation (remerkleable) | https://github.com/ethereum/remerkleable |
| Nim reference (nim-ssz-serialization) | https://github.com/status-im/nim-ssz-serialization |
| Go differential oracle (dynamic-ssz) | https://github.com/pk910/dynamic-ssz |
| Geth engine-API SSZ transport PR #35171 | https://github.com/ethereum/go-ethereum/pull/35171 |
| Cohort six Nimbus Pureth proposal (prior work) | https://github.com/eth-protocol-fellows/cohort-six/blob/master/projects/pureth-eip-7919-nimbus.md |
| Pureth guide | https://pureth.guide/ |
