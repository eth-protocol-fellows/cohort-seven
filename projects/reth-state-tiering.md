# State Tiering using EIP-8188 and EIP-8295 for Reth 

Implementing Ethereum state tiering through `last_written_block` and `last_written_period` signal to allow for efficient state management.

## Motivation

Ethereum's state is ever evolving. Being one of the most used cryptocurrencies worldwide, there is a constant influx of account data, storage slot data and multiple other things. According to the article "[Not All State is Equal](https://ethereum-magicians.org/t/not-all-state-is-equal/25508)" which contains a detailed analysis of the Ethereum state from **genesis** block to block **22,431,083**, the Ethereum state is very uneven: most contracts go inactive right after creation, a majority of storage slots are never touched again after their first write, and a small number of contracts and factories account for most of the state. Out of the full node state around **80%** has been unaccessed for over a year ([EIP-8188 ACD Slides](https://weiihann.github.io/20260521-eip8188-acd/))

This proposal intends to form the baseline for splitting the Ethereum state into a "hot" and "cold" one based on a write-age signal. In other words, "cold" state would be states which have not had a write operation in a long time and "hot" state would be those which have had a recent write operation. This allows individual clients to optimize the commit-critical mutable path, say things like trie updates, state-root computation and other work associated with repeatedly rewriting state.

## Project description

