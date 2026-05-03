# Cross-zkVM Proof Aggregation using Cairo's Recursive STARK Architecture

*Adapting Cairo's SHARP proof aggregation system to enable cross-zkVM proof composition for Ethereum L1 validity proofing.*

## Motivation

Ethereum's long-term roadmap ([The Verge](https://vitalik.eth.limo/general/2024/10/23/futures4.html)) targets SNARKifying the EVM - making Ethereum L1 validity-provable via ZK proofs. Multiple teams have made serious progress on zkEVM proving: [SP1](https://github.com/succinctlabs/sp1), [RISC Zero](https://github.com/risc0/risc0), [Jolt](https://github.com/a16z/jolt), [ZisK](https://github.com/0xPolygonHermez/zisk), and [OpenVM](https://github.com/openvm-org/openvm) can all generate proofs of Ethereum block execution. Some are approaching real-time.

The bottleneck that gets less attention is what happens *after* proof generation. Verifying a STARK proof on Ethereum L1 costs approximately 5 million gas; a SNARK costs approximately 1 million gas. Submitting individual block proofs at this cost is economically unviable. The solution is aggregation - combining many proofs into one before L1 submission - but no cross-compatible aggregation standard exists. Each zkVM produces proofs in an incompatible format.

This is precisely the problem Cairo's SHARP (Shared Prover) has solved at the L2 level. SHARP aggregates thousands of Cairo program proofs into a single on-chain proof daily using recursive STARKs. The architecture is proof-system-agnostic by design: a Cairo bootloader circuit verifies sub-proofs without caring what the sub-programs computed. This project investigates whether that mechanism can be adapted to aggregate proofs from RISC-V based zkVMs (SP1, RISC Zero) before Ethereum L1 submission.

## Project description

Design and prototype a Cairo-based proof aggregation layer that accepts outputs from at least one external zkVM (SP1 as primary target) and produces a single aggregated proof verifiable on Ethereum L1.

The core idea: write a Cairo circuit that acts as a verifier for SP1's proof format. Running this circuit through SHARP produces an aggregated STARK proof. The L1 then verifies one aggregated proof instead of many individual proofs, dramatically reducing total verification cost.

This is research-plus-prototype, not a production system. The goal is to determine whether the architecture is technically viable and identify what obstacles - cryptographic, tooling, or standards - stand in the way of practical deployment.

## Specification

**Phase 1 - Research and architecture (Month 1):**
- Survey existing proof aggregation approaches: [SHARP](https://docs.starkware.co/starkex/stark-cryptography/sharp-shared-prover.html), [STARKPack](https://medium.com/nethermind-eth/starkpack-aggregating-starks-for-shorter-proofs-and-faster-verification-8c78a5c26291), [Plonky2](https://github.com/0xPolygonZero/plonky2) recursion, [Nova](https://github.com/microsoft/Nova) folding
- Document the exact proof format produced by SP1 (RISC-V + Plonky3 backend)
- Define the interface the aggregation circuit must implement
- Identify the main technical risk: can a Cairo circuit verify a Plonky3 proof without prohibitive in-circuit cost?

**Phase 2 - Cairo aggregator circuit (Month 2):**
- Write a Cairo circuit (targeting [Stwo prover](https://github.com/starkware-libs/stwo)) that verifies an SP1 proof
- Start with a simplified proof format if full SP1 verification is too large
- Unit tests for each verification step

**Phase 3 - Integration and testnet (Month 3):**
- Run the aggregator on [Ephemery testnet](https://ephemery.dev)
- End-to-end test: generate SP1 proof of simple EVM computation → aggregate via Cairo circuit → submit to L1

**Phase 4 - Benchmarking (Month 4):**
- Measure: aggregation proving time, peak memory, L1 verification gas cost vs. direct submission
- Identify primary bottleneck

**Phase 5 - Report (Month 5):**
- Viability recommendation with clear reasoning
- Present at [Devconnect Mumbai](https://devconnect.org), November 2026

## Roadmap

| Month | Milestone |
|-------|-----------|
| June | SP1 proof format documented; architecture spec; blocker assessment |
| July | Cairo verifier circuit for SP1 compiling with unit tests |
| August | End-to-end prototype on Ephemery |
| September | Full benchmarking published |
| October | Final report drafted |
| November | Devconnect presentation |

## Possible challenges

**In-circuit verification cost:** Verifying a Plonky3 proof inside a Cairo STARK circuit may be prohibitively expensive. Primary risk - assessed in Month 1. Fallback: target a simpler proof format first.

**Proof format stability:** SP1's proof format may change. Mitigation: pin to a specific version.

**Scope creep:** SP1 only, one aggregation level. No expansion.

## Goal of the project

Working prototype aggregating at least one SP1 proof via Cairo circuit + published benchmarks + clear viability recommendation. A well-documented negative result is equally valid.

## Collaborators

### Fellows
None at this stage.

### Mentors
Open to anyone working on zkVM proving, proof systems, or The Verge roadmap.

## Resources

- [EIP-6800](https://eips.ethereum.org/EIPS/eip-6800) - Ethereum Verkle tree state
- [Vitalik: The Verge](https://vitalik.eth.limo/general/2024/10/23/futures4.html)
- [SHARP documentation](https://docs.starkware.co/starkex/stark-cryptography/sharp-shared-prover.html)
- [Stwo prover](https://github.com/starkware-libs/stwo)
- [SP1 zkVM](https://github.com/succinctlabs/sp1)
- [STARKPack](https://medium.com/nethermind-eth/starkpack-aggregating-starks-for-shorter-proofs-and-faster-verification-8c78a5c26291)
- [EPF cohort 6 zkVM benchmarking](https://github.com/eth-protocol-fellows/cohort-six/tree/master/projects)
- [EPF cohort 6 ZK-EVM research](https://github.com/eth-protocol-fellows/cohort-six/tree/master/projects)
