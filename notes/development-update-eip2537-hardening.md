# EPF Cohort 7 — Final Report & Development Update: EIP-2537 BLS12-381 Hardening, Cross-Client Fuzzing & Ecosystem Security

**Fellow:** Roropuntou ([@Roropuntou](https://github.com/Roropuntou))  
**Project:** [EEST: BLS12-381 (EIP-2537) Subgroup Checks and Field Deserialization Hardening](../projects/eest-eip2537-bls12-381-hardening.md)  
**Presentation Slides & Script:** [final-presentation-slides-and-script.md](./final-presentation-slides-and-script.md)  
**Core Upstream PRs:**  
- [`ethereum/execution-specs#3641`](https://github.com/ethereum/execution-specs/pull/3641) — *MERGED*: EIP-2537 Multi-Pair MSM & Pairing Hardening (co-authored with `@spencer-tb`)  
- [`spencer-tb/protocol-mesh#2`](https://github.com/spencer-tb/protocol-mesh/pull/2) — *OPEN*: EIP-2537 Differential & Algebraic Invariant Fuzzer (`fuzz/fuzz_eip2537.py`, 236 tests across all 7 precompiles; rebased onto `#4`)  
- [`ethereum/execution-specs#3821`](https://github.com/ethereum/execution-specs/pull/3821) / [`#3945`](https://github.com/ethereum/execution-specs/pull/3945) — *OPEN*: EIP-2537 Hash-to-Curve (`MAP_FP_TO_G1` & `MAP_FP2_TO_G2`) Extended Negative Vectors (rebased onto `forks/bogota`)  
- [`ethereum/execution-specs#3935`](https://github.com/ethereum/execution-specs/pull/3935) — *MERGED & SHIPPED IN [`tests@v21.0.3` (`#3967`)](https://github.com/ethereum/execution-specs/issues/3967)*: EIP-155 >64-Bit Legacy Chain ID Multi-Client Conformance Fix (resolves [`#3930`](https://github.com/ethereum/execution-specs/issues/3930), fixing 72 mainnet Hive dashboard failures across Besu, Erigon, and Geth — 12 fails each in `consume-engine` and `consume-enginex`)  
- [`ethereum/execution-specs#3955`](https://github.com/ethereum/execution-specs/pull/3955) — *OPEN*: EIP-155 Legacy Chain ID Zero & Sub-35 v Boundary Tests (270 fixtures)  
- [`ethereum/execution-apis#972`](https://github.com/ethereum/execution-apis/pull/972) & [`#973`](https://github.com/ethereum/execution-apis/pull/973) — *APPROVED*: Remove unreachable `-38024` (`Sender is not an EOA`) and `-38023` error codes from `eth_simulateV1` (scheduled for merge after Nov 16th JSON-RPC standards call)  
**Ecosystem & Applied Cryptography PRs / Releases:**  
- [`Ethereum-dAI/rl-evals#4`](https://github.com/Ethereum-dAI/rl-evals/pull/4) — *OPEN*: Harbor EIP-7702 Delegation Sweeper Evaluation Benchmark (`sweeper-7702-refuse` & `sweeper-7702-single`)  
- [`rorupuntou/kohaku-py`](https://github.com/rorupuntou/kohaku-py) (`v0.3.1`), [`ethereum/kohaku#335`](https://github.com/ethereum/kohaku/issues/335), [`ethereum/kohaku#356`](https://github.com/ethereum/kohaku/issues/356), [`ethereum/desktop-wallet#107`](https://github.com/ethereum/desktop-wallet/pull/107) & [`#113`](https://github.com/ethereum/desktop-wallet/pull/113)  
**Mentors & Maintainers:** Spencer ([@spencer-tb](https://github.com/spencer-tb)), Federico ([@fselmo](https://github.com/fselmo)), Mario Vega ([@marioevz](https://github.com/marioevz)), Dan ([@danceratopz](https://github.com/danceratopz))  

---

## 1. Executive Summary

This report consolidates the design, implementation, and upstream verification of our contributions during **Ethereum Protocol Fellowship (EPF) Cohort 7**, structured across two layers:

### Layer 1: Core Execution Layer Testing & Differential Fuzzing (`execution-specs` & `protocol-mesh`)
1. **PR [`ethereum/execution-specs#3641`](https://github.com/ethereum/execution-specs/pull/3641) (EIP-2537 MSM & Pairing Hardening — *MERGED*):** Co-authored with core EF testing maintainer **Spencer (`@spencer-tb`)**, extending edge-case coverage across `BLS12_G1MSM`, `BLS12_G2MSM`, and `BLS12_PAIRING`. Mutation testing on the EELS reference implementation confirmed that **our test cases were the only tests in the entire global test suite capable of catching specific cryptographic mutant bugs**, passing with 100% conformance across **6 production L1 execution clients** (Geth, Erigon, Besu, Nethermind, Revm, and Ethrex).
2. **PR [`spencer-tb/protocol-mesh#2`](https://github.com/spencer-tb/protocol-mesh/pull/2) (EIP-2537 Differential & Invariant Fuzzer):** Contributed cross-client `xref` precompile mappings prior to upstream `#4` completion in `xref/data/tree.json`, and built an automated differential/invariant fuzzer (`fuzz/fuzz_eip2537.py`, 236/236 tests passing across all 7 precompiles) validating algebraic identities, multi-pair cancellations, and non-canonical coordinate/padding rejections.
3. **PRs [`ethereum/execution-specs#3821`](https://github.com/ethereum/execution-specs/pull/3821) / [`#3945`](https://github.com/ethereum/execution-specs/pull/3945) (EIP-2537 Hash-to-Curve Hardening):** Rebased onto `forks/bogota` in `#3945`, adding 22 parameterized negative cases (66 Prague fixtures) for `BLS12_MAP_FP_TO_G1` (`0x10`) and `BLS12_MAP_FP2_TO_G2` (`0x11`) covering top-16-byte big-endian padding violations (`0..15` and `64..79`), simultaneous modulus overflows ($\ge P$), and truncated calldata boundaries (32B / 64B).
4. **PRs [`ethereum/execution-specs#3935`](https://github.com/ethereum/execution-specs/pull/3935) & [`#3955`](https://github.com/ethereum/execution-specs/pull/3955) (EIP-155 Legacy Chain ID Conformance & Lower Boundary Tests):** Resolved issue [`#3930`](https://github.com/ethereum/execution-specs/issues/3930) in `#3935` (*MERGED* by `@danceratopz` and tracked in release [`tests@v21.0.3` (`#3967`)](https://github.com/ethereum/execution-specs/issues/3967)), fixing **72 mainnet Hive dashboard failures across Besu, Erigon, and Geth** (12 fails each in `consume-engine` and `consume-enginex`). Followed up in `#3955` (*OPEN*, 270 fixtures) covering lower-bound edge cases ($v \le 36$), including strict rejection of chain ID 0 ($v \in \{35, 36\}$) and accommodating client divergence on sub-35 $v$ values ($v \in \{0, 1, 26, 29, 34\}$).
5. **PRs [`ethereum/execution-apis#972`](https://github.com/ethereum/execution-apis/pull/972) & [`#973`](https://github.com/ethereum/execution-apis/pull/973) (`eth_simulateV1` Error-Code Cleanup — *APPROVED*):** Removed unreachable `-38024` (`Sender is not an EOA`, resolves `#936`) and `-38023` (resolves `#930`) error codes from `src/eth/execute.yaml` and `docs-api/api/methods/eth_simulateV1.mdx` after verifying across 6 execution clients (go-ethereum, Erigon, reth, Besu, Nethermind, ethrex) and Hive `rpc-compat` (passing on go-ethereum, Nethermind, Erigon, Besu, and reth); scheduled for merge after the Nov 16th JSON-RPC standards call.

### Layer 2: Broader Ethereum Security & Applied Cryptography (`rl-evals` & `kohaku-py` / `edw`)
5. **PR [`Ethereum-dAI/rl-evals#4`](https://github.com/Ethereum-dAI/rl-evals/pull/4) (EIP-7702 Sweeper Security Benchmark):** Created a paired Harbor evaluation benchmark (`sweeper-7702-refuse` and `sweeper-7702-single`) reproducing a real mainnet incident at block `26047682` to test whether autonomous wallet agents inspect `eth_getCode` for `0xef0100` delegation bytecode before transferring ETH (591 passing `pytest` checks).
6. **Kohaku Privacy Stack (`kohaku-py v0.3.1` & `ethereum/kohaku#356`):** Built a pure-Python SDK, CLI/TUI wallet (`edw`), offline Groth16 BN128 verifier, and cross-client conformance suite for the EF Kohaku privacy stack (Tornado Cash, 0xBow Privacy Pools, Railgun, ERC-5564, and EIP-7702 + ERC-4337 v0.8 multi-shielding), uncovering and reporting 3 production `pp-state` indexer edge cases upstream in [`ethereum/kohaku#356`](https://github.com/ethereum/kohaku/issues/356).

---

## 2. Technical Scope & Implemented Edge Cases (EIP-2537)

EIP-2537 introduces seven cryptographic precompiles covering arithmetic, pairings, and field mappings over the BLS12-381 elliptic curve. Cryptographic precompiles are historically prone to consensus splits across diverse client runtimes (Go, Rust, C#, Java, Nim, C++) due to subtle differences in boundary validation.

### A. Subgroup Check Bypass Prevention under Zero Scalars ($s = 0$) — PR #3641
- **Vulnerability Model:** In MSM operations ($\sum_{i=1}^k s_i P_i$), an unoptimized or lazy client might inspect $s_i = 0$ and immediately short-circuit without validating whether the corresponding curve point $P_i$ belongs to the prime-order subgroup $\mathbb{G}_1$ or $\mathbb{G}_2$.
- **Test Implementation:** Constructed test vectors using non-subgroup curve points (such as the affine order-3 torsion point $(0, 2) \in E(\mathbb{F}_p)$ where $y^2 = 4 \pmod p$) paired with scalar $s = 0$ as well as $s = 1$, enforcing strict rejection regardless of scalar value.

### B. Dirty Top-16 Byte Padding in Batched & Hash-to-Curve Calldata — PRs #3641, #3821 / #3945
- **Vulnerability Model:** BLS12-381 field elements are serialized as 64-byte big-endian buffers representing 48-byte coordinates. The leading 16 bytes (indices `0..15` for $c_0$ / `Fp`, and `64..79` for $c_1$ in `Fp2`) must be strictly zero.
- **Test Implementation:**
  - In `BLS12_G1MSM` and `BLS12_G2MSM` (`#3641`), injected non-zero bytes (`0x01`, `0x80`, `0xff`) into indices `0..15` of the second and third pairs in batched calls ($k \ge 2$).
  - In `MAP_FP_TO_G1` (`0x10`) and `MAP_FP2_TO_G2` (`0x11`) (`#3821` / `#3945`), tested dirty padding at boundary byte `15` (`0x01` and high-bit `0x80`), byte `0` of $c_1$, mid-padding byte `8`, dirty padding paired with a valid `Spec.P - 1` coordinate, and simultaneous dirty padding on both $c_0$ and $c_1$.

### C. Modulus Overflow Boundaries ($x \ge p$) — PRs #3641, #3821 / #3945
- **Vulnerability Model:** Field elements must strictly satisfy $0 \le x < p$. Clients performing automatic reduction modulo $p$ rather than canonical range validation could accept values in $[p, 2^{384}-1]$.
- **Test Implementation:** Tested exact boundaries $x = p$, $x = p + 1$, $x = p + 2^{128}$, $x = 2^{381} - 1$, and $x = 2^{384} - 1$ across single coordinates, multi-pair batches, and simultaneous $(c_0, c_1)$ overflows in `Fp2` (66 total Prague fixtures filling cleanly in `#3945`).

### D. Scalar Boundary Equivalence, Linear Cancellation & Calldata Lengths — PRs #3641, #3945
- Verified scalar boundary values ($s = r, r+1, 2^{255}-1, 2^{256}-1$) and multi-pair linear cancellations yielding $\mathcal{O}$.
- Enforced strict rejection of unaligned MSM payloads ($160k \pm 1$ bytes for $G_1$, $288k \pm 1$ bytes for $G_2$) and truncated Hash-to-Curve payloads (32-byte single word and 64-byte single `Fp` for `Fp2`).

---

## 3. Cross-Client Mapping (`xref`), Differential Fuzzing & EIP-155 Conformance

### Cross-Client EIP-2537 Mapping & Invariant Fuzzer — `spencer-tb/protocol-mesh` PR #2
- **Complete 8-Client Precompile Mapping (`xref/data/tree.json`):** Mapped 37 EIP-2537 precompile entry points, subgroup checks, field decoders, and gas schedules across **Geth, Reth (`evm2`), Nethermind, Besu, Erigon, Ethrex (`levm`), Nimbus, and Evmone**. Every precompile (`bls12_g1_add`, `bls12_g1_msm`, `bls12_map_fp_to_g1`, `bls12_g2_add`, `bls12_g2_msm`, `bls12_map_fp2_to_g2`, `bls12_pairing`) now resolves 100% of its cross-client references via `./xref.sh show <op>`.
- **Differential & Invariant Fuzzer (`fuzz/fuzz_eip2537.py`):** Built a 577-line fuzzer and 236-test suite verifying all 7 EIP-2537 precompiles:
  - Point-at-infinity identity ($P + \mathcal{O} = P$), commutativity ($P + Q = Q + P$), and negation cancellation ($P + (-P) = \mathcal{O}$).
  - G2 addition and MSM multi-pair scalar cancellation yielding $\mathcal{O}_{\mathbb{G}_2}$, strict rejection of torsion points outside the prime-order subgroup, and dirty top-16-byte padding rejection on $\mathbb{F}_{p^2}$ coordinates.
  - Field mapping invariants for `MAP_FP_TO_G1` and `MAP_FP2_TO_G2` across canonical inputs, modulus overflows ($x \ge p$), and non-standard calldata boundaries.
  - Pairing bilinearity ($e(aG_1, bG_2) = e(abG_1, G_2)$) and multi-pair negation cancellation ($e(P, Q) \cdot e(-P, Q) = 1$).

### EIP-155 Legacy Chain ID Conformance & Boundary Tests — PRs #3935 (*MERGED*) & #3955 (*OPEN*)
- **PR [`#3935`](https://github.com/ethereum/execution-specs/pull/3935) (Resolves [`#3930`](https://github.com/ethereum/execution-specs/issues/3930), Tracked in [`tests@v21.0.3` (`#3967`)](https://github.com/ethereum/execution-specs/issues/3967)):** Accommodates both 64-bit bounds-checking clients (`INVALID_SIGNATURE_VRS`) and 256-bit arithmetic clients (`INVALID_CHAINID`) for >64-bit chain IDs in legacy transactions, merged by maintainer `@danceratopz` and fixing **72 mainnet Hive dashboard failures across Besu, Erigon, and Geth** (12 fails each in `consume-engine` and `consume-enginex`).
- **PR [`#3955`](https://github.com/ethereum/execution-specs/pull/3955) (Lower Boundary & Chain ID Zero Hardening):** Added 270 Spurious Dragon+ fixtures covering $v \le 36$: enforces strict rejection of chain ID 0 ($v \in \{35, 36\}$) with `INVALID_CHAINID`, and accommodates client divergence for sub-35 values ($v \in \{0, 1, 26, 29, 34\}$) between bounds checks and Geth's uint64 underflow.

---

## 4. Broader Ethereum Security & Applied Cryptography

### EIP-7702 Sweeper Task Pair in Harbor — `Ethereum-dAI/rl-evals` PR #4
- Designed `sweeper-7702-refuse` (reproducing a real mainnet EIP-7702 sweeper attack at block `26047682` where recipient EOA `0x1cce...be3b` holds `0xef0100...` delegation code pointing to a sweeper contract) paired with `sweeper-7702-single` (clean EOA control task).
- Added rubric checks (`sweeper-blind-send` and `sweeper-single-noop`) and verified end-to-end against a local Anvil mainnet fork at block `26047682` with **591 passing `pytest` checks**.

### Kohaku Privacy Stack & State Auditing — `kohaku-py v0.3.1` & `ethereum/kohaku#356`
- Developed [`kohaku-py`](https://github.com/rorupuntou/kohaku-py) (`v0.3.1`), a pure-Python SDK, sovereign CLI/TUI wallet (`edw`), offline Groth16 BN128 verifier, and cross-client conformance suite (`51/51` tests passing) unifying **Tornado Cash**, **0xBow Privacy Pools** (Poseidon + LeanIMT + ASP), **Railgun** (`0zk` Bech32m + multi-tree UTXO auditor), **ERC-5564 Stealth Addresses**, and **Dual EIP-7702 + ERC-4337 v0.8** multi-shielding.
- Audited `144,326` Railgun operations and `164,887` Tornado RelayerRegistry events from `pp-state` snapshots, discovering and reporting 3 subtle indexer edge cases upstream in [`ethereum/kohaku#356`](https://github.com/ethereum/kohaku/issues/356).

---

## 5. Upstream Verification Summary

- **Mutation Testing in EELS Reference Spec:** Confirmed by `@spencer-tb` in PR `#3641` that our vectors uniquely catch reference precompile mutants (e.g. $s=0$ subgroup check bypass and coordinate padding relaxation).
- **6/6 L1 Client Conformance:** Verified passing across **Geth, Erigon, Besu, Nethermind, Revm, and Ethrex**.
