# EIP-8304: Trustless Log and Transaction Index for Nethermind

Verifiable log lookups for the Nethermind execution client that allows light clients to prove that a query result is correct and complete without trusting the RPC provider.

## Motivation

Ethereum logs are the primary mechanism through which smart contracts expose events to offchain consumers. Wallets reconstruct token balances from `Transfer` events, bridges relay crosschain messages via log proofs, and indexing services like The Graph build entire query layers on top of `eth_getLogs`. But unlike account state where a Merkle proof against `stateRoot` can authenticate any storage slot, there is no protocol level mechanism to prove that a log query result is both genuine and complete.

The original per-block bloom filters embedded in block headers were designed for this purpose, but they are now severely oversaturated on mainnet. A bloom filter for a typical block matches nearly every address, making it effectively useless for filtering. Worse, even a working bloom filter can only prove that a block might contain matching logs, it cannot prove completeness across a block range, and it cannot prove that zero logs were omitted from a result set.

This creates a trust problem which is users who cannot afford to run a full node will have to trust their RPC provider to execute `eth_getLogs` honestly. The provider could omit results, return stale data, or silently filter events. There is no cryptographic recourse.

EIP-8304 addresses this by introducing a hierarchical index table structure whose roots are committed to a system contract during block processing. Since these roots become part of the world state, anyone who trusts the state root (obtainable from the consensus layer) can verify log query proofs end to end. The system contract pattern follows the precedent established by EIP-4788 (beacon block roots) and EIP-2935 (historical block hashes).

This project implements EIP-8304 in the Nethermind execution client from index table generation during block processing, through system contract integration and persistent storage, to a proof-serving RPC endpoint and an accompanying verification library.

### Why Nethermind

Nethermind is the second-most used Ethereum execution client. Client diversity is a protocol level safety property, if only one client implements a consensus change, a bug in that client becomes a consensus failure. EIP-8304 modifies block validation rules, so having different independent implementations per client producing identical table roots for the same chain history is the ideal for spec correctness. Also diversified client implementation can find out specification ambiguities that might not have been caught by a single client). Personally, I recently contributed to Nethermind client so I'm familiar with the codebase.

The existing `BeaconBlockRootHandler` (EIP-4788) and `BlockhashStore` (EIP-2935) provide patterns for system contract integration. The `ISszCodec<T>` interface in `Nethermind.Serialization.Ssz` provides zero-allocation SSZ merkleization. The `ILogIndexStorage`/`LogIndexFilterVisitor` infrastructure provides a log query evaluation framework that the proof-serving layer can build on. And the Autofac-based plugin system allows the index subsystem to be wired as a self-contained module without modifying the core block processing pipeline.

## Project Description

The project consists of some components that together form a complete verification path from a trusted state root to a verified log object:

1. Index Table Engine: Generates, sorts, hashes, and merges index entries from blocks and receipts. Computes SSZ table roots that are deterministic and identical across all conforming implementations.

2. System Contract Integration: Writes table roots into the EIP-8304 index contract during block processing via a system call from `Address.SystemUser`, following the same lifecycle as EIP-4788. Handles fork activation, contract deployment, sync recovery, and chain reorganizations.

3. Proof Index and RPC: Maintains a persistent proof-serving database (separate from the consensus-critical path) that supports bounded-range log queries with inclusion, exclusion, and completeness proofs. Exposes an experimental RPC endpoint.

4. Verification Library: A standalone .NET library that verifies the complete proof chain: state proof, table root, index proof, block/transaction entries, receipt proof, verified log. This library has no dependency on Nethermind internals and can be consumed independently.

Only index entry generation and SSZ root computation are consensus-critical and are pure functions of a block's transactions and receipts. The proof-serving database and RPC are addititionals that can be rebuilt from chain data and does not affect block validation.

## Specification

### Index Entry Generation

For each processed block, the engine generates a set of index entries from the block's transactions and receipts. Seven entry types are defined, each with a variable length binary encoding:

