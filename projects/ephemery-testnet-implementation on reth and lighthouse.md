# Native Ephemery Testnet Support for Reth and Lighthouse

Enabling a native Ephemery testnet integration for Reth and Lighthouse

## Motivation

[Ephemery](https://github.com/ephemery-testnet/ephemery-resources) is an automatically resetting testnet that provides an alternative environment for short-term testing of applications, validators, and breaking changes in client implementations.

Ephemery testnet is important because by design it avoids issues like state bloating, unlike other testnets such as [Sepolia](https://sepolia.dev/) and [Holesky](https://holesky.dev/) that constantly accumulate state values and run into validator set issues over time.

The core challenge with Ephemery is that its design requires a new genesis to be generated for every network iteration. Clients like [Reth](https://github.com/paradigmxyz/reth) and [Lighthouse](https://github.com/sigp/lighthouse) cannot do this natively today, which means anyone trying to bootstrap a node has to manually download and configure the correct genesis themselves. This makes the experience significantly worse compared to other testnets where you just pass a network flag and launch. This project will solve this by integrating [EIP-6916](https://eips.ethereum.org/EIPS/eip-6916) natively into these clients.

## Project description

This project implements native [EIP-6916](https://eips.ethereum.org/EIPS/eip-6916) support in [Reth](https://github.com/paradigmxyz/reth) and [Lighthouse](https://github.com/sigp/lighthouse). Currently, anyone trying to run an Ephemery node on these clients has to manually download and configure the correct genesis for each network iteration. We want to change that.

Prior contributor WIP PRs exist for both clients but the implementation is still incomplete. This project picks up that work, identifies what gaps remain, and completes it. The primary goal is native genesis generation, so that both clients can deterministically produce the correct genesis state for any given network iteration and connect to Ephemery by simply passing a flag.

As a stretch goal, the project will also explore implementing the automatic reset function, which involves detecting when the 28-day period has expired, cleaning the database, regenerating a new genesis, and rediscovering peers.

## Specification

The implementation follows the specification defined in [EIP-6916](https://eips.ethereum.org/EIPS/eip-6916) and the [Ephemery testnet specs](https://github.com/ephemery-testnet/ephemery-resources/blob/master/specs.md). There are two levels of client support for Ephemery:

### Basic Support (Genesis Function) - Primary Goal

Basic support requires the client to determine the current network specs and generate the correct genesis. This is enough to participate in the network for short-term testing.

Ephemery starts from a hardcoded `genesis_0` but on each client startup, the client must derive the current genesis rather than using the base one directly. The derivation works as follows:

**Execution Layer (Reth)**

On startup, the client calculates the current period iteration from the base genesis timestamp and the current time:

- `i = int((current_timestamp - genesis_0.timestamp) / period)`
- `chainId = genesis_0.chainId + i`
- `genesis_timestamp = period * i + genesis_0.timestamp`

A genesis.json will be hardcoded into Reth's genesis resources folder alongside other predefined networks. Since most of the genesis content (alloc, contract state, configuration) is constant across all periods, and the only fields that change every iteration are the chainId and the timestamp, on startup, the genesis struct is loaded and run through a function that overwrites these two fields to match the current period. The corrected genesis is what gets passed to the node launcher.

One notable difference is that in Reth, predefined networks are registered as `NamedChain` variants, but Ephemery's chainId changes every period so it cannot be registered this way. Instead, the chain will use `ChainKind::Id` with the dynamically calculated chainId. This means the chain identification, display name, and any logic that depends on chain matching needs to work through the raw ID rather than a named variant.

**Consensus Layer (Lighthouse)**

The CL side similarly requires changing some of the network configuration to be calculated dynamically, in addition to adding the genesis state file for the beacon chain.

At startup, a config update function calculates the current period iteration using the same formula as the EL:

- `i = (current_timestamp - genesis_0_timestamp) / period`
- `depositChainId = genesis_0_chainId + i`
- `depositNetworkId = genesis_0_networkId + i`
- `minGenesisTime = genesis_0_timestamp + (i * period_in_seconds)`

These three fields are overwritten to reflect the current Ephemery period, regardless of what is in the configuration file.

The approach on Lighthouse is to bundle a base `ephemery.yaml` with `genesis_0` values and then run it through an update function at startup that overwrites the `depositChainId`, `depositNetworkId`, and `minGenesisTime` based on the calculated iteration. The genesis.ssz and bootnodes are fetched remotely from `ephemery.dev/latest/` since both change every period and cannot be bundled. A default checkpoint sync URL will also be set so users don't need to provide one. The corrected config and fetched genesis state are then passed to the beacon chain initializer. Before use, the downloaded state is deserialized and validated by checking that fields like `genesis_time` and `depositChainId` match what the local period calculation produced, ensuring that the fetched state is correct and consistent with the current network iteration.

### Full Support (Reset Function) - Secondary Goal

Full support means the client can also follow the reset process and always sync the latest chain, even during a reset. This is needed for running infrastructure, genesis validators, and bootnodes.

When the current timestamp exceeds `genesis_timestamp + period`:

- The network stops accepting new blocks
- The database and state are discarded
- A new genesis is generated using the genesis function above
- The client re-initializes from the new genesis

This is a harder problem since it requires the client to handle a full state rollback without restarting, and client architecture may not support this. This will require research into how each client's storage and initialization layers can accommodate a runtime reset.

### Prior Work and Gaps

Previous fellows contributed PRs to both [Reth](https://github.com/paradigmxyz/reth/pull/5124) and [Lighthouse](https://github.com/sigp/lighthouse/pull/4764) but neither reached completion. The Reth PR had the genesis function mostly working but had broken tests and touched base functionality that needed team review. Since then, Reth's chainspec architecture has changed a lot, so the integration path needs to be reworked from scratch. On the Lighthouse side, the main blocker was the checksum and genesis_validators_root verification during state download. These checks assume static values, but Ephemery changes them every iteration, so they either need to be made optional for Ephemery or handled dynamically. Tests were also incomplete on both sides and the reset function was never attempted.

## Roadmap

- **Weeks 1-5 (June - mid July):** Going through both codebases to understand how the chainspec, storage, and CLI layers work. Reviewing the prior WIP PRs to see what was done and what went wrong. Also setting up Ephemery nodes manually on both clients to get a feel for the current bootstrapping flow.

- **Weeks 6-10 (mid July - late August):** Implementing the genesis function on both clients in parallel.
  - Execution (Reth): Loading the base genesis.json, calculating the current period iteration, updating the chainId and timestamp, registering bootnodes, and adding Ephemery as a CLI network flag.
  - Consensus (Lighthouse): Bundling the base ephemery.yaml, implementing the config update function for depositChainId/depositNetworkId/minGenesisTime, setting up remote fetching for genesis.ssz and bootnodes, configuring a default checkpoint sync URL, and adding the CLI flag.
  - Targeting first PRs for both clients by last week of August.

- **Weeks 11-14 (September - early October):** Testing and validating genesis on both clients.
  - Making sure the EL and CL genesis outputs are compatible and that a node can bootstrap onto Ephemery with just a flag.
  - Adding tests and test fixtures that cover genesis derivation across different period iterations, with CI compatibility.
  - Working through PR feedback from the Reth and Lighthouse maintainers.
  - Beginning feasibility research into the reset function, understanding how each client handles shutdown, database cleanup, and re-initialization internally.

- **Weeks 15-17 (October):** If the reset is feasible, starting a proof of concept on one or both clients. Otherwise, focusing on hardening the genesis implementation and addressing any remaining PR feedback.

- **Weeks 18-19 (late October - first week of November):** Wrapping up remaining review feedback and getting PRs to a mergeable state.

## Possible challenges

- The EL and CL sides of this implementation are quite different. The execution side deals mainly with chainId and timestamp updates, while the consensus side has to handle dynamic config overwrites, remote fetching of genesis.ssz and bootnodes, and working around the checksum and genesis_validators_root verification that assumes static values. What works on one side won't necessarily apply to the other.

- Both codebases are very large and complex, which makes it hard to fully understand all the parts that need to change and how they interact with each other.

- Client architecture may make the reset function not feasible at all. The reset requires a full runtime state rollback without restarting the process, and the clients were not necessarily designed to support this. This is the biggest unknown in the project and why the reset is treated as a stretch goal.

- Testing the reset functionality is tricky since the 28-day cycle means waiting almost a month to test on the live network. A local Ephemery setup with a shorter period would be needed to iterate faster.

## Goal of the project

This project is considered successful when:

- Reth and Lighthouse can both run the Ephemery testnet natively through a simple CLI flag without any manual configuration.
- Both clients can deterministically generate the correct genesis state for any given network iteration.
- The EL and CL genesis outputs are compatible and a full node pair can bootstrap onto Ephemery out of the box.
- Test fixtures and CI-compatible tests are in place covering genesis derivation across different period iterations.
- The implementation aligns with the [EIP-6916](https://eips.ethereum.org/EIPS/eip-6916) specification, and any findings during the process are fed back to improve the spec.

Beyond the core goals, if time permits:

- Both clients can detect when the 28-day period has expired, clean their databases, regenerate a new genesis, and resync automatically.
- Peer discovery is properly handled during resets, discarding old peers and rediscovering new ones on the new network.

## Collaborators

### Fellows

- [Isaac Akhigbe](https://github.com/isaac-akhigbe) - Execution layer (Reth)
- [Mary Odior](https://github.com/maryodior) - Consensus layer (Lighthouse)

### Mentors

[Mario Havel](https://github.com/taxmeifyoucan)

## Resources

- [EIP-6916: Ephemery Testnet Specification](https://eips.ethereum.org/EIPS/eip-6916)
- [Ephemery Testnet Specs](https://github.com/ephemery-testnet/ephemery-resources/blob/master/specs.md)
- [Ephemery Resources Repository](https://github.com/ephemery-testnet/ephemery-resources)
- [Reth Ephemery Issue](https://github.com/paradigmxyz/reth/issues/4340)
- [Lighthouse Ephemery Issue](https://github.com/sigp/lighthouse/issues/4664)