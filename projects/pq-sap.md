# Post-Quantum Stealth Addresses as an ERC-5564 Scheme Extension

The project will specify, implement, benchmark, and test a post-quantum stealth address scheme as a new [ERC-5564](https://eips.ethereum.org/EIPS/eip-5564) scheme ID, with client-agnostic test vectors, a reference library, and a written security analysis.

## Motivation

Ethereum is preparing for the post-quantum transition well in advance: proposals like EIP-8141 aim to support multiple signature schemes and make the transition seamless.

Stealth address cryptography is already widely used ([Fluidkey](https://www.fluidkey.com/), [Cloacked](https://www.clkd.xyz/), [Umbra](https://umbraprivacy.com/), and other ERC-5564-based protocols) to preserve privacy while keeping human-readable handles — it is much cheaper than heavier privacy systems and offers better UX for compartmentalizing funds.

To display a balance, a wallet downloads the announcement logs and in linear time checks a one-byte view tag to skip non-matching entries without decoding everything. On a candidate match, it computes the shared secret `S = v·R`, confirms the payment, and with the spending key converts it into the private key controlling the address. All of this data is public and permanent: it can be collected now and decoded later. With a quantum computer running Shor's algorithm to recover the viewing and spending keys of every stealth payment ever made ("harvest now, decrypt later", NIST IR 8547).

This project addresses that at the ERC level: a new scheme ID working against the existing ERC-5564/ERC-6538 contracts. Onchain spending from a post-quantum key awaits protocol-level signature support (e.g., EIP-8141) and is scoped out of the core deliverable. Recent discussions like Native UTXOs presuppose exactly such an ERC-5564 scheme for their post-quantum, key-only recipients. And the sender-names-the-address model constrains the derivation constructions this project evaluates.

We’ll measure speed, gas costs and e2e implementation of a stealth address protocol based on 4337 with pq-safe key-exchange and zk-prooved ownership to enable cheaper spending during the pq transition according to [pq.ethereum.org](http://pq.ethereum.org)

## Project description

**Proposed solution.** Using lattice-based key encapsulation: Learning With Errors (LWE) and its Module variant (MLWE), the basis of NIST's ML-KEM — for the key-exchange step, we will produce a new ERC-5564 extension: a post-quantum-ready scheme ID. Per [ePrint 2025/112,](https://eprint.iacr.org/2025/112) the LWE approach *In some cases* brings an increase in parsing speed together with future-proofing the Native UTXO design and its ERC-5564 use case.

The parsing trick: the errors mixed into the polynomial equations are easy for the recipient to discover. After plugging in their real secret values, the error falls out as a subtraction from the publicly broadcast value. Without the key, separating the noise from the structure is the hard problem, for quantum computers too. We can also generate a proof of such statement for spending instead of using full-pq signature.

Log verifiability provided by EIP-8304 (a stretch goal, below) would be the cherry on top allowing to verify completeness proofs that a scan returned all announcements, none omitted.

The work divides into one research question and one engineering deliverable.

**Research question: sender-side address derivation.** In the SECP256K1 scheme the sender builds the recipient's one-time address from the meta-address and a shared secret (`P = K + hash(S)·G`) without ever seeing the private key. That doesn't move over cleanly to FIPS 204/205. An ML-DSA key *can* be shifted additively, but unlike an EC point the shift grows the secret's norm (so the signer needs a wider bound, `β' = τ·2η`) and it has to land on the rounded public key. Hash-based keys can't be rerandomized at all. We looked at two constructions.

* **(A)** Borrow key-blinding techniques from the signature literature (Tor onion services use them) to shift an ML-DSA key by a value derived from the shared secret; the stealth address is the hash of the blinded verification key. This keeps the ERC-5564 property that the sender computes the address. Signing works, and the signatures pass stock FIPS 204 verifiers (dilithium-py, liboqs) and the ZKNOX verifier deployed on Sepolia (its level-2 Dilithium2 profile). A fresh error term `e'` per stealth key kills the linkability, and it costs the sender nothing. There's a test suite, conformance vectors, a live receive-and-spend on Sepolia through an ERC-4337 account, and a Lean 4 proof (no sorries) of the blinding correctness identity, the widened bound, and the fact that ownership and signing-key possession are the same thing. What's left for A isn't engineering, it's the security write-up: mainly the widened-`z` distribution at `β' = τ·2η`, and unlinkability across payments.

* **(B)** Use ML-KEM only for the shared secret and have the recipient derive a fresh standardized keypair from it. Only standardized operations, but the sender can't compute the address on its own, so the announcement flow changes and unlinkability needs its own analysis. Fine for account-transfer flows, not for sender-names-the-address ones. We keep it as the fallback if A's security argument doesn't hold up.

**Threat model.** One thing shapes both choices. Ethereum's own [PQ roadmap](http://pq.ethereum.org) points out that harvest-now-decrypt-later hits confidentiality, not ownership: recording announcements today doesn't let anyone steal funds later. So the urgent piece is the ML-KEM detection layer, which protects who-paid-whom permanently. Spending is a live threat, and it migrates through account abstraction whenever a quantum computer actually shows up, which is the path the EF roadmap takes anyway. That's why A is the forward-looking spend path, the ZK proof is its cheaper on-chain version, and B is the fallback, all on top of the same confidentiality core that's the real priority. The reason for picking A and the reason for dropping B are both deliverables.

**Deliverables:**

* A draft spec for the new ERC-5564 scheme ID: meta-address encoding, announcement format (ML-KEM ciphertext as the ephemeral key, plus view tag), and the chosen key derivation. There's a working draft already, along with a running log of the design decisions; it freezes after the security analysis and a round of community review.
* Test vectors as versioned JSON in a public repo, including ones that must fail: wrong view tag, malformed ciphertext, wrong recipient. Any implementation can check itself against them without touching our code. This one's done. The v0 vectors are in the repo, deterministic, with the negative cases, and an independent TypeScript port already matches them byte for byte.
* A reference library: scanning, viewing keys (can see payments, can't spend), and proof of possession, meaning you sign a challenge with the derived key to show you control the address, no transaction needed. Done in Python, with a TypeScript scanning client alongside it. The crypto primitives come from spec-faithful pure-Python implementations (kyber-py, dilithium-py) for the executable spec, cross-checked against the audited liboqs; we only write the protocol layer on top. There's also a separate Noir prototype for the zero-knowledge ownership version of the proof.
* Security analysis in pq is quite complicated topic and prooving that something is safe now does not meant that no new attack would be discovered in NIST approved list. we will try to make the best effort by relying on Lean4 proofs and existiing academic research.

* Benchmarks as a runnable harness. We reproduced the 2025/112 scan experiment from 0x3327/pq-sap (about 23.8 µs per announcement at 80k, the number behind the ~66.8%-vs-Curvy figure) and ran our scheme against a DKSAP baseline on the same machine. The Curvy run and the L2 pricing are still to do. The cost report is partly there: an ML-KEM-768 ciphertext is 1,088 bytes where today's ephemeral key is 33, and we've measured the on-chain side (account deployment around 6.2M gas, roughly 8.2M per ML-DSA verification, ~22 kB public key). What's left is writing it up with mainnet and L2 gas at current schedules.

**Stretch goals** — optional, none part of the core scope:

An end-to-end spend demonstration. The near-term route is done. We spent from a stealth address on Sepolia through an ERC-4337 account (Kohaku pq-account, ZKNOX verifiers). Their factory derives the account address via CREATE2 from the public keys, so the stealth address is just the counterfactual account address of the derived key, spendable today with no protocol change. It works with construction A, since the sender computes the derived pubkey. The whole receive-and-spend went through on Sepolia, and the per-address costs (deployment, ~22 kB ML-DSA pubkey storage, 1.7–8.4M gas per verification depending on the scheme) go into the cost report. Two things we don't paper over: ZKNOX's on-chain verifier is a level-2 Dilithium2 profile, not our default ML-DSA-65, and its account wants an ECDSA signature next to the PQ one, so what we showed is the plumbing working end to end, not a pure post-quantum spend. The long-term route, native spending via EIP-8141 (still a Draft, not scheduled for a fork), stays out of scope unless it reaches a devnet during the cohort.

Verifiable announcement discovery for light clients against a proving-capable log index (EIP-7745 / EIP-8304, under the Pureth umbrella), if such an index shows up on a devnet. This is also where the sub-linear scanning question sits. Beating the linear scan is an old open problem (see the [ethresear.ch](http://ethresear.ch) "improving stealth addresses" thread), and a post-quantum version of it would reuse the same ownership-proof machinery from the spend side.

## Specification

As a base reference we'll use the paper published by the [folks from Serbia](https://eprint.iacr.org/2025/112) and take their reference implementation (`0x3327/pq-sap`, Rust) as the starting point. What it doesn't have and we’ll compliment is a real ERC-5564 scheme ID, a key derivation that lands on a standardized signature scheme, conformance vectors, and execution benchmarks. The full flow in Python pseudocode.

```python
# --- Setup (recipient) -------------------------------------------------
k, K = MLKEM768.keygen()          # spending keypair
v, V = MLKEM768.keygen()          # viewing keypair
meta_address = encode(SCHEME_ID, K, V)   # published via ENS or ERC-6538

# --- Send (sender) -----------------------------------------------------
K, V = resolve(meta_address)
R, S = MLKEM768.encaps(V)         # R: ciphertext = ephemeral pubkey, S: shared secret
stealth_address = derive_stealth_address(K, S)   # construction A or B
view_tag = H(S)[0:VIEW_TAG_BYTES]
announcer.announce(SCHEME_ID, stealth_address, R, view_tag)
send_assets(stealth_address)

# --- Scan (recipient, or any viewing-key holder) -----------------------
for ann in announcements_since(last_scan):
    S_i = MLKEM768.decaps(v, ann.R)
    if H(S_i)[0:VIEW_TAG_BYTES] != ann.view_tag:
        continue                  # cheap rejection path
    if derive_stealth_address(K, S_i) == ann.stealth_address:
        record_payment(ann, S_i)

# --- Spend (recipient only; requires spending key k) -------------------
p = derive_stealth_key(k, S)      # one-time private key
assert verifies(SIG_SCHEME, pubkey(p), sign(p, msg), msg)
# On-chain spend semantics: out of core scope (see Project description).

# --- Prove possession (no transaction, no protocol support needed) -----
proof = prove_possession(p, challenge)               # sign(p, H(challenge, stealth_address))
assert verify_possession(stealth_address, challenge, proof)
```

**Notes:**

The viewing key v alone detects and views payments; spending needs sk\_spend too. Standard viewing/spending separation.

Because the shared secret S itself, not its hash, derives the address here, view tags longer than 1 byte stay safe. The paper's own numbers say it barely matters though: full-hash tags bought about 0.7% over 1 byte at 1M announcements. So 1 byte is the default.

ML-KEM-768 by default, and the spec stays parameter-agile across the 512/768/1024 sets. VIEW\_TAG\_BYTES = 1. One caveat that came out of building the spend path: on-chain you're stuck with whatever verifier is actually deployed, which today means a level-2 Dilithium2 profile, not our level-3 default.

Decisions the spec still has to close, in order: (1) construction A vs. B. A is the working default now (built, tested, spent on-chain, partly proved in Lean), and B stays as the fallback; the final call is gated on the security write-up. (2) target signature scheme for derived keys: ML-DSA family by default, since it matches the deployed verifiers, with SLH-DSA documented as the alternative if we end up on B. (3) which parameter level, given the deployed-verifier reality above. (4) announcement encoding.

## Roadmap

Worth saying up front: it’s 2026, code is cheap. A lot of the engineering already exists as a prototype, which is what most of this write-up is reporting. So the estimates below are really about finishing and hardening each piece.

* Benchmark reproduction + key-blinding bibliography — mostly done. The paper's numbers from 0x3327/pq-sap are reproduced. What's left is finishing the key-blinding reading (Tor v3 blinding, Eaton et al. Latincrypt 2021 and the work citing it) and locking the A/B call with mentors. Call it a week of the original three.

* v0 spec + conformance vectors — done, not frozen yet. The executable Python spec and the versioned JSON vectors with negatives are in the repo. What remains is a review round with mentors and the community, then the freeze. Everything after builds against the frozen spec, same as before.

* Security analysis — ~4–5 weeks, Unlinkability across payments, sender can't derive the spending key, viewing/spending separation, the widened-z distribution at β' = τ·2η, parameter analysis. Formal verification with lean4 using vcvio for algebraic and game-based testing.

* Reference library — done in v0, Python plus a TypeScript scanning client, built against the vectors. Remaining work is polish and keeping it in sync as the spec freezes. 

* Benchmarks + cost report — ~2 weeks. The scan benchmarks and the on-chain gas numbers are measured already. What's left is L2 pricing, mainnet, and writing the report up.

* ERC draft + wrap-up — ~2 weeks. PR to ethereum/ERCs, conformance run, final report and Devcon presentation.

A few things landed ahead of the plan too: the Lean proofs, the Noir zero-knowledge ownership prototype, and a live receive-and-spend on Sepolia (a stretch goal). So the shape shifted. The engineering came in early, which frees cohort time for the security analysis and the ERC draft and leaves more room for the stretch goals. Weekly dev updates per EPF process, everything published as I go.

## Possible challenges

* Construction A may not survive its own security analysis. The engineering risk is basically gone now: it's built, it signs, it spends on-chain, and the correctness identity is proved in Lean. What's left is the actual security argument, and key blinding for lattice signatures is still active research. The specific worry is the widened-z distribution at β' = τ·2η, which leaks differently than a plain Dilithium signature. We keep B (standardized operations only) as a real fallback, and a well-documented negative result on A still counts as a deliverable.

* The protocol surface moves. Which post-quantum signature scheme Ethereum picks, and when accounts get signature agility, are still open. We [hit a live version of this](https://sepolia.etherscan.io/address/0xB9443280E49d728301384Fdbd07Eea8628d1b135): the only PQ verifier deployed on Sepolia is a level-2 Dilithium2 profile, not our level-3 default, and EIP-8141 is still on the way. So the spend path is a moving target. The core scheme doesn't lean on any of it, which is the whole reason the protocol-dependent parts sit in the stretch goals.

* Announcements are ~1 KB instead of 33 bytes. An ML-KEM-768 ciphertext is 1,088 bytes, and the cost report gives the numbers at current gas schedules with L2 profiles included.

* Scanning is linear in registry size. View tags cut the constant, not the asymptotics, and the benchmarks show the curve across registry sizes. Beating linear is an old open problem (the [ethresear.ch](http://ethresear.ch) "improving stealth addresses" thread), and a post-quantum version of it is future work, not something we're promising here.

* We don't roll our own crypto. The executable spec binds to spec-faithful pure-Python ML-KEM and ML-DSA (kyber-py, dilithium-py) and cross-checks against the audited liboqs; our code there is the protocol layer. Two honest exceptions: the TypeScript scanning client re-implements the ML-DSA polynomial math because the audited JS library doesn't expose it, and the Noir circuit implements the ownership relation directly. Both are validated against the conformance vectors, which are the security core of the suite. The negative vectors do the heavy lifting.

## Goal of the project

Success means these things exist publicly:

1. An ERC draft with a registered ERC-5564 scheme ID for the post-quantum scheme.
2. Conformance vectors, negative vectors included, that any independent implementation can validate against.
3. A reference library that passes them: sender flow, scanning, viewing keys, proof of possession.
4. The security analysis of the chosen derivation, including lean4 proofs with vcvio with the rejection rationale for the alternative. 
5. Benchmarks and the cost report against the elliptic-curve and lattice baselines.


## Collaborators

### Fellows

Skas

### Mentors

[Tamaghna](https://github.com/RazorClient) expressed interest in contributing

## Resources

**Papers and articles**

* Mikić, Srbakoski, Praška — _Post-Quantum Stealth Address Protocols_, ePrint 2025/112: [https://eprint.iacr.org/2025/112](https://eprint.iacr.org/2025/112)
* Mikić, Srbakoski, Praška — *More Efficient Stealth Address Protocol* (hybrid Curvy/MLWE, deliberately not quantum-secure to stay Ethereum-compatible — the gap this project closes): [https://arxiv.org/abs/2504.06744](https://arxiv.org/abs/2504.06744)
* asanso — *Towards practical post quantum stealth addresses* (CSIDH-based, ethresear.ch, April 2023 — earliest PQ stealth design for Ethereum; isogeny-based, pre-standardization): [https://ethresear.ch/t/towards-practical-post-quantum-stealth-addresses/15437](https://ethresear.ch/t/towards-practical-post-quantum-stealth-addresses/15437)
* *Post-Quantum Threats to Ethereum Privacy* (ethresear.ch, March 2026 — harvest-now-decrypt-later and why privacy migration is more time-sensitive than authentication migration): [https://ethresear.ch/t/post-quantum-threats-to-ethereum-privacy/24450](https://ethresear.ch/t/post-quantum-threats-to-ethereum-privacy/24450)
* Eaton et al. — *Post-Quantum Key-Blinding for Authentication in Anonymity Networks*, Latincrypt 2021 (primary prior art for construction A)
* Lyubashevsky — *Basic Lattice Cryptography: The concepts behind Kyber (ML-KEM) and Dilithium (ML-DSA)* (tutorial, ePrint)
* Wahrstätter et al. — _BaseSAP: Modular Stealth Address Protocol for Programmable Blockchains_, IEEE TIFS 2024
* Mikić, Srbakoski — *Elliptic Curve Pairing Stealth Address Protocols* (Curvy): [https://arxiv.org/abs/2312.12131](https://arxiv.org/abs/2312.12131)
* Buterin — _An incomplete guide to stealth addresses_: [https://vitalik.eth.limo/general/2023/01/20/stealth.html](https://vitalik.eth.limo/general/2023/01/20/stealth.html)
* *Native UTXOs on Ethereum* (ethresear.ch, July 2026): [https://ethresear.ch/t/native-utxos-on-ethereum/25368](https://ethresear.ch/t/native-utxos-on-ethereum/25368)
* *VCVio: Verified Cryptography in Lean via Oracle Effects and Handlers* ePrint 2026/899: [https://eprint.iacr.org/2026/899.pdf](https://eprint.iacr.org/2026/899.pdf)

**Standards and specifications**

* ERC-5564 — Stealth Addresses: [https://eips.ethereum.org/EIPS/eip-5564](https://eips.ethereum.org/EIPS/eip-5564)
* ERC-6538 — Stealth Meta-Address Registry: [https://eips.ethereum.org/EIPS/eip-6538](https://eips.ethereum.org/EIPS/eip-6538)
* EIP-8141 — Frame Transactions (Draft): [https://eips.ethereum.org/EIPS/eip-8141](https://eips.ethereum.org/EIPS/eip-8141)
* NIST FIPS 203 (ML-KEM), FIPS 204 (ML-DSA), FIPS 205 (SLH-DSA); NIST IR 8547 (PQC transition)
* Tor rend-spec-v3, key-blinding appendix (the deployed Ed25519 blinding design)
* Ethereum quantum-resistance overview: [https://ethereum.org/roadmap/future-proofing/quantum-resistance/](https://ethereum.org/roadmap/future-proofing/quantum-resistance/)

**Implementations and benchmarks**

* `0x3327/pq-sap` — ePrint 2025/112 reference implementation and scan benchmarks (Rust): [https://github.com/0x3327/pq-sap](https://github.com/0x3327/pq-sap)
* `ethereum/kohaku` `packages/pq-account` — ERC-4337 post-quantum accounts (ZKNOX ML-DSA/Falcon verifiers, deployed on Sepolia and Arbitrum Sepolia); the near-term spend route: [https://github.com/ethereum/kohaku/tree/master/packages/pq-account](https://github.com/ethereum/kohaku/tree/master/packages/pq-account)
* PQClean / liboqs and the pq-crystals reference code — audited ML-KEM/ML-DSA implementations the library binds to
* `GiacomoPope/kyber-py`, `dilithium-py` — spec-faithful Python implementations, used for vector generation and cross-checking
* ScopeLift stealth-address-sdk / Umbra and Fluidkey deployments — DKSAP baseline behavior and announcer usage in production
