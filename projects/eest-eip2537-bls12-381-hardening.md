# EEST: BLS12-381 (EIP-2537) Subgroup Checks and Field Deserialization Hardening

Hardening execution client conformance on cryptographic precompiles through edge-case test vectors and strict canonical deserialization enforcement.

## Motivation

Ethereum Execution Layer clients historically present divergences in cryptographic precompiles at boundary conditions. EIP-2537 introduces support for BLS12-381 curve operations across seven precompiles (`BLS12_G1ADD`, `BLS12_G1MSM`, `BLS12_G2ADD`, `BLS12_G2MSM`, `BLS12_PAIRING`, `BLS12_MAP_FP_TO_G1`, `BLS12_MAP_FP2_TO_G2`).

Critical vulnerabilities and consensus breaks arise from two primary vectors:
1. **Subgroup Membership Divergence**: Points lying on the elliptic curve $E(\mathbb{F}_p)$ or $E'(\mathbb{F}_{p^2})$ whose order is not the prime order $r$ (such as cofactor torsion points, e.g. the affine order 3 point $(0, 2) \in E(\mathbb{F}_p)$). If a client fails to enforce subgroup checks inside `BLS12_PAIRING`, it becomes susceptible to small-subgroup attacks.
2. **Non-Canonical Deserialization**: Coordinates are formatted as 64-byte big-endian fields representing 48-byte integers. Bytes 0..15 must be zero, and the integer must satisfy $0 \le x < p$. Subtle differences in how clients handle boundary cases ($x = p$, $x = p + 1$, $x = 2^{381} - 1$, $x = 2^{384} - 1$, or dirty top-16 bytes) result in consensus state-root splits between Go (Geth), Rust (Reth), C# (Nethermind), and Java (Besu).

## Project description

This project designs, implements, and integrates an exhaustive suite of negative, boundary, and fuzzing test cases within `execution-spec-tests` (EEST / EELS), targeting:
* Comprehensive verification of strict coordinate deserialization across all BLS12-381 operations.
* Subgroup membership verification in `BLS12_PAIRING` using both small-order torsion points and uncompressed randomized non-subgroup curve points.
* Multi-pair combinatorial permutations ($k=1, 2, 3$) ensuring fault isolation when valid and invalid subgroup points are combined.
* Validation against real execution client transition tools (`evm t8n`) and upstream PR submission into `ethereum/execution-specs`.

## Specification

The test suite is structured around three technical pillars:

### 1. Canonical Field Coordinate Deserialization
* **Top-16 Byte Padding Verification**: Validates that any non-zero byte in indices $0 \dots 15$ of coordinate fields triggers immediate precompile failure (`REVERT` / call status 0).
* **Modulus Range Boundaries**: Evaluates inputs where coordinates equal $p$, $p+1$, $p + 2^{128}$, $2^{381}-1$ (maximum 381-bit scalar), and $2^{384}-1$ across both $\mathbb{F}_p$ ($G_1$) and $\mathbb{F}_{p^2}$ ($G_2$).

### 2. Subgroup Check Validation (`BLS12_PAIRING`)
* **Small-Order Curve Points**: Generates deterministic non-identity points of order dividing the cofactor $h_1$ (e.g. $(0, 2)$ where $y^2 = 4 \pmod p$) and tests rejection in pairing checks.
* **Cofactor / Isogeny Point Fuzzing**: Parametrizes multi-pair pairings combining valid generator elements with torsion points.
* **Negative Permutations**: Verifies that failure in any single pair ($i \in \{1 \dots k\}$) guarantees total precompile rejection.

### 3. Multi-Scalar Multiplication Validation (`BLS12_G1MSM` / `BLS12_G2MSM`)
* **Subgroup Enforcement**: Verifies strict rejection of non-subgroup points (including small-order $(0, 2)$ points) under both zero scalars ($s=0$) and non-zero scalars ($s=1$), preventing lazy bypass in client implementations.
* **Dirty Padding & Modulus Overflow in Batches**: Tests dirty top-16 byte padding and $x \ge p$ coordinates in single-pair and multi-pair batches ($k=2, 3$).
* **Calldata Length Misalignment**: Confirms rejection of unaligned payloads ($160k \pm 1$ for G1, $288k \pm 1$ for G2).
* **Scalar Modulo Equivalence & Cancellation**: Tests boundary scalars ($s=r, r+1, 2^{256}-1$) and multi-pair linear cancellations to the point at infinity ($\mathcal{O}$).

### 4. Arithmetic & On-Curve Validation (`BLS12_G1ADD` / `BLS12_G2ADD`)
* Verifies that addition operations correctly process valid curve points regardless of subgroup membership, confirming that subgroup restrictions apply strictly where specified by EIP-2537.

## Roadmap

| Milestone | Timeline | Deliverables | Status |
|---|---|---|---|
| M1: Environment & Baseline Pairing Suite | Week 1–2 | Bare-metal Nix Flake environment, local harness with `evm t8n`, first 47 unit test vectors (141 fixtures). | **Completed** |
| M2: MSM Hardening & Upstream Integration | Week 3–4 | Extended suite with G1MSM/G2MSM padding, subgroup bypass prevention, boundary scalars, total 447 fixtures. Upstream PR [#3641](https://github.com/ethereum/execution-specs/pull/3641) submitted. | **Completed** |
| M3: Multi-Client Differential Fuzzing | Week 5–6 | Run generated fixtures against Nethermind, Besu, and Reth `t8n` backends; document cross-client consensus behavior. | In Progress |

## Possible challenges

* **Upstream Migration ("The Weld")**: The migration of `execution-spec-tests` into `ethereum/execution-specs` requires tracking the unified monorepo branch structures.
* **Client Transition Tools Differences**: Ensuring the test fixtures pass uniformly across diverse transition backends (Geth `evm`, Nethermind `t8n`, Reth).

## Goal of the project

1. 100% test coverage for non-canonical coordinate encodings and subgroup boundary violations on EIP-2537 in the official Ethereum test suite.
2. Official PR merged into the Ethereum execution test repository.
3. Verification that all major L1 execution clients produce identical state transitions under adversarial cryptographic calldata.

## Collaborators

### Fellows
* roru (`@roru`)

### Mentors
* EEST / Testing Team (Mario Vega, danceratopz)

## Resources
* EIP-2537 Specification: https://eips.ethereum.org/EIPS/eip-2537
* Execution Spec Tests (EEST): https://github.com/ethereum/execution-spec-tests
* Ethereum Execution Layer Specification (EELS): https://github.com/ethereum/execution-specs
* Local Test Suite: [`execution-spec-tests/tests/prague/eip2537_bls_12_381_precompiles/test_bls12_subgroup_and_deserialization_edge_cases.py`](file:///home/roru/execution-spec-tests/tests/prague/eip2537_bls_12_381_precompiles/test_bls12_subgroup_and_deserialization_edge_cases.py)
