# Reth: Partial Statefulness and State Expiry Prototype

A partial state mode for Reth that retains all accounts, selectively stores contract storage and bytecode, makes unavailable state explicit, and uses EIP-7928 Block-Level Access Lists as a foundation for maintaining retained state.

## Motivation

Ethereum execution clients currently operate under the assumption that a full node stores the complete live execution state locally. As the state grows, this increases disk requirements, sync costs, hardware requirements, and the long term burden of operating a node.

Research into state expirty and partial statelessness challenges a fundamental client assumption:

> What happens when valid Ethereum state exists globally but is intentionally unavailable to a particular node?

This project explores that question in Reth.

The prototype will introduce a partial-state mode in which Reth retains account information for all accounts while selectively downloading and storing contract and bytecode only for configured contracts or address ranges.

The most important correctness requirement is that unavailable state must not be silently interpreted as empty state.

Reth must be able to distinguish between:
* an account or storage value that is present and empty;
* state that has been downloaded and is locally available;
* state that may exist but was intentionally not retained;
* state tequired for execution but currently unavailable.

This distinctions affects Reth's storage providers, snap synchronization, RPC methods, EVM execution, transaction-pool validation, Engine API behaviour, reorganization handling, and future support for state expiry.
[EIP-7928](https://eips.ethereum.org/EIPS/eip-7928) will also be investigated as a mechanism for identifying state accessed or modified by each block and for maintaining a node's subset after its initial partial-state synchronization.

## Project Description

The project will implement and document a research prototype of partial statefulness in Reth.

The prototype will:
* download account information for all accounts;
* apply a configurable retention policy to each account;
* download storage and bytecode only for tracked contracts;
* persist retained accounts, storage, and bytecode into Reth's database;
* record whether omitted storage and bytecode are unavailable rather than empty;
* propogate state-unavailability errors through provider, RPC, and execution paths;
* ingest and retain EIP-7928 Block-Level Access Lists;
* investigate where BAL data can maintain tracked state after synchronization;
* measure downloaded state, skipped state, database size, and synchronization behavior.

The goal is to build the client-side infrastructure required to experiment with:
* partial state retention;
* state availability 
* state expiry
* selective snap synchronization;
* BAL-driven state maintenance;
* unavailable state RPC semantics;
* the assumption Reth currently makes about complete local state.

For every account, the node retains account-level information such as:
* nonce;
* balance;
* storage root;
* bytecode hash.

For tracked contracts, the node additionally retrieves and stores:
* contract bytecode;
* relevant storage slots.

For untracked contracts, the corresponding storage and bytecode are marked unavailable.

**Scope Summary**

| Area | Role in this project |
| --- | --- |
| Core Project | Add a working partial-state synchronization and availability path to Reth | 
| First success target | Synchronize all accounts on a local network while retaining storage and bytecode for one configured contract |
| Core correctness target | An RPC request for omitted state returns an explicit unavailable-state error instead of zero or empty code |
| Strong-success extension | Use BAL data to update retained state across new canonical blocks and shallow reorganizations | 
| Stretch work | Txpool policy, Engine API behavior, dynamic retention filters, deeper reorganization recovery |

## Research Questions 

**Selective synchronization**

Can Reth's snap synchronization pipeline retrieve all accounts while requesting storage and bytecode only for selected contracts?

**State availability**

How should Reth represent the difference between an empty value and a value that was intentionally not downloaded?

**Provider architecture**

Which Reth storage and provider interfaces currently assume complete local state, and how can those assumptions be made explicit?

**RPC and execution semantics**

How should Reth behave when `eth_call`, `eth_estimateGas`, or transaction validation touches unavailable state?

**State maintenance**

Can EIP-7928 Block-Level Access Lists provide enough information to maintain a configured subset of state after initial synchronization?

**Verification**

Which correctness guarantees can a partial-state node provide locally, and which require proofs, witnesses, BAL extensions, or other protocol support?

**Practical benefit**

How much storage, network traffic, and synchronization work can be avoided under different retention policies?

## Current State Map

| Area | Current state | Remaining Project Work | 
| --- | --- | --- |
| Partial-state CLI | Initial `--partial-state` support added | Finalize options and validation | 
| Contract configuration | Inline addresses and JSON file support started | Persist in `reth.toml` | 
| Contract filters | `ContractFilter`, `ConfiguredContractFilter`, and `AllowAllContractFilter` added | Add snap compatible and future range filters |
| Partial-state types | `PartialState` foundation added | Connect to node lifecycle | 
| BAL history | `BalHistory` foundation added | Persistence, indexing, retention, and reorg integration | 
| Startup visibility | Partial state startup logging added | Expose complete active retention policy | 
| Storage API | Initial partial storage interfaces added | Finalize availability semantics | 
| Partial snap writer | Partial snap writer | Complete database persistence and checkpoints |
| Snap network access | Fetch-client work started | Complete account, storage, code, and trie request paths |
| Partial downloader | Initial implementation started | Filtering, request queues, retry logic, and checkpoints |
| Progress reporting | Initial logging started | Metrics for downloaded and skipped state |
| RPC availability | Not yet integrated | Add explicit errors for unavailable storage and code |
| Execution awareness | Not yet integrated | Propagate unavailable-state failures through `revm` |
| BAL application | Not yet implemented | Filter and apply relevant block updates |
| Reorganization handling | Not yet implemented | Add shallow reorg behavior and deep reorg fallback |
| Devnet validation | Not yet completed | Local and Kurtosis testing |

## Specification

1. **Configuration and Retention Policy**

Partial-state mode will be configurable through CLI flags and `reth.toml`.

Initial configuration will support:
* enabling partial-state mode
* specifying tracked contract addresses inline
* loading tracked addresses from a JSON file;
* configuring BAL-history retention;
* selecting an allow-all policy for comparison and testing.

Example conceptual configuration:

```
[partial-state]
enabled = true
contracts-file = "./tracked-contracts.json"
bal-retention = 128
```

Helper APIs will expose operations such as:

`is_partial_state_enabled`
`is_partial_state_contract_tracked`
`partial_state_bal_retention`

The node will log the active policy during startup so that partial-state operation cannot be mistaken for ordinary full-state operation.

Future filter implementations may support:

* address ranges;
* account-hash ranges;
* storage ranges;
* recently accessed contracts;
* dynamically updated retention sets.

These are not required for the first working prototype.

2. **Contract Filters**

A filter abstraction will determine which state should be retained.

Initial implementations include:

`ContractFilter`
`ConfiguredContractFilter`
`AllowAllContractFilter`

Two forms of filtering may be required:

* address-based filtering for user configuration and RPC behavior;
* hash-based filtering for snap-sync account ranges and database keys.

The filter must be available to:

* the partial snap downloader;
* the partial snap writer;
* provider APIs;
* RPC methods;
* BAL processing;
* execution and transaction-validation paths.

3. **Partial Snap Downloader**

The downloader will request account ranges from compatible snap peers.

For each returned account, it will:

* decode the account;
* persist its account-level information;
* determine whether the account is tracked;
* queue its bytecode request when tracked and code is present;
* queue its storage-range requests when tracked and storage is present;
* skip those requests when the account is untracked;
* update downloaded and skipped counters;
* persist synchronization progress.

The downloader will expose progress information similar to:

```
Syncing: partial state download in progress 
state=... 
accounts=... 
slots=... 
slots_skipped=... 
codes=... 
codes_skipped=...
```

The implementation will track:

* account ranges completed;
* accounts persisted;
* storage requests sent;
* code requests sent;
* storage slots downloaded;
* storage slots skipped;
* bytecode objects downloaded;
* bytecode objects skipped;
* bytes downloaded;
* failed peer requests;
* retry attempts;
* database growth.

4. **Partial Snap Writer**

A provider backed partial snap writer will persist:
* account records;
* selected storage slots;
* selected bytecode;
* synchronization checkpoints;
* availability information required by the prototype.

The first implementation should reuse Reth’s existing provider and database abstractions where possible rather than introducing an independent state database.

The writer must support restart and resume behavior so partial synchronization does not always begin from the first account range.

5. **Explicit State Availability**

Database absence alone cannot indicate whether state is empty.

For example, a missing storage entry could mean:

* the storage slot is zero;
* the slot was not downloaded;
* the node has not yet synchronized the relevant range;
* the state was removed by pruning;
* the underlying database is inconsistent.

The prototype will introduce explicit availability semantics.

A conceptual model is: 

```
enum StateAvailability<T> {
    Available(T),
    AvailableEmpty,
    Unavailable,
}
```

The exact Rust representation may differ according to Reth's provider interfaces.

Potential provider errors include:

`StorageUnavailable(address, slot)`
`BytecodeUnavailable(address, code_hash)`
`AccountStateUnavailable(address)`
`PartialStateSyncIncomplete`

The implementation must preserve these distinctions across storage, provider, RPC, and execution boundaries.

6. **RPC Behavior**

RPC methods that depend only on retained account information should continue to operate normally.

Examples include:

`eth_getBalance`
`eth_getTransactionCount`

Methods requiring potentially omitted state must be partial-state aware.

`eth_getStorageAt`

For tracked and available storage, return the normal value.

For intentionally omitted storage, return an explicit unavailable-state error.

It must not return zero merely because the slot was not stored locally.

`eth_getCode`

For available bytecode, return the normal code.

For an account known to have code whose bytecode was not retained, return an unavailable-state error.

It must not return `0x` as though the account were an externally owned account.

`eth_call`

Return a state-unavailable error if execution touches unavailable:

* bytecode;
* storage;
* nested-call state;
* dependent contract state.

`eth_estimateGas`

Return a state-unavailable error when estimation cannot be completed using retained state.

The prototype will define an internal error taxonomy and map it to clear JSON-RPC errors. The error should communicate that the request failed because of the node’s local retention policy, not because the requested state is empty or the transaction reverted.

7. **Execution Awareness**

The database interface used by `revm` must preserve unavailable-state failures.

The project will identify where Reth currently performs operations equivalent to:

`missing value -> default value`

and determine how those paths should behave under partial-state mode.

The initial execution target is not arbitrary execution over missing state. It is explicit failure when execution requires data that is unavailable.

The project will investigate:

* propagation through revm database traits;
* nested contract calls;
* bytecode loading;
* storage reads;
* state overrides;
* call tracing;
* RPC error conversion.

8. **Transaction-Pool and Engine API Behavior**

Transaction-pool and Engine API integration will be treated as a gated extension after storage, sync, and RPC behavior work reliably.

Questions include:

* Should a transaction requiring unavailable state be rejected?
* Should it be retained but marked locally unverifiable?
* Should the node defer validation until the state is retrieved?
* Can a partial-state node safely participate in local block building?
* How should engine_newPayload behave when execution requires unavailable state?
* Should partial-state mode disable specific validator or builder roles?

The minimum prototype may fail closed and document unsupported paths.

A stronger prototype will add an explicit policy for these cases.

9. **Block-Level Access Lists**

EIP-7928 BALs will be used as the foundation for post-synchronization maintenance experiments.

The implementation will add:

* BAL ingestion;
* block-to-BAL indexing;
* configurable retention;
* persistence across restarts;
* filtering of BAL entries by tracked contract;
* integration with canonical-chain updates;
* shallow-reorganization handling.

The research will determine:

* which account updates can be identified from BAL data;
* which storage changes can be maintained;
* whether the BAL contains sufficient resulting values;
* what additional data must be requested;
* whether proofs or witnesses are needed;
* how newly accessed untracked state should be represented.

BALs will not be treated as a complete solution in advance. Determining their limits is part of the project.

10. **Reorganization Handling**

BAL and update history will be retained for a configurable number of blocks.

For reorganizations within that retention window, the prototype will attempt to:

* identify reverted canonical blocks;
* identify affected tracked accounts and slots;
* revert or reconstruct retained updates where sufficient information exists;
* process BALs from the new canonical branch;
* update the partial-state checkpoint.

For deeper reorganizations, the node may need to:

* re-download tracked state from a new checkpoint;
* restart partial synchronization;
* request additional proofs or witnesses;
* stop and report that safe recovery is not possible locally.

The node must fail explicitly rather than continue with state that it knows may be inconsistent.

## Roadmap

**Weeks 1–5 — Research, Foundations, and Proposal**

Complete the initial research and implementation groundwork:

* study the Geth partial-state prototype and relevant state-expiry research;
* map the corresponding Reth storage, provider, sync, RPC, and execution paths;
* define the initial partial-state architecture and retention model;
* add initial CLI and tracked-contract configuration;
* introduce contract-filter, partial-state, and BAL-history foundation types;
* begin the partial snap writer, network fetch support, downloader, and progress logging;
* define the core prototype boundary;
* establish the first-success target;
* prepare the branch and pull-request strategy;
* separate core, strong, and stretch deliverables.

**Deliverable:** an accepted proposal, documented architecture, initial implementation foundations, and an agreed technical scope.

**Weeks 6–7 — Configuration and Storage Foundations**

Complete:

* `reth.toml` integration;
* contract-filter interfaces;
* helper APIs;
* provider-backed partial snap writer;
* synchronization checkpoints;
* startup logging;
* configuration and writer tests.

**Deliverable:** a node can start with a persisted partial-state policy and persist selected state through Reth providers.

**Weeks 8–11 — Partial Snap Downloader**

Complete:

* account-range requests;
* account decoding;
* tracked-account selection;
* storage request queues;
* bytecode request queues;
* skipped-state accounting;
* retries and checkpoints;
* progress metrics.

**Deliverable:** a standalone downloader can retrieve all accounts while selectively retrieving storage and bytecode.

**Weeks 12–14 — Node Sync Integration**

Integrate the downloader with:

* Reth’s node lifecycle;
* a real target state root;
* network peer selection;
* synchronization progress;
* database persistence;
* restart behavior.

Run the first controlled local-network test.

**Deliverable:** a Reth node performs a partial-state bootstrap against a local devnet.

**Weeks 15–17 — State Availability, RPC, and Execution**

Add:

* explicit availability types;
* provider errors;
* `eth_getStorageAt` behavior;
* `eth_getCode` behavior;
* initial ``eth_call` propagation;
* initial `eth_estimateGas` propagation;
* tracked versus untracked tests.

**Deliverable:** unavailable state is no longer silently interpreted as empty at provider and RPC boundaries.

**Weeks 18–19 — BAL Maintenance and Reorganizations**

Add:

* BAL persistence;
* retention configuration;
* tracked-state filtering;
* canonical-chain processing;
* shallow-reorganization experiments;
* documentation of missing BAL information.

**Deliverable:** at least one demonstrated BAL-based retained-state maintenance path, with its limitations documented.

**Weeks 20–21+ — Devnet Validation**

Run:

* local devnet or Kurtosis tests;
* full-state versus partial-state comparison;
* database-size measurements;
* downloaded-data measurements;
* restart tests;
* RPC correctness tests;
* shallow-reorg tests.

Prepare:

* reproducible runbook;
* final report;
* presentation;
* code-path and architecture documentation;
* follow-up issues.

Deliverable: reproducible prototype and measurement report.

## Testing Strategy

**Unit Tests**

Test:

* configuration parsing;
* address and hash filters;
* tracked and untracked account decisions;
* state-availability values;
* provider error propagation;
* BAL retention;
* checkpoint persistence.

**Integration Tests**

Test:

* account-range persistence;
* selective storage downloading;
* selective bytecode downloading;
* synchronization restart;
* explicit storage-unavailable errors;
* explicit bytecode-unavailable errors;
* nested execution touching unavailable state.

**Devnet Tests**

Run at least:

1. a normal full-state Reth node;
2. a partial-state Reth node tracking no contracts;
3. a partial-state Reth node tracking one contract;
4. a partial-state Reth node tracking several contracts.

Compare:

* database size;
* downloaded bytes;
* synchronization duration;
* storage slots retained;
* bytecode retained;
* RPC behavior;
* restart behavior;
* canonical-block processing.

## Possible Challenges

**Reth’s architecture differs from Geth**

The Geth partial-state prototype is a useful reference, but Reth has different provider, sync, database, static-file, pruning, and pipeline abstractions. The project cannot be implemented as a direct port.

**Snap synchronization assumes eventual completeness**

Parts of snap synchronization may assume that all storage and code will eventually be downloaded. These assumptions must be identified and isolated.

**Missing state may be mistaken for empty state**

This is the main correctness risk. Returning zero storage or empty bytecode for unavailable state could produce incorrect RPC responses and execution results.

**Complete state-root verification may be unavailable**

A node storing only part of the state may not be able to reconstruct the complete state root using its retained database alone.

The prototype must clearly state its trust and verification assumptions.

**BALs may be insufficient by themselves**

BALs may identify accessed or modified state without containing every value, trie node, or proof required to update retained state.

**Execution dependencies are dynamic**

A transaction sent to one tracked contract may call an untracked contract or read an unavailable slot. Retention cannot always be determined from the transaction recipient alone.

**RPC compatibility**

Existing applications may expect standard methods to always return a value. New unavailable-state errors must be clear and stable.

**Reorganization recovery**

Safe rollback may require state values in addition to access information. Deep reorganizations may require re-synchronization.

**Interaction with pruning and storage changes**

Partial-state behavior may interact with:

* pruning;
* static files;
* snapshots;
* trie storage;
* storage v2;
* database compaction.

Some interactions may remain prototype-only during the cohort.

## Goal of the Project

The project is successful if Reth has a working, tested, and documented partial-state prototype.

**Minimum Success**

* Partial-state configuration is persisted and visible at startup.
* The node downloads and stores account information for all accounts.
* Storage and bytecode are downloaded only for configured contracts.
* Downloaded and skipped state are reported through logs or metrics.
* Retained data is persisted through Reth’s provider interfaces.
`eth_getStorageAt` and `eth_getCode` distinguish unavailable state from empty state.
* The prototype runs on a controlled local network.
* Architectural assumptions and unresolved correctness questions are documented.

**Strong Success**

* Partial synchronization is integrated into Reth’s node sync lifecycle.
* Synchronization can restart from a persisted checkpoint.
* `eth_call` and `eth_estimateGas` propagate unavailable-state failures.
* BAL history is persisted and filtered for tracked contracts.
* At least one retained-state update path is driven by BAL information.
* Shallow-reorganization behavior is demonstrated.
* Full-state and partial-state nodes are compared using disk, bandwidth, and synchronization measurements.
* One or more focused PRs are merged or under review.

**Stretch Success**

* Explicit transaction-pool policy for locally unverifiable transactions.
* Engine API behavior for unavailable execution state.
* On-demand retrieval of missing tracked state.
* Dynamic or recently-accessed contract filters.
* Address-range or hash-range retention policies.
* More complete BAL-based rollback and reorganization recovery.
* Interoperation with an external partial-state implementation.
* Wider Kurtosis or research-devnet testing.

## Deliverables

The project will produce:

* partial-state CLI and reth.toml configuration;
* configurable contract filters;
* a provider-backed partial snap writer;
* a partial snap downloader;
* partial sync integration;
* explicit state-availability types and errors;
* partial-state-aware RPC behavior;
* initial execution-awareness support;
* BAL-history storage and processing;
* shallow-reorganization experiments;
* synchronization and storage metrics;
* local devnet or Kurtosis configurations;
* a full-state versus partial-state benchmark;
* architecture and limitation documentation;
* a final report and demonstration.

## Collaborators

**Fellow**

* [Ifeoluwa Oderinde](https://github.com/owanikin)

**Mentors**
Reth mentor — to be confirmed

## Resources
[Reth repository](https://github.com/paradigmxyz/reth)
[EIP-7928: Block-Level Access Lists](https://eips.ethereum.org/EIPS/eip-7928)
[Validity-Only Partial Statelessness](https://ethresear.ch/t/a-pragmatic-path-towards-validity-only-partial-statelessness-vops/22236)
[Geth partial-state prototype](https://github.com/ethereum/go-ethereum/pull/33764)
[Local Reth branch / pull request](https://github.com/paradigmxyz/reth/compare/main...owanikin:reth:owanikin/partial-statefulness-and-state-expiry)

