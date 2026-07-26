# Etheorem: machine-checked proofs for Ethereum's consensus spec, in Lean 4

**Fellows:** Ivan Anishchuk ([@IvanAnishchuk](https://github.com/IvanAnishchuk)) and
Raj Gill ([@irajgill](https://github.com/irajgill)) · **Mentor:** Leo Lara
([@leolara](https://github.com/leolara))

**Tagline:** Prove key correctness properties of Ethereum's consensus-layer
specification in Lean 4, on an executable spec that already passes the official
conformance vectors at the current fork frontier (Fulu and Gloas, with Heze in
review).

> Note: "Lean 4" here is the **theorem prover**, not the post-quantum "lean chain" /
> Beam chain. Etheorem formalizes the current production consensus spec; it is
> unrelated to (and complementary with) the lean-consensus client effort.

Etheorem is an executable formalization of Ethereum's consensus layer in
Lean 4. The same definitions run, pass the official conformance vectors at the current
fork frontier, and carry machine-checked proofs. Normally the running code, the tests,
and the proofs live in separate artifacts that drift apart. Here they're one set of
definitions, so a proof and a passing vector describe the same spec.

This is a joint proposal from two fellows, Ivan Anishchuk
(@IvanAnishchuk) and Raj Gill (@irajgill). etheorem is an existing upstream project.
Leo Lara started it and offered it as a cohort-seven Fellowship idea, so we're
contributors here, not the owners. Our Fellowship work is one thing, the *proofs*. The
spec already runs and passes the official vectors; what it doesn't have yet is
machine-checked proofs that it holds the properties the protocol depends on, and that's
the project.

It splits by layer, which is why we propose it together. Raj takes the SSZ layer,
finishing the three central serialization theorems so they hold for every type the codec
actually implements rather than the subset proved so far. Ivan takes the consensus layer
above it, the state-transition, merkleization, shuffle and fork-choice proofs, and wrote
the verification roadmap both lanes work from. Keeping the spec
conformance-green as the forks move is the oracle every proof is checked against, so
that upkeep comes with the work. Other contributors land their own pieces alongside.

## Motivation

Ethereum's consensus rules are the protocol every client must follow, one
specification, implemented identically across all of them. Today that protocol lives as
prose and executable Python pseudocode, trusted mainly through testing. Testing is
necessary, but it only exercises the cases someone thought to generate, so the
protocol's guarantees stay implicit, checked on the examples and unproven in general. A
machine-checked formalization is a spec you can run, test against the same vectors the
clients do, and *prove* properties of, so its guarantees hold for every input.

This is not abstract. Ethereum stopped finalizing twice in May 2023, four epochs without
finality on the 11th and nine on the 12th, participation dropping close to the two-thirds
supermajority the chain needs to finalize at all, when a surge of valid attestations to old
checkpoints forced the majority consensus client into repeated, expensive beacon-state
regeneration until it fell behind
([Prysm/Offchain Labs post-mortem](https://medium.com/offchainlabs/post-mortem-report-ethereum-mainnet-finality-05-11-2023-95e271dfd8b2);
[cascading-effects analysis](https://hackmd.io/kfBDZe9xTdObMN2gDJjXYQ)).

Prior formal verification of the beacon chain is real but partial, and none of it is
current. The ConsenSys Dafny effort (TACAS 2022, now archived read-only) proved SSZ
round-trip and injectivity and state-transition refinement for Phase 0, with its overflow
bounds assumed via axiom lemmas. Runtime Verification proved the
deposit contract's incremental Merkle tree (CAV 2020) and Gasper's accountable safety in
Coq, on an abstract model. Both efforts left concrete open ground. Neither ever verified
`is_valid_merkle_branch`, neither proved the shuffle is a permutation, and both left
merkleization resting on an uninterpreted hash. And neither tracks the protocol
as deployed in 2026; as of mid-2026 no Lean 4 formalization of the current consensus
spec existed at all.

[etheorem](https://github.com/etheorem/etheorem) already covers the ground that closes
this gap. It's an executable spec of SSZ, the beacon-chain containers and state
transitions, and fork choice, at the Fulu and Gloas forks, conformance-tested against the
official consensus-spec-tests. Its cryptographic layer is already well proved, a pure-Lean
SHA-256, roughly 65 SSZ theorems (round-trip, injectivity, bit-packing), and the
Poseidon2 proofs, with an auditable axiom footprint. That verified hash is what lets merkleization close
against a concrete hash instead of the uninterpreted one both prior efforts settled for.

Two gaps remain, one per layer. The SSZ theorems are strong but they don't yet reach
every type. The codec serializes and decodes containers that mix fixed- and
variable-size fields, but those containers sit outside the predicate the central
theorems quantify over, so the wire format the beacon blocks and states actually use is
implemented and unproved. The consensus layer above
SSZ is nearly bare. Two proof sites on main (a `uint64` bound and the builder-index
round-trip) sit against roughly sixty fork-choice and state-transition functions with no
proofs attached. The roadmap covers both.

Gloas (headlined by EIP-7732 ePBS) is targeted for H2 2026 and Heze (with FOCIL,
EIP-7805) is forming behind it, so the spec surface is moving while the correctness bar
rises. The work is already underway.
etheorem tracks the fork frontier with conformance green on both presets (the Heze layer
in review), and the roadmap's first proof target, `isValidMerkleBranch` completeness down
to etheorem's pure-Lean SHA-256, is in hand as the worked example the rest of the roadmap
follows.

## Project description

The project is the proofs, machine-checked correctness properties on etheorem's
executable consensus spec, and the roadmap Ivan wrote is the plan for it.

**Raj's lane, the SSZ layer.** SizzLean carries three central theorems: serialization
round-trips (`decode_encode`), serialization is injective (`serialize_injective`), and
an encoded value never exceeds the schema's size bound (`encode_size_le_max`). Each is
proved by cases over SSZ's type constructors, and the work is closing the arms one type
at a time until nothing the codec implements is left outside. The bitvector and bitlist
arms and the wide 128- and 256-bit integer arms are merged; mixed-field containers, the
hardest arm, are in review now. One arm remains behind it, `vector` and `list` over
variable-size elements, which reuses the same offset-table machinery element-wise; past
that, the ground the executable decoder opens as it grows.

**Ivan's lane, the consensus layer.** Everything above SSZ, in rising difficulty: the
structural and arithmetic guarantees, then merkleization, then the validator shuffle,
then fork choice and the FFG layer. The first target, completeness of Merkle-proof
verification, is already done and the rest follow its shape.

The lanes meet at merkleization, where the consensus proofs lean on SSZ's guarantees.
That is why this runs as two sides of one project. Mouz (@Mouzayan)
lands Gloas consensus proofs off her own list alongside us. All three of us align
through the roadmap so nobody double-proves.

Conformance is the ground underneath. etheorem passes the official vectors today, and
keeping it green as forks land is what anchors the proofs and keeps the spec current, so
that upkeep rides with the proving.

**Method.** The proving workflow is tool-assisted, with the Lean kernel as the trust
anchor. A proof carries the same guarantee no matter how it was produced, so the work
leans on Lean's automation and machine-drafted candidate proofs where they help, and the
kernel checks every step with every assumption written down. That workflow is what makes
this scope realistic at two people, and documenting it as a reproducible pipeline is
itself a deliverable other Lean 4 efforts can reuse.

Ivan also keeps a small experimental line alongside the proofs, using etheorem to
cross-check a real Rust client (moonglass). Mainly that means extraction, lifting a Rust
kernel into Lean and proving it matches the spec, with differential testing as a broader
check. Prototypes exist; he pushes it and reports what it finds.

## Specification

**The SSZ proofs (Raj).** Make the three central theorems total over the codec. Each is
stated by cases on SSZ's type constructors, so every unproved arm is a type the spec can
serialize but nobody has proved anything about. The one in review is the container that
mixes fixed- and variable-size fields, which is the shape `BeaconBlockBody` and
`BeaconState` themselves take. It is also the hardest. The decoder recovers field
boundaries from an offset table the encoder wrote, so the round-trip proof has to walk
that table and show the recovered regions tile exactly the encoder's output. Admitting
the constructor means carrying its preconditions too, the 32-bit offset bound the
encoder silently assumes among them. Extending `SupportedBounded` and the machine-checked
subset theorem in step keeps the predicates from drifting apart. One arm remains behind the containers,
`vector` and `list` over variable-size elements, the same offset-table codec walked
element-wise over a homogeneous sequence instead of a heterogeneous field list; closing
it is what makes the theorems total. Past that, the decoder's own scope is the
limit, and the proofs follow it out as it grows.

**The consensus proofs (Ivan).** Prove correctness properties of the spec above SSZ, in
order of rising difficulty so partial progress still counts. They start easy, the
arithmetic that stays in range
(`increaseBalance` never wraps, total balance under 2^64) and the structures that stay
consistent (validator and balance lists in lockstep across the transition), where earlier
projects already charted the way. Merkleization is next, completeness of
`isValidMerkleBranch` down to the pure-Lean hash, and the cached Merkle tree agreeing with
the spec root. A generalized-index library, in progress, carries that same completeness
proof from deposits out to the branches real consumers open, the ones a light client
verifies a header against, and the blob and data-column sidecar inclusion proofs. Then the
new ground. The validator shuffle (`computeShuffledPermutation`) is a true permutation,
never proved before, and past it sit the fork-choice store invariants and a start on
Casper FFG accountable safety. The full ordered list, with acceptance criteria and prior-art tags, is in
[etheorem#35](https://github.com/etheorem/etheorem/pull/35).

**Conformance.** Keep etheorem passing the official conformance vectors as the fork
frontier moves: land the Heze (FOCIL) fork layer, currently in review; absorb each spec
alpha and keep the full suite green; and account for every skipped vector explicitly, so a
coverage gap can never hide as a silent pass. That suite is the oracle every proof is
checked against, so this is the same work as the proofs.

**Cross-check (experimental, Ivan).** etheorem also gets used to cross-check moonglass, a
Rust consensus client. The real work is extraction, lifting a small Rust kernel into Lean
and proving it computes what the spec says, each proof a checkable result whether or not
the whole client is ever wired in. Extraction only pays off once etheorem has the
protocol proofs to match against, so the proofs come first and the cross-check rides on them.
Differential testing over shared and mutated inputs is the broader, cheaper companion, clean
so far. AWS's Cedar ships a Lean model with differential testing against its Rust
([arXiv:2407.01688](https://arxiv.org/abs/2407.01688)). The EF's zkEVM Verification
Project runs the same Charon+Aeneas→Lean pipeline these extraction proofs use
([arXiv:2605.30106](https://arxiv.org/abs/2605.30106)).

**Stack:** Lean 4 (pinned v4.29.1) + Lake; consensus-spec-tests pinned per re-pin
(currently v1.7.0-alpha.11); the existing FFI SHA-256 / BLS / KZG shims; Rust
(moonglass, ssz_rs), Charon + Aeneas for extraction; Python for the conformance and
differential drivers.

## Related work and differentiation

Why Lean 4, and not the tools the prior efforts reached for. Lean expresses an executable
spec and its machine-checked proofs in one language, so the same definitions that run
against the conformance vectors are the ones that carry proofs. It also sits in a live
Ethereum-adjacent Lean ecosystem, Nethermind's EVMYulLean on the
execution side and the Nyx leanSpec on the consensus side, so idioms and tooling carry
over. The earlier beacon-chain work chose Dafny and Coq; those
results are cited below and reused wherever they chart the path, and this formalization
tracks the current spec they predate.

- **ConsenSys / Dafny eth2.0 beacon chain** (Cassez, Fuller, Asgaonkar; TACAS 2022;
  archived read-only 2026): SSZ round-trip and injectivity for basic types, bitlists,
  and bitvectors, but not the mixed-field containers the beacon chain actually
  serializes; state-transition refinement; overflow bounds assumed via axiom lemmas
  (not discharged); fork-choice store invariants; FFG accountable safety under a fixed
  validator set. Merkleization
  root-equals-spec stayed differential-tested with `hash()` uninterpreted;
  `is_valid_merkle_branch` and a real shuffle were never proved. Phase 0 only.
- **Runtime Verification** (K + Coq): the deposit-contract incremental-Merkle proof
  (CAV 2020); Gasper accountable safety and plausible liveness in Coq on an abstract
  model with dynamic validator sets; a K state-transition model that was
  conformance-tested but never proved.
- **Apalache / TLA+**: bounded model-checking of 3SF accountable safety (Konnov et
  al. 2025), which stops short of deductive proof.
- **Nethermind EVMYulLean / Verified-zkEVM stack**: the adjacent Lean 4 ecosystem on
  the execution side; etheorem builds alongside, on the consensus layer.
- **Nyx Foundation `formal-leanSpec`**: an independent Lean 4 formalization of the
  post-quantum leanSpec (the lean-consensus / Beam-chain minimal spec). Parallel and
  complementary: it follows the future PQ spec, etheorem follows the spec running on
  mainnet now and the forks next in line.

This is the only Lean 4 formalization of the *current*
production consensus spec, conformance-tested at the fork frontier, now growing
machine-checked proofs in the exact spots prior efforts left open
(serialization total over the codec, merkle-branch completeness, a real shuffle,
merkleization over a verified hash).

## Roadmap

Cohort runs June to mid-November, kickoff at week 1 (mid-June); week 5 is mid-July,
where this proposal sits. Already underway from onboarding: six PRs merged between us,
three more in review (the Heze fork layer and the two mixed-field container PRs), the
first consensus proof done and the roadmap written, and the client cross-check and
extraction proofs prototyped. The two lanes run in parallel, month by month:

| Month (weeks) | Ivan, consensus layer | Raj, SSZ layer |
|---|---|---|
| **July** (wk 5–7) | Finalize + present this proposal; land Heze, the Merkle-branch completeness proof, and the roadmap upstream; start overflow safety + slot/epoch invariants | Land the mixed-field container arms, the hardest arm and the shape `BeaconState` takes |
| **August** (wk 8–12) | Finish the structural + arithmetic invariants; cached-tree merkleization equivalence; keep conformance green as the spec moves | Close the last composite arm, `vector` / `list` over variable-size elements (the container proof's offset-table machinery, element-wise); the three central theorems then hold for every type the codec implements. Package the size-bound and offset lemmas as the interface the merkleization proofs consume |
| **September** (wk 13–16) | **The big one:** prove the validator shuffle is a real permutation, with its committee cluster | Lift the theorems to the user surface: un-gate the `SSZ.roundtrip` corollary and instantiate round-trip + injectivity on the real Fulu / Gloas container schemas, `BeaconBlockBody` and `BeaconState` included; write up serializer non-malleability, now total over the codec |
| **October** (wk 17–20) | Fork-choice store invariants + FOCIL inclusion-list validity on the in-flight Heze layer; further merkleization proofs; begin the abstract FFG layer behind accountable safety | Decoder soundness, the reverse direction: an accepted buffer re-serializes to itself, one accepted encoding per value (prove it, or tighten the decoder where it is lenient; either result stands). Admit the Heze containers to the proved surface; start shrinking `decode_encode`'s axiom footprint toward the kernel trio |
| **November** (wk 21–22) | Consolidation: an axiom audit, the documented tool-assisted proving pipeline, the final report, the Devcon presentation. *Stretch:* a start on Casper FFG accountable safety in Lean 4 | Joint: the axiom audit covers both layers; joint final report |
| **Continuous** | Conformance kept green at each re-pin; the moonglass cross-check poked at as a background sideline | Both: dev updates at least every two weeks; docs + cleanup as filler |

*The schedule carries slack on purpose: the September shuffle is the likely slip point (the
first genuinely greenfield result), November's consolidation weeks double as buffer if an
earlier tier runs long, and the tiered success criteria below absorb the rest.*

## Possible challenges

- **Proof effort is open-ended**, especially the shuffle and deep-consensus targets.
  Mitigation: the targets escalate from templated (following prior Dafny work) to
  greenfield, so a partial run still lands standing results; the Merkle-branch proof
  already demonstrates the merkleization proof shape end to end; anything landed is a
  first for the current spec. If a target turns out intractable in the window, dropping
  it down a tier is a deliberate, documented call (the tiered success criteria are built
  for exactly that).
- **The fork frontier moves.** New spec alphas re-shape containers under the work.
  Mitigation: keeping conformance green absorbs this by design. Pin every claim to a
  named spec and vector release, and keep the skip accounting explicit.
- **Serialization proofs can only reach as far as the decoder does.** The central
  theorems quantify over a predicate, and a type outside that predicate is unproved even
  when the codec handles it. Closing the mixed-field container arm is exactly this
  problem; whatever the decoder does not yet implement stays out until it does. Stated
  openly to keep the proof claims honest.
- **Two fellows on one codebase can collide or drift.** Mitigation: the split is by
  layer, not by ticket, so the lanes touch different packages, and both work from the
  one roadmap. Where they do couple it runs one way, the consensus proofs leaning on
  SSZ results, and the exact form is for the proofs to settle.
- **The "Lean" name collision** (theorem prover vs lean-chain / Beam). Mitigation:
  disambiguate once, up front, as this proposal does.

## Goal of the project

By November, etheorem is a more thoroughly machine-checked reference for the current
consensus spec than it is today. Conformance stays green across the frontier through
Heze; its SSZ layer carries the three central theorems over every type the codec
implements; and the consensus layer above carries a body of proofs guided by the
roadmap. In concrete terms: serialization proved total over the codec, the structural and
arithmetic invariants, the Merkle-branch worked example, real progress on the shuffle and
fork-choice and FOCIL, and honest reporting where the deep-consensus targets and the
cross-check land short. The commitment is the progress and the documented pipeline, not
an exact theorem count, and the list flexes with the spec and the work.

Success at minimum: etheorem stays conformance-green as the frontier advances, with the
Heze (FOCIL) fork layer landed; the mixed-field container arms close, so round-trip,
injectivity and the size bound hold for the containers the beacon chain actually
serializes; the verification roadmap and the Merkle-branch completeness proof, down to
the pure-Lean SHA-256, land upstream; and a bounded set of the roadmap's early proofs
completes, the structural and arithmetic invariants, overflow safety and slot/epoch
monotonicity and length preservation among them. Each is a first for the current
consensus spec at that layer.

Target: the proof work climbs the roadmap. The big one is shuffle-is-a-permutation with
its committee cluster, the first proof of a real beacon-chain shuffle. Alongside
it, fork-choice store invariants, further serialization and merkleization proofs, and
FOCIL inclusion-list validity on the in-flight Heze layer (a satisfied inclusion list
forces the required transactions into the block).

Stretch: the deep consensus results the roadmap points at, a start on Casper FFG
accountable safety in Lean.

The moonglass cross-check runs in the background as experimental research, mainly extraction
proofs that lift small Rust kernels into Lean, with a broader differential check alongside.
The prototypes are there and the work continues.

Impact: etheorem's consensus layer gains machine-checked proofs in Lean 4, in spots
earlier efforts left open; etheorem grows into a more complete, fork-current executable
reference client teams can diff against; and the documented tool-assisted proving
pipeline lowers the bar for the next contributor.

## Collaborators

### Fellows

This is a joint proposal from two cohort-seven fellows, split by layer:

- **Ivan Anishchuk** ([@IvanAnishchuk](https://github.com/IvanAnishchuk)): the consensus
  layer, the verification roadmap, and conformance.
- **Raj Gill** ([@irajgill](https://github.com/irajgill)): the SSZ layer, the three
  central serialization theorems.

etheorem is a shared effort wider than the two of us. **Mouz**
([@Mouzayan](https://github.com/Mouzayan)) lands Gloas consensus proofs off her own
candidate list, and we align with her through the roadmap.

### Mentors

- **Leo Lara** ([@leolara](https://github.com/leolara)), etheorem author; proposed
  the Etheorem project idea upstream (cohort-seven PR #80).

## Resources

- Etheorem: https://github.com/etheorem/etheorem (start with CONTRIBUTING.md)
- Upstream project-idea entry: https://github.com/eth-protocol-fellows/cohort-seven/blob/main/projects/project-ideas.md
- Ivan's upstream work: etheorem PRs
  [#4](https://github.com/etheorem/etheorem/pull/4),
  [#5](https://github.com/etheorem/etheorem/pull/5) (merged),
  [#6](https://github.com/etheorem/etheorem/pull/6) (Heze, in review),
  [#22](https://github.com/etheorem/etheorem/pull/22) (fork-choice throw-faithfulness),
  [#35](https://github.com/etheorem/etheorem/pull/35) (the verification roadmap).
  The Merkle-branch completeness proof is implemented and goes upstream next.
- Raj's upstream work: etheorem PRs
  [#10](https://github.com/etheorem/etheorem/pull/10) (bitvector/bitlist arms),
  [#18](https://github.com/etheorem/etheorem/pull/18) and
  [#21](https://github.com/etheorem/etheorem/pull/21) (the wide `uintN` 128/256 arms and
  the machine-checked subset claim), all merged;
  [#28](https://github.com/etheorem/etheorem/pull/28) +
  [#29](https://github.com/etheorem/etheorem/pull/29) (mixed-field containers, in
  review), closing issue
  [#27](https://github.com/etheorem/etheorem/issues/27)
- moonglass (the Rust client): https://github.com/brech1/moonglass · cross-check scratch
  (experimental): [consensus-diff](https://github.com/IvanAnishchuk/consensus-diff),
  [moonglass-runner](https://github.com/IvanAnishchuk/moonglass-runner)
- Consensus specs: https://github.com/ethereum/consensus-specs (SSZ:
  `ssz/simple-serialize.md` on `master`)
- consensus-spec-tests: https://github.com/ethereum/consensus-spec-tests (pinned
  v1.7.0-alpha.11)
- Charon: https://github.com/AeneasVerif/charon · Aeneas:
  https://github.com/AeneasVerif/aeneas
- Prior FV: ConsenSys `eth2.0-dafny` (TACAS 2022); Runtime Verification
  deposit-contract proof (CAV 2020) and Gasper-in-Coq; Apalache/TLA+ 3SF
  (arXiv 2501.07958)
- Forks: EIP-7732 (ePBS, Gloas), EIP-7805 (FOCIL, Heze), EIP-8282 (builder
  execution requests); https://ethereum.org/roadmap/glamsterdam/
- Dev updates:
  [cohort-seven development-updates rows](https://github.com/eth-protocol-fellows/cohort-seven/blob/main/development-updates.md)
  for both of us · Ivan's [hosted updates](https://ivananishchuk.github.io/eth-protocol-fellowship/)
  · Raj's on [hackmd](https://hackmd.io/@irajgill)