| Entry Type | Type ID | Content | Encoding Length |
|---|---|---|---|
| Block | 0 | Block hash | 42 bytes |
| Transaction | 1 | Tx hash | 50 bytes |
| Log Address | 2 | Address | 38 bytes |
| Log Topic[0-3] | 3-6 | Topic hash | 50 bytes |

All fields are big-endian encoded. Entries are lexicographically sorted by their binary representation, which naturally clusters entries of the same type and value together a property used by range based inclusion and completeness proofs.

Block entries follow a one-block delay rule: block N's table contains the block entry for block N-1 (since N's own hash is unknown during N's processing).

The entry generation in Nethermind would extend the post transaction processing phase in `BlockProcessor`, operating on the same `TxReceipt[]` array that is already available:

```csharp
// Nethermind.Consensus/IndexTables/IndexEntryGenerator.cs

public static class IndexEntryGenerator
{
    /// <summary>
    /// Generates all index entries for a single block from its header, transactions, and receipts.
    /// Block entries use a one-block delay: the entry for block N-1 is included in block N's table.
    /// </summary>
    public static void GenerateEntries(
        BlockHeader header,
        Transaction[] transactions,
        TxReceipt[] receipts,
        Hash256? parentBlockHash,
        IList<IndexEntry> entries)
    {
        // Block entry for parent (one-block delay)
        if (!header.IsGenesis && parentBlockHash is not null)
        {
            entries.Add(IndexEntry.CreateBlock(parentBlockHash, header.Number - 1));
        }

        ulong cumulativeLogCount = 0;
        for (int txIndex = 0; txIndex < receipts.Length; txIndex++)
        {
            TxReceipt receipt = receipts[txIndex];

            // Transaction entry
            entries.Add(IndexEntry.CreateTransaction(
                transactions[txIndex].Hash!,
                header.Number,
                (uint)txIndex,
                (uint)cumulativeLogCount));

            // Log entries
            LogEntry[] logs = receipt.Logs ?? [];
            for (int logIndex = 0; logIndex < logs.Length; logIndex++)
            {
                LogEntry log = logs[logIndex];

                entries.Add(IndexEntry.CreateLogAddress(
                    log.LoggersAddress, header.Number, (uint)txIndex, (uint)logIndex));

                ReadOnlySpan<Hash256> topics = log.Topics;
                for (int t = 0; t < topics.Length && t < 4; t++)
                {
                    entries.Add(IndexEntry.CreateLogTopic(
                        t, topics[t], header.Number, (uint)txIndex, (uint)logIndex));
                }
            }
            cumulativeLogCount += (ulong)logs.Length;
        }
    }
}
```

### Table Root Computation

Entries are sorted lexicographically, individually hashed with SHA-256, and the resulting hashes are merkleized as an SSZ `List[Hash32, entry_count]`. Nethermind's existing `Merkle` infrastructure in `Nethermind.Serialization.Ssz` handles the tree computation:

```csharp
// Nethermind.Consensus/IndexTables/IndexTableRoot.cs

public static class IndexTableRoot
{
    /// <summary>
    /// Computes the SSZ table root from sorted index entries.
    /// Each entry is SHA-256 hashed, then merkleized as an SSZ List[Hash32].
    /// </summary>
    public static Hash256 ComputeRoot(ReadOnlySpan<IndexEntry> sortedEntries)
    {
        Span<byte> entryBuffer = stackalloc byte[MaxEntryLength];

        using ArrayPoolList<Hash256> leafHashes = new(sortedEntries.Length);
        foreach (ref readonly IndexEntry entry in sortedEntries)
        {
            int len = entry.Encode(entryBuffer);
            leafHashes.Add(Sha256.Compute(entryBuffer[..len]));
        }

        Merkle.MerkleizeList(leafHashes.AsSpan(), (ulong)sortedEntries.Length, out UInt256 root);
        return new Hash256(root);
    }
}
```

### Multi-Level Table Merging

