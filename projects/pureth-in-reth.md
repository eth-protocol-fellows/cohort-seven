# Pureth in Reth: SSZ Execution Blocks and Query Language

## Motivation

Execution layer data is mostly consumed through JSON-RPC providers and indexers right now. A wallet can ask for a receipt or a log but it usually has to trust the server response. With RLP receipts and the Merkle Patricia Trie, a proof can show that an encoded receipt is included in a block. It cannot directly prove one field inside that receipt, such as `status` or a log `address`, without the verifier decoding the whole receipt.

Pureth is exploring a different direction which is SSZ shaped execution data, SSZ Merkle roots and proofs where a client verifies only the data it needs. SSZ gives a typed object a deterministic Merkle tree and generalized index(gindex) for each field. So a response can carry a value, its gindex, and the sibling hashes needed to reconstruct the object root.

We plan to prototype selected Pureth style SSZ execution data in Reth through one shared pipeline. One track produces the agreed SSZ execution objects and roots, while the other builds the query and proof layer over those roots. The goal is a narrow but verifiable path from execution data to a field proof which can grow with Pureth.

## Project Overview

We are working in one shared Reth fork through two connected tracks:

1. **SSZ execution data and roots.** Experimental SSZ representations for selected execution objects, their roots, and the root/object backend for the query layer.
2. **SSZ Query Language and proofs.** An experimental Reth RPC capability that accepts a field path, resolves it against an agreed SSZ object, calculates its gindex, and returns the value with a Merkle proof.

```text
Reth execution data / deterministic sample objects
└── agreed experimental SSZ object schema + (schema specific)prototype root
    └── RootProvider
        └── SSZ-QL parser + path validator
            └── schema aware gindex resolver
                └── Merkle proof generator
                    └── response
                        ├── value
                        ├── gindex
                        ├── proof nodes
                        ├── root
                        └── root context
```

The tracks meet at `RootProvider`. That lets the query layer work with deterministic sample objects first & then switch to the experimental Reth backend later without changing the request/response model.

## Scope and Trust Model

