# Hive Spec-Test Hardening for Lean Consensus

Auditing leanSpec fixtures and widening Hive assertions to enforce them.

## Motivation

The primary objective of the Lean Consensus 2026 plan is to transition from research feasibility to production-grade implementation. Its milestones include a stable devnet of at least five distinct client implementations, monthly devnet launches, and a long-running 10,000-validator devnet. Every one of those milestones depends on independent clients computing identical state transitions.

The shared testing infrastructure for post-quantum clients lives in `ethereum/hive`'s lean simulator. I audited both fixture collections across all three families. I then reviewed how the mature Ethereum testing stack — `consensus-specs`-test-formats and their client consumers, using Lighthouse's `ef_tests` as the reference — solves the same problems. The result is that the oracle is weaker than it looks, and that the fix follows an established design.

- devnet-5 pins a top-level `postStateRoot` on all 59 successful state-transition cases. It is a hash commitment to the full post-state, and Hive ignores it. The leanSpec generator documents it as the deliberate second tier of a two-tier design: "Authored expectations cover only some fields, so pin the full-state root too. The root closes the divergence gap across every other field."
- 49 of 59 successful devnet-5 cases enforce exactly one field, `post.slot`. All 22 `test_justification` cases and all 12 `test_finalization` cases assert nothing about justification or finalization.
- No mature fixture consumer grades a client on its own verdict: consensus-specs ships the full post-state, and Lighthouse compares it directly, with a field-by-field diff on mismatch. Meanwhile, lean's pipeline relies on client self-verification. The client checks its own stateroot, and Hive only learns of its verdict.
- The gaps are ecosystem-wide. Ream's native spec-test suite never reads `postStateRoot` or `storeSnapshot`, asserts the same four summary fields Hive does, and carries stale fork-choice check names that fail open (the same bug class as the six dead assertion branches this project has already fixed in Hive)

At 10,000 validators across five clients, an undetected state divergence takes days to debug. The same bug caught by a pinned fixture is a one-line CI failure.

## Project description

The project comprises three pillars.

**Pillar 1: Complete the fixture audit.** Measure what the fixture collection asserts across all three driver families (state-transition, fork choice, and verify signatures) and both devnet profiles.

