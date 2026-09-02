# Reth: Partial Statefulness and State Expiry Prototype

A partial-state mode for Reth that stores all account records while retaining storage and bytecode only for selected contracts.

## Motivation

Ethereum full nodes store the complete live execution state. As the state grows, disk usage, synchronization costs, and the hardware required to run a node increase.

The Geth partial-state prototype reports that storing all accounts while skipping contract storage and bytecode when no contracts are tracked can reduce disk usage from approximately ~640 GiB to ~59 GiB.

This project explores how to implement a similar prototype in Reth. Reth has different provider, database, and sync abstractions, so the implementation cannot be directly ported from Geth.

The main correctness requirement is:
> State that was not downloaded must not be interpreted as empty state.

## Project Description

The project will add an experimental partial-state mode to Reth. In this mode, Reth will download and store account records for all accounts. A configurable contract filter will then determine which contracts should have their storage and bytecode downloaded.

For tracked contracts, the node will retain: the account record, contract storage and contract bytecode. For untracked contracts, the node will retain the account record but skip its storage and bytecode.

The prototype will also make skipped state explicit. For example, requesting storage for an untracked contract should return an unavailable state error rather than zero.

The first success target is a local demonstration with:

- one normal full-state Reth node;
- one partial-state Reth node;
- one configured tracked contract.


The partial-state node should store every account, retain the tracked contract's storage and bytecode, reject requests for unavailable state, and use less disk space than the full-state node.