The baseline target is structured receipts and logs, informed by [EIP 6466](https://eips.ethereum.org/EIPS/eip-6466). The first receipt prototype uses a simplified local schema. It is EIP 6466 informed, not EIP 6466 compatible.

Withdrawals are the early root computation and test vector milestone. They are modelled after [EIP 6465](https://eips.ethereum.org/EIPS/eip-6465).

Structured transaction field proofs come later. Paths like `.transactions[0].to` need [EIP 6404](https://eips.ethereum.org/EIPS/eip-6404) and [EIP 8016](https://eips.ethereum.org/EIPS/eip-8016). These define the structured transaction model and stable semantics across transaction variants.

[EIP 7807](https://eips.ethereum.org/EIPS/eip-7807) is relevant, but it is not enough by itself for receipt field queries. It encodes the execution payload using SSZ, while transaction and receipt data stay at the encoded byte list/root level. It can commit to a receipt item, but it does not expose structured receipt fields such as `.receipts[5].status`.

Those fields need an agreed structured receipt schema. This is why the QL baseline uses EIP 6466 informed receipts instead of EIP 7807 alone.

A proof shows that a value or subtree is included at a stated gindex under a supplied root for a declared SSZ schema. It does not by itself show that the root is canonical Ethereum data.

The response will identify the schema, proof encoding, and root context. Root context can be deterministic test data, the shared Reth prototype, or a future anchored source.

## Track 1: SSZ Execution Data Production in Reth

This is the execution data and root production side.

### Withdrawals: root computation baseline

The first milestone is an EIP 6465 shaped `Withdrawal` representation and root calculation over an SSZ list.

- `WithdrawalSSZ`
- `withdrawals_root_ssz()`
- `tree_hash_root()`

This is the smallest selected object. It gives us a way to validate conversion, Merkleization, saved test examples, and the `RootProvider` contract before moving into receipts.

> The Rust sketch below is illustrative pseudocode. Exact EIP 6465 compatible roots need the selected SSZ implementation to use `ProgressiveContainer` and `ProgressiveList` semantics.

```rust
#[derive(Debug, Clone, Encode, Decode, TreeHash)]
pub struct WithdrawalSSZ {
    pub index: u64,
    pub validator_index: u64,
    pub address: Address,
    pub amount: u64,
}

fn withdrawals_root_ssz(withdrawals: &[Withdrawal]) -> B256 {
    let ssz_list: Vec<WithdrawalSSZ> = withdrawals.iter()
        .map(WithdrawalSSZ::from)
        .collect();
    ssz_list.tree_hash_root().into()
}
```

### Receipts and logs: shared proof target

The main execution data milestone is an experimental receipt/log model informed by EIP 6466. The first prototype uses a simplified single receipt container to keep the implementation moving. It is not EIP 6466 compatible. Full EIP 6466 compatibility needs the `Receipt` `CompatibleUnion` with `BasicReceipt`, `CreateReceipt`, and `SetCodeReceipt` variants.

The local prototype model includes:

- A simplified `ReceiptSSZ` container;
- `gas_used`, local `success`, `logs`, and `contract_address` as fields. EIP 6466 uses `status`, and its fields differ by variant;
- No transaction sender (`from_`) field;
- A `LogSSZ` container with `address`, up to four `topics`, and `data`; and
- An experimental receipt list root computed as `hash_tree_root(ProgressiveList<ReceiptSSZ>)` for the agreed prototype schema. This does not claim to replace Ethereum's canonical block header `receipts_root`.

Reth's existing receipt data is mapped into this schema, then the receipt list root, object, and shared provider path are built around it.

- `ReceiptSSZ`
- `LogSSZ`
- `tree_hash_root()`
- `RootProvider`

```rust
#[derive(Debug, Clone, Encode, Decode, TreeHash)]
pub struct ReceiptSSZ {
    pub r#type: TransactionType,
    pub success: bool,
    pub gas_used: u64,
    pub contract_address: Option<Address>,
    pub logs: ProgressiveList<LogSSZ>,
}
```

> This is also an illustrative local schema, not an exact EIP 6466 type. The exact Reth and Alloy insertion points will be documented when implementation starts instead of guessed here.

### Transactions and execution blocks

EIP 6404-style transactions and EIP 8016 `CompatibleUnion` support come after the receipt path works.

- `SszTransaction`
- `CompatibleUnion`
- `TxEnvelope`

These have a bigger compatibility surface across legacy, access list, fee market, blob, and set code transaction forms.

> The code snippet below only shows the intended shape of the transaction model for understanding. The actual implementation will follow the current EIP 6404 schema and EIP 8016 `CompatibleUnion` semantics.

```rust
#[derive(Debug, Clone, Encode, Decode, TreeHash)]
#[ssz(enum_behaviour = "compatible_union")]
pub enum SszTransaction {
    Legacy(LegacyTransactionSSZ),
    Eip2930(Eip2930TransactionSSZ),
    Eip1559(Eip1559TransactionSSZ),
    Eip4844(Eip4844TransactionSSZ),
}
```

Full EIP 7807 execution block support is extension work. It depends on the progressive SSZ stack and broader EL/CL protocol decisions, so the QL milestone does not depend on it.

## Track 2: SSZ Query Language and Proof Serving in Reth

This is the Reth side SSZ-QL (query and proof)layer.

### Initial API

The prototype adds one experimental RPC method, provisionally named `pureth_query`. The existing receipt response also gets an extra `sszProof` field with the proof subfields.

The request selects a block, object kind, and one path:

```json
{
  "block_hash": "0x<32-byte-block-hash>",
  "object": "receipts",
  "path": "[5].logs[0].address",
  "include_proof": true
}
```

The block hash selects one specific block, even if it later becomes noncanonical. The method returns an error when the block or requested root is unavailable. Test data uses a deterministic block hash with the same shape.

Paths are relative to the selected object:

```text
[5].logs[0].address
```

Use `.field` for fields and `[0]`, `[1]`, and so on for list items. Since `object` is `receipts`, this asks for the first log address in receipt six.

v0 accepts one exact, nonempty path with named fields and non-negative list indexes. Wildcards, filters, ranges, and transaction field paths are outside the baseline. Invalid paths are rejected before data traversal or proof generation.

### Response and proof format

The response includes:

```json
{
  "value_ssz": "0x<ssz-serialized-value>",
  "path": "[5].logs[0].address",
  "gindex": "<decimal-gindex>",
  "proof": ["0x<32-byte-sibling-node>"],
  "proof_format": "merkle_branch_v0",
  "schema_id": "pureth-receipt-v0",
  "root": "0x<32-byte-root>",
  "object": "receipts",
  "block_hash": "0x<32-byte-block-hash>",
  "root_context": "deterministic test data"
}
```

`schema_id` tells a verifier which SSZ schema to use. It reconstructs the target SSZ node from `value_ssz` and the path, then uses the sibling nodes to reproduce `root`. For example- an address is returned as 20 serialized bytes and right-padded with 12 zero bytes to form one 32byte SSZ chunk before verification. `proof_format` records the sibling node ordering used by the response.

Test vectors save a known request, value, root, gindex, and proof. They catch mismatches in paths, serialization & proof output.

The same proof subfields will also be returned inside an extra `sszProof` field in the existing receipt response.

### Query engine

The QL implementation will:

- Register the experimental RPC method in Reth and add `sszProof` to the existing receipt response
- Parse and validate the path into field/index tokens
- Obtain the selected object and root from `RootProvider`
- Traverse the declared SSZ schema and resolve the value
- Derive the corresponding gindex using the same schema and Merkleization rules as the producer
- Produce the requested Merkle proof
- Return the schema id, proof format and root context so callers can verify the response & know whether they are checking test data or the Reth prototype or future anchored source.

`DummyRootProvider` serves deterministic receipt/log sample objects first. `RethRootProvider` uses the experimental receipt backend when its root is available.

## Shared Interface and Verification

The tracks freeze a small interface before implementation:

- `ObjectKind`: initially `withdrawals` and `receipts`; transactions are added only after their schema is ready.
- `RootProvider`: given `block_hash` and `ObjectKind`, return the SSZ object, root, schema revision, and root context.
- Schema revision: every test object/root pair identifies the exact schema used to construct it.
- Verifier: a small independent check that combines the returned value and proof nodes, then confirms that they reconstruct `root` at the stated gindex.

`RootProvider` is an internal interface. The public response exposes the `object`, `schema_id`, `proof_format`, block hash, root, and root context needed for verification.

The verifier tests show that one known good proof works and that changing the value, proof, or root makes verification fail.

## Roadmap

The formal roadmap starts in Week 7 and follows these rough phase ranges:

### Phase 0: Shared foundation (Weeks 7-9)

- Agree the receipt/log schema revision, test data format, `ObjectKind`, `RootProvider`, and root context.
- Create deterministic test vectors and a small independent verifier.
- The progressive type dependencies (`sigp/ethereum_ssz` PR #67 and `sigp/tree_hash` PR #43) are a hard prerequisite for Phase 1 withdrawal and receipt work. The Phase 1 query layer prototype has no dependency on these PRs and can proceed in parallel.

### Phase 1: Independent prototypes (Weeks 10-13)

**Execution data and roots**

- Implement the withdrawal conversion/root prototype and compare it against available reference vectors.
- Define the experimental EIP 6466 informed receipt/log model and produce deterministic receipt roots.

**SSZ query language and proofs**

- Add the experimental Reth RPC surface and parser.
- Implement schema aware traversal, gindex derivation, `DummyRootProvider`, & proof generation.
- Check the saved test vectors end to end: the returned value and proof must reconstruct the expected root.

### Phase 2: Reth integration (Weeks 14-17)

- Map selected Reth receipt data into the agreed experimental schema.
- Evaluate EIP 6404/EIP 8016 transaction support only after the receipt pipeline is stable.
- Implement `RethRootProvider` for the shared receipt object.
- Prove selected receipt/log fields against a root produced by the Reth fork.
- Add RPC and integration tests for `pureth_query` and the receipt response proof field.

### Phase 3: Extensions (Weeks 18-20+)

- Extend receipt/log path coverage and improve proof handling where useful.
- Evaluate EIP 7807, system contract anchoring, and CL support only as separately confirmed stretch work.

## Testing, Limits, and Risks

The project tests each layer: parser tests validate paths, gindex/value tests check traversal, root/proof tests check verification, and RPC tests cover the full flow. The key integration test proves a receipt/log field against a Reth fork root using the agreed schema, not only test data.

**STEEL and execution spec tests.** We will track the Specifications and Testing for the Ethereum Execution Layer (STEEL) workflow and use any relevant execution spec test vectors as they become available for the selected EIPs. Until then our deterministic SSZ object/root/proof vectors are the reference suite for this prototype & they should be versioned so they can later be compared with or contributed to a wider execution spec test effort.

**Kurtosis devnet validation.** Once the Reth root producing path and the required EL/CL payload plumbing are working, we will run a reproducible local Kurtosis Ethereum devnet. This tests the modified Reth node during block production and exercises the query RPC against produced blocks. It is a late integration check, not a baseline claim of cross client interoperability or protocol anchoring.

v0 accepts only supported object kinds and one path, with caps on depth, indexes, response size, and proof size. Invalid or oversized queries fail before traversal or proof generation.

## Goals

- Produce deterministic SSZ execution object roots in a Reth fork, beginning with withdrawals and structured receipt/log sample objects.
- Additive Reth query/proof interface for structured receipt/log paths.
- Return values, gindices, proofs, roots, and root context that independent verifier can check.
- Integrate the QL layer with the Reth produced receipt root when the shared data path is ready.

## Collaborators

- **[Parth Singh:](https://github.com/ParthSinghPS)** SSZ execution data representations, root production, Reth fork data mapping, and `RethRootProvider`.
- **[Arsh:](https://github.com/ArshLabs)** SSZ-QL RPC, path parsing, gindex and proof generation, and `RootProvider` integration.

**Mentors:** [Tamaghna Choudhuri](https://github.com/RazorClient) and [Etan Kissling](https://github.com/etan-status).

## Resources

- [EIP 7919: Pureth Meta](https://eips.ethereum.org/EIPS/eip-7919)
- [EIP 6465: SSZ withdrawals root](https://eips.ethereum.org/EIPS/eip-6465)
- [EIP 6466: SSZ receipts](https://eips.ethereum.org/EIPS/eip-6466)
- [EIP 6404: SSZ transactions](https://eips.ethereum.org/EIPS/eip-6404)
- [EIP 7807: SSZ execution blocks](https://eips.ethereum.org/EIPS/eip-7807)
- [EIP 7495: SSZ ProgressiveContainer](https://eips.ethereum.org/EIPS/eip-7495)
- [EIP 7916: SSZ ProgressiveList](https://eips.ethereum.org/EIPS/eip-7916)
- [EIP 8016: SSZ CompatibleUnion](https://eips.ethereum.org/EIPS/eip-8016)
- [SigP `ethereum_ssz` PR #67](https://github.com/sigp/ethereum_ssz/pull/67)
- [SigP `tree_hash` PR #43](https://github.com/sigp/tree_hash/pull/43)
- [STEEL execution spec tests](https://steel.ethereum.foundation/docs/execution-specs/tests/)
- [Kurtosis private Ethereum networks](https://geth.ethereum.org/docs/fundamentals/kurtosis)
- [Jun and Nando's SSZ-QL work in Prysm](https://hackmd.io/@junsong/Hyd7ElRJZx)
- [Reth RPC crates](https://github.com/paradigmxyz/reth/tree/main/crates/rpc)
