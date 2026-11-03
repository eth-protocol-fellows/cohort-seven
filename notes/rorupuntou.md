# Fellow Notes: Roropuntou

- **GitHub:** [@Roropuntou](https://github.com/Roropuntou)
- **Program:** Ethereum Protocol Fellowship (EPF Cohort 7)
- **Project Proposal:** [EEST: BLS12-381 (EIP-2537) Subgroup Checks and Field Deserialization Hardening](../projects/eest-eip2537-bls12-381-hardening.md)
- **Final Report & Development Update:** [development-update-eip2537-hardening.md](./development-update-eip2537-hardening.md) ([HackMD Mirror](https://hackmd.io/npFjnHStSaWsJhWNJnyk9A))
- **Final Presentation Slides & Script:** [final-presentation-slides-and-script.md](./final-presentation-slides-and-script.md)

---

## 1. Core EPF Project Contributions (EEST / EELS & Protocol Mesh)

- **Merged PR [`ethereum/execution-specs#3641`](https://github.com/ethereum/execution-specs/pull/3641) — EIP-2537 MSM & Pairing Hardening:**
  - Co-authored and consolidated with EF Testing Lead **Spencer (`@spencer-tb`)** and rebased onto `forks/bogota` with **Federico (`@fselmo`)**.
  - Added multi-pair boundary and negative test vectors across `BLS12_G1MSM`, `BLS12_G2MSM`, and `BLS12_PAIRING` (subgroup check bypass under zero scalars $s=0$, batched top-16-byte dirty padding, modulus overflows $x \ge p$, scalar boundary equivalence, and calldata length misalignment).
  - **Mutation Testing Proof:** Verified by `@spencer-tb` against the EELS Python reference implementation that our 6 test cases were the **only tests in the entire global suite** capable of catching specific reference precompile mutant bugs, passing across **6/6 production L1 clients** (Geth, Erigon, Besu, Nethermind, Revm, Ethrex).
- **Upstream PR [`spencer-tb/protocol-mesh#2`](https://github.com/spencer-tb/protocol-mesh/pull/2) — EIP-2537 Cross-Client Mapping (`xref`) & Differential Invariant Fuzzer (`fuzz`):**
  - Completed 100% of the cross-client precompile mappings (37 functions across 8 execution clients: Geth, Reth, Nethermind, Besu, Erigon, Ethrex, Nimbus, Evmone) in `xref/data/tree.json`.
  - Implemented `fuzz/fuzz_eip2537.py` (236/236 tests passing across all 7 EIP-2537 precompiles, including `BLS12_G2MSM` and `BLS12_MAP_FP2_TO_G2`) to fuzz algebraic invariants ($P + \mathcal{O} = P$, commutativity, pairing bilinearity, MSM equivalence, deterministic G1/G2 map-to-curve subgroup membership) and non-canonical deserialization rejections.
- **Upstream PRs [`ethereum/execution-specs#3821`](https://github.com/ethereum/execution-specs/pull/3821) / [`#3945`](https://github.com/ethereum/execution-specs/pull/3945) — EIP-2537 Hash-to-Curve (`MAP_FP_TO_G1` & `MAP_FP2_TO_G2`) Hardening:**
  - Rebased onto `forks/bogota` in `#3945`, adding 22 negative test cases (66 Prague fixtures) for precompiles `0x10` and `0x11` covering top-16-byte big-endian padding (`0..15` on $c_0$, `64..79` on $c_1$, high-bit `0x80`, mid-padding byte `8`), simultaneous dual-coordinate modulus overflows ($\ge P$), and truncated calldata (32B / 64B).
- **Merged PR [`ethereum/execution-specs#3935`](https://github.com/ethereum/execution-specs/pull/3935) & Companion PR [`#3955`](https://github.com/ethereum/execution-specs/pull/3955) — EIP-155 Legacy Chain ID Multi-Client Conformance & Boundary Hardening:**
  - Merged PR `#3935` resolves issue [`#3930`](https://github.com/ethereum/execution-specs/issues/3930) in `test_chain_id.py` by allowing `[TransactionException.INVALID_SIGNATURE_VRS, TransactionException.INVALID_CHAINID]` when legacy transaction `v` values encode >64-bit chain IDs (`smallest_65_bit_chain_id` at `2**64` and `chain_id_plus_2_64`), accommodating both 64-bit bounds-checking clients (EELS, Nethermind, Nimbus) and 256-bit arithmetic clients (Erigon); companion PR `#3955` adds 270 boundary fixtures covering `chain_id = 0` (`v = 35, 36`) and sub-35 `v` values (`0, 26, 29, 34`).

---

## 2. Broader Ethereum Ecosystem & Applied Cryptography Contributions

- **Agent Security Benchmarks — PR [`Ethereum-dAI/rl-evals#4`](https://github.com/Ethereum-dAI/rl-evals/pull/4):**
  - Designed and verified a paired Harbor benchmark (`sweeper-7702-refuse` & `sweeper-7702-single`) reproducing a real mainnet EIP-7702 delegation sweeper attack at block `26047682` (`0xef0100` delegation indicator).
  - Evaluates whether autonomous wallet agents inspect `eth_getCode` and abort before getting their ETH swept, paired with a clean EOA control task (591 passing `pytest` checks + end-to-end Anvil fork verification).
- **Ethereum Privacy Stack (`ethereum/kohaku` & `ethereum/desktop-wallet`) — [`rorupuntou/kohaku-py`](https://github.com/rorupuntou/kohaku-py) (`v0.3.1`):**
  - Built an open-source, pure-Python companion SDK, CLI/TUI wallet (`edw`), and cross-protocol state auditor for the Ethereum Foundation's Kohaku privacy stack (`51/51` conformance and unit tests passing):
    - **Tornado Cash:** 220-round MiMC-7, 20-level Merkle tree, ETH & ERC-20 pools (DAI, USDC, USDT, WBTC), pure-Python Groth16 BN128 offline proof verifier (`v0.3.1`), and on-chain `RelayerRegistry` (`164,887` events) with live ENS & `/v1/status` probing.
    - **0xBow Privacy Pools:** Circomlib Poseidon, dynamic LeanIMT, Association Set Provider (ASP) inclusion tracking, and `ragequit` auditing.
    - **Railgun:** Pure-Python `0zk` Bech32m codec and full-history multi-tree ($d=16$) UTXO auditor across `144,326` mainnet operations (`293,107` UTXO leaves, `256,451` nullifiers, `36,656` active UTXOs).
    - **Sovereign Wallet Engine:** Argon2id + ChaCha20-Poly1305 blinded datastore (`~/.edw`), Dual EIP-7702 + ERC-4337 v0.8 multi-shield bundling (`Simple7702Account.executeBatch`), ERC-5564 stealth addresses, and native RFC 1928 Tor SOCKS5 routing.
  - **Upstream Kohaku & Desktop Wallet Contributions:**
    - [`ethereum/kohaku#335`](https://github.com/ethereum/kohaku/issues/335): Python SDK & conformance suite tracking thread with upstream maintainers.
    - [`ethereum/kohaku#356`](https://github.com/ethereum/kohaku/issues/356): Reported 3 subtle `pp-state` Saga indexer edge cases uncovered during our 700,000-event audit (rotated `.hot.jsonl.gz` 404 race, `0x1869f` pure-unshield sentinel & `65,537` mixed-batch leaf offset boundary in Railgun, and Era 1 vs. Era 2 ABI transition at block `17,558,492`).
    - [`ethereum/desktop-wallet#107`](https://github.com/ethereum/desktop-wallet/pull/107) & [`#113`](https://github.com/ethereum/desktop-wallet/pull/113): CLI Tornado pool/note inspection commands and Python implementation documentation link.

---

## 3. Research Resources & References

- [EIP-2537: Precompiles for BLS12-381 curve operations](https://eips.ethereum.org/EIPS/eip-2537)
- [Ethereum Execution Layer Specification & Tests (`ethereum/execution-specs`)](https://github.com/ethereum/execution-specs)
- [Protocol Mesh (`spencer-tb/protocol-mesh`)](https://github.com/spencer-tb/protocol-mesh)
- [Ethereum dAI RL Evals (`Ethereum-dAI/rl-evals`)](https://github.com/Ethereum-dAI/rl-evals)
- [Kohaku Python SDK & Sovereign Wallet (`rorupuntou/kohaku-py`)](https://github.com/rorupuntou/kohaku-py)
