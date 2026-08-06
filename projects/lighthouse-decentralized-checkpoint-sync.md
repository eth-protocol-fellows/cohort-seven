# Lighthouse Decentralized Checkpoint Sync

Trust-minimized checkpoint sync for Ethereum full nodes via light client bootstrap and verifiable state backfill.

## Motivation

Checkpoint sync today requires nodes to trust a checkpoint provider — a URL or other source that supplies the state needed to bootstrap. That trust assumption is at odds with a chain that claims to be decentralized.

This project implements a trust-minimized checkpoint sync pathway. A new node bootstraps as a light client using a hardcoded, network-verified block root (e.g., the first Altair block root), syncs forward to the present using the existing light client protocol, obtains cryptographic proof of a recent agreed-upon checkpoint state, and then backfills that state from the p2p network in verifiable chunks.

This work primarily affects the sync protocol of full nodes and extends the responsibilities of light client data providers in the network, who will now be required to store and serve certain data they were not required to previously.

## Project description

Our proposed solution is a multi-phase trust-minimized checkpoint sync strategy, inspired by the light client (LC) sync protocol and guided by ongoing work on light client data backfill ([EIP-7658](https://eips.ethereum.org/EIPS/eip-7658)). The flow for a joining node is:

1. **Light Client Bootstrap:** The node starts with a trusted block root baked into the client (e.g., the first Altair block root). It requests a `LightClientBootstrap` and initializes a `LightClientStore`.
2. **Forward Sync:** The node requests `LightClientUpdates` by range, syncing forward from the trusted root to the present. It now has a verified recent `beacon_block_root`.
3. **Checkpoint Discovery:** The node requests a `LightClientBeaconSnapshot`, a structure containing a recent, agreed-upon state root and a Merkle proof connecting it to the verified block header.
4. **State Backfill:** The node fetches the `BeaconState` at that state root in fixed-size chunks from multiple peers. Each chunk comes with a Merkle proof verifying its inclusion in the state tree. The node verifies each chunk independently and reassembles the full state.
5. **Full Node Activation:** Once the state is fully fetched and verified, the node has everything it needs to transition to full node duties.

The core of this project, however, is not just the high-level flow — it is the Lighthouse client infrastructure required to make this flow possible. Our research into the Lighthouse codebase revealed that the storage schema and serving endpoints for light client data already exist (`DBColumn::LightClientUpdate`, `SyncCommitteeBranch`, `SyncCommittee`, and the p2p `LightClientUpdatesByRange` endpoint). The critical gap is that historical data is not persisted due to the following barriers:

**Recency guard in `import_block_update_metrics_and_events`:**

```rust
// Do not write to the cache for blocks older than 2 epochs, this helps reduce writes to
// the cache during sync.
if block_delay_total < self.slot_clock.slot_duration() * 64 {
    // Store the timestamp of the block being imported into the cache.
    self.block_times_cache.write().set_time_imported(
        block_root,
        current_slot,
        block_time_imported,
    );
}
```

This is an optimization. When Lighthouse is syncing old blocks, it skips notifying the light client server to avoid extra computation. The assumption was: "nobody needs light client data for old blocks." That assumption now stands as a barrier to what this project aims to achieve.

**Storage gap in `get_light_client_bootstrap`:**

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

Even if you synced old blocks, Lighthouse doesn't retroactively compute and store the `sync_committee_branch`. It only stores this for the blocks you processed while already synced.

The combined effect: a Lighthouse node that synced from genesis or checkpoint cannot serve `LightClientBootstrap` or historical `LightClientUpdates` because it never generated or stored the proofs.

Our solution is a trust-minimized checkpoint sync strategy borrowed from/inspired by the LC sync strategy:

1. **Make Lighthouse "collect" historical light client data** — The consensus spec tests verify that a client can construct the full sequence of light client objects, but Lighthouse currently throws away historical data. We need to ensure Lighthouse can generate and persist:
   - `LightClientUpdate` for every sync committee period
   - `sync_committee_branch` for finalized checkpoint blocks
   - `LightClientBootstrap` data for historical finalized blocks

2. **Design the backfill API** — Once nodes have the data, we need a way to request it. This is the first TBD endpoint and it should look something like:

   ```
   GET /eth/v1/beacon/light_client/updates/backfill?from_period={u64}&count={u64}
   ```

   Or a libp2p protocol. The response would be a batch of `LightClientUpdates`. Key design questions:
   - Should this be a REST API (Beacon-API) or a libp2p gossip/rpc protocol?
   - How do you rate-limit it? (Historical updates could be megabytes)
   - How does a peer prove it has the data vs. lying?

3. **Pass the spec tests** — There are test vectors in the consensus-specs repo. The goal is to make Lighthouse generate the expected outputs for all historical periods, not just the current one.

## Specification

### Phase 1: Historical Light Client Data Collection in Lighthouse

**The Problem:** `recompute_and_cache_updates()` in `LightClientServerCache` already computes Merkle proofs, constructs `LightClientUpdates`, and stores `SyncCommitteeBranches` to the database. However, it is only invoked for recent blocks because `import_block_update_metrics_and_events()` gates the light client server channel as explained earlier. Furthermore, `cache_state_data()` runs for all blocks but only persists to an in-memory LRU cache of size 32, meaning proofs are computed and then discarded for historical blocks.

**The Solution:** Implement a post-sync backfill task that walks the finalized chain from the Altair fork to the present and reuses the existing `recompute_and_cache_updates()` logic to populate the database for all historical sync committee periods.

- **Trigger:** After the node transitions from `SyncState::Syncing` to `SyncState::Synced`, spawn a background task (`lc_backfill`) that iterates finalized blocks period-by-period.
- **Resumability:** Before processing a period, check `store.get_light_client_update(period)`. If it exists, skip.
- **Optimization:** Rather than loading the state for every block in a period (~8,192 slots), iterate block headers via `get_blinded_block()`, identify the block with the best `SyncAggregate` participation in that period, and only load the state and call `recompute_and_cache_updates()` for that candidate. This reduces expensive state loads from thousands per period to one per period.
- **Bootstrap Data:** Ensure `store_sync_committee_branch()` and `store_sync_committee()` are also called during backfill so that `get_light_client_bootstrap()` works for any historical finalized checkpoint, not just recent ones.
- **Fallback for On-Demand Generation:** For nodes that cannot run the full backfill (e.g., due to pruned states), modify `get_light_client_bootstrap()` and `get_light_client_updates()` to fall back to on-demand computation: if the DB entry is missing, load the archived state from the freezer DB, compute the proof or update, store it, and return it. The pattern for this already exists in `get_or_compute_prev_block_cache()`.
- **Testing:** Ensure Lighthouse passes the consensus-specs light client data collection tests. Currently, Lighthouse lacks the data collection coverage because it cannot reconstruct the full historical sequence.

**Data Collection Test Handler (Aarish)**

The `light_client_data_collection` test handler has been implemented and the handler follows Lighthouse's `LoadCase` + `Case` trait pattern:

- `LoadCase` reads `initial_state.ssz_snappy` into a Beacon state, parses `steps.yaml` using typed structs, and loads `SignedBeaconBlock` objects using fork-aware SSZ deserialization via `from_ssz_bytes_by_fork`
- `Case` initializes a `BeaconChainHarness` from the initial state, processes `NewBlock` steps by importing blocks and calling `recompute_and_cache_updates` directly on `light_client_server_cache` using the block's own sync aggregate
- `NewHead` step handling is in progress — will verify the light client cache against expected values loaded from SSZ files
- Draft PR: [sigp/lighthouse#9666](https://github.com/sigp/lighthouse/pull/9666)

### Phase 2: LightClientBeaconSnapshot Endpoint

Design and prototype an endpoint to serve a recent, agreed-upon state root with a Merkle proof connecting it to the `beacon_block_root` that the light client already trusts.

The snapshot contains: `beacon_block_root`, state root, and a Merkle proof that state root is the correct field in the `BeaconBlockHeader`. Since the block header is already trustlessly known from LC sync, the proof allows the node to verify the state root without trusting the endpoint provider.

This requires nodes to collect and serve the snapshot data, and likely requires an addition to the Beacon API or a new libp2p protocol.

### Phase 3: Beacon Sync Protocol — State Chunking

Design the protocol for fetching the `BeaconState` in verifiable chunks. The proposed approach is:

- Chunk by top-level `BeaconState` fields (and possibly sub-chunk large fields like `validators` by index range). This aligns with SSZ's natural Merkle tree structure, allowing each chunk to be verified with standard Merkle proofs against known generalized indices.
- Parallel fetching from multiple peers, with immediate rejection and re-request of any chunk that fails verification.
- The chunking strategy should align with SSZ's natural tree layout to avoid extra hashing.

### Phase 4: Integration & End-to-End Testing

- Wire the components together: LC bootstrap, forward sync, snapshot, chunk fetch, state assembly.
- Test that a new peer can join the network, validate cryptographically all the way to the present, and transition to full node duties without trusting a checkpoint URL.
- Write spec tests and integration tests for the backfill API.

## Roadmap


| Phase    | Timeline     | Deliverables                                                                                                                             | Fellow(s) Responsible |
| -------- | ------------ | ---------------------------------------------------------------------------------------------------------------------------------------- | --------------------- |
| Phase 1a | Week 7 – 8   | Implement post-sync backfill task; remove historical data gaps in `LightClientUpdate` and `SyncCommitteeBranch` storage                  | Roheemah              |
| Phase 1b | Week 8 – 11  | Implement on-demand fallback for `get_light_client_bootstrap` and `get_light_client_updates`; pass consensus-specs data collection tests | Roheemah, Aarish      |
| Phase 2  | Week 10 – 12 | Design and prototype `LightClientBeaconSnapshot` endpoint (state root + Merkle proof)                                                    | Yee                   |
| Phase 3  | Week 12 – 14 | Design beacon sync chunking protocol (fixed-size chunks with Merkle multi-proofs); prototype p2p endpoint                                | Aarish                |
| Phase 4  | Week 14 – 16 | Integration testing: end-to-end trustless checkpoint sync; spec test compliance; documentation                                           | Roheemah, Aarish, Yee |




## Possible challenges

- **State Availability for Backfill:** The backfill task requires loading historical `BeaconStates` from the freezer DB. If a node pruned states before backfill completed, or if it checkpoint-synced and never had old states, it cannot generate historical light client data. We must document that backfill requires either archive node configuration or fetching missing data from peers (the backfill API).
- **Performance During Sync:** While the backfill task runs in the background, loading and hashing old states is CPU and I/O intensive. We must ensure it yields to validator duties and does not starve the node of resources. The task should be pausable and low-priority.
- **Channel Overflow:** If we remove the recency guard entirely instead of using post-sync backfill, the light client server channel (currently bounded to 32 slots) will overflow during fast sync, dropping events. The post-sync backfill approach avoids this, but we must verify that the `prev_block_cache` (also size 32) does not become a bottleneck if we repurpose the flow.
- **Beacon Sync Chunk Proofs:** Designing Merkle multi-proofs for arbitrary fixed-size byte ranges of an SSZ container is non-trivial. The proofs must be efficient to generate (without rehashing the entire state) and compact enough to not negate the benefit of chunking.
- **EIP-7658 Scope:** EIP-7658 defines a hard-fork mechanism for trustlessly proving that a served `LightClientUpdate` is the canonical "best" one. This project operates before such a hard fork, meaning peers must trust that the update served is the best available. We must be clear about this trust assumption and design the protocol to be forward-compatible with EIP-7658.

## Goal of the project

**Minimum Viable Goal**

- Lighthouse generates, stores, and serves historical `LightClientUpdates` for all sync committee periods from Altair to present.
- Lighthouse stores `SyncCommitteeBranch` and `SyncCommittee` for all historical finalized checkpoints, enabling `get_light_client_bootstrap` for any valid block root.
- Lighthouse passes all consensus-specs light client data collection tests.

**Stretch Goals**

- A working `LightClientBeaconSnapshot` endpoint that serves a recent state root with a Merkle proof against a trustlessly known block header.
- A working pathway for nodes to fetch and verify the `BeaconState` in fixed-size chunks with Merkle proofs, completing the trustless checkpoint sync loop.
- A documented backfill API (p2p or REST) allowing nodes to discover and fetch missing historical light client updates from peers.

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
