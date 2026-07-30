# Hive Spec-Test Hardening for Lean Consensus

Auditing and tightening lean-specs fixtures and widening Hive assertions to enforce them.

## Motivation

The Lean Consensus 2026 primary objective is to transition from research feasibility to production-grade implementation, with a stable devnet of at least 5 distinct client implementations, monthly devnet launches, and a long-running 10,000-validator devnet as milestones. Every one of those milestones depends on independent clients' ability to compute identical state transitions.

The shared testing infrastructure for post-quantum clients lives in `ethereum/hive`’s lean simulator. In its current state, that oracle is weaker than it looks. I audited the devnet-4 state-transition fixture collection:

- Of 45 successful cases, all pin `post.slot`, but only 5 pin `latestBlockHeaderStateRoot`, the hash commitment to the full post-state. For the other 40, a client can compute a wrong post-state and still pass.
- Fixtures pin fields the test-driver response contract cannot carry, i.e. `justifiedSlots` (25/45) and `latestJustifiedSlot` (36/45). The harness silently drops those checks: these assertions are never enforced.
- The driver response (`StateTransitionPost`) exposes only 4 summary fields, so even a fully-pinned fixture could only ever be checked at that structure.

The major risk the gap poses is, at 10,000 validators and 5 or more devnets, a state divergence bug that goes undetected becomes a devent issue. But the same bug caught by a comprehensively pinned spec test is a single-line CI failure pointing to a specific client and fixture.

## Project description

The project comprises three verticals.

**Pillar 1:** Complete the fixture audit across all three families (state-transition, fork choice and verify signatures) and both devnet profiles. This includes an intentionality analysis: confirm if the gaps are delibrately scoped for the scenario or the fixtures evolved that way to into gaps that need to be plugged. The devnet 4 state-transition audit is already done and produced the numbers above.

**Pillar 2:** Tighten the leanSpec fixture generator in the pytest suite. Currently, the generator computes the entire post-state for every case but only records a fraction of it.

**Pillar 3:** Widen the response schema and enforce it. Extend the test-driver response so it can carry what fixtures pin. The field list will be co-designed with [Mohit Grover](https://github.com/groverInnovate), a fellow working on XMSS-lifecycle. His work consumes the same contract.


## Specification

**Pillar 1.** Extract the fixture collection from the Docker image, run a field-pinning analysis per family and per scenario directory, and produce a public audit report consisting of per-family pinning tables. Identify pinned but uncarried list. Confirm from authors if the intention behind is delibrate. The devnet-4 and devnet-5 collections differ in layout and source (pinned ReamLabs/lean-spec-tests commit vs leanSpec release artifacts) and are audited separately.

**Pillar 2.** Locate where the generator constructs each case's `post` and change the recording policy to always include the post-state root and scenaria relevant fields for successful cases. Regeneration then propagates the tightened pins to the published fixture artifacts consumed by hive.

**Pillar 3**:
1. A short schema doc defining the widened driver response, co-authored with Mohit, signed off by mentors and client teams. Structure: full post-state root and diagnostic scalars (localization), all derived from the leanSpec `State` definition.

2. Extend Hive's `StateTransitionPost` (and the fork-choice `DriverSnapshot` where applicable) with the new fields as optional, and extend the assertion blocks in `spec_assets.rs` to enforce them when present and pinned.

3. Client-side endpoint changes are implemented as per the schema doc as PRs split between the two of us. We'll implement Rust clients first during the fellowship and expand to clients written in other programming languages.

Open questions I am investigating:

- Where each client implements the `/test_driver/*` endpoints.
- The intentionality verdict, which decides whether pillar 2 is framed as a policy proposal ("always pin the root") or a gap fix.

## Roadmap

| Milestone | Timeline | Deliverables |
|---|---|---|
| Audit complete | Mid-July to mid-August | (i) Pinning census for all three families on both devnets. (ii) Intentionality feedback received from the fixture authors. (iii) Public audit report. (iv) First leanSpec generator PRs merged (always pin the post-state root), with regenerated fixtures flowing into Hive. |
| Contract schema | August to early September | (i) Schema document co-authored with Mohit and signed off by mentors and client teams. (ii) Client adoption begins. |
| Enforcement in Hive | September to mid-October | (i) `StateTransitionPost` and assertion extensions merged with the optional-field rollout. (ii) Pinned-but-dropped checks are enforced as clients adopt the schema. |
| Wrap-up | Mid-October to Devcon | Final report and presentation. |

Throughout the fellowship, I share (bi)weekly updates on the Ream dev call and the EPF standup, and contribute small maintenance PRs to the lean simulator as the devnets evolve.

## Possible challenges

- **Rust depth.** I am ramping up Rust. The mitigation is structural. The first two milestones are Python and design work, and the hive enforcement extend existing, well-shaped code paths.
- **Multi-client coordination.** Pillar 3 requires seven client teams to adopt a schema change. Mitigations: backwards-compatible optional struct fields, mentor sign-off before circulating the schema, and value at partial adoption, as checks tighten per client as each one ships the new schema.
- **Spec churn.** Lean is evolving, and that means monthly devnet launches regenerate fixtures. This strengthens pillar 2, because a fix at the generator level re-emits with every regeneration. It also means audit numbers must be re-derived for each pinned collection version.
- **Intentionality verdict.** If sparse pinning turns out to be deliberate scenario-scoping, pillar 2 reframes from a gap fix to a policy proposal. The state-root argument survives either way, but the framing of the solutions and review path change.
- **Overlap management.** Mohit and I need to coodinate the change request to client teams

## Goal of the project

Success, measurable at the end of the fellowship:

1. **Audit:** pinning coverage is measured and published for all three fixture families on both devnets, and the intentionality question settled.
2. **Generator:** every successful state-transition fixture pins the post-state root (from 5 of 45 to 45 of 45 on the devnet-4 equivalents), plus the agreed diagnostic fields.
3. **Contract and enforcement:** the widened driver response is specified, adopted by clients, and enforced by hive. No field a fixture pins is silently dropped.

## Collaborators

### Fellows

Richard Gregory

[Mohit Grover](https://github.com/groverInnovate) is working on XMSS key-lifecycle edge cases and signature-aggregation interop. Our proposals interlock. We plan to co-design the driver-contract schema.

### Mentors

[Derek Sorken](https://github.com/Dsorken) and [Kolby ML](https://github.com/KolbyML)

## Resources

- Project repo: https://github.com/ethereum/hive (lean simulator: `simulators/lean/`, SDK: `hivesim-rs/`)
- Spec and fixture generator: https://github.com/leanEthereum/leanSpec
- devnet-4 fixture artifacts: https://github.com/ReamLabs/lean-spec-tests
- Lean Consensus 2026 plan: https://hackmd.io/@tcoratger/ryS1ElrWbx
- Lean Hive dashboard: https://hive.leanroadmap.org/