The EIP mandates five table levels with sizes `[1, 4, 16, 64, 256]`. Higher-level tables are produced by k-way merging the sorted entries of their constituent lower-level tables. Since entries are already sorted, the merge is a standard sorted-merge operation.

Higher-level tables are published with a delay (`TABLE_SIZES[i]/4` blocks) to allow asynchronous background merging without adding latency to block processing. The merge scheduler must handle:

- Deadlines: Level-i merges must complete within `TABLE_SIZES[i]/4` blocks.
- Persistence: Intermediate tables survive client restarts.
- Reorgs: Invalidated tables are discarded and regenerated from the new canonical chain.

### System Contract Integration

Table roots are written to the index contract via a system call, following the EIP-4788 pattern already implemented in Nethermind's `BeaconBlockRootHandler`. The call uses `Address.SystemUser` as sender, some gas limit, zero value, and 96 bytes of calldata (`first_block || table_size || table_root`).

In the block processing pipeline, index table system calls execute after all user transactions and after existing post-execution system operations:

Integration point in `BlockProcessor.ProcessBlock`:

1. Pre-execution: EIP-4788, EIP-2935
2. User transactions
3. Post-execution: rewards, withdrawals, execution requests
4. EIP-8304 index table root commitments
5. State root computation

The contract's storage layout uses a ring buffer of 1024 slots per table level:

```
storage_slot = table_size × 1024 + (first_block/table_size) % 1024
```

This covers approximately the last 262,144 blocks at the highest table level. (256 blocks per table × 1024)

### Fork Activation

A new `IsEip8304Enabled` flag on `IReleaseSpec` gates all EIP-8304 logic. The index contract address and ring buffer size are configurable per-chain via `Eip8304ContractAddress` and related spec properties, following the established pattern for EIP-2935 and EIP-4788:

```csharp
bool IsEip8304Enabled { get; }
Address? Eip8304ContractAddress { get; }
```

### Proof Format and RPC

The proof-serving RPC endpoint accepts bounded block-range queries with address and topic filters and returns paginated results where each page contains:

- The query parameters and reference execution block hash
- Table roots and their state proofs
- Sorted index entries covering the queried range
- Inclusion/exclusion boundary entries for completeness verification
- Receipt proofs for each matched log (required to verify `log.data`, which EIP-8304 does not directly commit to)

The proof chain from trusted state root to verified log:

```
State Root
  → eth_getProof for index contract storage slot
    → Table root (SSZ List[Hash32] root)
      → Sorted entries + Merkle proof
        → Matching entries with (blockNumber, txIndex, logIndex)
          → Block entry → target block header hash
            → Receipt proof against receiptsRoot
              → Verified log (address, topics, data)
```

## Roadmap

### Phase 0: Foundation and Spec Alignment (Weeks 1-2)

- Read and document every open question in the EIP-8304 specification, particularly around entry encoding lengths, SSZ list semantics, and deployment transaction parameters.
- Study Nethermind's existing system contract implementations (`BeaconBlockRootHandler`, `BlockhashStore`) and the `BlockProcessor` pipeline in depth.
- Set up a local devnet (Nethermind + a CL client via Kurtosis) for testing.
- Implement the `IndexEntry` type with variable-length binary encoding (38/40/50/52 bytes) and round-trip tests.
- Build an independent reference root calculator as a standalone CLI tool that takes blocks + receipts and produces expected table roots this serves as a differential testing oracle throughout the project.

### Phase 1: Single-Block Index Tables and System Contract (Weeks 3-5)

- Implement `IndexEntryGenerator` generation of all seven entry types from a block's transactions and receipts.
- Implement `IndexTableRoot` lexicographic sort, SHA-256 per-entry hashing, SSZ `List[Hash32]` merkleization using `Nethermind.Serialization.Ssz`.
- Add `IsEip8304Enabled` and `Eip8304ContractAddress` to `IReleaseSpec` and wire activation through `ChainSpecBasedSpecProvider`.
- Implement `IndexTableHandler` the system contract caller that posts level-0 (single-block) table roots to the index contract during `BlockProcessor.ProcessBlock`, positioned after execution requests and before state root computation.
- Deploy the index contract bytecode at genesis in the devnet chain spec.
- Validate the level-0 roots produced by Nethermind match the reference root calculator for the same blocks.
- Open a draft upstream PR for initial review by maintainers 

