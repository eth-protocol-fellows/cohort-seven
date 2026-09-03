# Time-series analysis on execution states

Analysis of temporal trends in Ethereum state growth, specifically the read/write patterns and activity span of each state.

## Motivation

Ethereum is moving toward higher throughput by increasing gas limit ([100+M gas/block](https://blog.ethereum.org/2026/02/18/protocol-priorities-update-2026)) and blob parameters ([EIP-7732](https://eips.ethereum.org/EIPS/eip-7732)). At the same time, Ethereum community has been exploring the mechanism to slow down state growth and reduce hardware requirements for node operators. Recent proposals such as state tiering ([EIP-8037](https://eips.ethereum.org/EIPS/eip-8037), [EIP-8188](https://eips.ethereum.org/EIPS/eip-8188), [EIP-8295](https://eips.ethereum.org/EIPS/eip-8295), and [EIP-8296](https://eips.ethereum.org/EIPS/eip-8296)), or new trie structures ([EIP-8297](https://eips.ethereum.org/EIPS/eip-8297)) also require a better understanding of how Ethereum state evolves in the wild.

This project aims to provide insights on how Ethereum states have changed by analyzing the historical data. Specifically, we will provide analysis on how read-heavy and write-heavy states, dead storage, and access on publicly known contracts have evoloved and how their seasonality or trend has been changed.
The results can serve as empirical references for ongoing discussions around state tiering, partitioned-binary tree, and future storage-related protocol upgrades.

## Project description

This project consists of two major parts.

**The first half** reproduces the analysis presented in [*Not All State is Equal*](https://ethereum-magicians.org/t/not-all-state-is-equal/25508). Reproducing the existing results establishes a reliable analysis pipeline and validates the methodology before extending it.

**The second half** expands to time-series analysis. We will analyze the temporal changes in :
1. Read-access count and write-access count of each state (smart contracts, storage slots). We will identify periods when read-heavy or read-and-write-heavy states are dominant.
2. Number of dead states (i.e. states that have not been active for over 1 year). We can also divide it into read-age and write-age for deeper understanding.
3. Access count of unique contracts/storage slots [identified in prior work](https://ethresear.ch/t/the-anatomy-of-ethereum-s-state-access/25317#p-60995-who-sits-at-the-top-19). This will show the access trends across the period.

We will discover long-term trends, seasonal behavior, and periods of intensive state growth. For the rest of the time, we will propose improvements to state-tiering ([EIP-8295](https://eips.ethereum.org/EIPS/eip-8295)) and partitioned binary tree ([EIP-8297](https://eips.ethereum.org/EIPS/eip-8297)).

## Specification

These are the research questions we want to explore:
- Is there a seasonality in Ethereum state evolution? If so, what is it?
- How many 'one-block active' accounts have ever been revisited? Is there a correlation during specific periods?
- What distinctive cycles exist during the history of Ethereum?

### Data Collection
The data will be collected using a local Ethereum archive node together with publicly accessible data.
- Network data will be primarily collected from [Xatu dataset](https://github.com/ethpandaops/xatu-data/tree/master) (e.g. `canonical_execution_storage_diffs` for write counts, `canonical_execution_storage_reads` for read counts). To store data from the EthPandaOps ClickHouse database, we will set up Clickhouse database locally.
- From archive node, we will collect recent state data that is not covered by Xatu dataset, by tracing state changes for each block using Erigon trace APIs.
-  To identify the contracts, we will use labels from [prior work](https://ethresear.ch/t/the-anatomy-of-ethereum-s-state-access/25317#p-60995-who-sits-at-the-top-19) by weiihann, and public datasets like [Eth-labels](https://github.com/dawsbot/eth-labels) and [Kaggle](https://www.kaggle.com/datasets/hamishhall/labelled-ethereum-addresses).

### First half (H1)

Produce an analysis pipeline based on the previous work.

- Deploy and synchronize an Ethereum archive node
- Set up Clickhouse database and write scripts for data collection
- Reproduce and validate the results from [*Not All State is Equal*](https://ethereum-magicians.org/t/not-all-state-is-equal/25508)

Specifically, the analyses to be reproduced are:
| # | Title | Description |
| :---: | :---: | :---: |
| 1 | Portion of template-driven contracts | Portion of template vs. Unique bytecodes |
| 2 | Portion of factory-generated contracts | Portion of non-factory vs. factory-individual vs. factory-multi |
| 3 | Storage occupation by deployer address | Distribution of storage slots by deployer address |
| 4 | Smart contracts owned by deployer address | Distribution of deployed smart contracts owned by deployer address |
| 5 | Smart contract sized by bytecode| Distribution of deployed smart contracts categorized by its bytecode size |
| 6 | Stateful vs. Stateless contracts| Portion of stateful vs. stateless smart contracts|
| 7 | Storage slot access count| Distribution of number of access count for each storage slots|

</br>

### Second half (H2)
Second half includes new time-series analysis built on the pipeline from the first half.

#### Access-heavy vs. Write-heavy
For read and write counts, Erigon JSON-RPC method supports `trace_replayTransaction`, `trace_replayBlockTransactions`, and `trace_block`. From there, we can get opcode-level traces (e.g. CALL, SLOAD, SSTORE, SELFDESTRUCT, CREATE) for each storage slot. In database, we will store the number of SLOAD and SSTORE of each storage slot for read and write, respectively.

To count the read/write number of each state, accumulate the SLOAD and SSTORE count after each block. This will show the growing speed for each access type.

#### Number of dead states
We first divide two different types of ages: read-age, write-age. Each age is calculated as follows:
```
read_age = current_block_number - last_read_block_number
```
Similarly,
```
write_age = current_block_number - last_written_block_number
```
Since write operation automatically requires read access to the state, we can also assume that `read_age = access_age`.

To measure the read/write age of each state, update the last_read_block_number/last_written_block_number after each block. We define 'dead state' as *last_read_age > 2,618,875 (~ 1 year)*. To calculate the ratio of 'dead state', we can divide the number of dead states by the total number of explicit states.

If we want to show age distribution, instead we can divide the age range into :
- 0–1 day
- 1–7 days
- 1 week–1 month
- 1–6 months
- 6–12 months
- 1–2 years
- \>2 years

Furthermore, we can measure 'revisit' pattern using these datasets. We can set the threshold such that `revisit = True` if `gap = last_read_block - second_last_read_block > threshold`.

#### Access of unique address
There are public identifiers for some Ethereum addresses ([The Anatomy of Ethereum’s State Access](https://ethresear.ch/t/the-anatomy-of-ethereum-s-state-access/25317#p-60995-who-sits-at-the-top-19), [Eth-labels](https://github.com/dawsbot/eth-labels), and [Kaggle](https://www.kaggle.com/datasets/hamishhall/labelled-ethereum-addresses)). Using these we can categorize each address into labels (e.g. token contract, DEX, builder address).

For each label, we can measure total read/write access count. We can analyze which services account for the majority of transaction accesses. From there, we can analyze the trend of the state access across the period.


</br>

Overall, we will discover the cycle, seasonality, or trend from these time-dimensional changes.

After that, we can give a guide for ongoing research on state-tiering ([EIP-8295](https://eips.ethereum.org/EIPS/eip-8295)) and partitioned binary tree ([EIP-8297](https://eips.ethereum.org/EIPS/eip-8297)). These proposals are closely related to the state structure, so insightful analysis can help consolidating the upgrades.

## Roadmap

The project is divided into four milestones.

### Phase 1: Setup and H1 analysis (Week 7 - 10)

The first milestone focuses on establishing a reliable analysis pipeline.

- Deploy and synchronize an Ethereum archive node
- Build the database and pipeline
- Reproduce the published figures and validate the results from *Not All State is Equal*

**Deliverable**: A reproducible analysis pipeline together with the reproduced results from the prior work.

### Phase 2: First half of H2 analysis (Week 11 - 14)

Once the reproduction is completed, we will move on to time-series analysis.

- Access-heavy vs. Write-heavy state analysis
- Number of dead states and revisit pattern analysis

**Deliverable**: A comprehensive time-series characterization of Ethereum execution state evolution.
### Phase 3: Second half of H2 analysis & Provide suggestions (Week 15 - Week 18)

The rest of the H2 analyses will be covered for Phase 3. Also, we will explore EIP-8295 and EIP-8297. From there, we will derive suggestions how to incorporate temporal trend of state evolution.

**Deliverable**: Suggestions for ongoing research

### Phase 4: Documentation and publication (Week 19+)

The final milestone focuses on making the work reproducible and useful for the community.

- Organize analysis code and documentation
- Publish datasets and figures
- Summarize findings and discuss implications for ongoing state management proposals

**Deliverable**: Public repository containing source code, datasets, and a final report.

## Possible challenges

I have already set up a synced archive node. Possible engineering challenge will be understanding the codebase of the node's implementation to know where to collect data. I will go through Erigon APIs to trace the state change, which can take some time.

Another challenge is accurately identifying write operations at the storage-slot level. Unlike access information, write information must be inferred by comparing execution states before and after transaction execution, which may require additional preprocessing and optimization.

Some analyses also require identifying the real-world entities behind deployers, factory contracts, or template bytecodes. While many major protocols can be identified using public datasets and manual inspection, a portion of addresses will likely remain unlabeled.


## Goal of the project

The project will be considered successful if it:

- reproduces the key findings of previous Ethereum state analysis,
- discovers new trend in Ethereum state evolution (e.g. seasonality)
- provides meaningful guide for bleeding edge research

## Collaborators

### Fellows 

Solo project by Aiden ([@sgtSong](https://github.com/sgtSong))

### Mentors

Ng Wei Han ([@weiihann](https://github.com/weiihann))

## Resources

- Discussions on state-expiry - https://ethereum-magicians.org/tag/state-expiry
- *Not All State Is Equal* - https://ethereum-magicians.org/t/not-all-state-is-equal/25508
- Existing implementation - https://github.com/weiihann/ethereum-state-analysis
- The Anatomy of Ethereum's state access - https://ethresear.ch/t/the-anatomy-of-ethereum-s-state-access/25317
- EIP-8037 - https://eips.ethereum.org/EIPS/eip-8037
- EIP-8188 - https://eips.ethereum.org/EIPS/eip-8188
- EIP-8295 - https://eips.ethereum.org/EIPS/eip-8295
- EIP-8296 - https://eips.ethereum.org/EIPS/eip-8296
- EIP-8297 - https://eips.ethereum.org/EIPS/eip-8297