# Ream: Beacon Sync Modernization and PeerDAS Sampling

Robust head synchronization, optimistic execution-engine validation, and Fulu PeerDAS sampling for the Ream consensus client.

## Motivation

Ethereum consensus clients must synchronize seamlessly from genesis or checkpoints up to the canonical head, maintain live tracking of unfinalized slots, and adapt to the architectural requirements of the upcoming Fulu upgrade (PeerDAS). 

In [Ream](https://github.com/ReamLabs/ream), a Rust-based Ethereum consensus client, synchronization and post-Dencun/Fulu data availability handling face critical operational challenges:
1. **Incomplete Range Synchronization ([ReamLabs/ream#1550](https://github.com/ReamLabs/ream/issues/1550))**: Currently, checkpoint sync terminates once the finalized checkpoint is reached without transitioning into an active head-sync phase. Consequently, blocks between the finalized checkpoint and the canonical head are omitted, RANDAO reveals fail to land locally (causing validator committee divergence), and valid gossip attestations get rejected.
2. **Synchronous Execution Bottlenecks**: Full payload validation blocks beacon sync progress when the execution client lags behind. An optimistic sync pipeline is needed to import blocks optimistically and handle safe rollbacks if the execution layer flags an invalid payload.
3. **PeerDAS Integration & Sampling Mechanics ([ReamLabs/ream#1105](https://github.com/ReamLabs/ream/issues/1105))**: The Fulu fork transitions blob sidecars to data columns and peer-sampling matrices. Client sync routines must not crash on zero-blob blocks post-Fulu and must correctly implement column discovery, extended sample count calculations, and column fetching (`DataColumnSidecarsByRange`).

This project solves these fundamental synchronization and data-availability gaps, transforming Ream into a production-grade, spec-compliant client capable of tracking live mainnet/testnet heads and validating Fulu PeerDAS networks.

## Project description

This project focuses on three interconnected tracks in the Ream consensus client:

1. **Head Synchronization & Peer Classification**:
   - Implement a post-finalization head sync phase within `ream_syncer` that bridges the gap between the finalized checkpoint and live network head.
   - Refactor peer management to cluster peers by fork/head agreement rather than using flat unweighted slot averages, preventing synchronization deadlocks on forked peers.
   - Implement peer reputation and progressive penalty scoring to avoid permanent premature bans of healthy peers.

2. **Optimistic Sync Engine & Deferred Execution Verification**:
   - Enable beacon block importing before execution payload verification completes (`PayloadStatusV1::Syncing`/`Optimistic`).
   - Maintain an optimistic root tracking table in `BeaconDB` to isolate speculative state transitions.
   - Implement robust rollback mechanisms for cascading block, blob, and state pruning upon receiving `PayloadStatusV1::Invalid`.

3. **PeerDAS Sampling Mechanics & Fulu Column Synchronization**:
   - Implement core peer-sampling functions according to the Fulu consensus specification (`get_extended_sample_count`, hypergeometric cumulative distributions, and combinatorics).
   - Adapt `ream_syncer` to handle post-Fulu slot transitions without expecting legacy blob sidecars.
   - Integrate `DataColumnSidecarsByRange` and `DataColumnSidecarsByRoot` request-response routines for PeerDAS sync.

## Specification

### 1. Head Sync & Range Syncer State Machine
- **Transition Logic**: In `ream_syncer::block_range`, after `checkpoint_sync` confirms the finalized epoch, range sync transitions to `HeadSync` mode rather than terminating with `Stage Finished`.
- **Target Slot Resolution**: `PeerManager` will track peer status responses by `(finalized_root, head_root)`. The syncer selects canonical head targets based on fork-weight consensus rather than global `max(slot)`.
- **Gossip Attestation Hand-off**: When the distance `head_slot - current_slot < SLOTS_PER_EPOCH`, the syncer hands off block ingestion to the live gossip manager while retaining backfill capabilities.

### 2. Optimistic Sync and Rollback Pipeline
- **Execution Interface**: Interfacing with the Engine API via `engine_newPayloadV3/V4` and `engine_forkchoiceUpdatedV3`.
- **State Handling**: If `status == SYNCING`, the block is imported optimistically, marking the block root in `OptimisticRootsTable`.
- **Invalid Payload Cascade Pruning**:
  ```rust
  fn purge_invalid_chain(&mut self, bad_root: &H256) -> Result<(), SyncError> {
      let descendants = self.db.get_descendant_roots(bad_root)?;
      for root in descendants.iter().rev() {
          self.db.remove_block(root)?;
          self.db.remove_blobs(root)?;
          self.db.remove_state(root)?;
          self.optimistic_roots.remove(root)?;
      }
      self.fork_choice.remove_root(bad_root);
      Ok(())
  }
  ```
- **Deadlock Mitigation**: Transaction guards around table read/write locks in `redb` to prevent lock contention during bulk block removals.

### 3. Fulu PeerDAS Peer Sampling
- **Sampling Calculation**: Implement hypergeometric CDF and binomial coefficient computation to derive extended sample counts based on custody subnets:
  $$\text{CDF}(k; N, K, n) = \sum_{i=0}^{k} \frac{\binom{K}{i}\binom{N-K}{n-i}}{\binom{N}{n}}$$
- **Column Syncer Branch**:
  - For slots $\ge \text{FULU\_FORK\_EPOCH} \times \text{SLOTS\_PER\_EPOCH}$, bypass blob sidecar validation.
  - Dispatch column range queries via P2P RPC `DataColumnSidecarsByRange` to reconstruct custody matrices.

## Roadmap

The project is structured across four phases over the fellowship duration:

### Phase 1: PeerDAS Mathematical Primitives & Optimistic Sync Baseline (Weeks 1-4) — *Completed / In Progress*
- [x] Implement `get_extended_sample_count`, `math_comb`, and `hypergeom_cdf` adhering to Fulu consensus specs ([PR #1575](https://github.com/ReamLabs/ream/pull/1575)).
- [x] Complete comprehensive unit test coverage with spec vectors and boundary conditions.
- [x] Rebase and resolve optimistic sync architecture with `OptimisticRootsTable` and deferred execution handling ([PR #1576](https://github.com/ReamLabs/ream/pull/1576)).

### Phase 2: Range Syncer Head Sync & Peer Management (Weeks 5-8)
- [ ] Implement `HeadSync` stage in `ream_syncer` to download blocks between finalized checkpoint and live head ([Issue #1550](https://github.com/ReamLabs/ream/issues/1550)).
- [ ] Implement peer clustering by fork digest and canonical chain root.
- [ ] Replace binary peer bans with a dynamic scoring engine (penalties, cooldowns, score decay).
- [ ] Verify head synchronization on Sepolia testnet.

### Phase 3: Post-Fulu Data Column Range Sync (Weeks 9-12)
- [ ] Fix zero-blob crash regression for post-Fulu blocks in `ream_network_manager`.
- [ ] Implement `DataColumnSidecarsByRange` and `DataColumnSidecarsByRoot` request-response protocols.
- [ ] Connect column fetching to the data availability verification boundary.
- [ ] Test cross-client column backfilling with mature clients (Lighthouse / Grandine).

### Phase 4: Devnet Validation, Kurtosis Testing & Hardening (Weeks 13-16)
- [ ] Deploy multi-node Ream setups running continuous sync in Kurtosis devnets.
- [ ] Validate reorg resilience, invalid payload recovery, and attestation propagation post-sync.
- [ ] Benchmark memory footprint and sync throughput during historical and head sync.

## Possible challenges

- **Spec Inconsistencies**: Fulu PeerDAS specifications and column networking protocols are actively evolving. Close tracking of `ethereum/consensus-specs` is required.
- **Peer Scoring Pitfalls**: Over-penalizing peers during transient network partitions or under-penalizing malicious peers can degrade sync throughput or invite DoS.
- **Database Lock Contention**: Rollbacks involving hundreds of unfinalized optimistic blocks require atomic batch operations in `redb` without blocking concurrent consensus reads.

## Goal of the project

The final objective is a production-ready sync engine for Ream that:
1. Seamlessly transitions from checkpoint sync to live head sync without human intervention or stalls.
2. Interoperates on live Ethereum testnets (Sepolia/Holesky) and Fulu PeerDAS devnets.
3. Optimistically tracks consensus heads while guaranteeing complete, deterministic recovery from invalid execution payloads.
4. Correctly validates and exchanges post-Fulu data columns without crashing or rejecting valid chain data.

## Collaborators

### Fellows
- **Alok** ([@alok-108](https://github.com/alok-108)) — Lead contributor: synchronization engine, optimistic sync, and Fulu PeerDAS sampling.
- Cross-collaboration with fellows working on the Ream consensus client:
  - **Daniel Pham** ([@perfogic](https://github.com/perfogic)) — Fulu full custody & DA decoupling.
  - **Hans Vuong** ([@vuonghuuhung](https://github.com/vuonghuuhung)) — Sepolia sync testing & Issue #1550 author.
  - **Tosin** ([@tosynthegeek](https://github.com/tosynthegeek)) — Kurtosis & cross-client testing.

### Mentors
- [Shariq Naiyer](https://github.com/shariqnaiyer)
- [Kolby](https://github.com/KolbyML)

## Resources

- [Ream Repository](https://github.com/ReamLabs/ream)
- [PR #1575: Fulu get_extended_sample_count Implementation](https://github.com/ReamLabs/ream/pull/1575)
- [PR #1576: Optimistic Sync Implementation](https://github.com/ReamLabs/ream/pull/1576)
- [Issue #1550: Head sync in the range syncer](https://github.com/ReamLabs/ream/issues/1550)
- [Ethereum Consensus Specs: Fulu](https://github.com/ethereum/consensus-specs/tree/master/specs/fulu)
- [PeerDAS Devnet Specifications](https://notes.ethereum.org/@ethpandaops/peerdas-devnet-6)
