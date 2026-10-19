# Lighthouse Decentralized Checkpoint Sync

Trust-minimized checkpoint sync for Ethereum full nodes via light client bootstrap and verifiable state backfill.

## Motivation

Checkpoint sync today requires nodes to trust a checkpoint provider — a URL or other source that supplies the state needed to bootstrap. That trust assumption is at odds with a chain that claims to be decentralized.

This project implements a trust-minimized checkpoint sync pathway. A new node bootstraps as a light client using a hardcoded, network-verified block root (e.g., the first Altair block root), syncs forward to the present using the existing light client protocol, obtains cryptographic proof of a recent agreed-upon checkpoint state, and then backfills that state from the p2p network in verifiable chunks.

This work primarily affects the sync protocol of full nodes and extends the responsibilities of light client data providers in the network, who will now be required to store and serve certain data they were not required to previously.

## Project description

Our proposed solution is a multi-phase trust-minimized checkpoint sync strategy, inspired by the light client (LC) sync protocol and guided by Etan's [decentralized CL sync specification](https://hackmd.io/@etan-status/decentralized-cl-sync), which supersedes the earlier [EIP-7658](https://eips.ethereum.org/EIPS/eip-7658) approach by eliminating the hard-fork requirement and defining concrete p2p endpoints for epoch-level backfill and state snap sync. The flow for a joining node is:

1. **Light Client Bootstrap:** The node starts with a trusted block root baked into the client (e.g., the first Altair block root). It requests a `LightClientBootstrap` and initializes a `LightClientStore`.
2. **Forward Sync:** The node requests `LightClientUpdates` by range, syncing forward from the trusted root to the present. It now has a verified recent `beacon_block_root`.
3. **Checkpoint Discovery:** The node requests a `LightClientBeaconSnapshot`, a structure containing a recent, agreed-upon state root and a Merkle proof connecting it to the verified block header.
4. **State Backfill:** The node fetches the `BeaconState` at that state root in fixed-size chunks from multiple peers. Each chunk comes with a Merkle proof verifying its inclusion in the state tree. The node verifies each chunk independently and reassembles the full state.
5. **Full Node Activation:** Once the state is fully fetched and verified, the node has everything it needs to transition to full node duties.

The critical gap: historical light client data was never generated in Lighthouse, and there is a single root cause for this.

The function `import_block_update_metrics_and_events` is designed so that it only notifies the light client server for blocks that are within 32 slot-durations of the current time:

```rust
// Do not trigger light_client server update producer for old blocks, to extra work
// during sync.
if self.config.enable_light_client_server
    && block_delay_total < self.slot_clock.slot_duration() * 32
    && let Some(mut light_client_server_tx) = self.light_client_server_tx.clone()
    && let Ok(sync_aggregate) = block.body().sync_aggregate()
    && let Err(e) = light_client_server_tx.try_send((
        block.parent_root(),
        block.slot(),
        sync_aggregate.clone(),
    ))
{
    warn!(
        error = ?e,
        "Failed to send light_client server event"
    );
}
```

This code is an intentional optimization: it ensures Lighthouse avoids the extra computation of generating light client updates while racing to catch up during initial sync or backfill. This keeps the live sync path efficient by only processing recent blocks relevant to light client data consumers.

However, this singular recency guard causes two key downstream effects:

