Scalable, adaptive, lean, and resourceful native execution-sharding for the ethereum network

## Motivation

Rollups are not optimized for efficient load-balancing due to the requirement for social coordination, and the security trade-offs and loss of decentralization stemming from the need for a centralized sequencer and prover brings into question the notion that rollups scale "Ethereum" at all. What is needed as a scalability solution that incorporates the benefits of parallel horizontal scale while remaining bound by the technical decentralization of the L1, and keeping the coordination of this parallel scale seamless to the user and native to mainnet itself.

## Project description

Delegated Execution Sharding (DES) scales Ethereum mainnet by inheriting the benefits of parallel processing that rollups do while keeping load-balancing native to mainnet itself and coupling the coordination of the entire execution pipeline to the L1. 

Practical Delegated Execution Sharding (PDES) integrates this paradigm within the current EVM model by working around issues stemming from non-deterministic EVM execution paths and load balancing using sharded mempools and recursive zkEVM aggregation.

## Specification

To keep the scope of this project light, my PDES proof-of-concept will follow the following architecture:

* ***Virtual mempool:*** network nodes use recursive STARK aggregation to synthesize a "virtual mempool," a kind of sharded mempool structure that allows nodes to coordinate load-balancing of transactions ad-hoc on the networking layer by deterministically partitioning transactions into distinct *execution columns*.

* ***Execution columns:*** the *execution committee* for slot N will gossip their virtual mempool during slot N-2 in order to synchronize the state of their virtual mempools; once the underlying transactions are synced they can all deterministically partition them into distinct columns of parallel execution.

* ***Execution committee:*** the execution committee is the set of multiple proposers (with one main proposer) that synchronize their virtual mempools, and then, within slot N-1, execute and prove the computation that they've been randomly delegated from a distinct execution column. 

* ***Sub-blocks:*** once the execution committee's individual proposers have executed the execution columns' transactions and created a zkEVM validity proof, they package it into a sub-block that they broadcast to the main proposer who creates one aggregate proof broadcasted at the beginning of slot N.
## Roadmap

**Week 5-12:**
* Initial research and high-level architectural specification in place with rough implementation parameters in place.
**Week 12-15**:
* Underlying simulations and results determined based on set implementation parameters.
**Week 16-18**:
* Analysis of results, security trade-offs and limitations of PDES proposal.
**Week 19-21**:
* Explore potential other architectures and ways in which the simplicity of PDES can be extended and reworked from the ground up.
## Possible challenges

There are many technical challenges PDES faces:

* zkEVM proving times not being optimized enough for the complexities of PDES.
* The overhead of synchronizing local mempool views being too high and too asynchronous.
* The incentives not being robust within the execution committees.
* The game theory devolving down to an economy of scale to produce higher amounts of throughput in a centralized manner.

Managing other scaling burdens such as state growth and data bandwidth also pose problems to any scaling solution that only increases the capacity for execution.

## Goal of the project

My goal is to specify a simplistic DES architecture to test and analyze its assumptions, while also using it as a proof-of-concept for the viability of the underlying paradigm. I'd also like to explore a more high-level overview of the DES paradigm, while also exploring other potential implementations.

## Resources

Lean Execution: a holistic approach to secure, efficient, adaptive, and resourceful execution throughput to scale the world-computer:
https://ethresear.ch/t/lean-execution-a-holistic-approach-to-secure-efficient-adaptive-and-resourceful-execution-throughput-to-scale-the-world-computer/25374

Why Homogenizing the Execution of the World-Computer Beats Scaling Through Fragmentation:
https://ethresear.ch/t/why-homogenizing-the-execution-of-the-world-computer-beats-scaling-through-fragmentation/24860

