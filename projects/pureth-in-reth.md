# Pureth in Reth: SSZ Execution Blocks and Query Language

## Motivation

Execution layer data is mostly consumed through JSON-RPC providers and indexers right now. A wallet can ask for a receipt or a log but it usually has to trust the server response. With RLP receipts and the Merkle Patricia Trie, a proof can show that an encoded receipt is included in a block. With this MPT proof, checking a field such as `status` or a log `address` still requires the verifier to have the full encoded receipt, but not to decode every field.

Pureth is exploring a different direction which is SSZ shaped execution data, SSZ Merkle roots and proofs where a client verifies only the data it needs. SSZ gives a typed object a deterministic Merkle tree and generalized index (gindex) for each field. So a response can carry a value and the hashes needed to reconstruct the object root. The verifier derives the gindices from the request and agreed schema instead of just taking them from the server response.

We plan to prototype selected Pureth style SSZ execution data in Reth through one shared pipeline. One track will produce the agreed SSZ execution objects and roots, while the other will build the query and proof layer over those roots. The goal is a verifiable path from execution data to field proofs which can grow with Pureth. We will extend this into transaction and block objects, then the selected live Reth integration.

## Project Overview

We will build this together in one shared Reth fork. The work has two connected tracks:

1. **SSZ Execution Data and Roots.** Experimental SSZ representations for selected execution objects and their roots, with a backend that provides both to the query layer. This also includes the agreed builder, Engine and block identity work.
2. **SSZ Query Language and Proofs.** An experimental RPC service in Reth that will accept selections, resolve them against an agreed SSZ object, and return the values with Merkle proofs. The query layer and verifier will use the same schema and Merkleization rules as the producer.

```text
Reth execution or supported provider inputs
                |
                v
validated objects + retained component trees
                |
                v
payload and summary commitments, with a coherent source snapshot
                |
                v
bounded selection -> proof generation -> response
                |
                v
client checks positions derived from the request against an expected root
```

The tracks will meet first at `RethRootProvider` for receipts, then at a shared execution source for the selected objects. The query layer will work with deterministic sample objects first & then the experimental Reth backend. Each response must use the object, tree and root from the same acquisition.

## Scope and Trust Model

The first receipt prototype will use a simplified local schema. It will be EIP 6466 informed, not EIP 6466 compatible. This will  basically give us the initial receipt and log pipeline before then replacing it with the selected EIP receipt format and root.

