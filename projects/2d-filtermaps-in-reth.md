# FilterMaps log index for eth_getLogs in Reth

A client-side two-dimensional filter map index that replaces the per-block bloom scan in Reth's `eth_getLogs`.

## Motivation

`eth_getLogs` in Reth walks every header in the requested range and tests its 2048-bit bloom filter. That is the `logs` path in `crates/rpc/rpc/src/eth/filter.rs`. The cost is O(range) even when the query matches nothing, so a wide historical query is slow no matter how few logs come back.

Blooms also do not help much when they do match. Mainnet blooms are saturated, so a match usually means fetching and re-checking receipts that turn out to contain nothing relevant. The filter is doing very little filtering.

Geth replaced this with filter maps in `core/filtermaps` and now answers the same queries by looking up rows in an index instead of scanning headers. Reth has no equivalent.

## Project description

I am implementing the filter maps data structure from EIP-7745 in Reth as a local search index, following Geth's `core/filtermaps` as the reference implementation.

The important scoping point is that "log index" means two different things, and I am only doing one of them.

A **consensus commitment** is a structure the protocol commits to, so a light client can prove a log query result is correct and complete. EIP-7745 does this with `log_index_root` in the header. EIP-8304 does it with index table roots in a system contract. These two compete, and 8304 is the newer design from the same author.

A **local search index** is a database on the node's own disk that makes finding logs fast. Nothing is proven. This is what Geth's `core/filtermaps` is, and it is what I am building.

Neither EIP gives a node a local search index. EIP-8304's in-protocol tables cover only about 2^18 blocks in a ring buffer, roughly 36 days, and everything older is deferred to a ZKP-backed history index contract that the EIP explicitly does not define. Its tables are also sorted for proof generation over 1 to 256 block ranges, so serving a ten-million-block `eth_getLogs` from them would mean walking tens of thousands of tables.

So the index I am building is needed whether 7745 lands, 8304 lands, or neither does. Geth demonstrates this directly: it ships filter maps on by default with no EIP activated.

**Out of scope**, stated explicitly so it does not get read back in: SSZ containers, `log_index_root`, Merkle proofs, EIP-7745 consensus conformance, wire protocol changes, and op-reth.

## Specification

### The index

Every log contributes searchable values to a global sequence: one address value, then one value per topic. Each value gets a position in that sequence, the log value index. Positions are grouped into filter maps of `VALUES_PER_MAP` values each, and maps are grouped into epochs of `MAPS_PER_EPOCH` maps.

A map is a grid of `MAP_HEIGHT` rows by `MAP_WIDTH` columns. A value lands in a row derived from the value hash and the epoch, and a column derived from the value hash and its log value index. The row is stable across a whole epoch, which is what makes a lookup one contiguous read instead of a scatter. Rows that overflow spill into higher mapping layers with larger row limits.

A query hashes the searched address or topic, reads that row across the maps covering the block range, and turns the column entries back into candidate log value indices. Those resolve to candidate blocks.

### Value index space

I am matching Geth's layout exactly, including the parts that look like implementation detail.

Geth reserves one log value index slot per block as a delimiter. The slot is consumed but never hashed into a row, so it is not searchable. There are no transaction entries. EIP-7745 instead defines block entries and transaction entries as searchable values, `sha2(block_hash + 0x02)` and `sha2(tx_hash + 0x01)`.

That means the two value spaces are not the same space. The EIP burns one slot per block plus one per transaction, Geth burns one per block. Different positions produce different column indices, so the resulting maps are incompatible and cannot be converted without reindexing.

I am going with Geth's layout because it makes Geth usable as a correctness oracle, which is the single most valuable thing available to this project. The layout sits behind one set of parameters so switching to the EIP layout later is just a reindex, not a redesign.

### Storage

Three MDBX tables:

- `FilterMapRows`, keyed `(epoch, row, map)`. This ordering puts one row across all maps of an epoch next to each other, so a lookup is one cursor walk. Same idea as `TxNumber` and `BlockBodyIndices`.
- `FilterMapMeta`, holding the index parameters and the indexed range state.
- `LogValueIndices`, mapping block number to its first log value index.

Not static files. Static file segments in Reth are addressed by block number ranges throughout, `StaticFileTargets` is `RangeInclusive<BlockNumber>` for every segment. Filter map rows are addressed by `(epoch, row, map)`, which has no usable relationship to block number, so a static file segment would mean extending core storage infrastructure as a side project. Rows near the head also get overwritten as maps re-render, and static files are append-only. I will measure on-disk size on mainnet and revisit static files for completed epochs if the number justifies it.

Parameters go in `FilterMapMeta` and are validated on open. If stored parameters do not match configured parameters the index refuses to serve reads and rebuilds. 

### Building the index

The index is built by a background service, not a pipeline stage.

A stage does not work here. Reth only runs the pipeline when the gap to the target exceeds `MIN_BLOCKS_FOR_PIPELINE_RUN`, which is `EPOCH_SLOTS` — see `exceeds_backfill_run_threshold` in `crates/engine/tree/src/tree/mod.rs`. A node at the tip keeps `backfill_sync_state` idle and never invokes the pipeline. So a new stage would never run on an already-synced node, and nobody resyncs a node to gain an index. A stage-only index is empty for essentially every existing user.

The service subscribes to canonical state notifications from `crates/chain-state/src/notifications.rs` for the head, and walks history for the tail. Head and tail are the same loop with different bounds, which also collapses what would otherwise be two write paths into one. `PrunerService` and `StaticFileProducer` are already wired as background services in the node builder, so this follows an existing pattern rather than inventing one.

The tail is indexed backwards from the head, following Geth, so recent blocks become searchable within minutes of startup rather than after a full historical pass. Combined with bloom fallback for unindexed ranges, the index is useful immediately and degrades to current behaviour otherwise.

### Reorgs and history bounds

The index never does fine-grained deletes. That invariant covers both ends.

On a reorg the indexed range shrinks and the affected maps are left in place, to be overwritten by the next render. This is Geth's approach. #18305 instead deleted row by row across every map in the affected range, seeking each of `MAP_HEIGHT` rows individually, which on a deep reorg is millions of cursor operations.

At the tail there are two independent bounds, matching Geth:

- A history cutoff, derived from receipt availability via the `PruneSegment::Receipts` checkpoint. The index refuses to extend below it, the same way Geth errors with "cannot start indexing before history cutoff point". The index is never more optimistic than the data behind it, so `index ⊆ available receipts` holds by construction.
- A configurable rolling window, `--log-index.history`, mirroring Geth's `--history.logs`. As the head advances the tail target advances with it.

Tail deletion happens one whole epoch at a time and is resumable across restarts, with partial progress persisted in the range state. Geth does this with `tailPartialEpoch` and `cleanedEpochsBefore` for the same reason.

### Read path

A `LogIndexProvider` trait resolves a filter to candidate block numbers. Matchers cover the three cases: a single value, any of a set (address or topic alternatives), and a sequence (address followed by topics at fixed offsets).

`eth_getLogs` uses the index for indexed ranges and falls back to the bloom scan otherwise. The other three filter methods — `new_filter`, `get_filter_logs`, `get_filter_changes` — reach the same range scan, and `get_filter_changes` re-scans on every poll, so they get wired up too.

### Correctness

The index is only a hint. The final answer always comes from re-filtering real receipts, so a wrong hint makes a query slower and cannot make it wrong. The dangerous failure is a false negative, dropping a block that genuinely matched, and that is what the tests target.

Three layers:

1. **Geth-derived test vectors.** A small Go program runs `core/filtermaps` over a fixed sequence of blocks and emits the resulting rows and candidate sets as JSON. The vectors are committed and run in CI with no Go dependency at test time. This is the layer that pins the value index space, and it is the layer that would have caught the delimiter divergence.
2. **Differential test against the bloom path.** Same queries through the index and through the existing bloom scan over real chain data, comparing final logs. The two paths produce different candidate sets by design. The index is more precise, but both are supersets of the true matches, so after re-filtering receipts they must agree exactly.
3. **Ported Geth unit tests plus fuzzing** on the mapping math. Layer 1 matters because layers 2 and 3 both miss the specific defect that killed the prior attempt. Math tests pass with a wrong value space, because the mapping functions never see how positions were assigned. The differential test passes too, because a self-consistent wrong layout still agrees with its own blooms. Only the vectors pin the layout itself.

## Roadmap

Roughly 16 weeks. Work is cut into small PRs. Everything stays behind a flag so the default `eth_getLogs` path is unchanged until the last PR.

**Phase 1 — foundations (weeks 1–3)**

1. filter map parameters and mapping math
2. Geth-derived filter map test vectors
3. log value iteration over blocks and receipts

**Phase 2 — storage (weeks 4–5)**

1. filter map tables
2. map renderer
3. persist rendered maps and index range state

**Phase 3 — indexer service (weeks 6–9)**

1. background indexer service, head follow
2. extend index tail backwards over history
3. shrink indexed range on reorg
4. delete tail epochs beyond history window

**Phase 4 — read path (weeks 10–12)**

1. single, any and sequence matchers
2. `LogIndexProvider`
3. resolve `eth_getLogs` candidates from the index, off by default
4. differential test against the bloom path

**Phase 5 — completion (weeks 13–16)**

1. CLI flags and metrics
2. benchmark `eth_getLogs`, index versus bloom
3. use the index for the filter polling methods
4. enable the index by default

I am not serialising work on merges. Branches stack locally and rebase as things land, otherwise review latency sets the pace and most of these PRs never get written.

## Possible challenges

**The matcher is the hardest component and it is late.** Geth's `matcher.go` is 925 lines, layered lookups where a value can live in any of four layers depending on row fill, unions for alternatives, intersections with positional offsets for sequences. 

**Reorg handling.** Etan flagged this as the tricky part of EIP-7745 and Geth had a long reorg bug tail. Shrink-don't-delete removes the performance problem but the range bookkeeping still has to be exactly right, including across restarts mid-operation.

**Storage size.** A new index that meaningfully grows disk usage is a reasonable thing for a maintainer to object to, and "Geth does it too" is not a number. I need a measured mainnet figure early, since it also determines whether the rolling window default is sensible.

**Reth is not Geth, and only half of this ports.** The filter map structure itself is self-contained: parameters, row and column derivation, the renderer, and the matcher are pure functions over hashes and positions, with no real client coupling. That is the part the Geth vectors pin down, and that is the part where the port is mostly a transliteration. Everything around it has to be written against Reth’s own primitives, and that is where the differences start to matter.

Receipt availability is not a prefix in Reth. Geth has one history cutoff, so “indexable” basically means a contiguous range below the head. Reth has `PruneModes.receipts`, but also `receipts_log_filter`, a `ReceiptsLogPruneConfig` that retains receipts selectively by log filter. Under that config, a range can have some receipts and not others. Receipts can also resolve out of either static file segments or MDBX depending on the range, so log value iteration should go through `ReceiptProvider`, not a direct table walk. Log-filter-pruned ranges have to count as unindexed, otherwise the index can produce false negatives, which is the one failure mode that matters.

Canonical notifications can also run ahead of disk. `CanonStateNotification` carries `Arc<Chain>` with receipts already in memory, which is cheaper than Geth re-reading them. But Reth’s engine tree can hold `memory_block_buffer_target` blocks in memory and only persist past `persistence_threshold`, so a notification can describe blocks whose receipts are not on disk yet. Geth writes on insert and does not have this window. If the indexer commits rows for those blocks and the process dies, the index can end up ahead of the data it is only supposed to be a hint for. So the head-follow path should clamp to `PersistenceState::last_persisted_block`.