- **LightClientUpdate is never persisted for historical periods:** Because `recompute_and_cache_updates` is never invoked for old blocks, Lighthouse never generates or stores historical `LightClientUpdate` data.
- **`get_light_client_bootstrap` fails for historical roots:** As documented in the source:
  
  ```rust
  // we currently have no backfill mechanism for these values.
  // Therefore, sync_committee_branch and sync_committee are only persisted while a node is synced.
  #[allow(clippy::type_complexity)]
   pub fn get_light_client_bootstrap(
      &self,
      store: &BeaconStore<T>,
      block_root: &Hash256,
      finalized_period: u64,
      chain_spec: &ChainSpec,
   ) -> Result<Option<(LightClientBootstrap<T::EthSpec>, ForkName)>, BeaconChainError> {...}
```
  ```
  
  The function tasked with serving `get_light_client_bootstrap` only persists `sync_committee_branch` and `sync_committee` for the periods being actively synced. If a user requests historical roots (pre-checkpoint or unbackfilled blocks), it fails, simply because the persistence never occurred for those historical periods.

**Our solution does not touch this live import optimization.**  
Instead, we add a dedicated post-sync background task that leverages the exact same data persistence pathway—calling `recompute_and_cache_updates`, which in turn writes to `store_light_client_update`, `store_sync_committee_branch`, and `store_current_sync_committee`. This new task operates over historical periods after live sync has completed, bypassing the import guard entirely. This means Lighthouse can safely and efficiently generate and persist all historical light client data needed, without risking performance of the main sync path, and without loosening the guard that protects it.

1. **Make Lighthouse "collect" historical light client data** — The consensus spec tests verify that a client can construct the full sequence of light client objects, but Lighthouse currently throws away historical data. We need to ensure Lighthouse can generate and persist, for every period it has state available:
   - `LightClientUpdate` for every sync committee period
   - `sync_committee_branch` for finalized checkpoint blocks
   - `LightClientBootstrap` data for historical finalized blocks

2. **Design the backfill API** — Once nodes have the data, we need a way to request it from peers for periods they don't have locally. This is a libp2p request/response protocol, following the pattern of `LightClientDataBackfillByRange`'s spec. Key design questions already resolved by the spec:
   - Transport: libp2p req/resp
   - Rate limiting: capped at 256 epochs per request
   - Proof-of-honesty: every field in the response is independently Merkle-verifiable by the requester, so a lying peer is detectable, not merely trusted

3. **Pass the spec tests** — There are test vectors in the consensus-specs repo. The goal is to make Lighthouse generate the expected outputs for all periods it has state for, not just the current one.

## Specification

### Phase 1: Historical Light Client Epoch Data Collection & Serving

**The Problem:** Lighthouse already computes light client proofs for every block during sync, but the recency guard in `import_block_update_metrics_and_events` and the bounded channel (`LIGHT_CLIENT_SERVER_CHANNEL_CAPACITY = 32`) prevent historical data from reaching the database. Additionally, `get_light_client_bootstrap` explicitly lacks a backfill mechanism. The result: checkpoint-synced nodes cannot serve historical LC data to peers.

**The Solution** 
**Phase 1a - Local Collection:** Implement a post-sync backfill task, gated behind an explicit opt-in flag (off by default, following the existing `--archive`/`--reconstruct-historic-states` precedent). The task walks the finalized chain forward, from the node's earliest available state (per `store.get_historic_state_limits()`) toward the present. (Note: coverage is bounded by whichever `BeaconState`s the node actually retains. A checkpoint-synced, non-archive node cannot walk back to Altair; only an archive node can.) For each sync committee period, call the existing block-import light client update path — `recompute_and_cache_updates` — for every block in the period, in slot order. It already retains only the spec-best update per period via `is_better_light_client_update`, so no separate candidate-selection step is needed.

This task:
- Runs only when explicitly enabled via an opt-in flag, off by default
- Walks forward, from the node's earliest available state toward the present, to work with (not against) the `historic_state_cache`'s incremental replay
- Is resumable via a dedicated marker, set only once every block in a period has been processed.
- Is pausable and low-priority (yields to validator duties)
- Only works for periods where the node has the `BeaconState`
- Reuses Lighthouse's **existing** `DBColumn::LightClientUpdate` / `SyncCommitteeBranch` / `SyncCommittee` storage — this phase does not introduce a new DB column for `LightClientEpochData`. `LightClientEpochData` is a **wire/transport container** defined by the spec for the p2p endpoint below; we avoid committing local storage to a container shape that's still evolving. Note: this storage retains only the period's winning block, not per-epoch data for every slot — sufficient for Phase 1a's goal (`LightClientUpdate` history + bootstrap), but not by itself enough to serve the full `LightClientDataBackfillByRange` endpoint as specced (see Phase 1b scoping note below)

**Phase 1b - P2P Serving:** Implement the `LightClientDataBackfillByRange` libp2p endpoint as specified in spec draft:

```
/eth2/beacon_chain/req/light_client_data_backfill_by_range/0/

