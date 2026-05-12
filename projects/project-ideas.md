# Proposed projects

Project ideas proposed by client teams and mentors. These are projects that fit the EPF scope and would be the most valuable to the team.

If you are interested in the project, take your time to learn about it, the team it's coming from and related area of the protocol. Reach out to client organizers if you have any questions or need connection to a mentor.

## Previous cohorts

In project ideas from previous cohorts, you might find some up-to-date ideas which haven't been solved yet.

- [Project ideas in the sixth cohort](https://github.com/eth-protocol-fellows/cohort-six/blob/master/projects/project-ideas.md), [Projects executed](https://github.com/eth-protocol-fellows/cohort-six/blob/master/projects/)
- [Project ideas in the fifth cohort](https://github.com/eth-protocol-fellows/cohort-five/blob/master/projects/project-ideas.md), [Projects executed](https://github.com/eth-protocol-fellows/cohort-five/blob/master/projects/)
- [Project ideas in the fourth cohort](https://github.com/eth-protocol-fellows/cohort-four/blob/master/projects/project-ideas.md), [Projects executed](https://github.com/eth-protocol-fellows/cohort-four/blob/master/projects/)
- [Project ideas in the third cohort](https://github.com/eth-protocol-fellows/cohort-three/blob/master/projects/project-ideas.md), [Projects executed](https://github.com/eth-protocol-fellows/cohort-three/blob/master/projects/)
- [Project ideas in the second cohort](https://github.com/ethereum-cdap/cohort-zero/issues?q=is%3Aopen+is%3Aissue+label%3A%22help+wanted%22)
- [Project ideas in the first cohort](https://github.com/ethereum-cdap/cohort-one/issues?q=is%3Aissue+Project+idea)

## Ongoing wishlists

These are collections of projects proposed by some teams and might not be fully updated.

### Protocol Security tooling wishlist

By Fredrik

Ongoing wishlist of ideas for tools that will help with securing the protocol.

https://security.ethereum.org/wishlist

### PandaOps tooling wishlist

By Pari

Ongoing wishlist of ideas for tools that help with the development and testing of the protocol.

https://github.com/ethpandaops/tooling-wishlist

### RIG Opened Problems

By Barnabé Monnot

Explore Robus Incentives Group Opened Problems. Most relevant for EPF are tagged https://efdn.notion.site/ROPs-RIG-Open-Problems-c11382c213f949a4b89927ef4e962adf

## Proposed by client teams

Client teams will submit cohort 7 project ideas via pull request to this document. If you represent a client or research team and want to propose a project, open a PR adding a section here or reach out to the cohort organizers on Discord.

### Erigon: aglean — Lean client permutations and stability

By Giulio Rebuffo / Erigon

[aglean](https://github.com/erigontech/aglean) is a Python Lean Ethereum consensus client together with an AI-driven cross-language generation framework. Today the repository already has a working Python implementation, fixture verification, and a live local-devnet sync path. The next step is to develop it further into a stronger EPF project by adding actual implementation permutations and improving overall stability.

This project is about turning Aglean from a one-shot client-porting demo into a more serious client-diversity engine. In particular, the work could include:

- defining real interchangeable subspec permutations instead of a single baseline implementation,
- improving stability and reproducibility of generated clients and verification runs,
- strengthening fixture and devnet-based test harnesses for correctness and interoperability,
- and making it easier to try architectural alternatives without rewriting the entire client.

Examples of such alternatives include different storage backends, sync strategies, and other subsystem-level design choices that can expand the permutation space while keeping the same protocol behavior.

This project is a good fit for fellows interested in Lean Ethereum, client diversity, testing infrastructure, and AI-assisted client implementation.

### Hive: Black Box Interop Testing Post Quantum Lean Ethereum

By Kolby ML and Derek Sorken

The Ream team maintains the shared testing infrastructure for the post-quantum movement for Ethereum (Lean Consensus) at https://hive.leanroadmap.org/, a black-box testing framework that tests post-quantum clients against a common suite of scenarios.

The project would be fully in Rust 🦀 and can be found on https://github.com/ethereum/hive.

Every lean consensus client shipping runs through this testing framework, which means the work you do on this project directly impacts the lean ecosystem as it allows the post quantum effort to confidently scale devnets to larger validator sets and how quickly teams can ship devnets against an evolving Post Quantum Ethereum spec (https://github.com/leanEthereum/leanSpec).

This project would extend the testing infrastructure: adding new scenarios (spec tests, checkpoint tests, signature aggregation interop, state transition cross-checks, key lifecycle edge cases), onboarding new client implementations if required, and improving the framework itself, which involves adding fixtures, runtime errors, handling test failures, and maintaining dashboards that show the state of the clients' performance in the tests.

### Ream: Operating and Scaling PQ Interop Devnets

By Shariq Naiyer and Unnawut

The Ream team wants to operate and scale the post quantum interop devnets, which involves deploying post quantum clients (lean consensus clients) in realistic configurations, running them under load, and observing how they behave together, reporting findings, surfacing bugs from these runs, and fixing them.

Tooling to play around can be found at https://github.com/ReamLabs/leanstart 🦀.


### Ream: Execution Layer Integration for the Lean Chain

By Kolby ML and Shariq Naiyer

The Post Quantum spec is currently consensus-only. Ream's blocks carry only attestations, and there is no way for a wallet to submit a transaction against a lean network.

What made Ethereum different from what came before was that, for the first time, people could write programs backed by the same fundamental principles as Bitcoin. We think it would be cool to show that the same is true under PoC post-quantum testnet, where users can send transactions and deploy smart contracts on a chain finalized in seconds, backed by post-quantum security in Rust 🦀.

This project involves embedding reth (https://github.com/paradigmxyz/reth) directly into the Ream (https://github.com/ReamLabs/ream) binary as a library and driving it from inside the lean stack via reth's Execution Extensions (ExEx) framework. One process, no HTTP bridge between an Execution and Consensus. The success bar is a working "wallet → tx → block → confirmation" demo on a local lean devnet running as a single `ream lean_node` binary with reth embedded.

### Ream: Minimal PeerDAS Data Availability Client

By Shariq Naiyer and Kolby ML

Data Availability is the guarantee that the data behind a transaction has been published somewhere that any node can fetch and verify. Currently, there are only execution and consensus clients, and DA is often bundled with a consensus client. We want to build a dedicated DA client that can plug into a minimal beacon client and a generic interface, so it can also plug into a post-quantum consensus client in Rust 🦀.

This project builds a standalone `ream da-node` binary fully dedicated to the data layer: a Data layer client that custodies and serves the full 128 columns, and does the minimal amount of non-DAS work possible. No fork choice, no execution, no validator, just the data layer, decoupled from any one consensus runtime and runnable alongside a minimal consensus node.

