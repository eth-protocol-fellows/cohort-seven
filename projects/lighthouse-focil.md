# Implementation of FOCIL in Lighthouse

## Motivation

- **FOCIL (EIP-7805)** strengthens Ethereum's censorship resistance: a 16-validator committee broadcasts inclusion lists (ILs) each slot, and fork choice refuses to extend payloads that omit the timely IL transactions.
- FOCIL is scheduled for the **Heze** fork, layered on top of the changes introduced by [ePBS](https://eips.ethereum.org/EIPS/eip-7732) in Gloas.
- Lighthouse has no FOCIL support yet, though prior work exists in the [`focil` branch](https://github.com/sigp/lighthouse/tree/focil), a FOCIL implementation (in prototype phase) built on top of the Gloas changes. We can leverage this to build a production-ready implementation.

## Project description

Full FOCIL support will be implemented in Lighthouse in the following areas:

1. **IL production**: IL-committee validators fetching IL transactions from the execution layer (EL), signing, and broadcasting `SignedInclusionList`s before the slot's timeliness cutoff.
2. **Networking**: a new global `inclusion_list` gossip topic with spec-conformant validation (committee membership, at most two ILs per validator per slot, equivocation handling, signature checks), plus the `InclusionListByCommitteeIndices` req/resp protocol.
3. **Inclusion list store**: an in-memory store with equivocation detection and per-validator tracking - the core component shared by gossip validation, block production, fork-choice enforcement, and req/resp serving.
4. **Fork-choice enforcement**: recording whether each revealed payload included the timely ILs, and refusing to extend the ones that didn't.
5. **Block proposal**: proposers self-building IL-satisfying payloads, setting the bid's `inclusion_list_bits` from their own IL view, and skipping external bids whose bits are not inclusive of that view.
6. **Beacon API & validator client**: three new validator-client (VC) endpoints (IL duties, produce, publish), plus the VC duties/signing flow for the new IL committee duty.
7. **Interop (stretch)**: a local devnet run against another FOCIL-ready client (Lodestar), demonstrating IL propagation and censoring-payload rejection.

The full design (flows, store design, each change rationale per subsystem) is detailed in our [annotated design document](https://hackmd.io/@conache/HyDlczNXGl).

We will use the exploratory `focil` branch as a reference rather than a base branch: for each task, we assess what can be reused (or where a better approach exists), bring it up to the current spec, and land the changes as incremental PRs through a proper review cycle. The rest we build from scratch.

From our initial assessment of the branch so far:
- **Areas with reusable parts**: the IL store, gossip verification, req/resp serving, the validator duties and production flow, and block production.
- **Completely missing**: the bid-bits gossip check and IL backfilling.
- **Needs rework or completion**: spec drift (signing domain, `ProgressiveList` SSZ types, the engine API's IL-satisfaction integration, the `payload_attributes` SSE event), fork-choice satisfaction tracking, unfinished IL publish, testing and interop coverage.

We will follow the FOCIL specs across the CL and its EL integration:
- [CL spec (Heze)](https://github.com/ethereum/consensus-specs/tree/master/specs/heze)
- [Beacon API spec (PR #490)](https://github.com/ethereum/beacon-APIs/pull/490)
- [Engine API spec (PR #609)](https://github.com/ethereum/execution-apis/pull/609)

## Specification

### Types & constants

- Add the `InclusionList` and `SignedInclusionList` containers in `consensus/types`:

```rust
pub struct InclusionList<E: EthSpec> {
    pub slot: Slot,
    pub validator_index: u64,
    pub inclusion_list_committee_root: Hash256,
    pub transactions: ProgressiveList<Transaction<E::MaxBytesPerTransaction>>,
}

pub struct SignedInclusionList<E: EthSpec> {
    pub message: InclusionList<E>,
    pub signature: Signature,
}
```

- Extend `ExecutionPayloadBid` with `inclusion_list_bits: BitVector<InclusionListCommitteeSize>` (cascades into the Heze `BeaconState` SSZ layout).
- Add constants:
    - `INCLUSION_LIST_COMMITTEE_SIZE`
    - `DOMAIN_INCLUSION_LIST_COMMITTEE`
    - `INCLUSION_LIST_DUE_BPS`
    - `MAX_BYTES_PER_INCLUSION_LIST`
    - `MAX_REQUEST_INCLUSION_LIST`
    - `MAX_SIGNED_INCLUSION_LIST_SIZE`
- Wire the new containers into `ef_tests` (`ssz_static` spec tests).

### Consensus helpers

- Add `get_inclusion_list_committee(slot)` on `BeaconState`: returns the first `INCLUSION_LIST_COMMITTEE_SIZE` validators of the slot's concatenated beacon committees, wrapping back to the start if there are fewer (so a validator may repeat). Built on the existing `CommitteeCache`.
- Add `is_valid_inclusion_list_signature`: checks an IL's signature, with the domain computed at the IL's own slot epoch.

### Networking

- Add the new `inclusion_list` gossip topic:
  - Verification per the [spec's p2p rules](https://github.com/ethereum/consensus-specs/blob/master/specs/heze/p2p-interface.md), with the store insert as the final validation step.
  - Accepted ILs are forwarded to peers.
  - Accepted ILs are emitted on a new `inclusion_list` SSE topic.
- Add an `[IGNORE]` check on `execution_payload_bid` gossip: ignore any bid whose `inclusion_list_bits` don't mark every IL-committee member the node saw a valid inclusion list from for the previous slot.
- Add support for `InclusionListByCommitteeIndices` req/resp: the client fetches chosen committee members' ILs for a slot from peers, and the server answers those requests.
- Implement client-side backfill: if the node missed some of the previous slot's ILs on gossip (just before proposing a block, or right after syncing near a slot boundary), fetch the gaps, so its IL view is complete for the bid-bits check and block proposal.

### Engine API

Integrate Heze Engine API methods:
- `engine_getInclusionListV1`: fetch an inclusion list's transactions from the EL for IL production.
- `engine_forkchoiceUpdatedV5` and use `PayloadAttributesV5`: pass `inclusionListTransactions` to the EL so a self-built payload includes them for block production.
- `engine_newPayloadV6`: use the returned `inclusionListSatisfied`, which is the EL's verdict on the IL satisfaction of a payload, used for fork-choice enforcement.

### Inclusion list store

- Implement the `InclusionListStore`: an in-memory store of the inclusion lists received for the current slot and the two before it, keyed by `(slot, committee_root)` and pruned each slot. Its methods also operate on a given `(slot, committee)` pair. Core methods to add:
  - `process_inclusion_list`: validates and inserts an IL, detecting equivocation.
  - `get_inclusion_list_transactions`: returns the slot's IL transactions.
  - `get_inclusion_list_bits`: which committee members the node saw a valid IL from.
  - `is_inclusion_list_bits_inclusive`: whether the given bits cover every member the node itself saw an IL from.
  - `get_signed_inclusion_lists`: returns the stored signed ILs for chosen members.
- Expose `BeaconChain` wrappers over the store read methods: each takes a slot, derives that slot's committee and its root from the head state, and calls the store method. This lets callers like block production and the bid-bits check work with just the slot, instead of deriving the committee root themselves.

### Fork choice

- Add a `payload_il_satisfied` flag on `ProtoNode`. 
- Record the flag once per payload at envelope reveal, from `engine_newPayloadV6`'s `inclusionListSatisfied` response field (evaluated against the previous slot's IL transactions). Blocks imported during sync rather than from live gossip (including optimistic imports) default to satisfied: view-based enforcement does not apply retroactively.
- Update `should_extend_payload` to refuse extending IL-unsatisfied payloads.

### Block production

When a node proposes a block, it ensures its new block payload satisfies the ILs and carries no censoring bid:

- **Self-build:** the previous slot's IL transactions are passed to the EL via the `inclusionListTransactions` field of `PayloadAttributesV5` in the call to `engine_forkchoiceUpdatedV5`, making sure that the built payload includes them.
- **Self-bid:** set the bid's `inclusion_list_bits` from the node's own IL view.
- **Bid selection:** external bids whose `inclusion_list_bits` are not inclusive of the node's IL view are skipped by the node.
- Extend the `payload_attributes` SSE event with `inclusion_list_transactions`, so external builders know which ILs to include.

### Beacon API

Add validator endpoints for the IL committee duties:

- `POST /eth/v1/validator/duties/inclusion_list/{epoch}`: returns which of the provided validators are on the IL committee that epoch.
- `GET /eth/v1/validator/inclusion_list?slot={slot}`: returns the slot's IL transactions that the IL committee member assembles and signs.
- `POST /eth/v1/validator/inclusion_list`: accepts a signed inclusion list, verifies it, and publishes it on gossip (also emitting the SSE event).

### Validator client

- Extend the duties service to poll IL duties for the current and next epoch, cached by `(epoch, dependent_root)`.
- Implement an IL duties service, triggered per slot: fetch the IL transactions, assemble and sign the `InclusionList` under `DOMAIN_INCLUSION_LIST_COMMITTEE` signing domain, and submit it before the timeliness cutoff.

## Roadmap

**Phase 1 - Foundation (weeks 5-8)**

- Add the Heze IL types & constants
- Add the consensus helpers (committee derivation and signature verification)
- Add the `inclusion_list` gossip topic
- Add the `payload_il_satisfied` proto-array flag and its schema migration, landed inert (defaults `true`)
- Support `engine_getInclusionListV1`
- Add the Beacon API produce endpoint (`GET /eth/v1/validator/inclusion_list`), returning the IL to sign

**Phase 2 - Ingestion & validator duties (weeks 9-13)**

- Implement the `InclusionListStore` and the `BeaconChain` read wrappers
- Implement IL gossip verification and processing
- Forward accepted ILs to peers and emit the `inclusion_list` SSE event
- Add the Beacon API duties endpoint
- Add the Beacon API publish endpoint
- Implement the VC IL duties: poll duties, then assemble, sign, and submit the IL each slot before the cutoff

**Phase 3 - Serving, enforcement & block building (weeks 14-18)**

- Implement the `InclusionListByCommitteeIndices` server
- Support `engine_newPayloadV6`
- Implement fork-choice enforcement: record satisfaction at envelope reveal, and update `should_extend_payload` to orphan unsatisfied payloads
- Support `engine_forkchoiceUpdatedV5` / `PayloadAttributesV5`
- Add IL handling to block production: self-build IL-satisfying payloads, set the bid bits, and skip censoring external bids
- Extend the mock EL with the new engine methods

**Phase 4 - Completion & hardening (weeks 19-23)**

- Add the bid-bits check to `execution_payload_bid` gossip verification
- Implement the IL backfill client
- Add end-to-end tests, metrics, and catch up with any spec updates
- (Stretch) Run devnet interop with another FOCIL-ready client

## Possible challenges

- The FOCIL spec is still actively changing, so our work will need to adapt to spec updates.
- Gloas is still under active development in Lighthouse, and we need to make sure that our changes are correctly integrated with the latest Gloas work.
- Dependency on the EIP-7688's forward compatible consensus data structures work: `InclusionList.transactions` is a `ProgressiveList`, and that support is still in draft in Lighthouse, so we will adapt as it evolves.
- Interop and end-to-end testing depend on an EL client that supports the Heze engine methods. Although the support is emerging and is being worked on, for now we will test against Lighthouse's mock EL, which we will extend ourselves.

## Goal of the project

A spec-conformant FOCIL implementation in Lighthouse, ready for Heze devnets.

Success criteria:

1. IL-committee validators produce, sign, and broadcast timely ILs.
2. Nodes validate, store, propagate, and serve ILs per the p2p spec.
3. Fork choice refuses to extend a payload the EL reports as IL-unsatisfied.
4. Proposers build IL-satisfying payloads and reject censoring external bids.
5. All changes have extensive unit/integration tests and metrics coverage.
6. Stretch: devnet interop with another FOCIL-ready client (Lodestar).

## Collaborators

### Fellows

- **Cristian Conache** ([@conache](https://github.com/conache)) - IL production and block proposal updates:
  - Types & constants
  - Engine API integration (production side) and Beacon API endpoints
  - Validator client IL duties and signing
  - Block production, the bid-bits check, and the IL backfill client
- **Rahul Barman** ([@rahulbarmann](https://github.com/rahulbarmann)) - IL ingestion and fork-choice enforcement:
  - Consensus helpers (committee derivation and signature verification)
  - The `InclusionListStore` implementation
  - Gossip validation and propagation, and `InclusionListByCommitteeIndices` serving
  - `engine_newPayloadV6` integration and fork-choice enforcement

### Mentors

- **Eitan** ([@eserilev](https://github.com/eserilev)) - Lighthouse, Sigma Prime.

## Resources

- [Draft design document](https://hackmd.io/@conache/HyDlczNXGl)
- [FOCIL - EIP-7805](https://eips.ethereum.org/EIPS/eip-7805)
- [ePBS and FOCIL compatibility](https://ethereum-magicians.org/t/epbs-focil-compatibility/24777)
- [Beacon API spec PR](https://github.com/ethereum/beacon-APIs/pull/490)
- [Engine API spec PR](https://github.com/ethereum/execution-apis/pull/609)
- [Heze consensus specs](https://github.com/ethereum/consensus-specs/tree/master/specs/heze)
- [Lighthouse Heze boilerplate PR](https://github.com/sigp/lighthouse/pull/9573)
- [Lighthouse FOCIL tracking issue](https://github.com/sigp/lighthouse/issues/6660) and the exploratory [`focil` branch](https://github.com/sigp/lighthouse/tree/focil)
- [`ProgressiveList` support in Lighthouse (via the EIP-7688 PR)](https://github.com/sigp/lighthouse/pull/9450)