Request:  (start_epoch: Epoch, count: uint64)
Response: List[LightClientEpochData, MAX_REQUEST_LIGHT_CLIENT_EPOCH_DATA]
MAX_REQUEST_LIGHT_CLIENT_EPOCH_DATA := 256
```

Implementation follows the pattern of the existing `handle_light_client_updates_by_range` handler in `network_beacon_processor::rpc_methods.rs`:
- Validate the request (count ≤ 256, `start_epoch` and range must be finalized)
- Query stored `LightClientUpdate`/`SyncCommitteeBranch` data for the requested range and assemble each `LightClientEpochData` response entry
- Stream responses back via `SendResponse`

`LightClientEpochData` contains per-epoch raw block data (`sync_committee_bits`, `sync_aggregate_branch`, `finalized_checkpoint`) plus a nested `bootstrap_data: LightClientBootstrapData` container (`current_sync_committee`, `current_sync_committee_branch`, `execution_block_hash`, `execution_branch`) — everything a receiver needs to independently simulate `is_better_update` and verify the canonical best update for a period.

Key constraints:
- Only serves finalized data
- Fork context determined from last non-empty `block_data[i]` (or `epoch` if all empty)
- Response size capped at `MAX_REQUEST_LIGHT_CLIENT_EPOCH_DATA := 256` epochs per request.

**Storage scoping decision:** Phase 1a's storage (`LightClientUpdate`/`SyncCommitteeBranch`/`SyncCommittee`) retains only the winning block per sync committee period — not per-slot data for every epoch in that period. The `LightClientDataBackfillByRange` endpoint as drafted requests by individual `epoch`, and each `LightClientEpochData` response is expected to contain full per-slot `block_data` for that epoch, including epochs that did not win their period's `is_better_light_client_update` comparison. This is a real granularity gap: the endpoint's purpose (per the spec: "verifying every single field... then simulating is_better_update") is to let the requester independently recompute which block was best, rather than trust the server's selection — which requires access to the non-winning epochs' raw data too.

**MVP endpoint scope:** Phase 1a's storage (`LightClientUpdate`/`SyncCommitteeBranch`/`SyncCommittee`) retains only the winning block per sync committee period — not per-slot data for every epoch in that period. The `LightClientDataBackfillByRange` endpoint as specced requests by individual `epoch`, with each `LightClientEpochData` response expected to contain full per-slot `block_data`, including epochs that didn't win their period's `is_better_light_client_update` comparison — because the endpoint's purpose is letting the requester independently recompute which block was best, not trust the server's selection.

**For this project's MVP, Phase 1b serves period-best `LightClientUpdate` history** rather than the full per-epoch `LightClientEpochData` container. This requires no new storage, ships against the existing consensus-specs `light_client_data_collection` test format, and fits the project timeline. The trade-off: a requester has to trust the server's "best block" selection rather than independently verify it from raw per-epoch data.

**Future work (not being built in this project):** Full per-slot storage for every backfilled epoch, matching the endpoint as specced — roughly 256x more records than the MVP (1 per sync committee period → 1 per epoch, 256 epochs/period). Jeff is independently investigating this for Prysm; worth coordinating with him rather than designing it twice. This would also need to mirror whatever storage shape Nimbus settles on, given the spec is still evolving.

**Testing (Phase 1a/1b):**

1. **Unit tests**
   - `recompute_and_cache_updates` produces a correct `LightClientUpdate` and `SyncCommitteeBranch` for a known-good historical period
   - Constructing `LightClientEpochData` from stored `LightClientUpdate`/`SyncCommitteeBranch` data round-trips correctly against the container spec
   - Best-candidate selection picked correctly across a period with missed slots, using the full `is_better_light_client_update` ranking.

2. **Integration tests**
   - Post-sync backfill completes for all periods from earliest available state to present
   - Resume-on-crash: backfill continues from the last completed period rather than restarting
   - `get_light_client_bootstrap` succeeds for any historical finalized checkpoint the node has backfilled
   - On-demand fallback returns correct data for archive-mode nodes and correctly errors (or falls through to network) for non-archive nodes

3. **Consensus spec tests**
   - Pass `light_client_data_collection` test vectors (Aarish's draft PR #9666 implements the test handler)

4. **P2P endpoint tests**
   - Peer requests an epoch range and receives the expected `LightClientEpochData` list
   - Requests over `MAX_REQUEST_LIGHT_CLIENT_EPOCH_DATA` (256) are rejected
   - Fork context is correctly derived from the last non-empty `block_data[i]`, falling back to `epoch` when the whole range is empty
   - Requests for unfinalized data are rejected

5. **Adversarial tests** (design in Phase 1, exercise in Phase 4)
   - Peer returns incorrect `sync_committee_bits` → rejected on proof mismatch
   - Peer omits blocks in a period → detected via Merkle proof verification
   - Peer returns unfinalized or malformed data → rejected

### Phase 2: Verified State Acquisition & Checkpoint Bootstrap (Yee)
Build a verified acquisition path from an externally trusted finalized block root to checkpoint state data. Keep verification transport-independent, reuse Lighthouse’s existing infrastructure, and integrate authenticated snapshot summaries with the P2P state-parts downloader.
#### Phase 2.1 — Light-Client Consumer Core (PR1, implementation completed)
Implement a transport-independent light-client consumer core:
- Verify a bootstrap against an externally trusted finalized block root.
- Validate and process light-client updates, including committee rotation and supported fork transitions.
- Track checkpoint-eligible finalized headers separately from optimistic or timeout-forced progress.
- Cover the implementation with unit tests and official light-client sync conformance tests.
Output: VerifiedFinalizedHeader, exposing an authenticated beacon block root, state root, and slot/fork context.
#### Phase 2.2 — HTTP Light-Client Consumer (PR2)
Drive the consumer core using untrusted light-client REST providers:
- Define LightClientDataSource with fixture and HTTP implementations.
- Reuse Lighthouse’s HTTP client for bootstrap, updates-by-range, and latest finality updates.
- Implement bounded requests, retries, cancellation, and explicit invalid/unavailable/no-progress handling.
- Return a verified finalized header satisfying a locally configured freshness policy.
- Test the complete HTTP-to-verification path using real cryptography and Lighthouse-generated fixtures.

Acceptance:
```text
Trusted finalized block root
  → untrusted HTTP provider
  → verified bootstrap and processed updates
  → recent VerifiedFinalizedHeader
