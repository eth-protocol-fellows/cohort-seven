# Historical Light-Client (LC) Data Backfill in Prysm

Implement and evaluate an experimental, one-period historical light-client (LC) data backfill in Prysm, following the protocol sketch in the description of [Nimbus PR #8445](https://github.com/status-im/nimbus-eth2/pull/8445) by Etan Kissling. (That PR itself only adds a debug configuration flag, default off; the protocol design lives in its description.)

The project reconstructs the evidence needed to identify the canonical best `LightClientUpdate` for a completed sync-committee period, verifies that update without trusting the supplier, imports it atomically, and serves it through Prysm's existing light-client REST API.

## Motivation

Ethereum consensus clients commonly start from a recent checkpoint instead of replaying the chain from genesis. This makes startup practical, but it also means a checkpoint-synced node does not possess all historical post-states. Those states are normally required to derive old light-client updates, including their sync-committee and finality proofs.

Prysm already supports live light-client data generation and can serve retained updates through the standard APIs. Its ordinary block backfill, however, restores historical blocks, not the historical post-states used by the LC data-collection path. If a completed period's update was never generated, or has since been pruned, the updates-by-range interface cannot reconstruct it on its own.

Historical light-client data availability is one part of a larger decentralized checkpoint-sync pipeline. In that pipeline a **checkpoint consumer** (a freshly starting node) begins from a trusted block root, processes LC updates until it authenticates a recent finalized `state_root`, downloads the corresponding `BeaconState` from an untrusted **checkpoint provider**, and verifies that state against the authenticated root before initializing the client. The pipeline only works at scale if providers can actually serve the historical updates the consumer walks through — which is exactly what a checkpoint-synced Prysm node cannot do today.

This project therefore addresses the provider-side gap: recovering missing standard updates for completed periods.

### Roles

Four roles appear in this document and are kept distinct:

| Role                    | Meaning                                                                                                      |
|-------------------------|--------------------------------------------------------------------------------------------------------------|
| **Checkpoint consumer** | A freshly starting node walking LC updates to authenticate a recent state root. Out of scope here.           |
| **Checkpoint provider** | A node serving historical LC updates to consumers. The role whose gap this project closes.                   |
| **Backfill supplier**   | The untrusted party that supplies historical artifacts during backfill.                                      |
| **Backfill receiver**   | The node recovering its own history; runs a pure verifier over supplied artifacts before importing anything. |

A checkpoint provider becomes one by first acting as a backfill receiver.

### The security problem

The central question is not only whether a supplied update is _valid_ — it is whether it is the _best available_. An untrusted backfill supplier could return a valid but inferior update while withholding a stronger canonical candidate it holds.

The mechanism that makes completeness checkable rather than merely assumed is **sequencing**. The receiver first obtains the period's full `block_roots` list, committed and proven back to a trusted anchor, before it ever requests the per-slot participation data used to rank candidates. Because the reference list of "everything that should exist" is pinned up front, independently of the party supplying the per-slot data, a later gap in that data is structurally detectable rather than something the receiver must take on faith.

A second observation keeps the artifact set small: per-slot `sync_committee_bits` are Merkle-proven against block roots that are themselves proven canonical. Any such block was accepted by the chain's own state transition, which already verified its sync-aggregate signature. The receiver therefore does not need aggregate signatures for candidate ranking — only for the single winning update it imports.

## Relationship to existing work

| Reference                                                                      | Relationship                                                                                                                                                                                                                                                                                                                         |
|--------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| [Nimbus PR #8445](https://github.com/status-im/nimbus-eth2/pull/8445)          | The design this project implements. Provides the artifact shapes and the nine-step protocol sketch.                                                                                                                                                                                                                                  |
| [consensus-specs #3614](https://github.com/ethereum/consensus-specs/pull/3614) | Tracking the best `SyncAggregate` in `BeaconState` to enable backfill; the spec-side enshrinement counterpart Etan links from #8445.                                                                                                                                                                                                 |
| [consensus-specs #3553](https://github.com/ethereum/consensus-specs/pull/3553) | Canonical-only LC data collection. Establishes the canonicality requirement this verifier enforces.                                                                                                                                                                                                                                  |
| [EIP-7658](https://eips.ethereum.org/EIPS/eip-7658)                            | The EIP formulation of the same enshrinement direction as #3614 (tracking best sync data in `BeaconState`); currently Stagnant. This project follows the #8445 artifact model instead, which pins completeness to a committed `block_roots` list and requires no fork. Phase 0 includes writing an explicit mapping between the two. |

### The enshrinement alternative

Etan notes in #8445 that steps 4–6 (per-slot block data, exhaustiveness check, independent ranking) could be skipped entirely by enshrining `is_better_update` into the state transition function, which would also shrink `block_roots` needs to epoch-transition blocks only. That path was raised directly with Etan and set aside: enshrinement is a hard fork and a long process, whereas the artifact approach is deployable today at an estimated cost of roughly 1 GB/year (see the Resource bounds section).

This project therefore implements the artifact path, and Phase 0 records which components would be obsoleted if enshrinement later lands — the artifact types and producer would be, the verifier's canonicality and import logic largely would not.

## Scope

How this project maps onto the nine protocol steps sketched in #8445:

| #8445 step                                      | This project                                                                          |
|-------------------------------------------------|---------------------------------------------------------------------------------------|
| 1 — Sync a `LightClientStore`                   | Replaced by a pinned, pre-authenticated anchor fixture (see the Trust model section). |
| 2 — Fetch `LightClientPeriodData`               | **In scope** (SSZ file / in-process adapter instead of P2P).                          |
| 3 — Verify `anchor_update` best-ness            | **In scope**, with explicitly reduced semantics (see the Trust model section).        |
| 4 — Fetch `LightClientBlockData`                | **In scope** (same transport substitution).                                           |
| 5 — Verify exhaustiveness                       | **In scope**; core of the verifier.                                                   |
| 6 — Independent `is_better_update` ranking      | **In scope**; reuses Prysm's existing implementation.                                 |
| 7 — Obtain the standard update, match, seal     | **In scope**; matched update imported atomically and served via REST.                 |
| 8–9 — `LightClientBootstrapData` reconstruction | Deferred to follow-up work.                                                           |

### In scope

A vertical slice for **one completed Electra sync-committee period under the minimal preset**:

- Bounded, versioned, transport-neutral representations of `LightClientPeriodData` and `LightClientBlockData`.
- A deterministic producer for those artifacts, plus positive and adversarial fixtures.
- A pure verifier: canonical coverage, exhaustiveness, proof validation, independent `is_better_update` ranking, winner matching.
- Atomic verify-before-write import into Prysm's database, and round-trip serving through the existing updates-by-range REST endpoint.
- The path is disabled by default, behind a debug flag, matching the Nimbus default.

A sync-committee period is the span of slots served by a single sync committee (256 epochs on mainnet, about 27 hours; 8 epochs under the minimal preset). It is the natural unit because the LC protocol retains one best `LightClientUpdate` per period. Extending to a range is mostly repetition of the same mechanism, though it additionally requires chaining anchors across periods and walking the `historical_summaries` index — noted as follow-up work, not claimed as trivial.

### Out of scope

- `LightClientBootstrapData` **(steps 8–9 of #8445).** Reconstructing every `LightClientBootstrap` within the period is required for the full pipeline but is a separable artifact type with its own proof shape. Deferred to follow-up work.
- **P2P transport.** #8445 obtains the update via `light_client_updates_by_range/1` req/resp. This project uses deterministic SSZ files or an in-process adapter, and serves through REST. No libp2p protocol is defined or implemented.
- The checkpoint consumer, `BeaconState` transfer, and the full checkpoint-init flow.
- Multi-period and mainnet-preset operation.
- Producing artifacts from a node that no longer holds the relevant historical states.

### Open questions to close in Phase 0

These require confirmation with Etan before implementation begins, and are the main source of expected drift:

1. Exact semantics of `anchor_update` best-ness under an experimental (non-forward-synced) receiver — see the Trust model section.
2. Whether the reduced `LightClientOptimisticUpdate` form is fixed or still moving.
3. Artifact encoding and versioning, once mainnet size estimates are measured.
4. Confirmation of the derived `is_better_update` input semantics in the "Deriving `is_better_update` inputs" section — notably that `has_finality` holds for all non-genesis reconstructed candidates, and the attested-header derivation when a candidate's attested block lies in period P−1.

## Trust model

The receiver starts from an exact later `LightClientHeader` that has already been authenticated as finalized. In production this anchor comes from the node's own forward LC sync. In this project's experimental setting it is supplied as a pinned, pre-authenticated fixture, and its provenance is an explicit trust assumption.

The anchor header must be in period P+1 or later, sealing period P. The reason is structural: the `HistoricalSummary` covering P is only appended during the epoch transition after P's last slot, so no state inside P commits to P's complete `block_roots`.

**A note on `anchor_update` best-ness.** Step 3 of #8445 requires verifying that the supplied `anchor_update` is the best known update for P+x. That check presupposes step 3's stated assumption — that the client already has full knowledge of periods > P from its own forward sync or local import. A receiver operating from a pinned fixture does not have that knowledge, and "best known" is in general not a provable property (one cannot prove a counterparty holds nothing better).

This project therefore states the check precisely as: _the anchor is the best update among the candidates the receiver itself has synced for periods > P._ In the experimental setting that candidate set is exactly the pinned fixture, so the check degenerates to a trust assumption, and is implemented as an explicitly stubbed predicate with the production form documented alongside it. This is a deliberate, labelled reduction, not an oversight.

### Preset alignment

`SLOTS_PER_HISTORICAL_ROOT` equals `EPOCHS_PER_SYNC_COMMITTEE_PERIOD × SLOTS_PER_EPOCH` on both presets (mainnet 8192 = 256 × 32; minimal 64 = 8 × 8). Exactly one `HistoricalSummary` therefore covers exactly one sync-committee period. The entire anchoring chain — `block_roots` → `HistoricalSummary` → `anchor_update.header` — depends on this alignment, and the verifier asserts it at startup rather than assuming it.

## Specification

### Constants

| Name                                 | Mainnet                                                                  | Minimal |
|--------------------------------------|--------------------------------------------------------------------------|---------|
| `SLOTS_PER_HISTORICAL_ROOT`          | `8192`                                                                   | `64`    |
| `EPOCHS_PER_SYNC_COMMITTEE_PERIOD`   | `256`                                                                    | `8`     |
| `SLOTS_PER_EPOCH`                    | `32`                                                                     | `8`     |
| `SYNC_COMMITTEE_SIZE`                | `512`                                                                    | `32`    |
| `CURRENT_SYNC_COMMITTEE_PROOF_DEPTH` | `floorlog2(CURRENT_SYNC_COMMITTEE_GINDEX_ELECTRA)`                       | same    |
| `FINALIZED_ROOT_PROOF_DEPTH`         | `floorlog2(FINALIZED_ROOT_GINDEX_ELECTRA)`                               | same    |
| `STATE_ROOTS_PROOF_DEPTH`            | depth of `state_roots[SLOTS_PER_HISTORICAL_ROOT-1]` within `BeaconState` | same    |
| `HISTORICAL_SUMMARY_PROOF_DEPTH`     | depth of the period's `HistoricalSummary` within the anchor's state      | same    |
| `SYNC_COMMITTEE_BITS_PROOF_DEPTH`    | depth of `sync_aggregate.sync_committee_bits` within `BeaconBlock`       | same    |

The last four are computed from the Electra SSZ layout in Phase 0 and pinned as generated constants; they are listed here as named quantities rather than magic numbers.

### `FinalityTransitionProof`

```python
class FinalityTransitionProof(Container):
    header: BeaconBlockHeader
    finalized_checkpoint: Checkpoint
    finality_branch: Vector[Bytes32, FINALIZED_ROOT_PROOF_DEPTH]
```

Two instances bracket the point at which the finalized checkpoint's period crosses into P:

- `finality_transition_pre` — the last block whose finalized checkpoint predates the start of P. If no finalization into P occurred during P, this is `block_roots[SLOTS_PER_HISTORICAL_ROOT-1]`. Default value if P starts at genesis.
- `finality_transition_post` — the first block whose finalized checkpoint lies within P; its parent must be `finality_transition_pre` (or zero if the parent is genesis). Default value if `finality_transition_pre` is `block_roots[SLOTS_PER_HISTORICAL_ROOT-1]`.

This pair is sufficient to determine, for every candidate in P, whether its finalized header lies in P — the only finality property `is_better_update` needs — without a per-block finality proof.

### `LightClientPeriodData`

```python
class LightClientPeriodData(Container):
    first_header: BeaconBlockHeader
    block_roots: Vector[Root, SLOTS_PER_HISTORICAL_ROOT]
    finality_transition_pre: FinalityTransitionProof
    finality_transition_post: FinalityTransitionProof
    current_sync_committee: SyncCommittee
    current_sync_committee_branch: Vector[Bytes32, CURRENT_SYNC_COMMITTEE_PROOF_DEPTH]
    latest_state_root: Root
    latest_state_root_branch: Vector[Bytes32, STATE_ROOTS_PROOF_DEPTH]
    historical_summary_branch: Vector[Bytes32, HISTORICAL_SUMMARY_PROOF_DEPTH]
    anchor_update: LightClientOptimisticUpdate
```

- `first_header` — the header behind `block_roots[0]`. Enables exhaustiveness verification at the period's leading boundary by distinguishing a block that genuinely belongs to a prior period from withheld data.
- `block_roots` — the period's complete root list; the ground truth for exhaustiveness, proven against the recovered `HistoricalSummary`.
- `finality_transition_pre` / `finality_transition_post` — see above.
- `current_sync_committee` — the committee serving P; required to verify the winning update's signature. Its branch is proven relative to `state_roots[SLOTS_PER_HISTORICAL_ROOT-1]`, the last state of P, whose `current_sync_committee` is still P's.
- `latest_state_root` / `latest_state_root_branch` — the state root after P's final slot, proven relative to `htr(state_roots)`; together with `block_roots` this recovers the period's `HistoricalSummary`.
- `historical_summary_branch` — proves the recovered `HistoricalSummary` relative to `anchor_update.header`'s state root. This is the link that makes every other proof in the artifact terminate at the trusted anchor.
- `anchor_update` — a reduced `LightClientOptimisticUpdate` form of the best known update for period P+x (x ≥ 1, minimized). The anchor for the entire proof chain; its best-ness predicate is defined in the Trust model section.

### `LightClientBlockData`

```python
class LightClientBlockData(Container):
    slot: Slot
    sync_committee_bits: Bitvector[SYNC_COMMITTEE_SIZE]
    sync_committee_bits_branch: Vector[Bytes32, SYNC_COMMITTEE_BITS_PROOF_DEPTH]
```

- `slot` — the signature slot this entry describes; keyed against `block_roots[slot % SLOTS_PER_HISTORICAL_ROOT]`.
- `sync_committee_bits` — the participation bitvector from the block body's sync aggregate.
- `sync_committee_bits_branch` — Merkle proof of the bitvector relative to the block root at that slot.

### Skipped slots

When slot `s` is skipped, `block_roots[s % SLOTS_PER_HISTORICAL_ROOT]` repeats the previous entry. This cuts both ways:

- A claimed skip that does not correspond to a repetition is rejected — a supplier cannot fabricate skips.
- A supplier cannot suppress a real block, because a distinct root in the list with no matching `LightClientBlockData` is a detectable gap.
- The first entry, `block_roots[0]`, has no in-vector predecessor to compare against; the boundary is resolved with `first_header`, whose block may legitimately lie in a period before P and is then expected to carry no `LightClientBlockData` entry.

Validation therefore enforces: every distinct root has exactly one matching `LightClientBlockData` entry; repeated roots have none.

### Deriving `is_better_update` inputs

For each candidate at signature slot `s`:

| Input                         | Derivation                                                                                                                                                                             |
|-------------------------------|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `max_active_participants`     | `SYNC_COMMITTEE_SIZE` from the preset.                                                                                                                                                 |
| `num_active_participants`     | Population count of `LightClientBlockData[s].sync_committee_bits`.                                                                                                                     |
| `signature_slot`              | `s` — the slot whose block carries the sync aggregate.                                                                                                                                 |
| `attested_header.beacon.slot` | Scan `block_roots` backwards from `s−1` for the first entry distinct from its predecessor; that block is the most recent ancestor the aggregate attests to (not necessarily at `s−1`). |
| `has_relevant_sync_committee` | `period(attested_slot) == period(s)`, computed from slots; `first_header` resolves the period boundary case.                                                                           |
| `has_finality`                | True for all non-genesis candidates (to be confirmed in Phase 0; see open question 5).                                                                                                 |
| `has_sync_committee_finality` | `period(finalized) == period(attested)`, determined by the candidate's position relative to the `finality_transition_pre`/`finality_transition_post` boundary.                         |
| Final tiebreakers             | Larger `num_active_participants`, then smaller `attested_slot`, then smaller `signature_slot`.                                                                                         |

No aggregate signature is needed for ranking: the blocks carrying the bits are proven canonical, and the chain's own state transition already verified their aggregates when it accepted them.

## Verifier requirements

The verifier is a pure function — no I/O, no chain access, no direct database mutation. It must:

1. Reject malformed, unsupported, duplicate, reordered, or oversized input before any cryptographic work.
2. Bind the period to the exact trusted finalized header, and evaluate the `anchor_update` predicate as defined in the Trust model section.
3. Assert preset alignment and recover the `HistoricalSummary` from `block_roots`, `latest_state_root`, and `latest_state_root_branch`; verify it against the anchor.
4. Reconstruct canonical block and skipped-slot coverage for P from the committed `block_roots` list.
5. Reject any input with missing eligible candidates or unproven coverage gaps.
6. Verify the required ancestry, committee, finality, and Merkle relationships.
7. Run Prysm's existing `IsBetterUpdate` over the complete reconstructed candidate set.
8. Validate the proposed standard `LightClientUpdate` in that reconstructed context — including its sync-aggregate signature against `current_sync_committee` — and require it to match the independently computed winner.

Only a fully verified update is written, atomically. Importing an equivalent existing record is a no-op; a conflicting record follows a non-overwrite policy and surfaces an error.

## Resource bounds

Estimated figures, to be measured in Phase 0 and confirmed in Phase 4:

| Quantity         | Mainnet                                | Minimal           |
|------------------|----------------------------------------|-------------------|
| `block_roots`    | 8192 × 32 B = 256 KiB                  | 64 × 32 B = 2 KiB |
| Per-block data   | ~8 B slot + 64 B bits + branch ≈ 330 B | ≈ 270 B           |
| Per-period total | ≈ 2.6 MB (~8000 blocks)                | ≈ 19 KB           |
| Periods per year | ≈ 321                                  | —                 |
| **Per year**     | **≈ 0.9 GB**                           | —                 |

This is consistent with the roughly 1 GB/year figure discussed with Etan, and is the cost the enshrinement alternative would avoid.

## Roadmap

Twelve weeks (due to late of the submission of this proposal), phases non-overlapping, with an early Prysm integration slice to de-risk the client-side work:

| Phase                                     | Weeks | Deliverables                                                                                                                                                                                                                                                           |
|-------------------------------------------|-------|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| **0 — Protocol contract**                 | 1–2   | Close the open questions with Etan: anchor model and predicate semantics, period sealing, exhaustiveness mechanism, artifact encoding/versioning, generated proof-depth constants, resource-bound validation, EIP-7658/#3614 mapping, documented security assumptions. |
| **1 — Artifacts & producer**              | 3–4   | Electra/minimal SSZ containers; deterministic producer; positive fixtures.                                                                                                                                                                                             |
| **2 — Thin integration slice**            | 5     | Import path and REST round-trip with a hand-constructed, pre-trusted update; de-risks Prysm integration before the verifier exists.                                                                                                                                    |
| **3 — Pure verifier**                     | 6–8   | Canonical coverage, exhaustiveness, proof verification, input derivation, independent ranking, winner matching.                                                                                                                                                        |
| **4 — Adversarial fixtures & evaluation** | 9–10  | Adversarial suite (omitted candidates, forged skips, wrong anchors, oversized and reordered inputs); benchmarks; Nimbus mapping; reproducible demo.                                                                                                                    |
| **5 — Wire-up & hardening**               | 11    | Real verifier behind the import path; atomic verify-before-write; non-overwrite behaviour; flag defaults.                                                                                                                                                              |
| **Buffer**                                | 12    | Absorb protocol and client review feedback without scope expansion.                                                                                                                                                                                                    |

## Evaluation targets

Measured and reported in Phase 4:

- Wall-clock verification time for one minimal-preset period, with a projected mainnet estimate.
- Peak memory during verification.
- Measured artifact sizes versus the Resource bounds estimates, with variances explained.
- Every adversarial fixture rejected with zero database mutations.

## Possible challenges

1. **The design is not a ratified specification.** Details may move during review; Phase 0 exists to bound that drift, and the open-questions list makes the dependency explicit rather than open-ended.
2. **Proving absence is harder than proving presence.** Showing that a supplier did not omit a stronger canonical candidate is the hard part; the pre-committed `block_roots` list is what converts it from a trust question into a structural one.
3. **Reduced anchor best-ness in the experimental setting.** The stubbed predicate must either be accepted as a labelled trust assumption or extended with a later-period LC store, which would expand scope.
4. **Producer state availability.** Many nodes no longer hold the historical post-states needed to produce artifacts, which limits who can seed the network initially.
5. **Minimal-to-mainnet scaling.** Measurements may come in above projections, motivating more compact proof encodings or sharding artifact serving across peers.
6. **Prysm reviewer availability is unconfirmed.** The largest single risk to Phase 5. Mitigation: raise the integration approach with the Prysm team during Phase 0; if no reviewer is available, Phases 1–4 still stand alone as a specified, tested, fixture-backed verifier that any client can adopt, and integration ships as a draft PR rather than a merged one.

## Success criteria

- Prysm deterministically produces valid artifacts and the matching standard update for one completed Electra/minimal sync-committee period.
- The verifier reconstructs the complete canonical candidate set and selects the same winner as Prysm's live `IsBetterUpdate` path.
- Every adversarial fixture — invalid, incomplete, non-canonical, mismatched, oversized — is rejected without database mutation.
- A verified missing update is imported atomically; equivalent and conflicting existing records follow the defined non-overwrite behaviour.
- Prysm's existing updates-by-range REST endpoint returns the imported update byte-identically.
- Artifact sizes and verification costs are measured and published, with mainnet projections stated.

## Collaborators

### Fellows

- Jeff Chung ([jeffoodchain](https://github.com/jeffoodchain))
- Kapil (TBD)

### Mentors

- Etan Kissling ([etan-status](https://github.com/etan-status)), Nimbus.
* Bastin(TBD), Prysm.

## Resources

- [EPF Cohort 7 project idea: decentralized consensus-layer checkpoint sync](https://github.com/eth-protocol-fellows/cohort-seven)
- [Altair light-client sync protocol](https://github.com/ethereum/consensus-specs/blob/dev/specs/altair/light-client/sync-protocol.md)
- [Altair light-client full-node data collection](https://github.com/ethereum/consensus-specs/blob/dev/specs/altair/light-client/full-node.md)
- [Altair light-client P2P interface](https://github.com/ethereum/consensus-specs/blob/dev/specs/altair/light-client/p2p-interface.md)
- [Nimbus PR #8445 — historical LC data backfill design](https://github.com/status-im/nimbus-eth2/pull/8445)
- [consensus-specs #3614 — enable LC data backfill by tracking best `SyncAggregate`](https://github.com/ethereum/consensus-specs/pull/3614)
- [consensus-specs #3553 — canonical-only LC data collection](https://github.com/ethereum/consensus-specs/pull/3553)
- [EIP-7658 — Light client data backfill](https://eips.ethereum.org/EIPS/eip-7658)
- [Prysm repository](https://github.com/OffchainLabs/prysm)
- [EPF Cohort 5 project: Light Client Server Support in Prysm](https://github.com/eth-protocol-fellows/cohort-five)