Withdrawals are the early root computation and test vector milestone. They are modelled after [EIP 6465](https://eips.ethereum.org/EIPS/eip-6465).

Structured transaction field proofs will come later. These need the normalized transaction model in [EIP 6404](https://eips.ethereum.org/EIPS/eip-6404) and the compatible union rules in [EIP 8016](https://eips.ethereum.org/EIPS/eip-8016).

The combined scope will cover these selected parts and not the whole of Pureth meta collection:

- EIP 6465 withdrawal objects and progressive commitments.
- EIP 6466 Basic, Create and SetCode receipt formats, construction adapters and receipt list commitments under the selected interpretation.
- EIP 6404 normalized signed transactions, nested structures, codecs and individual roots and list roots.
- A payload and summary shaped around EIP 7807, using the selected typed transaction, receipt and withdrawal commitments.
- Finite query operations, shared proofs, component linkage and proof support in receipt responses.
- Client integration that follows the fork rules, validation against a separately running node and a compatible private devnet scenario with saved inputs and results.

[EIP 7807](https://eips.ethereum.org/EIPS/eip-7807) is relevant, but it is not enough by itself for receipt field queries. It encodes the execution payload using SSZ, while transaction and receipt data stay at the level of encoded byte lists and roots. It can commit to a receipt item, but it does not expose structured receipt fields such as `.receipts[5].status`.

Those fields need an agreed structured receipt schema. Our block experiment will combine typed EIP 6404 transactions, EIP 6466 receipts and EIP 6465 withdrawals with the EIP 7807 shaped payload. This changes the standalone EIP 7807 composition, so the producer, consumer and independent reference must agree on each field's schema and commitment.

A proof checks that a value or subtree belongs under the expected root for the agreed SSZ schema. The caller will need a trusted expected root from outside the response to connect that proof to Ethereum data.

Responses will identify the proof encoding and root context.

## Track 1: SSZ Execution Data Production in Reth

This is the execution data and root production side.

### Shared SSZ Support

We will reuse Reth's existing types and libraries for encoding, decoding and tree hashing where they fit the selected schema. Progressive containers, lists, byte lists and compatible unions will follow their pinned rules. The producer will retain the tree for the query layer, while an independent verifier can rebuild the root for checking.

### Withdrawals: Root Computation Baseline

The first milestone will be an EIP 6465 shaped `Withdrawal` representation and root calculation over a progressive SSZ list. This is the smallest selected object. It will give us a way to validate conversion, Merkleization, saved test examples, and the shared snapshot before moving into receipts.

The four ordered fields and their widths will be preserved. Encoding, decoding and the retained snapshot will support field, list and component proofs with the selected execution count limit.

### Receipts and Logs: Shared Proof Target

The main receipt milestone will be the selected EIP 6466 `Receipt` `CompatibleUnion` with `BasicReceipt`, `CreateReceipt`, and `SetCodeReceipt` variants. Their receipt selectors are Basic=1, Create=2 and SetCode=3, separately from transaction type bytes.

Reth's transaction, receipt and recovered sender data will be read in the same order and converted into this schema. Missing or malformed required inputs will reject the complete snapshot.

A failed top-level creation will keep the Create variant but record zero for the contract address. SetCode will keep each authorization position and duplicates, recording the recovered authority address or zero if recovery fails (recovery does not mean the authorization was accepted during execution).

Gas conversion will so follow the actual chain specification. This includes checked cumulative deltas, counts and types for transactions, receipts and senders, transaction gas limits and header bounds. Receipt gas charges and block resource gas will each follow the selected fork rules, preserving any difference between them.

Once the EIP provider and query path pass the equivalent tests, the active prototype schema, fixtures and wiring will thus be removed. Shared tree helpers will stay where we wanna reuse them, and the provider will return the one agreed EIP snapshot.

### Transactions and Execution Blocks

Support for transactions based on EIP 6404 and EIP 8016 `CompatibleUnion` will follow once the receipt path will work. These need to handle differences between legacy, access list, fee market, blob, and set code transaction forms.

There are ten normalized payload layouts: replayable legacy call and creation, chain-specific legacy call and creation, access list call and creation, dynamic fee call and creation, blob and SetCode.

The block work will thus build the agreed payload with 18 positions, its summary and component commitments. Replacing the selected fields with their roots must keep the payload and summary roots equal using schema aware `hash_tree_root`. Regular excess gas (`excess_gas.regular`) will be fixed at zero and nonzero values will be rejected.

The summary will replace transactions, withdrawals and BAL with their roots. Receipt and request roots will already be fields of the payload. Building a complete retained source will also require the receipt and request bodies. 

The remaining fields need actual sources: state and parent references, ordinary and blob gas and fees, slot, full execution requests, withdrawals and RLP block access list bytes derived during execution. Empty BAL uses the RLP encoding of an empty list. Historical conversion will support blocks with all the required inputs available.

Header and body construction will use the same retained payload snapshot and check that they agree. The block type and storage codec will provide the representation needed for the later canonical identity integration. Block RPC responses under the selected rules will expose `logsBloom` with all bits set to one, separately from the changes to receipt responses below.

### Integration Milestones

| Milestone | Work and required evidence |
| --- | --- |
| Level 1: representation | Complete the mapping of actual execution inputs, codecs and retained roots. Check agreement between payload and summary roots & run independent reference checks and rejection tests. |
| Level 2: builder and Engine | Build so according to the fork rules, use the selected payload carrier and validate execution and commitments when receiving payloads. Test payload building and the Engine API `getPayload` and `newPayload` calls, rejecting incompatible inputs. |
| Level 3: canonical client identity | Sealing and parent identity, hash and number indices, forkchoice, reorgs, durable restart, ordinary RPC, log and receipt references, BLOCKHASH for eligible ancestors and applicable parent history behavior. |

also we will adapt Lighthouse to derive the same execution summary from its payload inputs and check the advertised block identity. Reusable object and commitment code will be separated from Reth storage dependencies. 
Both clients will share that core, with independent reference checks for commitment calculations and integration checks using both clients. The selected route will keep native beacon carriers and native signed RLP transactions, using JSON Engine messages authenticated with JWT.

## Track 2: SSZ Query Language and Proof Serving in Reth

This is the Reth side SSZ-QL (query and proof)layer.

### Operations

The query layer will support these operations:

| Surface | Intended operations and boundary |
| --- | --- |
| Receipt and log selections | Values, whole elements, lengths, variant checks, presence checks, half-open ranges and data slices. Duplicate requests retain their results. |
| Shared proofs | Derive target and context positions from the request, and use multiproofs that share repeated tree positions. Reject missing, conflicting, extra or unused witness material. |
| Complete bounded filters | AND predicates over receipt sender, status and gas used by each transaction, and log addresses and topics at specified positions, with log predicates applying to the same log. Important checks to be done including no matches, within one block's receipt list. |
| Component selections | Supported transaction, withdrawal, request and execution summary selections, borrowing retained producer trees. Execution payload queries will cover whole objects and BAL queries will cover committed bytes. Separate component proofs will link transaction, withdrawal, request and BAL roots to the expected execution summary. |
| Associations | Authenticate transaction and receipt roots under one expected execution summary, prove the same requested transaction position and check the native transaction hash derived from the proved transaction. |
| Receipt response | Package whole receipt `sszProof` responses and bind them to the native committed fields. Omit the field when disabled, return null when unsupported and requested, integrate this with ordinary receipt RPCs. |

Receipt field proofs will check the selector and active fields, and verification will cover the supported variants and committed SSZ fields.

### Initial API

The prototype will start with `pureth_query`. We will extend the service with `pureth_transactionReceipt` for the association between a transaction and its receipt. The existing receipt response will also get an extra `sszProof` field when proof support is enabled.

The request selects a block, object kind, and the paths and operations to run. The example below show the inner request with a placeholder block hash. Named JSON-RPC params will wrap it in a `request` field, or a positional call can supply it as the single parameter:

```json
{
  "block_hash": "0x0101010101010101010101010101010101010101010101010101010101010101",
  "object": "receipts",
  "selection": {
    "selections": [
      {"path": "[0].status", "operation": {"kind": "value"}},
      {"path": "[0].logs[0].data", "operation": {"kind": "slice", "start": 1, "end": 3}}
    ],
    "include_proof": true
  }
}
```

The block hash selects one specific block, even if it later becomes noncanonical. Discarded sources will remain queryable while retained in memory, with no guaranteed readback after restart. Paths are relative to the selected object. Use `.field` for fields and `[0]`, `[1]`, and so on for list items.

### Response and proof format

The response will include the block hash, object, root context, block status and the selection results in request order. The selection will contain root, proof format, values and proof material. Multiproof targets and helpers will contain hashes, with their positions derived by the verifier from the request and agreed schema.

The service will start with receipts, then extend `pureth_query` to transactions, withdrawals, execution requests, BAL, execution summaries and whole payloads. Receipt queries will acquire receipt sources independently of complete payloads. Component responses will use their expected component roots, with separate proofs for summary linkage. Associations between transactions and receipts will include the required proofs under the same execution summary.

Individual and block receipt responses will share the source backend and optionally include `sszProof` for the SSZ receipt. After activation, receipts will expose `authorities`, omit `logsBloom` and use positions within each receipt, while earlier blocks retain legacy indexing. The proof switch will control only the added proof field: omitted when disabled, null when unsupported and requested, and an error if an expected proof cannot be produced.

### Query Engine and Limits

The QL implementation will:

- Register the experimental RPC method in Reth and add `sszProof` to the existing receipt response
- Parse and validate the selections as field tokens, index tokens and operations
- Obtain the selected object, retained tree and root from the agreed provider or snapshot
- Traverse the agreed SSZ schema and resolve the values
- Derive the corresponding gindices using the same schema and Merkleization rules as the producer
- Produce the requested Merkle proof for the independent verifier to check against its expected root
- Return the proof format and root context so callers can verify the response & know what source they are checking.

## Roadmap

The roadmap will begin near to week7-8 phase

| Period | Object production and live sources | Block formats, queries and verification | Integration and validation |
| --- | --- | --- | --- |
| Weeks 7-9 | Study pinned types, source mappings and reusable libraries and establish shared contracts and withdrawal foundation. | Study paths, position composition and client verification and agree trust and failure boundaries. | Validate the shared types and interfaces. |
| Weeks 10-13 | Buildd withdrawal objects and deliberately simplified prototype receipt objects, trees and a provider. | Build parser, resolver and first proof flow for selected fields over deterministic snapshots. | Preserve the prototype results and record the simplified schema used. |
| Weeks 14-16 | Read historical Reth data through one consistent provider view, add typed error causes, and implement the selected EIP receipt producer and reference checks. | Integrate the provider and query layer, tighten error handling and proof context checks, and extend supported receipt selections. | Review the provider and query interfaces and extend the agreed receipt operations. |
| Post Week 16 | Finish integrating the selected receipt provider, codec and tree and begin legacy transaction layouts and retained withdrawal interfaces. | Expand receipt queries to cover values, whole objects, lengths, variant checks, presence checks, ranges and byte slices. Add bounded requests with multiple selections and establish gas, fee and summary formats. | Reconcile producer and consumer sources and independently check the join. |
| Week 17 | Complete ten transaction layouts and preservation checks, reconcile current receipt construction and gas accounting, and expose components from real execution data. | Complete shared multiproofs, sparse and complete filters, component and transaction queries, receipt proof packaging, payload and request codecs and native block types. | Validate the combined implementation and replacement of superseded code.  |
| Week 18 | Integrate builder and Engine receiving rules, the selected fork and transport, and source persistence. Start canonical and compatibility tests. | Integrate sealing, block identity and real RPC and EVM callers. Finish the remaining supported proof targets and component links, finish receipt, log and cache handling, and test cancellation and bounds. | Check that the sources agree, using direct process runs and a compatible client route. |
| Week 19 | Connect the compatible Lighthouse route, pin both clients and fork inputs, and exercise production, import, activation, restart and canonical replacement. | Check queries against produced blocks, receipt proofs, component links, filters and associations. Measure construction, acquisition, serving and verification separately. | Validate produced blocks and independently check selected roots and proofs. |
| Week 20 | Final review and documentation | Complete the reports and also the presentation using the recorded devnet and benchmark results. | Review the final report and presentation. |

## Testing, Limits, and Risks

The project will test each layer: parser tests will validate paths, gindex and value tests will check traversal, root and proof tests will check verification, and RPC tests will cover the selected flow. The key integration test will prove a receipt or log field against a Reth fork root using the agreed schema and real inputs.

Exact bytes and nested roots, object roots and list roots will be compared with the pinned independent reference. This includes variants, order, malformed data, boundaries and missing inputs. Proof tests will check context for selectors and active fields, positions of selected fields, authenticated lengths, padding, ranges, unused witnesses and complete filters, including cases with no matches. They will also check rejection of out-of-bounds selections.

**STEEL and Execution Spec Tests.** We will track the Specifications and Testing for the Ethereum Execution Layer (STEEL) workflow and use relevant execution spec test vectors as they become available for the selected EIPs. Until then, our versioned SSZ object, root and proof vectors will be the reference suite for this prototype, ready for later comparison with or contribution to the wider execution spec test effort. We will record our fixtures and the applicable STEEL and EEST tests run separately.

**Direct Process Checks.** We will record real execution captures, supplied fixtures, unit and integration tests, the local HTTP example and a separate executable process check, identifying the setup used for each.

**Kurtosis Devnet Validation.** We will run a reproducible local Kurtosis devnet with two modified Reth/Lighthouse pairs and validators. Checks will cover block production, finality, reorgs, restart recovery and selected queries and proofs against produced blocks. Dora will help us observe everything thats happening.

**Benchmarks and Resource Limits.** We will measure receipt construction, data access, query serving and proof verification separately, including response sizes and checks under invalid or concurrent requests.

## Goals

- Produce deterministic SSZ execution object roots in a Reth fork, beginning with withdrawals and structured receipt and log sample objects, then the selected EIP receipts, transactions and block composition.
- Additive Reth query and proof interface for structured receipt and log paths and the agreed expanded selections.
- Return values, proofs, roots, and root context that a caller can check against an independently expected root, deriving the gindices from the request and schema.
- Integrate the QL layer with the Reth produced receipt root when the shared data path is ready.
- Connect the selected block objects to actual builder, Engine API, canonical identity, storage, RPC and EVM paths, and check them through the agreed tests against a separately running node and a compatible devnet.

## Collaborators

- [**Parth Singh:**](https://github.com/ParthSinghPS) SSZ withdrawal, receipt and transaction representations, codecs and roots, Reth source and provider mapping, retained execution sources, block building and validation, storage integration and receipt specification clarification.
- [**Arsh:**](https://github.com/ArshLabs) SSZ-QL parsing, queries, proofs and filters, gas and fee commitments, payload, summary and request formats, native block types, query and receipt RPC integration, elcl compatibility, devnet validation and benchmarking.

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
- [EIP 7932: Secondary Signature Algorithms](https://eips.ethereum.org/EIPS/eip-7932)
- [EIP 7928: Block access lists](https://eips.ethereum.org/EIPS/eip-7928)
- [Lighthouse source](https://github.com/sigp/lighthouse)
- [ethPandaOps Ethereum package for Kurtosis](https://github.com/ethpandaops/ethereum-package)
- [Dora explorer](https://github.com/ethpandaops/dora)
- [SSZ reference specifications](https://github.com/ethereum/ssz-specs)
- [Engine API specifications](https://github.com/ethereum/execution-apis)
- [SigP `ethereum_ssz` PR #67](https://github.com/sigp/ethereum_ssz/pull/67)
- [SigP `tree_hash` PR #43](https://github.com/sigp/tree_hash/pull/43)
- [STEEL execution spec tests](https://steel.ethereum.foundation/docs/execution-specs/tests/)
- [Kurtosis private Ethereum networks](https://geth.ethereum.org/docs/fundamentals/kurtosis)
- [Jun and Nando's SSZ-QL work in Prysm](https://hackmd.io/@junsong/Hyd7ElRJZx)
- [Reth RPC crates](https://github.com/paradigmxyz/reth/tree/main/crates/rpc)
- [Reth source](https://github.com/paradigmxyz/reth)
- [Fork's integration branch](https://github.com/ArshLabs/reth/tree/pureth-integration)
- [Draft PR](https://github.com/paradigmxyz/reth/pull/26881)
- [Tracking issue](https://github.com/paradigmxyz/reth/issues/26880)
- [Cohort development updates](https://github.com/eth-protocol-fellows/cohort-seven/blob/main/development-updates.md)