```
This phase does not yet change Lighthouse’s production startup flow.
#### Phase 2.3a — Verified Checkpoint Handoff
Reuse Lighthouse’s existing checkpoint-sync download and initialization logic, adding a thin adapter from VerifiedFinalizedHeader:
- Fetch the state and matching block by authenticated roots using the existing HTTP client.
- Verify their roots against the authenticated header before any state advancement.
- Pass the verified data to the existing weak_subjectivity_state builder.
- Add integration tests ensuring mismatched data is rejected before handoff and valid data follows the existing initialization path.

#### Phase 2.3b — Snapshot Summary Verification & Handoff

Implement the proposed snapshot endpoint:

```text
/eth2/beacon_chain/req/beacon_state_summary/0/

Request:  block_root
Response: BeaconStateSnapshot { summary, state_branch }
```

- Generate fork-aware summaries that preserve the full state’s `hash_tree_root`, tested against real `BeaconState` fixtures.
- Verify the returned summary and its branch against the light-client-authenticated header’s `state_root`.
- Pass the authenticated summary and snapshot root to the P2P state-parts downloader, providing the commitments needed to verify parts and reconstruct the state.
- Retain the last two summaries and their associated parts data to support downloads across rollover.

**Deliverable:** a verified snapshot summary usable by the downloader. Coordinate the handoff with the downloader owner; parts scheduling and reconstruction remain separate responsibilities.

#### Phase 2.4 — Trusted Checkpoint Configuration & Startup

Connect the verified acquisition path to Lighthouse’s existing checkpoint initialization.

- Select the trusted root from `--trusted-block-root`, or bundled `trusted_checkpoint.txt` (`0x<block_root>:<epoch>`). Without either, use a known genesis root only if light-client bootstrap is supported there; otherwise require an explicit root.
- For the summary/parts route, require the reconstructed state’s root to match the authenticated snapshot root and obtain its matching block.
- Reuse `weak_subjectivity_state(...)` and the existing startup checks for both the direct-state and reconstructed-state routes.

**Deliverable:** an opt-in startup path from a trusted root to a verified checkpoint, followed by normal Lighthouse synchronization, without changing existing checkpoint-sync behavior.

### Phase 3: State Snap Sync — BeaconStatePartsByRange (Aarish)

Implement the state chunking protocol as specified:

```
/eth2/beacon_chain/req/beacon_state_parts_by_range/0/