### Phase 2: Multi-Level Table Merging (Weeks 6-8)

- Implement the k-way sorted merge for constructing higher-level tables from their constituent lower-level tables.
- Build the merge scheduler that tracks publication deadlines (`first_block + table_size - 1 + table_size/4`) and triggers merges for levels 1 through 4.
- Implement persistent table storage using a dedicated RocksDB column family, with the 1024-slot ring buffer eviction policy.
- Handle chain reorganizations: detect reorgs via canonical hash tracking, invalidate affected tables, and regenerate from the new canonical chain.
- Handle sync recovery: after snap/fast sync, regenerate required tables from the last `TABLE_SIZES[4] * 5/4 - 1 = 319` blocks.
- Handle client restarts: persist merge state and resume incomplete merges.
- Wire the full five-level system as an Autofac module (`IndexTableModule`) that registers all components without modifying existing block processing code.
- Validate all five table levels produce roots matching the reference calculator on the devnet.

### Phase 3: Proof Index and RPC (Weeks 9-11)

- Build the proof-index database a separate persistence layer (not on the consensus path) that stores sorted entries, entry-to-block mappings, and metadata needed for proof generation.
- Implement proof generation: inclusion proofs (contiguous match ranges in sorted entries), exclusion proofs (boundary entries proving no match exists), and completeness proofs (cross-table range coverage with predecessor/successor boundaries).
- Implement the experimental RPC endpoint supporting bounded block-range queries, address/topic filters, pagination, and empty-result proofs.
- Define the proof JSON schema with explicit versioning.
- Integrate with Nethermind's existing `IJsonRpcModule` infrastructure.

### Phase 4: Verification Library and Testing (Weeks 12-14)

- Build the standalone verification library (`Nethermind.IndexProof.Verifier`) with no dependency on Nethermind internals.
- Implement verification of the complete proof chain: state proof, table root, index entries, block/tx entries, receipt proof, verified log.
- Differential testing: compare Nethermind-generated roots against the reference calculator and (if available) another client's implementation for the same chain history.
- Write a Hive simulator test suite covering: correct root generation across all levels, reorg handling, sync recovery, restart resilience, malformed proof rejection, boundary conditions (empty blocks, max-entry blocks, ring buffer wraparound, genesis edge cases).
- Performance benchmarking: measure per-block overhead of index generation and system call, disk growth rate, proof generation latency, and proof sizes at each table level.
- Prepare the ppt presentation for showcase.

## Possible Challenges

Specification instability: EIP-8304 is a Draft. The contract address, deployment transaction, and encoding details are not finalized. Entry encoding lengths in the spec text differ between sections (the "entry types" table shows `4+32+8 = 44` bytes for block entries while the binary examples use a 2-byte type prefix giving 42 bytes). Mitigation strategy: pin to a specific spec commit, maintain the reference root calculator as the authoritative encoding oracle, and keep the encoding layer isolated behind an interface so changes propagate cleanly.

Cross-client root agreement: The entire value proposition of EIP-8304 depends on all clients computing identical table roots for the same chain. Differences in big-endian encoding, lexicographic sort order, SHA-256 input boundaries, or SSZ list length handling will produce silent root mismatches. For this we can carry out differential testing against the reference calculator at every development stage and byte-level logging of intermediate values (encoded entries, leaf hashes, partial trees).

Background merge complexity: Higher-level table merges are asynchronous but have hard deadlines. A level-4 merge (256 blocks) must complete within 64 blocks (~13 minutes). On resource-constrained nodes or during periods of high block density, this deadline may be tight. The merge must also be crash-safe (resumable after restart) and reorg-safe (invalidated upon canonical chain change). To fix this, we can use file-backed intermediate state, incremental merge checkpointing, and conservative deadline monitoring with early-warning logging.