This project contains a complete implmentation of the signals described in [EIP-8188](https://eips.ethereum.org/EIPS/eip-8188) and [EIP-8295](https://eips.ethereum.org/EIPS/eip-8295) for Reth following the [Geth Implementation](https://github.com/weiihann/go-ethereum/tree/feat/eip-8188/inject) by the original author. The following diagram contains a simplistic description of the hot-cold state separation.

```
hot state                           cold state
───────────────────────────         ─────────────────────────────────────────
standard MPT nodes (RLP)
+ stubs pointing to           ──→   blobs where each blob = one originally-stubbed subtree
```

The "hot" state is stored in a database where write access is easier and cheaper (RocksDB) and "cold" state is basically a simple append-only flat file. Commands that allow us to backfill the necessary data into the state and tier the states accordingly will be added to the `reth db` suite:

1. `reth db inject-periods` : This command is going to allow us to enter the necessary `period` data, as described in [EIP-8295](https://eips.ethereum.org/EIPS/eip-8295) into the state of the chain. 

2. `reth db convert-inactive` : This command is going to identify the inactive states, convert them into a format writable in the append-only flat file and connect the necessary stubs in the hot state to the blobs in the cold state. This hot-cold separation will ensure the tiering happens appropriately.

Some other support commands may be added as necessary.

## Specification

### Data Collection

To collect the necessary data, we will be using data collected through [Xatu](https://github.com/ethpandaops/xatu) and stored in ClickHouse. The ClickHouse instance allows us to extract necessary data using simple SQL queries.

#### Account Data
    
 To collect all account write events in the specified block range, we will query balance changes table (`canonical_execution_balance_diffs`), nonce updates table (`canonical_execution_nonce_diffs`), storage modifications table (`canonical_execution_storage_diffs`) and contract creations table (`canonical_execution_contracts`). We will then group the results by account address and select the max `block_number` for each account. This value will correspond to the account’s `last_written_block`, following the definition of “Account Modified” as defined under the “Account Rules” section in [EIP-8188](https://eips.ethereum.org/EIPS/eip-8188), since it represents the most recent block in which the account was modified.

#### Storage Slot Data

To collect all necessary storage slot data, we will look at `canonical_execution_storage_diffs` table to find which storage slot changes have taken place within the given range . We will ignore all write where the new value is zero (`0x0`, `0x00`, or `0`), because [EIP-8188](https://eips.ethereum.org/EIPS/eip-8188) only tracks storage slots still allocated after the write. We will then group the results by account address and storage slot, and select the maximum `block_number` for each pair. This value is the storage slot's `last_written_block`, i.e. the latest block in which that non-zero storage slot was modified.

From this `last_written_block` data, `last_written_period` is calculated as follows:

```
last_written_period = max(0, (last_written_block - PERIOD_START_BLOCK) // PERIOD_LENGTH)
```

This definition comes from [EIP-8295](https://eips.ethereum.org/EIPS/eip-8295). The `inject-periods` command described earlier uses these SQL queries to inject the period data into the state.

### Inactive Subtree Identification

The inactive-subtree identification routine finds **maximal inactive subtrees** in the Ethereum state trie according to the inactivity criteria described in [EIP-8188](https://eips.ethereum.org/EIPS/eip-8188). A subtree is considered **inactive** if all accounts (or storage slots, for a storage trie) in the subtree have an age greater than or equal to a user-defined threshold (`inactive_min_age`). The age of an object is determined by the `last_written_period` stored in the [EIP-8295](https://eips.ethereum.org/EIPS/eip-8295) snapshot metadata.

The identification algorithm makes one sequential depth-first-search (DFS) traversal of the account trie, while moving a snapshot iterator over the corresponding metadata. Because both iterators enumerate leaves in the same deterministic order, each account or storage slot is processed exactly once, with no need for random lookups. This makes the algorithm amenable to execution on mainnet-sized state and integration into clients such as Reth.

During traversal, a stack of internal-node frames is maintained. Each frame aggregates the inactivity status of its children in a bottom-up manner. Child subtrees that are completely inactive are temporarily stored as candidates. When an internal node has been fully processed, the algorithm determines whether all of its descendants are inactive. If so, the node itself becomes the representative inactive subtree and its candidate children are discarded. Otherwise, the node is mixed, and all previously collected inactive child candidates are emitted as maximal inactive subtree roots. This guarantees that the output contains the largest possible inactive subtrees without redundancy.

Before the traversal begins, the snapshot is synchronized to the latest committed state by flushing any in-memory or layered state to the persistent snapshot view. This makes sure that any recently modified accounts and storage slots are available to both the snapshot iterator and the trie traversal, thus preventing the stale metadata from having any kind of effect on subtree identification.

**Pseudocode**

```rust
fn identify_inactive_subtrees(trie: Trie, snapshot: Snapshot, min_age: u64) {
    let mut iterator = trie.node_iterator();
    let mut snapshot_iter = snapshot.leaf_iterator();
    let mut stack = Vec::new();

    while let Some(node) = iterator.next() {
        match node {
            Node::Internal => {
                stack.push(Frame::new(node.hash()));
            }

            Node::Leaf(leaf) => {
                let metadata = snapshot_iter.next().unwrap();

                assert_eq!(leaf.key(), metadata.key());

                let inactive =
                    current_period - metadata.last_written_period >= min_age;

                if let Some(parent) = stack.last_mut() {
                    parent.record_leaf(inactive);
                }
            }

            Node::EndInternal => {
                let frame = stack.pop().unwrap();

                if frame.all_children_inactive() {
                    if !frame.is_embedded() {
                        if let Some(parent) = stack.last_mut() {
                            parent.add_candidate(frame.root_hash());
                        } else {
                            emit(frame.root_hash());
                        }
                    }
                } else {
                    for subtree in frame.candidates() {
                        emit(subtree);
                    }

                    if let Some(parent) = stack.last_mut() {
                        parent.record_child_active();
                    }
                }
            }
        }
    }

    for contract in trie.contract_accounts() {
        if contract.storage_root() != EMPTY_ROOT {
            identify_inactive_subtrees(
                contract.storage_trie(),
                snapshot.storage_snapshot(contract.address()),
                min_age,
            );
        }
    }
}
```

After the inactive subtree has been identified, it gets stored in the cold state and a reference for it is stored in the hot state.

### Cold State Format
When a maximal inactive subtree is identified and moved to the cold file, it is replaced in the hot database by a compact 17-byte "primary stub". This stub acts as a filesystem pointer, allowing the client to resolve the data only when needed.

The Primary Stub has the following structure:

```
primary stub
───────────────────────────
marker + blobOffset + rootInBlob + rootSize
```

* Marker (1 Byte): 0x00 -> marker of the database entry as a cold state stub.

* Blob Offset (8 Bytes): Absolute byte position in the append-only file of the beginning of the subtree blob.

* Root in Blob (4 Bytes): Relative offset in this blob entry of the root node of the subtree.

* Root Size (4 Bytes): Byte Size of the root entry.

The actual Cold State File is comprised of concatenated **blobs**, which contain the following structure:

```
blob
───────────────────────────
- header
    versionNumber | reservedBytes | rootOffset | rootSize

- body
    Contains the serialized trie in post-order DFS format
```

**Header (16 bytes)**: The header contains some basic metadata about the blob. It includes the version number which indicates the version number of the blob, some reserved bytes, offset of the root node and size of the root. The offset allows the reader of the file to find the root of the serialized trie.

**Body**: The body of the blob contains the actual trie in post-order DFS format, meaning the children of the trie are written before the parent. Because of this post-order format, the root node becomes the last node of the blob.

**Trie Node Structure**

Each Trie node can be one of two types:

* **Full Node**: This node represents a branch node in the Merkle Patricia Trie. It contains 17 child slots, one for each nibble (`0-15`) and one value slot.

* **Short Node**: This node represents a leaf node. It contains the key part followed by one child slot.

**Child Slot Structure**

Each child slot contains a one-byte **kind** field that describes how the child is encoded. The possible kinds of the child slots include:

* **Empty**: means that there is no child.
* **Hashed Reference**: contains the child's Keccak-256 hash, as well as its offset and size within the blob.
* **Embedded Reference**: only includes the offset and size of the child, as the embedded nodes do not have any independent hashes.
* **Inline Value**: stores the serialized value directly within the node (typically the account or storage value associated with a leaf).

This layout preserves the complete Merkle Patricia Trie structure while allowing inactive subtrees to be stored as self-contained blobs that can later be reconstructed or materialized without requiring access to the active state database.

## Roadmap

| **Phase**                                                 | **Weeks** | **Deliverables**                                                                                                                                                                                                                      |
| --------------------------------------------------------- | --------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **1: Data Collection & Period Generation**                | **6**   | Implementation of the data collection pipeline with Xatu and ClickHouse and calculation of EIP-8188 and EIP-8295 metadata.
| **2: Period Injection (`reth db inject-periods`)**        | **7-8**   | Extension of the state schema with `last_written_period` and implementation of the `reth db inject-periods` command.          |
| **3: Inactive Subtree Identification**                    | **9-11**  | Implementation of the DFS algorithm for detecting inactive subtrees and detection of maximal inactive subtrees.        |
| **4: Cold State Conversion (`reth db convert-inactive`)** | **12-14** | Generation of the blob format and primary stubs, serialization of inactive subtrees into the cold state file and replacement of inactive subtrees in the hot state with stubs |
| **5: Testing, Benchmarking & Optimization**               | **15-18** | Test functionality, benchmark hot & cold states                |
| **(Stretch)**                                             | **18+**   | Extend for Ethrex as well if there is any extra time left.                                                          |


## Possible challenges

* **Uncertainty about Parameters**: [EIP-8188](https://eips.ethereum.org/EIPS/eip-8188) and [EIP-8295](https://eips.ethereum.org/EIPS/eip-8295) are very new EIPs and plenty of constant parameters like `blocks_per_period` and `inactive_min_age` are still TBD. Without those parameters, a lot of sample experimentation is necessary to figure out which are the optimal values for those parameters.

* **Performance at Mainnet Scale**: The inactive-subtree DFS traverses the full account trie, plus every contract's storage trie, in lockstep with a snapshot iterator. Mainnet state has hundreds of millions of accounts and billions of storage slots, so keeping the traversal, iterator synchronization, and internal-node frame stack performant enough to run in reasonable time (and without excessive memory use) is a real risk that will likely require profiling and iteration.

* **Crash Safety of Hot/Cold Conversion**: `reth db convert-inactive` needs to move a subtree out of the hot database into the append-only cold file and replace it with a stub, ideally atomically. If the process is interrupted mid-conversion, the hot and cold states could end up inconsistent (e.g. a stub pointing at a partially-written blob, or a subtree present in both places at once). Designing a safe commit protocol that tolerates crashes/restarts without corrupting state will be non-trivial.

## Goal of the Project

Success looks like:

* A complete implementation of the state tiering primitives described in **EIP-8188** and **EIP-8295** within Reth.
* A working `reth db inject-periods` command that computes and injects `last_written_period` metadata for all accounts and storage slots using data collected from Xatu.
* A working `reth db convert-inactive` command that correctly identifies maximal inactive subtrees, stores them in a proper format in the cold state file, and replaces them in the hot state with primary stubs that reference them in the cold state.
* A cold state format that properly preserves the original Merkle Patricia Trie structure and allows inactive subtrees to be accessed and materialized on demand.

## Collaborators

### Fellows 

[Astrion](https://github.com/astrion-coder)

### Mentors

[Ng Wei Han](https://github.com/weiihann)

## Resources

* [Not All State is Equal](https://ethereum-magicians.org/t/not-all-state-is-equal/25508)
* [EIP-8188: Last-Written Block for Accounts and Slots](https://eips.ethereum.org/EIPS/eip-8188)
* [Geth implementation of State Tiering](https://github.com/weiihann/go-ethereum/tree/feat/eip-8188/inject)
* [Xatu](https://github.com/ethpandaops/xatu)
* [EIP-8188 ACD Slides](https://weiihann.github.io/20260521-eip8188-acd/)