Request:  (start_chunk: uint64, count: uint64)
Response: List[BeaconStatePart, MAX_REQUEST_BEACON_STATE_PARTS]
MAX_REQUEST_BEACON_STATE_PARTS := 16
```

`BeaconStatePart` contains:
- `chunk_index: uint64`
- `data: ProgressiveByteList` — the actual chunk
- `branch: ProgressiveList[Bytes32]` — Merkle proof

Chunking is deterministic per chunk ID based on `ListSummary` fields. Each list has a defined "items per chunk" (e.g., validators: 2^12 per chunk, balances: 2^16 per chunk). Target: <0.5 MB per chunk.

The first chunk of a `ProgressiveList` contains all smaller subtrees that fit completely within the items-per-chunk budget. For example, if items-per-chunk is 32, the first chunk contains 1+4+16 = 21 items, and subsequent chunks contain 32 items each.

The node fetches the `BeaconStateSnapshot` first (Phase 2), then requests parts by range, verifies each chunk's Merkle proof against the summary, and reassembles the full state.

### Phase 4: Integration & End-to-End Testing

- Wire the components together: LC bootstrap, forward sync, snapshot, chunk fetch, state assembly.
- Test that a new peer can join the network, validate cryptographically all the way to the present, and transition to full node duties without trusting a checkpoint URL.
- Write spec tests and integration tests for the backfill API.
- Exercise the adversarial test cases designed in Phase 1 against a live testnet peer set.

## Phase interdependencies

```
Phase 1a/1b (Historical LC data)
  ├─ Depends on: Altair fork support already in Lighthouse
  ├─ Produces: locally verified LightClientUpdate/SyncCommitteeBranch data,
  │            served over the network as LightClientEpochData
  └─ Used by:   Phase 2 (links a verified block header to a state root)

Phase 2 (BeaconStateSnapshot — Yee)
  ├─ Depends on: Phase 1 (a verified recent block header to anchor to)
  ├─ Produces:  BeaconStateSummary + Merkle proof from block header to state root
  └─ Consumed by: Phase 3 (state chunk verification root)

Phase 3 (BeaconStatePartsByRange — Aarish)
  ├─ Depends on: Phase 2 (the state summary chunks are proved against)
  ├─ Produces:  verifiable BeaconState chunks
  └─ Consumed by: Phase 4 (full state reassembly)

Phase 4 (End-to-end)
  ├─ Orchestrates: Phases 1–3 in sequence
  ├─ Tests: full checkpoint-sync flow, including adversarial peers
  └─ Success: a new node joins without trusting a checkpoint URL