Consensus path isolation: The table root computation is consensus-critical it determines what gets written to the system contract and therefore affects the state root. The proof-serving database is not. These two paths should be strictly separated: the consensus path must never depend on the proof database, and a corrupted proof database must never affect block validation. We ought to have separate Autofac modules, separate storage backends, and explicit interface boundaries.

Proof completeness: Generating proofs that are not just valid but complete (proving that no matching results were omitted) is quite harder than inclusion proofs. Cross table queries, OR conditions on addresses/topics, topic wildcards, and empty results all need careful boundary proof construction. This could be achieved by starting with single table and single condition proofs and incrementally add complexity, with property-based fuzzing at each stage.

Performance overhead: Every block incurs at least one additional SSTORE for the level-0 table root, plus periodic SSTOREs for higher levels. The entry generation, sorting, hashing, and merkleization add CPU overhead to block processing. This could be mitigated by using zero-allocation patterns (stackalloc, ArrayPool, Span-based APIs), incremental SSZ tree updates where possible, and performance benchmarking against baseline block processing times.

## Goal of the Project

The project is deemed to be successful when:

1. Correct roots: Nethermind generates index table roots at all five levels that are byte-identical to the reference root calculator for the same chain history. If another EL client has an implementation (like Geth or Reth), cross client root agreement could be demonstrated on a shared devnet.

2. Consensus integration: The system contract is correctly updated during block processing, surviving sync recovery (snap sync, fast sync), client restarts, and chain reorganizations without producing incorrect roots or missing table commitments.

3. Verifiable proofs: The RPC endpoint serves log query proofs that the standalone verification library can validate end to end from state root to verified log contents and that it correctly rejects forged, incomplete, or malformed proofs.

4. Low overhead: The consensus-critical path adds negligible overhead to block processing. The proof-serving database is strictly separated from the consensus path and can be rebuilt from canonical chain data.

5. Coding guidelines: Code follows Nethermind's coding standards, it is wired via Autofac modules without modifying core block processing code, and has test coverage (unit, integration, and Hive), and is ready for review.

I already have prior experience contributing to Nethermind, where I implemented SSZ-REST Engine API (a 4.8× latency reduction vs JSON-RPC) in about 2 months. This is a speculation, but based on this I should be able to complete the project within the cohort duration.

The end result is a Nethermind node on a devnet that commits correct index table roots to the system contract and serves verifiable log proofs over RPC which allows any client with access to a trusted state root to query logs without trusting the node.

## Collaborators

### Fellows

- [Ritesh Das](https://github.com/Dyslex7c)

### Mentors

- [Tamaghna Choudhuri](https://github.com/RazorClient)
- [Etan Kissling](https://github.com/etan-status)

## Resources

- [EIP-8304: Trustless log and transaction index](https://eips.ethereum.org/EIPS/eip-8304) the specification
- [EIP-8304 discussion on Ethereum Magicians](https://ethereum-magicians.org/t/eip-8304-trustless-log-and-transaction-index/28824)
- [EIP-8304 specification gist by Zsolt Felföldi](https://gist.github.com/zsfelfoldi/55899871c8a569b3987611dc985361d8)
- [Nethermind repository](https://github.com/NethermindEth/nethermind) target codebase
- [EIP-4788: Beacon block root in the EVM](https://eips.ethereum.org/EIPS/eip-4788) system contract pattern reference
- [EIP-2935: Serve historical block hashes from state](https://eips.ethereum.org/EIPS/eip-2935) ring buffer storage pattern reference
- [EIP-7919: Pureth Meta](https://eips.ethereum.org/EIPS/eip-7919) broader trustless verification vision
- [8304 performance notes](https://gist.github.com/zsfelfoldi/3d22525b5732a9540bb9d9961ed5d238)
- [EIP-6466: SSZ receipts](https://eips.ethereum.org/EIPS/eip-6466)