There are also two smaller Reth-specific differences. Geth’s matcher fans out over goroutines and can block on reads freely. Reth’s `eth_getLogs` path is async, while MDBX reads are blocking and use a cursor per thread, so parallel row lookups need one read transaction per worker on a blocking pool. Also, `EthFilter` is generic over `EthApiTypes` and reused by `op-reth` and downstream nodes, while Geth has one concrete `FilterSystem`. So the index should enter through a trait with a no-op default, not as a hard field on the filter type.

None of these Reth-specific parts are covered by the Geth vectors. The vectors cover the pure FilterMaps logic, not Reth’s pruning model, persistence window, reorg behavior, async read model, or downstream generic API shape. So reorg and pruning behavior need their own tests instead of pretending the differential test covers them.

**Divergence from Geth.** Geth's filter maps are still being changed. My port is against current `master` and the committed vectors will drift. The vectors are regenerable by design, so this is maintenance rather than a rewrite.

## Goal of the project

Success is a filter map index that works in Reth and makes `eth_getLogs` faster on real chains. The index is built by a background service, correct across reorgs and history bounds, serves `eth_getLogs` behind a flag, and is proven equivalent to the bloom path by differential test. Then it is benchmarked against the current bloom scan on both a testnet and mainnet.

I want to run the same set of `eth_getLogs` queries, wide historical ranges, sparse addresses, common topics, empty results, against both paths on the same node and same data, and report the speedup per query shape on mainnet and on a public testnet.

The number to hit is the one in the tracking issue. Geth with its log index answers WETH transfer queries over EF treasury addresses in 0.55s over 100k blocks and 22s over 2.5M blocks, against 8.4s and 211s for Reth's bloom path. Success is parity with Geth on those shapes, on one machine, same chain data, both clients at the same head. I am stating that as an absolute rather than as a multiple of the Reth figures, because those predate concurrency work that has since landed. If the bloom path has improved the ratio shrinks while parity still holds, so the baseline gets re-measured rather than quoted.

Below roughly 5x on wide sparse queries the index has not earned its disk, and I would rather say that now than discover it at week 16. The structural claim underneath is the one that holds whatever the hardware is: latency should track the number of matches, not the width of the range. The bloom path reads one header per block, the index reads one row group per searched value per map. Latency against range width at a fixed match count is the plot that shows it.

The index also has to not make anything worse. Ranges too small to benefit, and unindexed ranges falling back to blooms, both stay within noise of today. Head indexing keeps up with a 12s slot with room to spare. And I report the mainnet on-disk figure in GB and as a fraction of receipt size, since by my own argument above that is the number a maintainer will actually ask for.

## Collaborators

### Fellows

[0xAysh](https://github.com/0xAysh)

### Mentors

TBD

## Resources


| Resource                         | Link                                                                                                                                       |
| -------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------ |
| Reth tracking issue              | [https://github.com/paradigmxyz/reth/issues/16999](https://github.com/paradigmxyz/reth/issues/16999)                                       |
| Prior attempt (closed, unmerged) | [https://github.com/paradigmxyz/reth/pull/18305](https://github.com/paradigmxyz/reth/pull/18305)                                           |
| Geth `core/filtermaps`           | [https://github.com/ethereum/go-ethereum/tree/master/core/filtermaps](https://github.com/ethereum/go-ethereum/tree/master/core/filtermaps) |
| EIP-7745                         | [https://eips.ethereum.org/EIPS/eip-7745](https://eips.ethereum.org/EIPS/eip-7745)                                                         |
| EIP-8304                         | [https://eips.ethereum.org/EIPS/eip-8304](https://eips.ethereum.org/EIPS/eip-8304)                                                         |
| Filter maps explainer            | [https://pureth.guide/filter-maps/](https://pureth.guide/filter-maps/)                                                                     |
| Log value index explainer        | [https://pureth.guide/log-value-index/](https://pureth.guide/log-value-index/)                                                             |