```

Phase 1 is independently useful on its own (it fixes a real gap in Lighthouse's existing LC serving today); Phases 2–3 build on it for the full trust-minimized checkpoint-sync pipeline.

## Roadmap

| Phase    | Timeline     | Deliverables                                                                                                                   | Fellow(s)        |
| -------- | ------------ | ------------------------------------------------------------------------------------------------------------------------------ | ---------------- |
| Phase 1a | Week 7 – 9   | Post-sync backfill task; store historical `LightClientUpdate` + `SyncCommitteeBranch` from earliest available state to present | Roheemah         |
| Phase 1b | Week 9 – 12  | `LightClientDataBackfillByRange` p2p endpoint; on-demand fallback; pass consensus-specs data collection tests                  | Roheemah, Aarish |
| Phase 2  | Week 12 – 14 | `BeaconStateSnapshot` endpoint; trusted checkpoint bootstrap                                                                   | Yee              |
| Phase 3  | Week 14 – 15 | `BeaconStatePartsByRange` endpoint; deterministic chunking                                                                     | Aarish           |
| Phase 4  | Week 15 – 16 | End-to-end integration; documentation                                                                                          | All              |

## Possible challenges

- **Protocol evolution risk:** Etan's HackMD spec is a draft and not yet optimal. The `LightClientEpochData` container, chunk sizes, and endpoint paths may change during implementation. We build defensively, pinning only the parts that are stable (the existing `LightClientUpdate` storage, the `is_better_update` ranking logic) and adapting the transport layer as the spec stabilizes.
- **State Availability for Backfill:** The backfill task requires loading historical `BeaconStates` from the freezer DB. Beyond ordinary pruning, coverage of the oldest history is a function of how many long-running, archive, or reconstructed nodes exist and continue to serve it. Hence, there's no protocol guarantee it survives forever, and this project builds the redistribution mechanism, not a guarantee of underlying availability.
- **Performance During Sync:** While the backfill task runs in the background, loading and hashing old states is CPU and I/O intensive. We must ensure it yields to validator duties and does not starve the node of resources. The task should be pausable and low-priority.
- **Channel Overflow:** If we remove the recency guard entirely instead of using post-sync backfill, the light client server channel (currently bounded to 32 slots) will overflow during fast sync, dropping events. The post-sync backfill approach avoids this, but we must verify that the `prev_block_cache` (also size 32) does not become a bottleneck if we repurpose the flow.
- **Beacon Sync Chunk Proofs:** Designing Merkle multi-proofs for arbitrary fixed-size byte ranges of an SSZ container is non-trivial. The proofs must be efficient to generate (without rehashing the entire state) and compact enough to not negate the benefit of chunking.
- **No BeaconState modifications required:** The revised spec avoids BeaconState modifications entirely (an improvement over the earlier EIP-7658 approach, which needed one). The trust model relies on Merkle-proved `LightClientEpochData` rather than enshrined state tracking, which is why this project does not require a hard fork.

## Goal of the project

**Minimum Viable Goal**

- Lighthouse generates, stores, and serves historical `LightClientUpdates` for all sync committee periods it has state available for.
- Lighthouse stores `SyncCommitteeBranch` and `SyncCommittee` for all historical finalized checkpoints it has backfilled, enabling `get_light_client_bootstrap` for those block roots.
- Lighthouse passes all consensus-specs light client data collection tests.

**Stretch Goals**

- A working `LightClientBeaconSnapshot` endpoint that serves a recent state root with a Merkle proof against a trustlessly known block header.
- A working pathway for nodes to fetch and verify the `BeaconState` in fixed-size chunks with Merkle proofs, completing the trustless checkpoint sync loop.
- A documented, spec-aligned `LightClientDataBackfillByRange` p2p endpoint allowing nodes to discover and fetch missing historical light client data from peers.

**Success Criteria**

This project will be considered successful when a new peer can join the Ethereum consensus network without trusting a checkpoint URL, using only:

- A hardcoded, network-verified block root
- The light client sync protocol
- Cryptographically verifiable proofs of state inclusion
- The standard p2p network for data availability

## Collaborators

### Fellows

- [Roheemah](https://github.com/AbolareRoheemah)
- [Aarish](https://github.com/aarishnaiyer)
- [Yee](https://github.com/yxz252426)

### Mentors

- [Etan](https://github.com/etan-status)

## Resources

- [Etan's decentralized CL sync spec](https://hackmd.io/@etan-status/decentralized-cl-sync)
- [Nimbus PR #8445 — historical LC backfill design](https://github.com/status-im/nimbus-eth2/pull/8445)
- [Aarish's draft PR — Data collection test handler](https://github.com/sigp/lighthouse/pull/9666)
- [Altair light client sync protocol](https://github.com/ethereum/consensus-specs/blob/master/specs/altair/light-client/sync-protocol.md)
- [Data collection test format](https://github.com/ethereum/consensus-specs/blob/master/tests/formats/light_client/data_collection.md)
- [Etan's Nimbus implementation reference](https://github.com/status-im/nimbus-eth2/blob/stable/tests/consensus_spec/test_fixture_light_client_data_collection.nim)
- [Lighthouse codebase](https://github.com/sigp/lighthouse)
- [Beacon API — getLightClientBootstrap](https://ethereum.github.io/beacon-APIs/#/Beacon/getLightClientBootstrap)
- [Beacon API — getLightClientUpdatesByRange](https://ethereum.github.io/beacon-APIs/#/Beacon/getLightClientUpdatesByRange)
- [Beacon API — getLightClientFinalityUpdate](https://ethereum.github.io/beacon-APIs/#/Beacon/getLightClientFinalityUpdate)
- [Beacon API — getLightClientOptimisticUpdate](https://ethereum.github.io/beacon-APIs/#/Beacon/getLightClientOptimisticUpdate)
- [Beacon API — event stream](https://ethereum.github.io/beacon-APIs/#/Events/eventstream)
 