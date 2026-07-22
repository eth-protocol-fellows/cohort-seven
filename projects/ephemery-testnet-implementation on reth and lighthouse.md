# Native Ephemery Testnet Support for Reth and Lighthouse

Enabling a native Ephemery testnet integration for Reth and Lighthouse

## Motivation

[Ephemery](https://github.com/ephemery-testnet/ephemery-resources) is an automatically resetting testnet that provides an alternative environment for short-term testing of applications, validators, and breaking changes in client implementations.

Ephemery testnet is important because by design it avoids issues like state bloating, unlike other testnets such as [Sepolia](https://sepolia.dev/) and [Holesky](https://holesky.dev/) that constantly accumulate state values, and run into multiple validator issues that make it harder for new stakers to test their setups.

The problem that Ephemery testnet is facing today is that validators that run Ephemery on [Reth](https://github.com/paradigmxyz/reth)/[Lighthouse](https://github.com/sigp/lighthouse) have to rely on manual configuration to set up, and there is no native integration of it on these clients. This project will solve this by integrating [EIP-6916](https://eips.ethereum.org/EIPS/eip-6916) natively into these clients.

## Project description

This project implements native Ephemery support in [Reth](https://github.com/paradigmxyz/reth) and [Lighthouse](https://github.com/sigp/lighthouse).

Prior contributor WIP PRs exist for both clients but the implementation is still incomplete and this project picks up that work and completes it.  Specifically, the clients will be able to:

- Deterministically generate the correct genesis state for the current network iteration
- Detect when the 28-day period has expired
- Automatically reset and regenerate a new genesis
- Sync to the latest Ephemery chain without manual intervention

This project will bring the Ephemery experience on these clients in line with what is already being done on other client pairs like Besu/Teku.

## Specification

The implementation follows the specification defined in [EIP-6916](https://eips.ethereum.org/EIPS/eip-6916) and the [Ephemery testnet specs](https://github.com/ephemery-testnet/ephemery-resources/blob/master/specs.md). The specification will be validated and possibly improved based on new discoveries during the implementation process.

There are two levels of client support for Ephemery:

- **Basic support** requires the client to determine the current network specs and generate the correct genesis state. This enables connecting to the network for short-term testing.
- **Full support** means the client can also follow the reset process and always sync the latest chain, even during a reset. This involves detecting when the 28-day period has expired, cleaning the database, regenerating a new genesis, and rediscovering peers on the new network.

This project targets full support for both [Reth](https://github.com/paradigmxyz/reth) and [Lighthouse](https://github.com/sigp/lighthouse). Both clients will expose Ephemery as a network option through a simple flag, and all genesis generation and reset logic will be handled internally without external tooling.

## Roadmap

- **Weeks 1-5 (June - mid July):** Diving deep into the Reth and Lighthouse codebases, Studying the chainspec, storage, CLI, and networking layers of both clients. Reviewing prior Ephemery-related WIP PRs and identifying what needs to be done.

- **Weeks 6-10 (mid July - late August):** Implementing the genesis function for both Reth and Lighthouse. This will cover the deterministic genesis generation and exposing Ephemery as a network option through a CLI flag. We will be targeting our first PR by last week of August.

- **Weeks 11-17 (September - mid October):** Implementing the reset function for both clients. This is the harder part, it covers period expiry detection, database cleanup, genesis regeneration, and peer rediscovery. We target the  reset PR with tests to be pushed by second to third week of October.

- **Weeks 18-19 (late October - first week of November):** Review feedback from the clients team, fix issues, and iterate on the PRs until they are ready to merge.

## Possible challenges

- Both codebases are very large and complex, which makes it challenging to fully understand all the parts that need to change and how they interact with each other.

- Reth and Lighthouse have very different architectures, so the implementation approach will differ between the two, some solutions that work on one client may not work on the other.

- Testing in the reset phase might be tricky since the 28-day cycle means we have to wait for almost a month to test.

- The reset touches multiple parts of each client, so the changes need to be carefully done to avoid breaking core functionality in the clients.

## Goal of the project

This project is considered successful when:

- Reth and Lighthouse can both run the Ephemery testnet natively through a simple CLI flag without any manual configuration.
- Both clients can generate the correct genesis state for the current network iteration.
- Both clients can detect when the 28-day period has expired, clean their databases, regenerate a new genesis, and resync automatically.
- Peer discovery is properly handled during resets, discarding old peers and rediscovering new ones on the new network.
- Full coverage test added for both clients.
- The implementation validates and aligns with the [EIP-6916](https://eips.ethereum.org/EIPS/eip-6916) specification.

## Collaborators

### Fellows 

[Isaac Akhigbe](https://github.com/isaac-akhigbe)

[Mary Odior](https://github.com/maryodior)

### Mentors

[Mario Havel](https://github.com/taxmeifyoucan) 

## Resources

- [EIP-6916: Ephemery Testnet Specification](https://eips.ethereum.org/EIPS/eip-6916)
- [Ephemery Testnet Specs](https://github.com/ephemery-testnet/ephemery-resources/blob/master/specs.md)
- [Ephemery Resources Repository](https://github.com/ephemery-testnet/ephemery-resources)
- [Reth Ephemery Issue](https://github.com/paradigmxyz/reth/issues/4340)
- [Lighthouse Ephemery Issue](https://github.com/sigp/lighthouse/issues/4664)