# EEST: BLS12-381 (EIP-2537) Subgroup Checks and Field Deserialization Hardening

Hardening execution client conformance on cryptographic precompiles through edge-case test vectors, cross-client specification mapping, differential invariant fuzzing, and strict canonical deserialization enforcement.

## Motivation

Ethereum Execution Layer clients historically present divergences in cryptographic precompiles at boundary conditions. EIP-2537 introduces support for BLS12-381 curve operations across seven precompiles (`BLS12_G1ADD`, `BLS12_G1MSM`, `BLS12_G2ADD`, `BLS12_G2MSM`, `BLS12_PAIRING`, `BLS12_MAP_FP_TO_G1`, `BLS12_MAP_FP2_TO_G2`).

Critical vulnerabilities and consensus breaks arise from two primary vectors:
1. **Subgroup Membership Divergence**: Points lying on the elliptic curve $E(\mathbb{F}_p)$ or $E'(\mathbb{F}_{p^2})$ whose order is not the prime order $r$ (such as cofactor torsion points, e.g. the affine order-3 point $(0, 2) \in E(\mathbb{F}_p)$). If a client fails to enforce subgroup checks inside `BLS12_PAIRING` or bypasses subgroup checks when a scalar is zero in `BLS12_G1MSM` / `BLS12_G2MSM`, it becomes susceptible to small-subgroup attacks and consensus splits.
2. **Non-Canonical Deserialization**: Coordinates are formatted as 64-byte big-endian fields representing 48-byte integers. Bytes `0..15` must be strictly zero, and the integer must satisfy $0 \le x < p$. Subtle differences in how clients handle boundary cases ($x = p$, $x = p + 1$, $x = p + 2^{128}$, $x = 2^{381} - 1$, $x = 2^{384} - 1$, or dirty top-16 bytes across batched pairs and $\mathbb{F}_{p^2}$ extension coordinates) result in consensus state-root splits between Go (Geth, Erigon), Rust (Reth, Ethrex, Revm), C# (Nethermind), Java (Besu), and Nim (Nimbus).

## Project description

This project designs, implements, and integrates an exhaustive suite of negative, boundary, and differential fuzzing test cases within `ethereum/execution-specs` (EEST / EELS) and `spencer-tb/protocol-mesh`, targeting:
- Comprehensive verification of strict coordinate deserialization across all BLS12-381 operations.
- Subgroup membership verification in `BLS12_PAIRING` and `BLS12_G1MSM` / `BLS12_G2MSM` using small-order torsion points and uncompressed non-subgroup curve points.
- Multi-pair combinatorial permutations ($k=1, 2, 3$) ensuring fault isolation when valid and invalid subgroup points are combined.
- Cross-client precompile function mapping (`xref`) across all 8 execution clients and automated differential/invariant fuzzing in `protocol-mesh`.
- Validation against real execution client transition tools (`evm t8n`) and upstream PR submissions into `ethereum/execution-specs` and related Ethereum security repositories.

## Specification

The test suite and verification tooling are structured around five technical pillars:

### 1. Canonical Field Coordinate Deserialization
- **Top-16 Byte Padding Verification**: Validates that any non-zero byte in indices $0 \dots 15$ of 64-byte coordinate fields triggers immediate precompile failure (`REVERT` / call status 0).
- **Modulus Range Boundaries**: Evaluates inputs where coordinates equal $p$, $p+1$, $p + 2^{128}$, $2^{381}-1$ (maximum 381-bit value), and $2^{384}-1$ across both $\mathbb{F}_p$ ($G_1$) and $\mathbb{F}_{p^2}$ ($G_2$).

### 2. Subgroup Check Validation (`BLS12_PAIRING`)
- **Small-Order Curve Points**: Generates deterministic non-identity points of order dividing the cofactor $h_1$ (e.g. $(0, 2)$ where $y^2 = 4 \pmod p$) and tests rejection in pairing checks.
- **Cofactor / Isogeny Point Fuzzing**: Parametrizes multi-pair pairings combining valid generator elements with torsion points.
- **Negative Permutations**: Verifies that failure in any single pair ($i \in \{1 \dots k\}$) guarantees total precompile rejection.

### 3. Multi-Scalar Multiplication Validation (`BLS12_G1MSM` / `BLS12_G2MSM`)
- **Subgroup Enforcement**: Verifies strict rejection of non-subgroup points (including small-order $(0, 2)$ points) under both zero scalars ($s=0$) and non-zero scalars ($s=1$), preventing lazy short-circuit bypass in client implementations.
- **Dirty Padding & Modulus Overflow in Batches**: Tests dirty top-16 byte padding and $x \ge p$ coordinates in single-pair and multi-pair batches ($k=2, 3$).
- **Calldata Length Misalignment**: Confirms rejection of unaligned payloads ($160k \pm 1$ for $G_1$, $288k \pm 1$ for $G_2$).
- **Scalar Modulo Equivalence & Cancellation**: Tests boundary scalars ($s=r, r+1, 2^{255}-1, 2^{256}-1$) and multi-pair linear cancellations to the point at infinity ($\mathcal{O}$).

