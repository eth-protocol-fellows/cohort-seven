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

Ongoing wishlist of ideas for tools that help with development and testing of the protocol. 

https://github.com/ethpandaops/tooling-wishlist

### RIG Opened Problems

By Barnabé Monnot

Explore Robus Incentives Group Opened Problems. Most relevant for EPF are tagged https://efdn.notion.site/ROPs-RIG-Open-Problems-c11382c213f949a4b89927ef4e962adf

## Proposed by client teams

Client teams will submit cohort 7 project ideas via pull request to this document. If you represent a client or research team and want to propose a project, open a PR adding a section here or reach out to the cohort organizers on Discord.

### Erigon: aglean — Lean client permutations and stability

By Giulio Rebuffo / Erigon

[aglean](https://github.com/erigontech/aglean) is a Python Lean Ethereum consensus client together with an AI-driven cross-language generation framework. Today the repository already has a working Python implementation, fixture verification, and a live local-devnet sync path. The next step is to develop it further into a stronger EPF project by adding actual implementation permutations and improving overall stability.

This project is about turning aglean from a one-shot client-porting demo into a more serious client-diversity engine. In particular, the work could include:

- defining real interchangeable subspec permutations instead of a single baseline implementation,
- improving stability and reproducibility of generated clients and verification runs,
- strengthening fixture and devnet-based test harnesses for correctness and interoperability,
- and making it easier to try architectural alternatives without rewriting the entire client.

Examples of such alternatives include different storage backends, sync strategies, and other subsystem-level design choices that can expand the permutation space while keeping the same protocol behavior.

This project is a good fit for fellows interested in Lean Ethereum, client diversity, testing infrastructure, and AI-assisted client implementation.

