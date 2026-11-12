# Teku Light Client Protocol

Implementing server-side light client protocol in Teku consensus client for resource-constrained nodes.

## Motivation

The Altair upgrade introduced the [light client sync protocol](https://github.com/ethereum/consensus-specs/blob/master/specs/altair/light-client/), which allows resource-constrained nodes to track Ethereum's state without replaying the full consensus chain. Instead of trusting a centralized provider, a light client can verify compact proofs and aggregated signatures from rotating sync committees, making it possible for low-resource infrastructure to follow Ethereum.

This protocol has two sides: light clients that consume data and beacon nodes that serve it. On the server side, a node must produce, store and share the light client data that lets other peers bootstrap, follow finalized checkpoints and track the latest head. Teku currently implements only part of this serving role. It can provide a client with initial bootstrap, but it cannot yet serve the ongoing updates a client needs to follow the chain. This matters for environments where running a full node is unrealistic but relying entirely on centralized infrastructure weakens Ethereum's trust model.

## Project Description

Teku already supports the initial light client bootstrap flow, which gives a light client a trusted starting point. However, bootstrap data alone is not enough. After a light client starts, it also needs fresh updates so it can keep following Ethereum over time.

The goal of this project is to complete Teku's server-side light client support so Teku can generate, store, and serve regular light client updates, finality updates, and optimistic updates through the standard network and API channels.

The project begins with a review of the partial work already done under [#6384](https://github.com/Consensys-Incorporated/teku/pull/6384) to establish what is in place, what is incomplete, and what regressed, before building on it.

The project is server-side only and covers REST, gossip, and Req/Resp networking across active forks from Altair onward, including the upcoming Gloas fork. Client-side light client sync, where Teku would act as a light client itself, is out of scope.

The project also persists light client updates and stores bootstraps in the database, so they can be served across restarts.

## Specification

The implementation follows the [consensus-specs](https://github.com/ethereum/consensus-specs/blob/master/specs/altair/light-client/), which evolves across forks (Altair → Capella → Deneb → Electra → Gloas). The key areas of work are:

### Data Types

The protocol defines several SSZ containers. Teku already has `LightClientHeader`, `LightClientBootstrap`, and `LightClientUpdate`. This project adds the missing types - `LightClientFinalityUpdate` and `LightClientOptimisticUpdate`.

* **Fork Transitions:** From Capella onward, the `LightClientHeader` includes an `ExecutionPayloadHeader` and a merkle proof to let light clients verify execution-layer headers. At Electra and Gloas, the generalized indices for sync committee and finality proofs shift due to new `BeaconState` fields, requiring fork-aware dispatch. The generalized indices will be derived from the concrete `BeaconState` schema of each fork.

### Server-Side Data Generation

* `create_light_client_update` — generates a full update from attested state and optional finalized block.
* `create_light_client_finality_update` — derives a finality update from a full update.
* `create_light_client_optimistic_update` — derives an optimistic update.
* `is_better_update` — the spec's comparison logic for selecting the best update per sync committee period.
* **Proof helpers** — Merkle branch computation for next sync committee and finalized checkpoint root. 
* **Gloas Canonical Execution Shift:** Under Gloas' ePBS rules, execution payload data in a beacon block is a future bid and might not become canonical. The canonical execution state is stored in the hash of the canonical execution payload. Thus, for post-Gloas blocks, the execution proof generates a merkle branch proving the parent block hash instead of the block's own execution payload root.

### Storage

* The best `LightClientUpdate` per sync committee period will be kept in memory and persisted in the database, along with the root of its signature block, so updates survive restarts and updates orphaned by a reorg can be removed.
* Updates will be retained for the `MIN_EPOCHS_FOR_BLOCK_REQUESTS` window and older periods will be pruned on finalization.
* Bootstraps for epoch boundary blocks will be built at block import and stored in the database once finalized, instead of being rebuilt from a state on every request.

### REST API

There are four standard Beacon API light client endpoints:

* `GET /eth/v1/beacon/light_client/updates` — currently returns 501, needs completion.
* `GET /eth/v1/beacon/light_client/finality_update` — new.
* `GET /eth/v1/beacon/light_client/optimistic_update` — new.
* `GET /eth/v1/beacon/light_client/bootstrap` — already exists, needs to be made spec-compliant.

All endpoints support JSON and SSZ response formats with fork `version` metadata.

### P2P Networking

Two new gossip topics broadcast updates to subscribed light clients:

* `light_client_finality_update`
* `light_client_optimistic_update`

Req/Resp RPC methods allow peers to request light client data on demand:

* `LightClientBootstrap`
* `LightClientUpdatesByRange`
* `LightClientFinalityUpdate`
* `LightClientOptimisticUpdate`

## Roadmap

| Phase | Weeks | Deliverables |
|-------|-------|--------------|
| 1: Spec Layer & Data Generation | 5–7 | Implement the missing light client update types and schemas with fork-aware generation logic for active forks Altair through Gloas. |
| 2: Store & Collection Wiring | 8–10 | Build the store, implement best-update selection logic, and hook up runtime collection to block-import and finalization events. |
| 3: REST API Completion | 11–14 | Complete the remaining endpoints to serve update data, supporting both JSON and SSZ formats. |
| 3.5: Persistence | 15–16 | Persist the best updates in the database and reload them on startup. |
| 4: P2P Gossip & Req/Resp Networking | 17–19 | Implement gossip managers to broadcast finality and optimistic updates to the network, and complete the remaining P2P Req/Resp RPC handlers. |
| 5: Testing & Fixing | 19–20 | Address issues, bugs, and client optimizations identified during interop testing. |

## Possible Challenges

* Electra and Gloas shifts generalized indices for sync committees and finality proofs, requiring epoch-aware dispatch.
* Historical data may not always be available on pruned or minimal nodes, which requires evaluation of the storage approach to ensure updates can be served reliably across restarts.
* The `is_better_update` predicate has non-obvious comparison tiebreakers that require exhaustive testing.
* The earlier work ([#6384](https://github.com/Consensys-Incorporated/teku/pull/6384)) surfaced a caching issue in how light client data is held and reused, so the storage and caching approach needs careful evaluation to serve updates efficiently without regressions.
* Checkpoint synced nodes start without the full block history, so building the bootstrap at the anchor and the finality branch of early updates needs special handling.

## Goal of the Project

Success looks like:
* Teku beacon nodes serve all four light client REST API endpoints with correct, spec-compliant data.
* Teku gossips `LightClientFinalityUpdate` and `LightClientOptimisticUpdate` to the P2P network.
* Teku responds to all four light client req/resp queries from peers.
* Implementation passes all relevant consensus-spec-tests and interoperates with light clients syncing from other beacon node implementations.
* Unit, reference, REST integration, and networking tests cover the full scope.

## Collaborators

### Fellows

* [Abhivansh](https://github.com/akronim26)

### Mentors

* [Paul Harris](https://github.com/rolfyone)

## Resources

* [Altair Light Client Spec](https://ethereum.github.io/consensus-specs/altair/light-client/sync-protocol/)
* [Capella Light Client Spec](https://ethereum.github.io/consensus-specs/capella/light-client/sync-protocol/)
* [Deneb Light Client Spec](https://ethereum.github.io/consensus-specs/deneb/light-client/sync-protocol/)
* [Electra Light Client Spec](https://ethereum.github.io/consensus-specs/electra/light-client/sync-protocol/)
* [Gloas Light Client Spec](https://ethereum.github.io/consensus-specs/gloas/light-client/sync-protocol/)
* [Teku Consensus Client Repository](https://github.com/Consensys-Incorporated/teku)
* [Teku Issue #4230 - Light Client Implementation Issue](https://github.com/Consensys-Incorporated/teku/issues/4230)
* [Teku Pull Request #6384 - Partial implementation of Light Client Protocol](https://github.com/Consensys-Incorporated/teku/pull/6384)
