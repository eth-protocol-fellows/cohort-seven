# Full Fulu Compatibility and Cross-Client Interoperability for Ream

End-to-end beacon-chain, PeerDAS, custody, and data-availability interoperability for the Ream consensus client.

## Motivation

The [**Fulu upgrade**](https://ethereum.org/roadmap/fusaka/) introduces PeerDAS, allowing nodes to store and serve only a portion of the blob data instead of requiring every node to store the full data. It also introduces deterministic proposer lookahead and configurable blob schedules, requiring updates across beacon state transition, block validation, proposer selection, and consensus-layer networking.

A Fulu-compatible client must support both beacon-chain and PeerDAS interoperability: processing blocks, attestations, and fork-choice updates while also validating, gossiping, requesting, reconstructing, and serving data columns.

This project began with the [**Ream: Minimal PeerDAS Data Availability Client proposal**](https://github.com/eth-protocol-fellows/cohort-seven/blob/main/projects/project-ideas.md). After taking a closer look at Ream, we found that its beacon client still has several gaps and mismatches with the consensus specifications, as well as limited end-to-end and cross-client test coverage. E.g: even running two Ream nodes together does not yet work reliably and still exposes errors.

This project therefore aims to complete Ream’s beacon-client implementation, add full PeerDAS support, and make Ream interoperable with mature clients such as Lighthouse and Prysm.

## Project description

We will implement and validate Fulu interoperability in Ream in six phases:

1. **Fulu specification gap analysis**: compare Ream's beacon-client implementation against the Fulu consensus specifications, identify missing or incorrect functionality which were described on spec, and produce a concrete implementation and testing backlog.

2. **Full-custody PeerDAS and DA decoupling**: implement a PeerDAS-compliant full-custody mode in which Ream stores and serves the complete blob dataset as data columns. Decouple data-availability responsibilities from the beacon-chain core behind explicit interfaces for column validation, storage, retrieval, reconstruction, serving, and availability tracking.

3. **Multi-node Ream beacon network**: implement end-to-end tests for networks containing multiple Ream beacon nodes, identify and fix protocol and implementation issues, and demonstrate that the nodes can reliably process blocks, agree on the canonical chain, and reach finality.

4. **Kurtosis and cross-client interoperability**: integrate Ream into a Kurtosis-based Ethereum network and run it alongside mature consensus clients such as Lighthouse and Prysm. Use these networks to identify, report, and fix incompatibilities across genesis, peering, synchronization, block and attestation processing, fork choice, and finality.

5. **PeerDAS custody mode**: extend the full-custody implementation to support configurable custody groups. A Ream node must derive its assignments and retrieve, verify, persist, reconstruct, and serve the columns required by its configured custody mode.

6. **Standalone DA node (optional)**: if Ream still benefits from a separate DA client after the beacon and DA boundaries are stabilized, extract the data-availability subsystem into a standalone process or integrate it with related standalone DA work.

## Specification

The current audit and implementation work are tracked in [ReamLabs/ream#1478](https://github.com/ReamLabs/ream/issues/1478).

| Area                                   | Current Ream baseline                                                                                                                                                                                                                                                                                                    | Work in this project                                                                                                                                                                                                                                                                                                              |
| -------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Beacon state and fork choice           | Fulu epoch processing was updated in [#1463](https://github.com/ReamLabs/ream/issues/1463), but node-level coverage is still limited.                                                                                                                                                                                    | Check proposer lookahead, blob schedules, block processing, and data-availability gating against the Fulu [beacon-chain](https://github.com/ethereum/consensus-specs/blob/master/specs/fulu/beacon-chain.md) and [fork-choice](https://github.com/ethereum/consensus-specs/blob/master/specs/fulu/fork-choice.md) specifications. |
| Column construction and reconstruction | The validator helpers for deriving columns from a block and an existing sidecar landed in [#1453](https://github.com/ReamLabs/ream/issues/1453) and [#1454](https://github.com/ReamLabs/ream/issues/1454). Matrix reconstruction and its spec tests were fixed in [#1449](https://github.com/ReamLabs/ream/issues/1449). | Connect these helpers to block production, column arrival, storage, and reconstruction instead of leaving them as isolated functions.                                                                                                                                                                                             |
| PeerDAS networking                     | Ream has data-column gossip validation and handlers for `DataColumnSidecarsByRoot` and `DataColumnSidecarsByRange`, but the proposer and synchronization paths do not yet exercise the complete flow.                                                                                                                    | Publish columns produced for a block, actively request columns missed on gossip, and connect range backfill and serving to the column store according to the Fulu [P2P specification](https://github.com/ethereum/consensus-specs/blob/master/specs/fulu/p2p-interface.md).                                                       |
| Custody                                | `get_validators_custody_requirement` was implemented in [#1460](https://github.com/ReamLabs/ream/issues/1460); configurable custody is not yet wired through discovery, subnet subscriptions, and storage.                                                                                                               | Start with full custody, then derive custody groups and make the advertised custody count, subscribed subnets, stored columns, and served columns agree.                                                                                                                                                                          |

### Beacon node behavior

Passing isolated **consensus-spec tests** is not enough to show that the beacon node works. The end-to-end harness introduced in [#1492](https://github.com/ReamLabs/ream/pull/1492) will first establish startup and peer connectivity, then add block propagation, head convergence, and finality. For example, [#1481](https://github.com/ReamLabs/ream/pull/1481) adds `test_beacon_nodes_produce_blocks_and_converge`, which runs two beacon nodes with two validator nodes and checks that the network produces blocks and reaches finality.

Later cases cover checkpoint synchronization, late joiners, restarts, and the Fulu fork boundary, including blob-parameter-only schedule changes. The tests compare the nodes' head and finalized checkpoint so failures can be traced to state transition, fork choice, synchronization, or networking.

### Column production, gossip, and retrieval

When a proposer receives blobs and KZG proofs for a block, Ream will use `get_data_column_sidecars_from_block` to construct the `DataColumnSidecar`s and publish each sidecar on its `data_column_sidecar_{subnet_id}` topic. Ream already has an incoming validation path; it will be checked against the ordered `p2p-interface.md` rules, including duplicate detection, subnet mapping, the signed block-header check, the commitment inclusion proof, and `verify_cell_kzg_proof_batch`. Only an accepted sidecar may enter the column store or update availability.

Gossip is the fast path, not the only path. If a required column does not arrive, the node will request it with `DataColumnSidecarsByRoot`. Range and checkpoint synchronization will use `DataColumnSidecarsByRange` to backfill columns that are still inside the serving window. The request path derives a peer's custody groups from its node ID and advertised custody count, validates every response before storage, and reports missing data to the availability checker instead of treating the block as available.

### Availability and reconstruction

Fulu changes `is_data_available` to operate on data-column sidecars. The [current gap analysis](https://hackmd.io/@perfogic/SJhswTmzzx) records Ream's check as local-storage-only, without an active fetch-on-miss path. This project will separate that check from block import. A block whose required columns are missing will remain pending. Arrival of a valid column will update the block's availability state and retry the import only when the node's custody requirement has been satisfied.

The [DAS core specification](https://github.com/ethereum/consensus-specs/blob/master/specs/fulu/das-core.md) says a node should reconstruct once it has at least 50% of the columns. Reconstruction is performed per blob: `recover_matrix` groups the available cells by row and calls `recover_cells_and_kzg_proofs` for each row. Cells received from peers must be verified against the block's commitments before they become reconstruction inputs; recovered cells and proofs are persisted and cross-seeded only after recovery succeeds. Tests will cover incomplete inputs, invalid cells and proofs, duplicate columns, successful recovery, and the case where reconstruction finishes after the corresponding block has already entered the pending queue.

### Custody and the DA boundary

The first PeerDAS milestone uses full custody so that every Ream node stores and serves all columns. This keeps custody selection out of the first interoperability test while the production, gossip, request/response, reconstruction, and availability paths are being debugged.

Configurable custody is added after the full-custody path works. The node will derive its groups with `get_custody_groups`, map them to columns with `compute_columns_for_custody_group`, advertise the same custody-group count in its ENR `cgc` field and `MetaDataV3`, and subscribe to the corresponding column subnets. Sync and pruning must preserve the columns the node has committed to serve.

Column verification, storage, retrieval, and reconstruction will sit behind a boundary used by the beacon node's availability checker. The beacon side retains pending blocks and decides when they may enter fork choice; the column backend reports which required data is available. If that boundary proves stable, the backend can later be reused by the optional standalone DA node tracked in [#1408](https://github.com/ReamLabs/ream/issues/1408).

### Interoperability testing

Following the [Fulu PeerDAS devnet process](https://notes.ethereum.org/@ethpandaops/peerdas-devnet-6), the same-client network is the entry gate for mixed-client testing. The [Kurtosis integration](https://github.com/ReamLabs/ream/issues/1445) will then run Ream with Lighthouse and Prysm. Beacon interoperability passes when the clients peer, synchronize, converge on the same head, and finalize. PeerDAS interoperability additionally requires cross-client data-column gossip and successful `DataColumnSidecarsByRoot` and `DataColumnSidecarsByRange` exchanges, including a late joiner that must backfill columns.

## Roadmap

The phases are ordered by dependency, with some overlap between testing, interoperability, and PeerDAS implementation.

**Phase 1 - Fulu specification gap analysis (weeks 0-4; completed prior work)**

- Compare Ream with the Fulu consensus specifications.
- Identify missing or incorrect beacon-client functionality.
- Create an implementation and testing backlog.

**Phase 2 - Full-custody PeerDAS and DA decoupling (weeks 5-9)**

- Store and serve the complete blob dataset as PeerDAS data columns.
- Implement data-column gossip and request/response protocols.
- Separate DA validation, storage, retrieval, and availability tracking from beacon logic.
- PeerDas compatibility when running Kurtosis on PeerDAS tests.

**Phase 3 - Multi-node Ream beacon network (weeks 6-10) - Parallel with Phase 2**

- Implement end-to-end tests for running multiple Ream beacon nodes.
- Sync Ream beacon nodes on Sepolia and Ethereum mainnet to expose bugs in live block synchronization, state processing, and networking.
- Identify and fix synchronization, block-processing, mis-matched in old specs and networking issues.
- Ensure the network can reliably produce and finalize blocks.

**Phase 4 - Kurtosis and cross-client interoperability (weeks 11-14)**

- Integrate and launch Ream with Kurtosis.
- Run Ream alongside Lighthouse and Prysm.
- Identify and fix beacon-chain and networking incompatibilities.

**Phase 5 - PeerDAS custody mode (weeks 15-18)**

- Implement custody-group calculation and subnet management.
- Support configurable custody requirements.
- Test column retrieval, verification, reconstruction, storage, and serving across clients.

**Phase 6 - Hardening and standalone DA node (weeks 18-21+, stretch)**

- Fix remaining interoperability issues and add metrics and documentation.
- Making sure everything run smooth with Prysm and Lighthouse.
- Work on making a seperate standalone DA node if needed (Based on Ream's roadmap) - this is 100% optional.

## Possible challenges

- Ream may have undocumented gaps or mismatches with the Fulu specifications.
- Cross-client failures can be difficult to reproduce and may depend on timing, networking, or configuration.
- Beacon-chain compatibility does not guarantee PeerDAS interoperability, which has separate gossip, custody, and request/response requirements.
- Decoupling DA from consensus-critical beacon logic must handle missing data, restarts, and partial failures safely.
- Full custody and reconstruction may require significant CPU, memory, bandwidth, and storage.

## Goal of the project

The goal is a Fulu-compatible Ream beacon client that can operate reliably in Ream-only and mixed-client networks.

Success criteria:

1. Multiple Ream nodes can synchronize, produce blocks, and reach finality.
2. Ream interoperates with Lighthouse and Prysm through Kurtosis.
3. Ream exchanges beacon-chain messages and PeerDAS data columns with other clients.
4. Ream supports both full-custody and configurable custody modes.
5. End-to-end tests cover beacon-chain and PeerDAS interoperability.
6. The DA subsystem is decoupled from the beacon-chain core and can support a future standalone DA node.

This establishes Ream as a Fulu-compatible client ready to participate in multi-client devnets.

## Collaborators

### Fellows

- **Daniel Pham** ([@perfogic](https://github.com/perfogic)) — currently full-node PeerDAS compatibility, currently decoupling data-availability logic from the beacon node and expanding end-to-end coverage for block production and checkpoint finality.
- **Tosin** ([@tosynthegeek](https://github.com/tosynthegeek)) — currently Kurtosis and mixed-client testing, currently running Ream nodes through Kurtosis to identify the first interoperability failures and specification mismatches.
- **Hans Vuong** ([@vuonghuuhung](https://github.com/vuonghuuhung)) — previously cross-workstream support, currently syncing a Ream beacon node on Sepolia and documenting bugs in block synchronization and processing.

These are current focus areas rather than fixed ownership boundaries. All three fellows contribute across workstreams: Daniel also works on Kurtosis testing and cross-client testing, while Tosin and Hans also contribute to PeerDAS and beacon-node tasks. Work will be rebalanced as dependencies and new bugs emerge.

### Mentors

- [Shariq](https://github.com/shariqnaiyer)
- [Kolby](https://github.com/KolbyML)

## Resources

- [Ream repository](https://github.com/ReamLabs/ream)
- [Ethereum consensus specifications](https://github.com/ethereum/consensus-specs)
- [Fulu consensus specifications](https://github.com/ethereum/consensus-specs/tree/master/specs/fulu)
- [PeerDAS specification](https://github.com/ethereum/consensus-specs/tree/master/specs/fulu)
- [Ethereum consensus networking specifications](https://github.com/ethereum/consensus-specs/tree/master/specs/phase0/p2p-interface.md)
- [Ethereum Kurtosis package](https://github.com/ethpandaops/ethereum-package)
- [Lighthouse](https://github.com/sigp/lighthouse)
- [Prysm](https://github.com/OffchainLabs/prysm)
- [Ream: Minimal PeerDAS Data Availability Client project idea](./project-ideas.md#ream-minimal-peerdas-data-availability-client)
