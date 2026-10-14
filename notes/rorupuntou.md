# Fellow Notes: Roropuntou

- **GitHub:** [@Roropuntou](https://github.com/Roropuntou)
- **Program:** Ethereum Protocol Fellowship (EPF Cohort 7)
- **Project Proposal:** [EEST: BLS12-381 (EIP-2537) Subgroup Checks and Field Deserialization Hardening](../projects/eest-eip2537-bls12-381-hardening.md)

---

## Development Updates

- **Development Updates & Upstream Contributions:**
  - Upstream PR [ethereum/execution-specs#3641](https://github.com/ethereum/execution-specs/pull/3641): EIP-2537 MSM and Pairing Hardening (co-authored with `@spencer-tb`, verified with mutation testing across 6 L1 clients).
  - Upstream PR [ethereum/execution-specs#3821](https://github.com/ethereum/execution-specs/pull/3821): EIP-2537 Hash-to-Curve (`MAP_FP_TO_G1` & `MAP_FP2_TO_G2`) boundary deserialization and modulus overflow hardening.
  - Development Update Report: [development-update-eip2537-hardening.md](./development-update-eip2537-hardening.md) ([HackMD](https://hackmd.io/npFjnHStSaWsJhWNJnyk9A))

---

## Research Resources & References

- [EIP-2537: Precompiles for BLS12-381 curve operations](https://eips.ethereum.org/EIPS/eip-2537)
- [Ethereum Execution Spec Tests (EEST)](https://github.com/ethereum/execution-spec-tests)
- [Ethereum Execution Layer Specification (EELS)](https://github.com/ethereum/execution-specs)
- [EELS Precompile Reference Implementation](https://github.com/ethereum/execution-specs/tree/master/src/ethereum/prague/vm/precompiled_contracts)