[EIP-7928](https://eips.ethereum.org/EIPS/eip-7928) Block-Level Access Lists may later be explored for maintaining retained state after the initial sync. BAL processing, reorg handling, txpool behaviour, and Engine API support are not part of the minimum project scope.

## Specification

**1. Configuration and Contract Filtering**

Complete the existing partial-state configuration in:

```
crates/node/core/src/args/partial_state.rs
crates/cli/commands/src/node.rs`
```

The configuration will support:
- enabling partial-state mode;
- providing tracked contract address;
- loading tracked addresses from a JSON file;
- displaying the active retention policy at startup.

The existing contract-filter abstraction will determine whether storage and bytecode should be retained for an account.

Initial filter implementations include:
```
ContractFilter
ConfiguredContractFilter
AllowAllContractFilter
```

**Deliverable**: Reth starts with a partial-state configuration and correctly identifies tracked and untracked contracts.


**2. Partial Snap Storage**

Complete the partial-state storage interfaces in:

```
crates/storage/storage-api/src/partial.rs
crates/storage/provider/src/partial_snap.rs
```

The partial snap writer will persist:
- all downloaded account records;
- storage belonging to tracked contracts;
- bytecode belonging to tracked contracts;
- sync progress required to resume the download.

The implementation will reuse Reth’s existing provider and database abstractions.

**Deliverable**: tests can persist all accounts while selectively persisting storage and bytecode.

**3. Selective Snap Downloading**

Complete the partial snap downloader and connect it to Reth's snap request path.

For each downloaded account, the downloader will:
- persist the account record;
- apply the configured contract filter;
- request storage when the contract is tracked;
- request the bytecode when the contract is tracked;
- skip storage and bytecode requests when it is untracked;
- record downloaded and skipped state.

Progress reporting will include:

```
accounts_downloaded
storage_slots_downloaded
storage_slots_skipped
bytecodes_downloaded
bytecodes_skipped
```

**Deliverable**: a local partial sync downloads every account while retaining detailed state only for configured contracts.

**4. Unavailable-State Handling**

In a full-state node, a missing db value may be interpreted as an empty value. That assumption is unsafe in partial-state mode because the value may simply not have been downloaded. The prototype will introduce explicit errors for unavailable storage and bytecode, for example:

```
StorageUnavailable(address, slot)
BytecodeUnavailable(address)
```

The first RPC methods covered will be:
```
eth_getStorageAt;
eth_getCode.
```

For retained state, these methods will behave normally.

For intentionally skipped state:
`eth_getStorageAt` must not return zero;
`eth_getCode` must not return empty bytecode;

They will instead return a clear unavailable-state error.

Support for `eth_call`, `eth_estimateGas`, txpool validation, and Engine API execution will only be considered after the initial provider and RPC behaviour works.

**Delivarable**: tests demonstrate that empty state and unavailable state produce different results.

**5. Measurement**

The prototype will be tested on a controlled local network.

The comparison will include:
- a normal full-state Reth node;
- a partial-state node tracking no contracts;
- a partial-state node tracking one or more contracts.

The project will measure:
- db size
- downloaded storage slots
- skipped storage slots
- downloaded bytecode
- skipped bytecode
- sync time

**Delivarable**: a short report comparing full-state and partial-state sync.


## Current Progress

The initial partial-state sync prototype is working.

Completed so far:
- added partial-state config and contract filters;
- implemented initial snap downloading and provider-backed persistence;
- selectively retained storage and bytecode for tracked contracts;
- added sync progress logs and skipped-state counters;
- tested the flow in Kurtosis and added automated snap-serving tests.

The next step is to validate the prototype using tracked and untracked contracts with real state changes.

### Roadmap

**Weeks 1–8 — Initial Prototype**
- implement config, filters, snap downloading, and persistence;
- add progress logging and automated network tests;
- demonstrate the flow in a two-node Kurtosis network.

**Deliverable**: working partial-state sync prototype.

**Weeks 9–12 — Selective Retention Validation**
- test tracked and untracked contracts;
- verify which storage and bytecode are retained or skipped;
- improve error handling and sync reliability

**Deliverable**: reproducible selective-retention test.

**Weeks 13–16 — Unavailable-State Handling**
- distinguish unavailable storage and bytecode from empty state
- update `eth_getStorageAt` and `eth_getCode`.
- add RPC and provider tests.

**Deliverable**: explicit unavailable-state behaviour.

**Weeks 17–21 — Measurement and Documentation**
- compare full-state and partial-state db sizes;
- collect sync metrics;
- document limitations and follow-up work;

**Deliverable**: tested prototype and measurement report.


## Possible Challenges

* Reth’s provider and sync architecture differs from Geth, so the prototype requires a Reth-native design.
* Snap sync may assume that all storage and bytecode are eventually downloaded.
* Missing state must not be treated as empty state.
* A partial-state node may not be able to reconstruct the full state root locally.
* Maintaining state after sync may require BALs, witnesses, or additional state requests.

## Goal of the Project

### Minimum Success

* synchronize all account records;
* retain storage and bytecode only for configured contracts;
* report downloaded and skipped state;
* persist retained state through Reth providers;
* distinguish unavailable state from empty state in `eth_getStorageAt` and `eth_getCode`;
* demonstrate the prototype on a local network;
* compare its database size with full-state Reth.

### Strong Success

* resume sync from a saved checkpoint;
* test multiple contract configurations;
* propagate unavailable-state errors into `eth_call`;

### Stretch Work

* BAL-based state maintenance;
* shallow-reorg handling;
* txpool and Engine API behaviour;
* on-demand retrieval of missing state.

## Collaborators

### Fellow

* [Ifeoluwa Oderinde](https://github.com/owanikin)

### Mentor

* Reth mentor — to be confirmed

## Resources

* [Reth repository](https://github.com/paradigmxyz/reth)
* [Geth partial-state prototype](https://github.com/ethereum/go-ethereum/pull/33764)
* [EIP-7928: Block-Level Access Lists](https://eips.ethereum.org/EIPS/eip-7928)
* [Validity-Only Partial Statelessness](https://ethresear.ch/t/a-pragmatic-path-towards-validity-only-partial-statelessness-vops/22236)
* [Local Reth branch](https://github.com/paradigmxyz/reth/compare/main...owanikin:reth:owanikin/partial-statefulness-and-state-expiry)
