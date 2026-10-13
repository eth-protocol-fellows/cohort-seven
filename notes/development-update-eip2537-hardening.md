# EPF Cohort 7 — Development Update: EIP-2537 BLS12-381 Precompile Hardening & Upstream Integration

**Fellow:** Roropuntou ([@Roropuntou](https://github.com/Roropuntou))  
**Project:** [EEST: BLS12-381 (EIP-2537) Subgroup Checks and Field Deserialization Hardening](../projects/eest-eip2537-bls12-381-hardening.md)  
**Upstream PR:** [ethereum/execution-specs#3641](https://github.com/ethereum/execution-specs/pull/3641)  
**Co-author / Maintainers:** Spencer ([@spencer-tb](https://github.com/spencer-tb)), Federico ([@fselmo](https://github.com/fselmo))  

---

## 1. Executive Summary

This update details the design, implementation, and upstream integration of an edge-case test suite for **EIP-2537 (BLS12-381 curve operations)** within the official Ethereum test harness (`execution-spec-tests` / `ethereum/execution-specs`). 

Our contributions landed upstream in **PR #3641**, consolidated and co-authored with core EF testing maintainer **Spencer (`@spencer-tb`)**. The PR extends edge-case coverage across multi-pair Multi-Scalar Multiplications (`BLS12_G1MSM`, `BLS12_G2MSM`) and Pairing operations (`BLS12_PAIRING`). 

Crucially, mutation testing on the EELS reference implementation confirmed that **our test cases were the only tests in the entire global test suite capable of catching specific cryptographic mutant bugs**. Furthermore, all test cases were verified and pass across **6 production Layer 1 execution clients**: **Geth, Erigon, Besu, Nethermind, Revm, and Ethrex**.

---

## 2. Technical Scope & Implemented Edge Cases

EIP-2537 introduces seven cryptographic precompiles covering arithmetic, pairings, and field mappings over the BLS12-381 elliptic curve. Cryptographic precompiles are historically prone to consensus splits across diverse client runtimes (Go, Rust, C#, Java) due to subtle differences in boundary validation.

Our test vectors in `test_bls12_subgroup_and_deserialization_edge_cases.py` systematically isolate and stress several boundary conditions:

### A. Subgroup Check Bypass Prevention under Zero Scalars ($s = 0$)
- **Vulnerability Model:** In MSM operations ($\sum_{i=1}^k s_i P_i$), an unoptimized or lazy client might inspect $s_i = 0$ and immediately short-circuit without validating whether the corresponding curve point $P_i$ belongs to the prime-order subgroup $\mathbb{G}_1$ or $\mathbb{G}_2$.
- **Test Implementation:** Constructed test vectors using non-subgroup curve points (such as the affine order-3 torsion point $(0, 2) \in E(\mathbb{F}_p)$ where $y^2 = 4 \pmod p$) paired with scalar $s = 0$ as well as $s = 1$. EIP-2537 strictly mandates that every input point must be validated for subgroup membership regardless of the scalar value. The tests enforce strict reversion/failure.

### B. Dirty Top-16 Byte Padding in Batched Calldata
- **Vulnerability Model:** BLS12-381 field elements are serialized as 64-byte big-endian buffers representing 48-byte coordinates. The leading 16 bytes (indices `0..15`) must be strictly zero. In batched operations ($k \ge 2$), clients could correctly validate the first pair but fail to check padding bytes in subsequent chunks due to improper pointer arithmetic or slicing offsets.
- **Test Implementation:** Injected non-zero bytes (e.g. `0x01`, `0x80`, `0xff`) specifically into indices `0..15` of the second and third pairs of `BLS12_G1MSM` and `BLS12_G2MSM` inputs, guaranteeing that any dirty padding anywhere in the payload triggers immediate rejection.

### C. Modulus Overflow Boundaries ($x \ge p$)
- **Vulnerability Model:** Field elements must strictly satisfy $0 \le x < p$. When deserializing 48-byte integers, a client using non-canonical reductions (e.g. automatic reduction modulo $p$) rather than strict canonical validation might accept values in the range $[p, 2^{384}-1]$.
- **Test Implementation:** Tested exact boundary conditions: $x = p$, $x = p + 1$, and $x = 2^{384} - 1$ across both $G_1$ and $G_2$ coordinates in multi-pair batches.

### D. Scalar Boundary Equivalence & Linear Cancellation
- **Vulnerability Model:** Scalars in MSM operations are 32-byte integers. While scalars are evaluated modulo curve order $r$, edge conditions at bit 255/256 ($s = r, r+1, 2^{255}-1, 2^{256}-1$) and cancellations yielding the point at infinity ($\mathcal{O}$) must produce canonical empty or identity outputs without crashing or out-of-bounds panics.
- **Test Implementation:** Verified scalar boundary values and linear multi-pair combinations where $\sum s_i P_i = \mathcal{O}$.

### E. Calldata Length Misalignment
- Ensured strict rejection of payloads that are not exact integer multiples of pair lengths ($160k \pm 1$ bytes for $G_1$, $288k \pm 1$ bytes for $G_2$).

---

## 3. Upstream Collaboration & Verification

### Monorepo Rebase & Retargeting to `forks/bogota`
- Our PR was initially created against `forks/amsterdam`. When the EF maintainers announced the feature freeze on the `amsterdam` development branch, maintainer **Federico (`@fselmo`)** retargeted and rebased the branch onto `forks/bogota`.

### Mutation Testing in EELS Reference Spec
- Core maintainer **Spencer (`@spencer-tb`)** integrated our vectors and performed mutation analysis against the EELS Python reference implementation.
- By intentionally introducing mutations into the reference precompiles (e.g. disabling subgroup checks when $s=0$ or relaxing coordinate padding checks), Spencer verified that **our 6 test cases were the only tests in the entire global test suite that caught those specific mutant bugs**.

### Multi-Client Execution Pass (6/6 Clients)
The test suite was run against the transition tools (`t8n`) of all major Ethereum L1 clients:
- **Geth** (Go) — PASSED
- **Erigon** (Go) — PASSED
- **Besu** (Java) — PASSED
- **Nethermind** (C#) — PASSED
- **Revm** (Rust) — PASSED
- **Ethrex** (Rust) — PASSED

This demonstrates 100% cross-client consensus conformance with zero false positives or client breakages.

---

## 4. Next Steps & Ongoing Work

1. **Hash-to-Curve Hardening (`MAP_FP_TO_G1` & `MAP_FP2_TO_G2`):**
   - Implement edge-case coverage for field element deserialization and sign-bit handling in Hash-to-Curve precompiles (`0x10`, `0x11`).
2. **Differential Fuzzing Harness:**
   - Develop automated differential fuzzing campaigns comparing client precompile execution under pseudo-randomized malformed calldata.
3. **EPF Cohort Tracking & Reporting:**
   - Maintain active synchronization with EEST upstream changes and document findings for the EPF final presentation.