**Pillar 2: Widen the response schema and enforce it, anchored on `postStateRoot`.** Extend the test-driver response so it can carry what fixtures pin: the client's computed post-state root, the justification and finalization fields the fork-choice response already returns, and a structured rejection code drawn from leanSpec's existing `RejectionReason` enum. The field list will be co-designed with [Mohit Grover](https://github.com/groverInnovate), whose XMSS-lifecycle work consumes the same contract. I ship the reference implementation in ream—the post-state root is about a one-line addition to ream's driver (`TreeHash` is already in scope at the response-construction site)—and the other teams review a working diff.

**Pillar 3: Generator cleanup and format documentation.** Draft that format specification for LeanSpec. LeanSpec has no equivalent of consensus-specs' `tests/formats/` documentation — the defense that would have prevented the key-name drift found in both Hive and ream.

## Specification

**Pillar 1.** Extract the fixture collection from the Docker image, run a field-pinning analysis per family and per scenario directory, and publish an audit report with per-family pinning tables and the pinned-but-uncarried list.

**Pillar 2.**
1. A short schema doc defining the widened driver response, co-authored with Mohit, signed off by mentors and client teams. Three groups of fields: (i) the client's computed post-state root, making verification independent rather than self-graded; (ii) the justification and finalization scalars for localization, which the fork-choice response already carries; (iii) a structured rejection code, so negative cases assert why a block was rejected and not merely that it was.
2. A reference implementation in ream: the driver response extension, plus companion fixes to ream's native suite (the stale check names, and reading `postStateRoot` and `storeSnapshot`).
3. Extend Hive's `StateTransitionPost` (and the fork-choice `DriverSnapshot` where applicable) with the new fields as optional, and extend the assertion blocks in `spec_assets.rs` to enforce them when present and pinned.
4. Harden the harness fail-closed, following Lighthouse's pattern: type the fixture-side `checks` object with `deny_unknown_fields` so a fixture key the harness does not model is a loud error instead of a silent drop, and destructure the checks struct exhaustively so adding a field forces a compile-time decision.
5. Adopt a coverage guard modeled on Lighthouse's accessed-file log: diff the cases shipped in the image against the cases loaded at startup, and fail or warn on the remainder against a checked-in allowlist whose entries name where each excluded family is covered. This replaces the weaker "warn on unconfigured prefixes" fix and turns the audit's division-of-labor question into a reviewed artifact.

**Pillar 3.**
1. Draft a `tests/formats/`-style specification for the lean fixture families in leanSpec, starting with `state_transition`: artifact list, field semantics, the validity convention, and the two-tier pinning contract. Consumers should be written against a format document, not against a sample fixture.

## Roadmap

The execution phase runs from July to the end of October. The milestones are sequenced Python-first: early work focuses on analysis and generator changes in leanSpec, and the Hive enforcement work lands later.

| Milestone | Timeline | Deliverables |
|---|---|---|
| Evidence complete | Mid-July to end of August | (i) Pinning census for all three families on both devnets, published. (ii) Prior-art review of consensus-specs, Lighthouse `ef_tests`, leanSpec, and ream. (iii) First Hive-only enforcement PR: fork-choice assertions read the keys the fixtures actually use. |
| Contract schema and reference implementation | September | (i) Schema document co-authored with Mohit, citing the prior art, signed off by mentors and client teams. (ii) Reference implementation in ream: driver `postStateRoot` plus native-suite companion fixes. (iii) Generator cleanup PRs opened in leanSpec. |
| Enforcement in Hive | September to mid-October | (i) `StateTransitionPost` and assertion extensions merged with the optional-field rollout. (ii) Every successful case graded on `postStateRoot`. (iii) Scenario-matched enforcement of justification and finalization fields. (iv) Structured rejection codes asserted. (v) Fail-closed hardening and the coverage guard. |
| Wrap-up | Mid-October to Devcon | (i) leanSpec format documentation, `state_transition` family first. (ii) Final report and presentation. |

## Possible challenges

- **Rust depth.** I am ramping up Rust, and more of the work sits in Hive than the original plan assumed. Mitigation, now partly demonstrated: the first enforcement PR extended an existing, well-shaped code path and shipped. The remaining harness work follows patterns in Lighthouse's `ef_tests`.
- **Multi-client coordination.** Pillar 3 needs 7 client teams to adopt the schema. Mitigations: backward-compatible optional fields, mentor sign-off before circulating, a working reference implementation in ream instead of a paper request, and value delivered even at partial adoption.
- **Spec churn.** Lean is evolving, and monthly devnet launches regenerate fixtures. This strengthens pillar 2 because a fix at the generator level re-emits with every regeneration, and it supports the coverage guard, which catches layout changes at every devnet bump.
- **Overlap management.** Mohit and I coordinate one combined contract request to client teams.

## Goal of the project

Success, measurable at the end of the fellowship:

1. **Evidence:** pinning coverage measured and published for all three fixture families on both devnets, the intentionality question settled, and the design validated against the mature stack.
2. **Post-state root:** every successful state-transition case is graded on `postStateRoot`, an independent comparison of the client's computed root against the fixture's, replacing today's self-verification. Target: 59 of 59 on the devnet-5 equivalents, from 0 today, with the reference client implementation shipped in ream.
3. **Scenario-matched enforcement:** justification scenarios assert justification fields and finalization scenarios assert finalization fields, so the 49 of 59 cases that today enforce only `post.slot` are graded on their own subject.
4. **Contract and enforcement:** the widened driver response is specified, adopted by clients, and enforced by Hive. No field-a-fixture pins are silently dropped
5. **Format documentation:** the lean fixture families have a written format specification in leanSpec, so the next consumer is written against a document rather than a sample fixture.

## Collaborators

### Fellows

Richard Gregory

[Mohit Grover](https://github.com/groverInnovate) is working on XMSS key-lifecycle edge cases and signature-aggregation interop. Our proposals interlock. We plan to co-design the driver-contract schema.

### Mentors

[Derek Sorken](https://github.com/Dsorken) and [Kolby ML](https://github.com/KolbyML)

## Resources

- Project repo: https://github.com/ethereum/hive (lean simulator: `simulators/lean/`, SDK: `hivesim-rs/`)
- Spec and fixture generator: https://github.com/leanEthereum/leanSpec
- Prior art: https://github.com/ethereum/consensus-specs (`tests/formats/`), https://github.com/sigp/lighthouse (`testing/ef_tests`), https://github.com/ReamLabs/ream (`testing/lean-spec-tests`, `crates/rpc/lean`)
- devnet-4 fixture artifacts: https://github.com/ReamLabs/lean-spec-tests
- Lean Consensus 2026 plan: https://hackmd.io/@tcoratger/ryS1ElrWbx
- Lean Hive dashboard: https://hive.leanroadmap.org/
