## Delegated Execution Sharding (DES) Project Proposal

Scalable, adaptive, lean, and resourceful native execution-sharding for the ethereum network

## Motivation

The rollup-centric-roadmap has provided short-term scalability for ethereum while simultaneously facilitating both data, memory, and execution-layer innovations and a widening of the experimentation and innovation of new ways of considering what constitutes ethereum. As a long-term scalability solution, however, the role of rollups is tenuous at best, and the notion that ideological and computational satellite networks outside of the technical and philosophical purview of mainnet are necessary to onboard the world is increasingly worrying.

Enshrining a more native form of data, memory, and execution sharding would solve these philosophical dilemmas while bringing first-class efficient and optimized scaling to ethereum mainnet, such that ethereum can accommodate the scale that is necessary while simultaneously remaining aligned with mainnet self-sovereignty.

## Project description

To achieve L1-native execution sharding, a number of architectural features are necessary in my eyes:

* A mechanism for coordinating execution committees in parallel to execute throughput
* Some form of delegated execution tree for organizing execution committees
* Some kind of native sequencing of coordinated user-actions
* An implementation of attestor-proposer-separation

Some other features that would or could be desirable for optimal scaling are as follows:

* State sharding
* New kind of more parallelizable state and/or execution
* State expiry
* SNARK/STARK-based CL signature aggregation

I'll be exploring how all of these features play a role, different variations and implementations of DES, and how various other proposals such as both based sequencing and native rollups can be interpreted in the context of a potential DES implementation.

I'll also do statistical research into the viability of a DES implementation by analyzing factors such as block-level parallelism and, bandwidth requirements, proving latency, and network synchronicity,

## Specification

There are a number of variations of delegated execution sharding I will be specifying, exploring the trade-off space and various optimizations of each specification; the variations I will be exploring include:

* The naive general case that limits the number of execution committees and maintains assumptions such as replicated state and memory
* The special-purpose case that uses new state constructs such as UTXOs to implement separate 'execution channels' in order to execute specific kinds of throughput in parallel
* The more rollup-centric case where based sequencing and native rollups are used to implement weakly-adaptive execution shards where both execution and proving is delegated from mainnet, but without the realtime fluidity of strongly-adaptive execution trees.
* The optimal general case where throughput is theoretically unbounded (and thus the number execution committees also), and where state and mempool sharding is used to optimize the potential throughput further. 

I will specify viable implementations for all four of these cases, while utilizing empirical data from mainnet and rollup execution to prove their viability, but also the various trade-offs of each kind of proposal.
## Roadmap

**Week 5-7:**
* Specify an implementation of the naive general case, with quantitative analysis of its technical overhead, various trade-offs and scalability improvements
  
**Week 8-10**:
* Specify an implementation of the special-purpose case, with analysis of technical overhead, trade-offs, and scalability improvements
  
**Week 11-13**:
* Specify an implementation of the rollup-centric case, with analysis of its differences to the current model, technical overhead, trade-offs, scalability improvements
  
**Week 14-16**:
* Specify an implementation of the theoretically optimal general case, with analysis of its technical overhead, trade-offs, major bottlenecks, theoretically optimal scalability improvements, and potential solutions to the various challenges
  
**Week 17-21**:
* Reflections, finishing touches, and potential work overflow buffer
  
## Possible challenges

There are many technical challenges implicit in trying to implement this form of execution sharding:

* Bandwidth overhead stemming from intra-committee communication of sub-blocks
* Proving overhead stemming from the large amount of zkEVM recursive proving and aggregation occurring within the execution tree
* Asynchronity within the memory layer leading to an adequate propagation of users' messages, meaning that the main benefits of parallelism cannot be achieved

Incorporating additional architectural changes such as state sharding adds a number of additional complexities that create more technical overhead for operating the chain. I general, the main challenge of DES is ensuring that the overhead of coordinating the execution sharding itself does not outweigh the benefits.

## Goal of the project

My goal is to practically and concretely specify a set of viable DES implementations while exploring their scalability gains and trade-off-space. I plan on going in-depth into the various potential implementations I previously outlined while pulling from real chain-data and technical realities to explore their viability.

## Resources

Lean Execution: a holistic approach to secure, efficient, adaptive, and resourceful execution throughput to scale the world-computer:
https://ethresear.ch/t/lean-execution-a-holistic-approach-to-secure-efficient-adaptive-and-resourceful-execution-throughput-to-scale-the-world-computer/25374

Why Homogenizing the Execution of the World-Computer Beats Scaling Through Fragmentation:
https://ethresear.ch/t/why-homogenizing-the-execution-of-the-world-computer-beats-scaling-through-fragmentation/24860

