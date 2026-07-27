# Time-series analysis on execution states

Understanding Ethereum's execution state evolution through fine-grained and time-series analysis to support future state management proposals.

## Motivation

Ethereum is moving toward higher throughput by increasing gas limit ([100+M gas/block](https://blog.ethereum.org/2026/02/18/protocol-priorities-update-2026)) and blob parameters ([EIP-7732]((https://eips.ethereum.org/EIPS/eip-7732))). At the same time, Ethereum community has been explored the mechanism to slow down state growth and reduce hardware requirements for node operators. Recent proposals such as [EIP-8037](https://eips.ethereum.org/EIPS/eip-8037), [EIP-8188](https://eips.ethereum.org/EIPS/eip-8188), [EIP-8295](https://eips.ethereum.org/EIPS/eip-8295), and [EIP-8296](https://eips.ethereum.org/EIPS/eip-8296) introduce new approaches to state management, including state tiering based on the last written block. These proposals require a better understanding of how Ethereum state evolves in practice.

This project aims to provied comprehensive characterization of execution states by extending prior work with fine-grained access analysis and time-series measurements. The results can serve as empirical references for ongoing discussions around state tiering, state expiry, statelessness, and future storage-related protocol upgrades.

## Project description

This project consists of two major parts.

**The first half** reproduces the analysis presented in [*Not All State is Equal*](https://ethereum-magicians.org/t/not-all-state-is-equal/25508). Reproducing the existing results establishes a reliable analysis pipeline and validates the methodology before extending it.

**The second half** expands the analysis in several directions:
1. Separating read access patterns from write access patterns, dividing read-heavy states and read-and-write heavy states.
2. Analyzing dependencies among storage owners, contract factories, deployment templates, and deployers to understand which entities dominate Ethereum state evolution.
3. Investigating different activity spans (if exists) between stateless and stateful smart contracts
4. Performing time-series analysis of newly created Ethereum states to identify long-term trends, seasonal behavior, and periods of intensive state growth.

Together, these analyses aim to provide a more complete picture of how Ethereum state evolves over time.

## Specification

The project will be implemented using a local Ethereum archive node together with custom analysis tooling.

### First half (H1)
Reimplement the analysis pipeline from the previous work.

- Deploy and synchronize an Ethereum archive node
- Write some codes to configure database from the node
- Rediscover the result and analyze the result

Specifically, the analysis aimed to reproduce are:
| # | Title | Description |
| :---: | :---: | :---: |
| 1 | Portion of template-driven contracts | Portion of template vs. Unique bytecodes |
| 2 | Portion of factory-generated contracts | Portion of non-factory vs. factory-individual vs. factory-multi |
| 3 | Storage occupation by deployer address | Distribution of storage slots by deployer address |
| 4 | Smart contracts owned by deployer address | Distribution of deployed smart contracts owned by deployer address |
| 5 | Smart contract sized by bytecode| Distribution of deployed smart contracts categorized by its bytecode size |
| 6 | Stateful vs. Stateless contracts| Portion of stateful vs. stateless smart contracts|
| 7 | Storage slot access count| Distribution of numder of access count for each storage slots|

</br>

### Second half (H2)

Second half includes new analysis built on the pipeline from the first half.

#### Access-age vs. Write-age
Prior work analyzed access patterns (read and write) as a whole. Specifically, storage slots are assessed by access time, rather than write-time. (Refer to the 7th row of above table, or [original post](https://ethereum-magicians.org/t/not-all-state-is-equal/25508#p-62266-how-actively-used-are-storage-slots-10))

To make granular result, we will produce
- Storage slot access pattern by access count and write count
    - Calculate the access count by reading each transactions inside the block, and mark it as 'written' by comparing execution state before and after the block 

#### Dependency among the states
The prior work showed extremely skewed distribution of Ethereum states.

To be specific,
- **Top 1000** smart contracts hold **~51(%)** of total storage slots
- **Top 5** factory contracts account for **~43(%)** of all deployed smart contracts
- **Top 10** template bytecode cover **~51(%)** of all deployed smart contracts
- **Top 500** deployers alone are responsible for **~57.4(%)** of all smart contracts
 
This implies that execution state is highly dependent on few storage slot owners, factory/template creators, and contract deployers. We will try to identify those addresses using public dataset and manual back-tracking.

As a result, we can label the addresses and see which labels are dominant across periods.

#### Activity spans of stateless and stateful contracts
Prior work showed very interesting result on activity span comparison of stateful vs. stateless smart contracts. (*Figure 19* in [prior work](https://ethereum-magicians.org/t/not-all-state-is-equal/25508))
- Stateless contracts show narrow and longer average activity span
- Stateful contracts show broad and shorter average activity span

The deeper analysis is yet exist, for example, the reason for different activity spans.

#### Time series Analysis of newly created states
This will be the main analysis for the project.

In [EIP-8037](https://eips.ethereum.org/EIPS/eip-8037), they measured weekly growth of Ethereum states as a whole. This lacks trend, seasonality, or cycle information of the state growths. Additionally, since significant portion of accounts are active for only few blocks, we want to know when do these accounts are generated intensively.

By conducting time series analysis, we can see which is the 'hot topics' in specific period, which season is the most storage-demanding, and when does 'zero-activity' accounts are concentrated.

Research question we want to solve:
- Is there a seasonality in Ethereum state evolution? If so, what is it?
- How many 'one-block active' accounts are ever been revisited? Is there a correlation with specific period?
- What distinctive cycles exist during the history of Ethereum?

By walking through these questions, we can get temporal context on state growth of Ethereum. For example, we can estimate the expected storage demand for upcoming events or issues. From there, we can estimate the gas price and ephemeral storage.

## Roadmap

The project is divided into four milestones.

### Phase 1: Setup and H1 analysis (Week 6 - 11)

The first milestone focuses on establishing a reliable analysis pipeline.

- Deploy and synchronize an Ethereum archive node
- Build the database and preprocessing pipeline
- Reimplement the methodology from *Not All State is Equal*
- Reproduce the published figures and validate the results

**Deliverable**: A reproducible analysis pipeline together with the reproduced results from the prior work.

### Phase 2: Fine-grained state analysis (Week 12 - 15)

Once the reproduction is completed, the pipeline will be extended to collect additional information that is not covered in the prior work.

- Separate read access and write access patterns
- Measure access-age and write-age distributions
- Analyze activity spans of stateless and stateful contracts
- Investigate dependency among storage owners, deployers, factory contracts, and template bytecodes

**Deliverable**: New datasets and statistical analyses describing fine-grained characteristics of Ethereum execution states.

### Phase 3: Time-series analysis (Week 16 - Week 20)

Using the datasets collected in the previous milestones, temporal analyses will be performed.

- Measure historical evolution of Ethereum execution states
- Identify trends, seasonality, and recurring cycles
- Analyze periods of intensive state creation
- Study revisit patterns of short-lived accounts and storage slots

**Deliverable**: A comprehensive time-series characterization of Ethereum execution state evolution.

### Phase 4: Documentation and publication (Week 21+)

The final milestone focuses on making the work reproducible and useful for the community.

- Organize analysis code and documentation
- Publish datasets and figures
- Summarize findings and discuss implications for ongoing state management proposals

**Deliverable**: Public repository containing source code, datasets, and a final report.

## Possible challenges

I already set up an synced archive node. Possible engineering challenge can be understanding a codebase of the node's implementation to know where to collect data. I will mostly discuss with AI in this part.

Another challenge is accurately identifying write operations at the storage-slot level. Unlike access information, write information must be inferred by comparing execution states before and after transaction execution, which may require additional preprocessing and optimization.

Some analyses also require identifying the real-world entities behind deployers, factory contracts, or template bytecodes. While many major protocols can be identified using public datasets and manual inspection, a portion of addresses will likely remain unlabeled.


## Goal of the project

The project will be considered successful if it:

- reproduces the key findings of previous Ethereum state analysis,
- provides new measurements on write-age and access-age distributions,
- characterizes temporal evolution of Ethereum state through time-series analysis,
- identifies important dependency structures that influence state growth,
- releases reproducible code and documentation for the Ethereum research community.

Ultimately, the project aims to provide empirical evidence that can support future discussions on state tiering, state expiry, and other Ethereum storage optimization proposals.

## Collaborators

### Fellows 

Solo project by [Aiden](https://github.com/sgtSong)

### Mentors

I have no mentors helping me out so far.

## Resources

- Discussions on state-expiry - https://ethereum-magicians.org/tag/state-expiry
- *Not All State Is Equal* - https://ethereum-magicians.org/t/not-all-state-is-equal/25508
- Existing implementation - https://github.com/weiihann/ethereum-state-analysis
- EIP-8037 - https://eips.ethereum.org/EIPS/eip-8037
- EIP-8188 - https://eips.ethereum.org/EIPS/eip-8188
- EIP-8295 - https://eips.ethereum.org/EIPS/eip-8295
- EIP-8296 - https://eips.ethereum.org/EIPS/eip-8296