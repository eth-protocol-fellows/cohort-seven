# Snapshot-Layer Components for Erigon's Federated History Network

Implementing three block-oriented pieces of the federation stack: `.seg` snapshot epoch-alignment, `era2ss` (era1 → `.seg` cross-client converter), and `ss2rpc` (snapshot-served BlockOnly JSON-RPC).

---

## Motivation

Ethereum's historical archive is served today by a small number of full archive nodes — a systemic risk as node counts shrink. The Erigon team has proposed a **federated history network** to remove this dependency, reusing existing infrastructure (era1, Erigon's `.seg` snapshot torrent swarm, HTTP mirrors) with a small set of glue pieces that make the archive self-certifying and multi-client.

That federation splits into two independent tracks. The Erigon team is directly building the distribution and trust layer, [chain.toml v2](https://github.com/erigontech/erigon/issues/20617). This project builds the block-oriented snapshot layer: aligning `.seg` files to era1 epochs, converting era1 to snapshot files cross-client, and serving snapshots as read-only JSON-RPC.

## Project description

Three deliverables that together form a **history appliance**: consume era1 from any client, produce `.seg` snapshots for the swarm, serve JSON-RPC. No Erigon full node required.

```
┌─────────────────────┐   ┌──────────┐   ┌───────────────────────────────┐
│ any client's era1   │   │          │   │ headers/bodies/txs            │
│ (geth/reth/nimbus)  │──►│  era2ss  │──►│   → epoch-rounded .seg/.idx   │
│ headers·bodies·txs  │   │   (D2)   │   │     (D1)                      │
│ ·receipts·TD        │   │          │   │ receipts → receipt domain     │
└─────────────────────┘   └──────────┘   │   + log index                 │
                                         └───────────────┬───────────────┘
                                                         │
                                                         ▼
                                         ┌────────────────────────────────┐
                                         │  ss2rpc — BlockOnly RPC (D3)   │
                                         │  reads .seg + receipt domain   │
                                         └────────────────────────────────┘
```

era1 carries five things per block — headers, bodies, transactions, receipts, TD — but they don't all land in the same store. Block-oriented data (headers/bodies/txs) goes to epoch-rounded `.seg`; receipts go to step-indexed **receipt domain files** with a log index; TD goes to the db (post-merge it is constant and derivable). D2 produces the `.seg` + receipt-domain + log-index outputs D3 needs.

**D1 — `.seg` epoch-rounding**. Realign Erigon's `.seg` files from ~500k-block segments to `N × 8192` blocks, matching era1's epoch size. Once boundaries match, era1 ↔ `.seg` conversion becomes bijective — the enabler for D2.

**D2 — `era2ss`**. The reverse of Erigon's shipped `ss2era`. Reads era1 from any client's export, writes Erigon-format block `.seg`/`.idx` (headers/bodies/txs) and, from era1's receipts, the step-indexed receipt domain files plus a log index. Any operator can now contribute to the snapshot swarm without running Erigon.

**D3 — `ss2rpc` (BlockOnly minimal)**. Read-only JSON-RPC served directly from `.seg`/`.idx` and the receipt domain files, no live DB / EVM / CL. ~12 methods covering block-explorer and subgraph-indexer usage, with `eth_getLogs` as the anchor — the most expensive method on public RPC providers. State-touching methods (`eth_call` etc.) return typed errors and are follow-up scope.

## Specification

### D1 — `.seg` epoch-rounding

Erigon's `.seg` files (headers / bodies / transactions, one file each) currently segment at ~500k-block boundaries chosen internally, with a recsplit MPHF `.idx` for O(1) block-to-offset lookup. The change: producers write at `N × 8192` block boundaries. Reasonable starting point: N = 64 (~524k blocks/file, close to current size); the value is a small config knob.

Readers accept both old and new formats for one release cycle (3–6 months) to allow the swarm to migrate. Writers emit new format only. `.idx` layout is unchanged; only offsets shift.

**Scope: block-oriented `.seg` only.** era1 files contain block-level data (headers, bodies, receipts, TD; no state), so `N × 8192 blocks` alignment only applies to block-indexed `.seg` files. Erigon's state-domain `.seg` (accounts/storage/code/commitment) is step-indexed — the aggregator's internal state-history granularity, with no 1:1 mapping to blocks — and is being aligned to power-of-2 step boundaries by the Erigon team as part of chain.toml v2 (PR [#20527](https://github.com/erigontech/erigon/pull/20527)). The two alignments are structurally independent, and ss2rpc's BlockOnly / stateless-execution modes never touch state `.seg`.

### D2 — `era2ss`

Once D1 lands, the pipeline is straight-line:

```
era1 (8192 blocks) ──parse──▶ headers, bodies, txs ──▶ {headers,bodies,txs}.seg + .idx
                              receipts             ──▶ receipt domain + log index
```

Receipts is **not** block-oriented `.seg` content. Erigon keeps receipts in step-indexed domain files, so era2ss writes those from era1's receipts and builds the log index that `eth_getLogs` reads. 

Two modes: one-shot CLI (`era2ss convert input.era1 out/`) and sidecar (watch a directory, auto-convert new era1 files, emit manifest delta). Testing: round-trip byte-equality era1 → `.seg` → era1; cross-check against Erigon-produced snapshots and public ethpandaops / nimbus mirrors.

### D3 — `ss2rpc` (BlockOnly minimal)

Reuse Erigon's existing read-only RPC handlers, wired to a **snapshot-only backend** that reads from snapshots only. A method whitelist enforces BlockOnly; anything requiring state returns `-32005 missing prestate`.

Method surface (~12):

- Block: `eth_getBlockByNumber`, `eth_getBlockByHash`, `eth_blockNumber`
- Transaction: `eth_getTransactionByHash`, `eth_getTransactionByBlockHashAndIndex`, `eth_getTransactionByBlockNumberAndIndex`
- Receipts: `eth_getTransactionReceipt`, `eth_getBlockReceipts`
- Logs: `eth_getLogs` 
- Utility: `net_version`, `eth_chainId`, `eth_syncing`


## Roadmap

**W1–6 · D1 `.seg` epoch-rounding.** Epoch-aligned `.seg` writer behind `--seg-epoch-aligned`, with dual-read reader compatibility so the swarm can migrate without a hard cutover. Ready for upstream PR (or internal-branch equivalent) by W6.

**W7–10 · D2 `era2ss`.** One-shot CLI and sidecar mode. Cross-verified byte-equal against Erigon-produced snapshots, using era1 inputs from geth / reth / nimbus and public mirrors.

**W11–16 · D3 `ss2rpc` (BlockOnly minimal).** Snapshot-only backend, ~12 read-only methods with `eth_getLogs`.

If above deliverables completed ahead of schedule, I will investigate open follow-up items in Erigon's chain.toml v2 track (issues #20619, #20933) and pick up a well-scoped sub-task.

## Possible challenges

**Choosing N + open-handle pressure (D1).** N = 64 is a starting point, not locked in. More files (higher N) means more file descriptors held open by the downloader / seeder — a real archive could hit OS limits at high N. 

**Byte-equal cross-client verification (D2).** `era2ss` output must match Erigon's own snapshot bytes. seg/compress dictionary encoding is deterministic but sensitive — any variation in field order, dictionary build, or record framing produces different bytes for the same logical content. 

**RPC method-surface coverage (D3).** Erigon exposes ~55 read-only methods across the `eth_` / `erigon_` / `ots_` / `debug_` / `web3_` / `net_` namespaces, and full coverage is out of reach in the slot. This project deliberately scopes to the ~12 core block/tx/receipt methods plus `eth_getLogs`. However, will try implement all.

**State-touch false positives (D3).** Some BlockOnly-classified handlers may transitively call state code, silently satisfying block queries by touching state. The runtime state-touch guard catches these at test time, but each case needs a fix — route around, patch upstream, or reclassify.


## Goal of the project

Three shipped artifacts:

1. Epoch-aligned `.seg` writer + reader in Erigon, with dual-read migration compatibility.
2. `era2ss` binary producing segments byte-equivalent to Erigon's own snapshots — from geth / reth / nimbus era1 exports.
3. `ss2rpc` binary serving ~12 read-only methods including `eth_getLogs`, parity-verified against full Erigon on a golden request set.

Out of scope:

- **chain.toml v2** — Erigon team's active state-domain-oriented effort.
- **Stateless execution in ss2rpc** — `eth_call` / `debug_traceCall` are follow-up work.
- **Manifest gossip** — largely landed in Erigon's PR [#20526](https://github.com/erigontech/erigon/pull/20526).
- **Beacon `.era` exporter** — separate CL-side track.

## Collaborators

### Fellows

Solo project.

### Mentors

Mark Holt.

## Resources

**Specs**: [era1](https://github.com/eth-clients/e2store-format-specs/blob/main/formats/era1.md) · [`.era`](https://github.com/eth-clients/e2store-format-specs/blob/main/formats/era.md) · [`.ere`](https://github.com/eth-clients/e2store-format-specs/blob/main/formats/ere.md) · [e2store](https://github.com/status-im/nimbus-eth2/blob/stable/docs/e2store.md) · [execution-apis](https://github.com/ethereum/execution-apis)

**Related Erigon issues (tracked, not touched)**: [#20617](https://github.com/erigontech/erigon/issues/20617) umbrella · [#20618](https://github.com/erigontech/erigon/issues/20618) baseline · [#20619](https://github.com/erigontech/erigon/issues/20619) target · [#20933](https://github.com/erigontech/erigon/pull/20933) producer-gate · [#20531](https://github.com/erigontech/erigon/issues/20531) merge-determinism · [#20587](https://github.com/erigontech/erigon/issues/20587) sparse snapshots