### 4. Hash-to-Curve (`BLS12_MAP_FP_TO_G1` / `BLS12_MAP_FP2_TO_G2`) & Arithmetic Validation
- **Hash-to-Curve Deserialization & Length Boundaries**: Validates rejection of dirty top-16-byte big-endian padding (`0..15` on $c_0$, `64..79` on $c_1$, and simultaneous violations), modulus overflows ($x \ge p$ up to $2^{384}-1$ and $p + 2^{128}$), and truncated calldata (32-byte and 64-byte inputs) on precompiles `0x10` and `0x11`.
- **Arithmetic & On-Curve Validation (`BLS12_G1ADD` / `BLS12_G2ADD`)**: Verifies that addition operations correctly process valid curve points regardless of subgroup membership, confirming that subgroup restrictions apply strictly where specified by EIP-2537.

### 5. Cross-Client Mapping (`xref`) & Differential Invariant Fuzzing (`protocol-mesh`)
- **Cross-Client Precompile Mapping**: Complete mapping of 37 EIP-2537 entry points and internal cryptographic helpers across 8 execution clients (Geth, Reth, Nethermind, Besu, Erigon, Ethrex, Nimbus, Evmone) in `xref/data/tree.json`.
- **Differential & Algebraic Invariant Fuzzer**: Automated fuzzer (`fuzz/fuzz_eip2537.py`, 236/236 tests passing across all 7 EIP-2537 precompiles) verifying point-at-infinity identity ($P + \mathcal{O} = P$), commutativity, pairing bilinearity ($e(aG_1, bG_2) = e(abG_1, G_2)$), multi-pair scalar cancellations ($G_1$ and $G_2$ MSM), deterministic map-to-curve subgroup membership (`MAP_FP_TO_G1` and `MAP_FP2_TO_G2`), and negative boundary rejections against the EELS reference spec.

## Roadmap

| Milestone | Timeline | Deliverables | Status |
|---|---|---|---|
| M1: Environment & Baseline Pairing Suite | Week 1–2 | Bare-metal Nix Flake environment, local harness with `evm t8n`, initial 47 unit test vectors (141 fixtures). | **Completed** |
| M2: MSM Hardening & Upstream Integration | Week 3–4 | Extended suite with `G1MSM`/`G2MSM` padding, subgroup bypass prevention ($s=0$), boundary scalars, total 447 fixtures. Upstream PR [`ethereum/execution-specs#3641`](https://github.com/ethereum/execution-specs/pull/3641) merged and verified via EELS mutation testing. | **Completed** |
| M3: Hash-to-Curve Hardening & Cross-Client Fuzzing | Week 5–6 | Extended `MAP_FP_TO_G1` (`0x10`) and `MAP_FP2_TO_G2` (`0x11`) negative test suite (22 parameterized cases / 66 fixtures in PR [`#3821`](https://github.com/ethereum/execution-specs/pull/3821) / rebased to `forks/bogota` in [`#3945`](https://github.com/ethereum/execution-specs/pull/3945)), plus 37-function cross-client `xref` mapping and 236-test EIP-2537 differential/invariant fuzzer across all 7 precompiles in [`spencer-tb/protocol-mesh#2`](https://github.com/spencer-tb/protocol-mesh/pull/2). | **Completed** |
| M4: Multi-Client Conformance, Ecosystem Security & Final Report | Week 7–8 | 6/6 L1 client conformance pass, merged EIP-155 >64-bit legacy chain ID multi-client exception fix ([`ethereum/execution-specs#3935`](https://github.com/ethereum/execution-specs/pull/3935) resolving [`#3930`](https://github.com/ethereum/execution-specs/issues/3930)) and boundary test suite ([`#3955`](https://github.com/ethereum/execution-specs/pull/3955)), EIP-7702 sweeper evaluation benchmark ([`Ethereum-dAI/rl-evals#4`](https://github.com/Ethereum-dAI/rl-evals/pull/4)), and EPF Final Report. | **Completed** |

## Possible challenges

- **Upstream Migration ("The Weld")**: The unification of `execution-spec-tests` and EELS into `ethereum/execution-specs` requires tracking evolving monorepo branch targets (`forks/amsterdam` $\rightarrow$ `forks/bogota`).
- **Client Transition Tools Differences**: Ensuring test fixtures and exception expectations pass uniformly across diverse client architectures (e.g. 64-bit bounds-checking vs. 256-bit arithmetic in legacy transaction decoding).

## Goal of the project

1. 100% test coverage for non-canonical coordinate encodings and subgroup boundary violations on EIP-2537 in the official Ethereum test suite.
2. Official PRs merged into `ethereum/execution-specs` and `spencer-tb/protocol-mesh`.
3. Verification that all major L1 execution clients produce identical state transitions under adversarial cryptographic calldata.

## Collaborators

### Fellows
- Roropuntou (`@Roropuntou`)

### Mentors
- EEST / STEEL Testing Team (Spencer `@spencer-tb`, Federico `@fselmo`, Mario Vega `@marioevz`, Dan `@danceratopz`)

## Resources
- EIP-2537 Specification: https://eips.ethereum.org/EIPS/eip-2537
- Ethereum Execution Layer Specification & Tests (`execution-specs`): https://github.com/ethereum/execution-specs
- Protocol Mesh (Cross-Client Mapping & Fuzzing): https://github.com/spencer-tb/protocol-mesh
- Ethereum dAI RL Evals (Harbor Benchmarks): https://github.com/Ethereum-dAI/rl-evals
