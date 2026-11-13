---
title: EEST - EIP-2537 BLS12-381 Precompile Hardening & Conformance
author: Roropuntou (EPF Cohort 7)
type: slide
slideOptions:
  theme: black
  transition: slide
---

# EEST: EIP-2537 Hardening
### BLS12-381 Subgroup Checks, Field Deserialization & Cross-Client Fuzzing

**Fellow:** Roropuntou ([@Roropuntou](https://github.com/Roropuntou))  
**Program:** Ethereum Protocol Fellowship (Cohort 7)  
**Core PRs:** `execution-specs` [#3641](https://github.com/ethereum/execution-specs/pull/3641) (Merged), [#3935](https://github.com/ethereum/execution-specs/pull/3935) (Merged), [#3821](https://github.com/ethereum/execution-specs/pull/3821)/[#3945](https://github.com/ethereum/execution-specs/pull/3945), [#3955](https://github.com/ethereum/execution-specs/pull/3955), `protocol-mesh` [#2](https://github.com/spencer-tb/protocol-mesh/pull/2) & `execution-apis` [#972](https://github.com/ethereum/execution-apis/pull/972)/[#973](https://github.com/ethereum/execution-apis/pull/973) (Approved)  
**Ecosystem Security:** `Ethereum-dAI/rl-evals` [#4](https://github.com/Ethereum-dAI/rl-evals/pull/4) & [`kohaku-py v0.3.1`](https://github.com/rorupuntou/kohaku-py)  

---

## 1. The Core Problem
### Why Cryptographic Precompiles Break Consensus

- **EIP-2537** introduces 7 new precompiles for BLS12-381 curve arithmetic (`0x0b`–`0x11`).
- Historically, elliptic curve precompiles are the **#1 source of consensus splits** across client implementations (Go, Rust, C#, Java, Nim, C++).
- **Two Major Vulnerability Classes:**
  1. **Non-Canonical Deserialization:** 48-byte coordinates padded into 64-byte big-endian buffers. If the top 16 bytes (`0..15`) are non-zero, or coordinate $\ge p$, the precompile must revert.
  2. **Subgroup Membership Validation:** Points on the curve that do not belong to the prime-order subgroup $r$ (torsion points like $(0, 2)$) must be strictly rejected in `PAIRING` and `MSM`.

---

## 2. Identified Vulnerabilities & Edge Cases

```text
Calldata Chunk (MSM Pair i >= 2):
+-------------------------+-------------------------+-------------------------+
| Dirty Padding (0..15)   | Coordinate (16..63) >= p| Scalar s = 0 (Bypass?)  |
| [ 0x01 ... 0x00 ]       | [ x >= Modulus p ]      | Short-circuit bug?      |
+-------------------------+-------------------------+-------------------------+
```

- **Subgroup Check Bypass under Zero Scalars ($s = 0$):**
  - In $\sum s_i P_i$, a lazy client might see $s_i = 0$ and skip checking if $P_i \in \mathbb{G}_1 / \mathbb{G}_2$.
  - EIP-2537 strictly mandates subgroup validation **regardless** of scalar values.
- **Batched Dirty Top-16 Byte Padding:**
  - In multi-pair operations ($k \ge 2$), clients often validate pair 1 but miss offset errors in pairs 2 & 3.
- **Modulus Overflow Boundaries:**
  - Exact boundary evaluation at $x = p$, $x = p + 1$, $x = p + 2^{128}$, $x = 2^{381}-1$, $x = 2^{384}-1$.

---

## 3. The Proof: Mutation Testing in EELS
### Merged Upstream PR #3641 with Spencer (@spencer-tb)

- Co-authored and consolidated with EF Testing Lead **Spencer (`@spencer-tb`)**.
- **The Experiment:** Spencer intentionally mutated the official Python EELS reference implementation:
  - Relaxed subgroup check when scalar $s = 0$.
  - Removed top-16 byte padding verification on subsequent chunks.
- **The Finding:**
  > **Our 6 test cases were the ONLY tests in the entire global test suite that caught those specific mutant bugs.**
- Without these tests, a client shipping with these regressions would have passed all existing Ethereum tests.

---

## 4. Multi-Client Conformance (6/6 L1 Clients)

- Test vectors were generated via `evm t8n` state-transition tools across **all 6 major production Ethereum L1 clients**:

| Client | Implementation Language | Result |
|:---|:---:|:---:|
| **Geth** | Go | **PASSED** (100%) |
| **Erigon** | Go | **PASSED** (100%) |
| **Besu** | Java | **PASSED** (100%) |
| **Nethermind** | C# | **PASSED** (100%) |
| **Revm** | Rust | **PASSED** (100%) |
| **Ethrex** | Rust | **PASSED** (100%) |

- **Zero false positives. 100% consensus conformance.**

---

## 5. Hash-to-Curve Hardening & Cross-Client Fuzzing
### PRs #3821 / #3945 & `spencer-tb/protocol-mesh` PR #2

- **Hash-to-Curve (`MAP_FP_TO_G1` `0x10` & `MAP_FP2_TO_G2` `0x11`):**
  - **The $c_1$ Padding Blindspot:** In $\mathbb{F}_{p^2}$ ($c_0 + c_1 u$), top-16 big-endian padding on $c_1$ (offsets `64..79`) was untested.
  - Added 22 negative cases (66 Prague fixtures) covering offsets `15`/`64`/`79`, high-bit `0x80`, mid-padding byte `8`, simultaneous $(c_0, c_1) \ge p$, and 32B/64B truncated calldata.
- **Cross-Client Mapping & Differential Fuzzer (`protocol-mesh#2`):**
  - Mapped **37 EIP-2537 functions across all 8 execution clients** in `xref/data/tree.json`.
  - Built `fuzz/fuzz_eip2537.py` (**236/236 tests passing** across all 7 EIP-2537 precompiles, including `BLS12_G2MSM` and `BLS12_MAP_FP2_TO_G2`) verifying algebraic invariants ($P + \mathcal{O} = P$, pairing bilinearity, MSM cancellation, deterministic G1/G2 map-to-curve subgroup membership) and malformed input rejections.

---

## 6. Broader Protocol & Security Contributions
### EIP-155 Conformance, EIP-7702 Benchmarks & Kohaku Privacy Stack

- **EIP-155 Legacy Chain ID Conformance (`execution-specs` PR [#3935](https://github.com/ethereum/execution-specs/pull/3935) Merged in [`tests@v21.0.3` (`#3967`)](https://github.com/ethereum/execution-specs/issues/3967) & [#3955](https://github.com/ethereum/execution-specs/pull/3955)):**
  - Merged PR [#3935](https://github.com/ethereum/execution-specs/pull/3935) resolves issue [#3930](https://github.com/ethereum/execution-specs/issues/3930) in `test_chain_id.py` for legacy transactions with >64-bit chain IDs in `v` (`2**64` and `chain_id + 2**64`), fixing **72 mainnet Hive dashboard failures across Besu, Erigon, and Geth** (12 fails each in `consume-engine` and `consume-enginex`, tracked in [`#3967`](https://github.com/ethereum/execution-specs/issues/3967)); companion PR [#3955](https://github.com/ethereum/execution-specs/pull/3955) adds 270 boundary fixtures covering `chain_id = 0` (`v = 35, 36`) and sub-35 `v` values (`0, 26, 29, 34`).
- **EIP-7702 Sweeper Security Benchmark (`Ethereum-dAI/rl-evals` PR [#4](https://github.com/Ethereum-dAI/rl-evals/pull/4)):**
  - Paired Harbor benchmark (`sweeper-7702-refuse` & `sweeper-7702-single`) reproducing a real mainnet EIP-7702 delegation attack (block `26047682`) to test if wallet agents inspect `eth_getCode` for `0xef0100` (591 `pytest` checks passing).
- **Kohaku Privacy Stack (`kohaku-py v0.3.1` & `ethereum/kohaku#356`):**
  - Pure-Python SDK, CLI/TUI wallet (`edw`), offline Groth16 BN128 verifier, and 700,000-event `pp-state` auditor across Tornado Cash, 0xBow Privacy Pools, Railgun, and EIP-7702 + ERC-4337 v0.8 multi-shielding.

---

## 7. Fellowship Experience & Thank You

- **Deep Dive into Cryptographic Consensus:**
  - Transitioned from studying historical Ethereum PoW & BTC Relay into active Core Protocol security testing, cross-client specification mapping, and applied zero-knowledge cryptography.
  - Navigated monorepo branch refactors ("The Weld") and upstream rebases onto `forks/bogota` and `forks/amsterdam`.
- **Special Thanks:**
  - **Mario Havel** & EPF Organizers.
  - **Spencer (`@spencer-tb`)**, **Federico (`@fselmo`)**, **Mario Vega (`@marioevz`)**, and **Dan (`@danceratopz`)**.

---

# Questions & Discussion

- **Core Protocol PRs:**
  - [`execution-specs#3641`: Multi-Pair MSM & Pairing Hardening (Merged)](https://github.com/ethereum/execution-specs/pull/3641)
  - [`execution-specs#3935`: EIP-155 >64-Bit Chain ID Conformance (Merged in `tests@v21.0.3` #3967)](https://github.com/ethereum/execution-specs/pull/3935) & [`#3955`: EIP-155 `chain_id=0` & Sub-35 `v` Boundaries](https://github.com/ethereum/execution-specs/pull/3955)
  - [`protocol-mesh#2`: EIP-2537 Differential & Invariant Fuzzer (236/236 Tests)](https://github.com/spencer-tb/protocol-mesh/pull/2)
  - [`execution-specs#3821`](https://github.com/ethereum/execution-specs/pull/3821) / [`#3945`: Hash-to-Curve Extended Negative Vectors](https://github.com/ethereum/execution-specs/pull/3945)
  - [`execution-apis#972`](https://github.com/ethereum/execution-apis/pull/972) & [`#973`: `eth_simulateV1` Unreachable Error Code Cleanup (Approved)](https://github.com/ethereum/execution-apis/pull/973)
- **Ecosystem Security & Privacy:**
  - [`Ethereum-dAI/rl-evals#4`: EIP-7702 Sweeper Agent Benchmark](https://github.com/Ethereum-dAI/rl-evals/pull/4)
  - [`rorupuntou/kohaku-py` (`v0.3.1`)](https://github.com/rorupuntou/kohaku-py) & [`ethereum/kohaku#356`](https://github.com/ethereum/kohaku/issues/356)
- **Cohort PR & Report:** [#453](https://github.com/eth-protocol-fellows/cohort-seven/pull/453) | [HackMD](https://hackmd.io/npFjnHStSaWsJhWNJnyk9A)

---

<!-- ===================================================================== -->
<!-- SPEAKER SCRIPT (WORD-FOR-WORD PRESENTATION GUIDE)                     -->
<!-- ===================================================================== -->

# 🎙️ SPEAKER SCRIPT (Guión palabra por palabra para la presentación)

> **Instrucciones:**
> Este guión está pensado para hablar de forma relajada y clara durante 8 a 10 minutos.
> No necesitas memorizarlo: puedes leerlo con tranquilidad mientras pasas las diapositivas.

---

### Slide 1: Introduction (Tiempo estimado: 45 seg)

*"Hello everyone! My name is Roru (Roropuntou on GitHub), and today I am excited to present my work for EPF Cohort 7: **EEST BLS12-381 Precompile Hardening, Cross-Client Fuzzing, and Ecosystem Security**.*

*During this fellowship, my primary focus has been on hardening client conformance and consensus security for EIP-2537 in `ethereum/execution-specs` across merged PR #3641 and PRs #3821 / #3945, as well as cross-client mapping and invariant fuzzing in `protocol-mesh` PR #2—alongside contributions to EIP-155 conformance (merged PR #3935 and companion PR #3955), EIP-7702 agent security in `Ethereum-dAI/rl-evals` (#4), and Ethereum's Kohaku privacy stack (`kohaku-py`).*

*Let's dive into the core problem we set out to solve."*

---

### Slide 2: The Core Problem (Tiempo estimado: 1 min 15 seg)

*"EIP-2537 introduces seven cryptographic precompiles to Ethereum for BLS12-381 curve operations—covering point additions, multi-scalar multiplications, pairings, and hash-to-curve mappings.*

*Historically, elliptic curve precompiles are among the most dangerous components in Ethereum execution clients. Because different clients are written in different languages—Go, Rust, Java, C#, Nim—even tiny differences in boundary checks can lead to catastrophic consensus splits.*

*We identified two primary vulnerability classes:*
*First, **non-canonical field deserialization**. In EIP-2537, 48-byte coordinates are encoded in 64-byte big-endian fields. The leading 16 bytes must be strictly zero, and the coordinate must be strictly less than modulus p. Any deviation must trigger an immediate EVM revert.*

*Second, **subgroup membership validation**. Points lying on the curve that do not belong to the prime-order subgroup r—such as small cofactor torsion points like (0, 2)—must be strictly rejected in Pairing and MSM operations."*

---

### Slide 3: Identified Vulnerabilities & Edge Cases (Tiempo estimado: 1 min 15 seg)

*"To harden the specification, we designed test vectors specifically targeting tricky boundary conditions that developers often overlook:*

*First: **Subgroup Check Bypass under zero scalars**. In an MSM operation, an optimizing developer might inspect scalar s = 0 and immediately skip the multiplication. However, EIP-2537 strictly mandates that the input point MUST still be validated for subgroup membership, even if s = 0. We created tests pairing torsion points with scalar 0 to enforce this rule.*

*Second: **Dirty top-16 byte padding in batched calldata**. In multi-pair operations, clients often correctly check the padding for the first point, but pointer arithmetic or slicing bugs might cause them to skip padding checks in subsequent chunks.*

*Third: **Modulus overflow boundaries**. We tested exact boundary numbers: x = p, x = p + 1, x = p + 2^128, x = 2^381 - 1, and 2^384 - 1 to ensure that non-canonical numbers are never silently reduced modulo p."*

---

### Slide 4: The Proof: Mutation Testing in EELS (Tiempo estimado: 1 min 15 seg)

*"Our first set of test cases was consolidated and merged upstream in PR #3641, co-authored with core EF testing maintainer Spencer.*

*The most rewarding validation of this work was a **mutation testing experiment** performed by Spencer against the official EELS Python reference implementation.*

*Spencer intentionally introduced bugs into the reference precompiles—such as disabling subgroup checks when s = 0 or relaxing top-16 padding checks.*

*The result was remarkable: **Our 6 test cases were the only tests in the entire global test suite that caught those mutant bugs.** Without our tests, those regressions would have silently gone unnoticed."*

---

### Slide 5: Multi-Client Conformance (Tiempo estimado: 1 min)

*"Of course, having tests catch bugs is only useful if the tests themselves are strictly conformant and don't introduce false positives.*

*We generated state-transition test fixtures and executed them across all six production Layer 1 clients: Geth, Erigon, Besu, Nethermind, Revm, and Ethrex.*

*The result: **100% pass rate across all 6 clients.** All clients produced identical state roots, confirming zero regressions and complete consensus alignment."*

---

### Slide 6: Hash-to-Curve Hardening & Cross-Client Fuzzing (Tiempo estimado: 1 min 30 seg)

*"Next, we expanded our work across two fronts:*

*First, in **PRs #3821 and #3945**, we targeted the Hash-to-Curve precompiles `MAP_FP_TO_G1` (`0x10`) and `MAP_FP2_TO_G2` (`0x11`), adding 22 negative test cases (66 Prague fixtures) covering top-16 big-endian padding on both c0 and c1, simultaneous modulus overflows, and truncated 32-byte and 64-byte inputs.*

*Second, in **`protocol-mesh` PR #2**, we completed 100% of the cross-client precompile mappings—37 functions across all 8 execution clients—and built an automated differential and algebraic invariant fuzzer (`fuzz_eip2537.py`) with 236 passing checks across all 7 EIP-2537 precompiles verifying pairing bilinearity, identity laws, G1 and G2 MSM cancellations, map-to-curve subgroup membership, and malformed coordinate rejection."*

---

### Slide 7: Broader Protocol & Security Contributions (Tiempo estimado: 1 min 15 seg)

*"Working deeply with execution specs and cryptography also enabled three broader ecosystem contributions:*

*1. In **merged `execution-specs` PR #3935** (shipped in `tests@v21.0.3`, #3967) and companion **PR #3955**, we resolved issue #3930 for EIP-155 legacy chain IDs over 64 bits—fixing **72 mainnet Hive dashboard failures across Geth, Besu, and Erigon**—and added 270 boundary fixtures covering `chain_id = 0` and sub-35 `v` values.*

*2. In **`Ethereum-dAI/rl-evals` PR #4**, we built a paired Harbor benchmark reproducing a real mainnet EIP-7702 delegation sweeper attack at block 26047682, testing whether autonomous wallet agents inspect `eth_getCode` for `0xef0100` before sending ETH.*

*3. Finally, we built and released **`kohaku-py` (`v0.3.1`)**, a pure-Python SDK, CLI/TUI wallet (`edw`), offline Groth16 BN128 verifier, and state auditor for the Ethereum Foundation's Kohaku privacy stack—auditing over 700,000 events across Tornado Cash, 0xBow Privacy Pools, and Railgun, and reporting 3 production `pp-state` indexer edge cases upstream in `ethereum/kohaku#356`."*

---

### Slide 8: Fellowship Experience & Thank You (Tiempo estimado: 45 seg)

*"Reflecting on this cohort, this was an transformative journey—starting from historical Ethereum PoW and BTC Relay mechanics, and progressing into core protocol cryptographic testing, cross-client fuzzing, and applied zero-knowledge privacy tooling.*

*I want to express my deepest gratitude to Mario Havel, Spencer, Federico, Mario Vega, Dan, and the entire EPF community for their mentorship and support.*

*Thank you very much, and I'm happy to answer any questions!"*
