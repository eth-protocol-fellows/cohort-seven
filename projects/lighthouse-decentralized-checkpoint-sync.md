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
**Phase 1a - Local Collection:** Implement a post-sync backfill task that walks the finalized chain backward, from the most recently finalized sync committee period to the node's earliest available *state* (per `store.get_historic_state_limits()`). (Note: this is bounded by whichever `BeaconState`s the node actually retains — a checkpoint-synced, non-archive node cannot walk back to Altair; only an archive node can. Backfill coverage is therefore a function of node configuration, not a project guarantee of full Altair-to-present coverage on every node type.) For each sync committee period, call the existing block-import light client update path — `recompute_and_cache_updates` — for every block in the period. It already retains only the spec-best update per period via `is_better_light_client_update`, so no separate candidate-selection step is needed; it stores the canonical `LightClientUpdate` and `SyncCommitteeBranch` for whichever block wins that comparison.

This task:
- Spawns once the node reaches `SyncState::Synced`, gated on `store.get_historic_state_limits()` rather than `BackFillState::Completed` — block backfill only guarantees historical *blocks* are available, not the *states* this task needs, so it doesn't need to wait for it and can run concurrently with it
- Walks backward, from the most recently finalized sync committee period toward the node's earliest available state, so the periods most likely to be requested by peers are backfilled first
- Is resumable (checks `store.get_light_client_update(period)` before processing)
- Is pausable and low-priority (yields to validator duties)
- Only works for periods where the node has the `BeaconState`
- Reuses Lighthouse's **existing** `DBColumn::LightClientUpdate` / `SyncCommitteeBranch` / `SyncCommittee` storage — this phase does not introduce a new DB column for `LightClientEpochData`. `LightClientEpochData` is a **wire/transport container** defined by the spec draft for the p2p endpoint below; we avoid committing local storage to a container shape that may still change. Note: this storage retains only the period's winning block, not per-epoch data for every slot — sufficient for Phase 1a's goal (`LightClientUpdate` history + bootstrap), but not by itself enough to serve the full `LightClientDataBackfillByRange` endpoint as specced (see Phase 1b scoping note below)

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

`LightClientEpochData` contains per-epoch raw block data including `sync_committee_bits`, `sync_aggregate_branch`, `finalized_checkpoint`, and `current_sync_committee` — everything a receiver needs to independently simulate `is_better_update` and verify the canonical best update for a period.

**Storage scoping decision:** Phase 1a's storage (`LightClientUpdate`/`SyncCommitteeBranch`/`SyncCommittee`) retains only the winning block per sync committee period — not per-slot data for every epoch in that period. The `LightClientDataBackfillByRange` endpoint as drafted requests by individual `epoch`, and each `LightClientEpochData` response is expected to contain full per-slot `block_data` for that epoch, including epochs that did not win their period's `is_better_light_client_update` comparison. This is a real granularity gap: the endpoint's purpose (per the spec: "verifying every single field... then simulating is_better_update") is to let the requester independently recompute which block was best, rather than trust the server's selection — which requires access to the non-winning epochs' raw data too.

**For this project's MVP, we're building Option A: scope the endpoint to what Phase 1a already stores** — serve period-best `LightClientUpdate` history rather than the full per-epoch `LightClientEpochData` container. This requires no new storage, ships against the existing consensus-specs `light_client_data_collection` test format (which does not test `LightClientEpochData`), and fits the project timeline. The trade-off is accepted for MVP: a requester has to trust the server's "best block" selection rather than independently verify it from raw per-epoch data, so this does not yet deliver the endpoint's full trust-minimization goal as drafted.

**Option B — full per-slot storage for every backfilled epoch, matching the spec as drafted — is the eventual path**, targeted as a stretch goal beyond this project's MVP scope (or a follow-on after EPF). It requires new storage design (roughly 32x more records than Option A) and should ideally mirror Nimbus's existing implementation (PR #8445) rather than Lighthouse inventing its own independently.

Key constraints:
- Only serves finalized data
- Fork context determined from last non-empty `block_data[i]` (or `epoch` if all empty)
- Rate-limited to 256 epochs per request

**On-Demand Fallback:** For periods not yet backfilled, modify `get_light_client_updates` and `get_light_client_bootstrap` to check the DB first, then fall back to computing from archived states if available (archive-mode nodes only). For pre-checkpoint data on a non-archive node, local generation is impossible — the node must request it from peers via the endpoint above.

**Testing (Phase 1a/1b):**

1. **Unit tests**
   - `recompute_and_cache_updates` produces a correct `LightClientUpdate` and `SyncCommitteeBranch` for a known-good historical period
   - Constructing `LightClientEpochData` from stored `LightClientUpdate`/`SyncCommitteeBranch` data round-trips correctly against the container spec
   - Best-candidate selection (highest sync aggregate participation) picked correctly across a period with missed slots

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

### Phase 2: BeaconStateSnapshot & Checkpoint Bootstrap (Yee)

Implement the `BeaconStateSnapshot` endpoint for state snap sync:

```
/eth2/beacon_chain/req/beacon_state_summary/0/

Request:  (block_root: Root)  # LightClientStore.finalized_header
Response: BeaconStateSnapshot
```

The `BeaconStateSnapshot` contains:
- `summary: BeaconStateSummary` — a mirror of `BeaconState` where all `List` / `ProgressiveList` fields are summarized as `ListSummary { items_root, num_items }`, preserving the same `hash_tree_root`
- `state_branch: ProgressiveList[Bytes32]` — Merkle proof for the summary

This allows a light client to obtain a compact, verifiable summary of the `BeaconState` at the start of a sync period, from which it can then request individual state parts.

The server must keep the last 2 summaries available to avoid rollover during ongoing downloads.

Also implement the trusted checkpoint bootstrap mechanism: if network metadata contains `trusted_checkpoint.txt` with `0x<block_root>:<epoch>`, light clients start syncing from this root; otherwise, use genesis (if post-Altair) or require `--trusted-block-root`.

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

- **Protocol evolution risk:** Etan's HackMD spec is explicitly marked as a draft ("TBD and likely not yet optimal"). The `LightClientEpochData` container, chunk sizes, and endpoint paths may change during implementation. We build defensively, pinning only the parts that are stable (the existing `LightClientUpdate` storage, the `is_better_update` ranking logic) and adapting the transport layer as the spec stabilizes.
- **State Availability for Backfill:** The backfill task requires loading historical `BeaconStates` from the freezer DB. If a node pruned states before backfill completed, or if it checkpoint-synced and never had old states, it cannot generate historical light client data for those periods. We document that full historical coverage requires either archive node configuration or fetching missing data from peers (the backfill API) — it is not something the local backfill task alone can guarantee on every node type.
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
 