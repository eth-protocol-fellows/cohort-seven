# Etheorem: machine-checked proofs for Ethereum's consensus spec, in Lean 4

**Fellows:** Ivan Anishchuk ([@IvanAnishchuk](https://github.com/IvanAnishchuk)) and
Raj Gill ([@irajgill](https://github.com/irajgill)) · **Mentor:** Leo Lara
([@leolara](https://github.com/leolara))

**Tagline:** Prove key correctness properties of Ethereum's consensus-layer
specification in Lean 4, on an executable spec that already passes the official
conformance vectors at the current fork frontier (Fulu and Gloas, with Heze in
review).

> Note: "Lean 4" here is the **theorem prover**, not the post-quantum "lean chain" /
> Beam chain. Etheorem formalizes the consensus spec running on mainnet and the forks
> next in line; it is unrelated to (and complementary with) the lean-consensus effort.

## Motivation

Ethereum's consensus layer has one specification, written as executable Python, and every
consensus client is expected to conform to it. Conformance is checked mostly by test vectors generated from that
spec. Clients run them, they pass, and that is the evidence. Vectors only cover the cases
someone generated, so the properties the protocol relies on stay checked by example and
unproven in general. A
machine-checked formalization is a spec you can run, test against the same vectors the
clients do, and *prove* properties of, so its guarantees hold for every input.

Ethereum specs keep changing form, toward something you can actually run and check. The
Yellow Paper set the pattern for the execution layer, prose and mathematics that clients
read and reimplemented. The consensus layer skipped that stage and has been executable
Python since Phase 0. The execution layer caught up later with
[EELS](https://github.com/ethereum/execution-specs), an executable Python reference
implementation whose test fixtures clients validate against.
Each step made the spec more directly runnable, and testable against what clients actually
ship. Lean 4 is the next step, a spec that still runs and still passes the vectors, and
that you can prove things about on top of that.

Prior formal verification of the beacon chain is real but partial. The ConsenSys Dafny
effort (TACAS 2022, now archived read-only) proved SSZ round-trip and injectivity and
state-transition refinement for Phase 0, with its overflow bounds assumed via axiom
lemmas. Runtime Verification proved the deposit contract's incremental Merkle tree
(CAV 2020) and Gasper's accountable safety in Coq, on an abstract model. Both left
concrete ground open. The Dafny model assumes `is_valid_merkle_branch` outright and does
not implement the spec's shuffle, so neither effort proved either one, and in both
merkleization rests on an uninterpreted hash.

None of those efforts tracks the protocol as deployed in 2026. There is Lean 4 work next door, Nyx's
`formal-leanSpec`, but it follows the lean-Ethereum spec rather than the one running on
mainnet, so as far as we know the current production consensus spec had no Lean 4
formalization before this one.

[etheorem](https://github.com/etheorem/etheorem) is where that gap can actually be
closed. It's an executable spec of SSZ, the beacon-chain containers and state
transitions, and fork choice, at the Fulu and Gloas forks, conformance-tested against the
official consensus-spec-tests. Its lower layers already carry machine-checked proofs: a
pure-Lean SHA-256 and the Poseidon2 proofs on the crypto side, roughly 65 SSZ theorems
(round-trip, injectivity, bit-packing) on the serialization side, all with an auditable
axiom footprint. That verified hash is what lets merkleization close against a concrete
hash instead of the uninterpreted one both prior efforts settled for.

Two gaps remain, one per layer. The SSZ theorems are strong but they don't yet reach
every type. The codec serializes and decodes containers that mix fixed- and
variable-size fields, but those containers sit outside the predicate the central
theorems quantify over, so the wire format the beacon blocks and states actually use is
implemented but not yet proved. The consensus layer above
SSZ is nearly bare. `main` currently carries nine theorems across four files: a `uint64`
bound, the builder-index round-trips, some PTC-window bounds. The fork layer they sit in
has over a hundred spec functions, nearly sixty in fork choice alone. The roadmap covers
both gaps.

Gloas (headlined by EIP-7732 ePBS) is targeted for H2 2026 and Heze (with FOCIL,
EIP-7805) is forming behind it, so the spec surface is moving while the correctness bar
rises. The work is already underway. The Heze layer is in review, and the roadmap's first
proof target, `isValidMerkleBranch` completeness down to etheorem's pure-Lean SHA-256, is
in progress as the worked example the rest of it follows.

## Project description

Etheorem is an executable formalization of Ethereum's consensus layer in
Lean 4. The same definitions run, pass the official conformance vectors at the current
fork frontier, and carry machine-checked proofs. Normally the running code, the tests,
and the proofs live in separate artifacts that drift apart. Here they're one set of
definitions, so a proof and a passing vector describe the same spec.

etheorem is an existing upstream project. Leo Lara started it and offered it as a
cohort-seven Fellowship idea, so we're contributors here, not the owners.

Our Fellowship work is one thing, the *proofs*. What the spec lacks is machine-checked
proofs of the properties the protocol depends on, and the roadmap is the plan for getting
them. The work splits by layer, which is why the two of us propose it
together. Other contributors work their own pieces alongside.

**Raj's lane, the SSZ layer.** SizzLean, etheorem's SSZ package, carries three central
theorems: serialization
round-trips (`decode_encode`), serialization is injective (`serialize_injective`), and
an encoded value never exceeds the schema's size bound (`encode_size_le_max`). The work is
closing them type by type until nothing the codec implements is left outside. The
bitvector and bitlist
arms and the wide 128- and 256-bit integer arms are merged; mixed-field containers, the
hardest arm, are in review now. One arm remains behind it, `vector` and `list` over
variable-size elements, which reuses the same offset-table machinery element-wise.

**Ivan's lane, the consensus layer.** Everything above SSZ, in rising difficulty: the
structural and arithmetic guarantees, then merkleization, then the validator shuffle,
then fork choice and the FFG layer. The first target, completeness of Merkle-proof
verification, is underway.

The lanes meet at merkleization, where the consensus proofs lean on SSZ's guarantees.
The roadmap keeps everyone aligned so nobody double-proves.

Conformance is the ground underneath. etheorem passes the official vectors today, and
keeping it passing as forks arrive happens alongside the proofs, not after them.

**Method.** The workflow is tool-assisted, with the Lean kernel as the trust anchor.
Statements and proofs are both drafted with tool help, including LLM-drafted candidates.
Statements get the closer scrutiny. A proof of the wrong statement is worth nothing and
the kernel will not catch that, so each statement is checked against the spec code it is
about, by code review and by tool-assisted audit. Proofs lean on Lean's automation first,
`simp` and `omega` for the arithmetic and structural obligations, `bv_decide` for the
bit-level ones.

For verification it doesn't matter how proofs are generated, they just need to be correct.
Every proof is kernel-checked, the build is `sorry`-free, and each result carries a
`#print axioms` audit, so the trust footprint is visible per theorem. The Merkle-branch
proof, still in progress, currently depends only on Lean's three kernel axioms. Where a
result does rest on an added axiom, we name it. That is what makes this scope realistic
for a small team, and the pipeline is worth writing down on its own, other Lean 4 efforts
can probably reuse it.

## Specification

**The SSZ proofs (Raj).** Make the three central theorems total over the codec. Each is
stated by cases on SSZ's type constructors, so every unproved arm is a type the spec can
serialize but nobody has proved anything about. The one in review is the container that
mixes fixed- and variable-size fields, which is the shape `BeaconBlockBody` and
`BeaconState` themselves take. It is also the hardest. The decoder recovers field
boundaries from an offset table the encoder wrote, so the round-trip proof has to walk
that table and show the recovered regions tile exactly the encoder's output. Admitting
the constructor means carrying its preconditions too, the 32-bit offset bound the
encoder silently assumes among them. One arm remains behind the containers, `vector` and
`list` over variable-size elements. It uses the same offset-table codec, applied element-wise
to a homogeneous sequence rather than to a heterogeneous field list, and closing it is
what makes the theorems total.

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
Differential testing over shared and mutated inputs is the broader, cheaper companion,
clean so far.

The pattern is established elsewhere. AWS's Cedar ships a Lean model with differential
testing against its Rust
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
- **Nyx Foundation `formal-leanSpec`**: an independent, actively developed Lean 4
  formalization of the lean-Ethereum spec. Parallel and complementary, it works the future
  line while etheorem follows what runs on mainnet now and the forks next in line.

As far as we know, this is the only Lean 4 formalization of the *current*
production consensus spec, conformance-tested at the fork frontier, now growing
machine-checked proofs in the spots prior efforts left open
(serialization total over the codec, merkle-branch completeness, a real shuffle,
merkleization over a verified hash).

## Roadmap

Cohort runs June to mid-November, kickoff at week 1 (mid-June); week 5 is mid-July,
where this proposal was written. Already underway from onboarding: six PRs merged between
us and five more open (the Heze fork layer, the fork-choice sweep, the roadmap, and the
two mixed-field container PRs), the first consensus proof in progress, and the client
cross-check and extraction proofs prototyped. The two lanes run in parallel, month by month:

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

- Proof effort is open-ended, especially the shuffle and deep-consensus targets.
  Mitigation: the targets escalate from templated (following prior Dafny work) to
  greenfield, so a partial run still leaves standing results; the Merkle-branch proof
  is already demonstrating the merkleization proof shape end to end. If a target turns out
  intractable in the window, dropping
  it down a tier is a deliberate, documented call (the tiered success criteria are built
  for that).
- The fork frontier moves. New spec alphas re-shape containers under the work.
  Mitigation: keeping the conformance suite passing absorbs this by design. Pin every claim to a
  named spec and vector release, and keep the skip accounting explicit.
- Serialization proofs can only reach as far as the decoder does. The central
  theorems quantify over a predicate, and a type outside that predicate is unproved even
  when the codec handles it. Closing the mixed-field container arm is this problem; whatever the decoder does not yet implement stays out until it does. Stated
  openly to keep the proof claims honest.
- Landing depends on upstream review. etheorem is not our repo, and the tiers below are
  written in terms of work merged, not work written, so review bandwidth is on the
  critical path. Mitigation: the path is already open, six PRs merged between us and five
  in flight, and our mentor is the project's author. We keep PRs small enough to review in
  one sitting and sequence them so a slow one does not block the next.
- Two fellows on one codebase can collide or drift. Mitigation: the split is by
  layer, not by ticket, so the lanes touch different packages, and both work from the
  one roadmap. Where they do couple it runs one way, the consensus proofs leaning on
  SSZ results, and the proofs themselves will settle what that interface looks like.

## Goal of the project

By November, etheorem is a more thoroughly machine-checked reference for the current
consensus spec than it is today. Conformance stays green across the frontier through
Heze; its SSZ layer carries the three central theorems over every type the codec
implements; and the consensus layer above carries a body of proofs guided by the
roadmap. The commitment is the progress and the documented pipeline, not an exact theorem
count, and the list flexes with the spec and the work.

**Success at minimum:** etheorem keeps passing the official vectors as the frontier advances,
with the
Heze (FOCIL) fork layer landed; the mixed-field container arms close, so round-trip,
injectivity and the size bound hold for the containers the beacon chain actually
serializes; the verification roadmap and the Merkle-branch completeness proof, down to
the pure-Lean SHA-256, land upstream; and a bounded set of the roadmap's early proofs
completes, the structural and arithmetic invariants, overflow safety and slot/epoch
monotonicity and length preservation among them. Each of those proofs is a first for the
current consensus spec at that layer.

**Target:** the proof work climbs the roadmap. The big one is shuffle-is-a-permutation with
its committee cluster, the first proof of a real beacon-chain shuffle. Alongside
it, fork-choice store invariants, further serialization and merkleization proofs, and
FOCIL inclusion-list validity on the in-flight Heze layer (a satisfied inclusion list
forces the required transactions into the block).

**Stretch:** the deep consensus results the roadmap points at, a start on Casper FFG
accountable safety in Lean.

The moonglass cross-check stays a background, experimental line. Prototypes are there, and
what it turns up gets reported either way.

**Impact:** the consensus layer gets machine-checked proofs in Lean 4 where earlier efforts
left off, and etheorem becomes a fork-current executable reference client teams can diff
against. The documented pipeline should make the next contributor's start cheaper than
ours was.

## Collaborators

### Fellows

Two cohort-seven fellows, split by layer:

- **Ivan Anishchuk** ([@IvanAnishchuk](https://github.com/IvanAnishchuk)): the consensus
  layer, the verification roadmap, and conformance.
- **Raj Gill** ([@irajgill](https://github.com/irajgill)): the SSZ layer, the three
  central serialization theorems.

etheorem is a shared effort wider than the two of us. **Mouz**
([@Mouzayan](https://github.com/Mouzayan)) lands Gloas consensus proofs off her own
candidate list.

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
  The Merkle-branch completeness proof is in progress and goes upstream next.
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
