# SSZ Query Language for Erigon

Implementing SSZ Query Language on Erigon execution client with proofs

## Motivation

Today, virtually every application that reads Ethereum does so by querying an RPC provider and simply believing the data it returns. When you call `eth_getBalance`, `eth_call`, or `eth_getLogs`, the response arrives with no cryptographic guarantee. The one escape hatch, `eth_getProof`, returns a MPT proof for account and storage slots, but it is narrow and method-specific: there is no uniform, composable way to request an arbitrary subset of an object's fields and receive a succinct, verifiable proof against a known root.

## Project description

SSZ is the right substrate for a general solution to the above stated problem. Unlike the MPT, every SSZ container, list, and vector has a deterministic Merkle tree layout with stable gindices. Because a field's position in the tree is statically derivable from the type schema, you can express a query as a set of paths into an SSZ object, resolve those paths to gindices, and return only the leaves plus the minimal set of sibling/branch nodes needed to recompute the root. This yields multiproofs: instead of N independent single-field proofs with heavily overlapping internal nodes, a single query returns the union of leaves and a deduplicated witness, verified in one pass against `hash_tree_root`.

An **SSZ Query Language** formalizes this: a compact, self-describing grammar for selecting fields that compiles to a gindex set. On the consensus side, the Prysm client has an in-progress SSZ-QL implementation ([OffchainLabs/prysm#15587](https://github.com/OffchainLabs/prysm/issues/15587)) that lets callers request specific fields of large `BeaconState`-family objects rather than fetching and re-hashing the entire structure. By resolving queries to gindices and returning batched multiproofs, it (a) sharply reduces response size and (b) reduces server-side work, since the client walks the Merkle tree once for the combined gindex set.

This project implements a new endpoint for SSZ Query Language with proof generation in **Erigon**. Delivering this on Erigon gives the ecosystem a production-grade execution-side reference for proof-backed data access.

## Specification

A new endpoint will be introduced which will accept POST requests over HTTP on the route `/eth/v0/execution/{block_id}/query`.

### Anchors

An anchor is a node from the merkle tree (can be both the `hash_tree_root` of the markle tree or a merkle subtree) that is trusted and proofs can be generated against it. **A request can have multiple anchors and every query leaf must be a descendant of the anchor provided.**

#### Examples

```
"anchor": "execution_block"
"anchor": "execution_payload.transactions"
"anchor": "execution_payload.receipts"
```

### Paths

A path is the primitive that addresses a specific location inside an anchored SSZ tree. This is the primitive that mentions which field we want to query.

#### Examples

```
.transactions[42].to
.transactions[42].value
.withdrawals[0:10]
.receipts[42].logs[0].topics[1]
```

### Filters

A filter is an expression evaluated on a list/vector and filters out elements to only return elements which match the criteria in the filter expression. Examples are `==, !=, >, <, in, &&, ||`.

#### Examples

```
"filter": "gas_price > 50e9 && to == 0xabc..."
"filter": "logs[*].topics[0] == 0xddf252ad..."
```

### Summaries

A summary replaces a subtree's expanded value with its `hash_tree_root`

#### Examples

```
{ 
    "anchor": "execution_payload",
    "path": ".transactions[42]",
    "summaries": [".input", ".access_list"] 
}
```

### Aliases

Alias names the current selection's index (or indices) so later can reference them. Back references use $<alias>.index inside a path.

#### Examples

```
{ 
    "path": ".transactions[*]", 
    "filter": ".to == 0xabc...", 
    "alias": "tx" 
},
{ 
    "path": ".receipts[$tx.index].logs" 
}
```

### Input Validation

For input validation, we need to check three conditions:
1. If anchor is valid
2. If the queried leaf is descendant of the anchor
3. If the filter is valid for the queried leaf

### Parameters

| Name          | Type    | Description                                                  |
|---------------|---------|-----------------------|
| `query`         | `object`  | A detailed query object.                                      |
| `include_proof` | `boolean` | Whether including a merkle proof or not. Default to false.    |
| `multiproof`    | `boolean` | Whether using multiproof method for efficiency. Default to false. |

### Response Structure

| Code | Description                                                          | Content              |
|------|-----------------------------------------------------------------------|----------------------|
| 200  | Success                                                                | Response with the schema |
| 400  | Invalid ID or malformed request                                       | N/A                  |
| 404  | Not found                                                              | N/A                  |
| 500  | Internal server error                                                  | N/A                  |
| 503  | Execution node is currently syncing and not serving requests on this endpoint | N/A         |

### Query Result

A successful query response returns an envelope containing `version`, `finalized`, and `data`.

- `version` — the EL fork/hardfork under which the queried object was produced (e.g. "cancun", "prague").
- `finalized` — whether the anchor block is finalized, per the execution node's latest forkchoice state.
- `data` — an array containing the query results, plus the resolved anchor root. In non-multiproof mode, this array contains one element per query item. In multiproof mode, it contains exactly one element.


| Field     | Type    | Description                                                        |
|-----------|---------|---------------------------------------------------------------------|
| version   | string  | The EL fork/hardfork version under which the queried object was produced. |
| finalized | boolean | Whether the anchor block is finalized per the execution node's forkchoice state. |
| data      | array   | Array of QueryResult objects, plus the resolved anchor root.       |

More detailed specs are available at [SSZ-QL specs for EL](https://hackmd.io/@SoarinSkySagar/rkwwvAMEzg)

## Roadmap

### Phase 1: Setup endpoint and write grammar parser to compile queries (week 6 - week 10)

- Register the new `POST /eth/v0/execution/{block_id}/query` route in Erigon's RPC layer
- Implement the query grammar parser for anchors, paths, filters, summaries, and aliases as defined in the specification.
- Perform input validation on dummy data (since EL blocks are not SSZ encoded right now): verify the anchor is valid, every queried leaf is a descendant of its anchor, and each filter is valid for the leaf it targets

### Phase 2: Wire up the API to Execution Blocks (week 11 - week 14)
- Ensure that [Erigon SSZ Execution Blocks and EIP-7807](./erigon-ssz-execution-blocks-eip-7807.md) project is complete to a point where actual EL block data can be used. We will be communicating regularly regarding our projects and the timelines so that it does not become a blocker to this project.
- Resolve `{block_id}` (block hash, number, `latest`, `finalized`) to the corresponding execution block and its SSZ-typed objects.
- Fetch the requested leaf values from Erigon's state for the compiled gindex set and return them (values-only, `include_proof=false`).
- Apply filter evaluation and alias back-references (`$<alias>.index`) to narrow list/vector results to matching elements.
- Populate the response envelope (`version`, `finalized`, `data`) with the resolved anchor root.

### Phase 3: Multiproof generation (week 15 - week 18)

- Generate single-leaf Merkle proofs against the anchor root when `include_proof=true`.
- Implement the `multiproof=true` path: walk the Merkle tree once over the combined gindex set and return the union of leaves plus a deduplicated witness.
- Support `summaries`, replacing an expanded subtree with its `hash_tree_root` so it contributes a single node to the proof.
- Ensure every proof is verifiable by recomputing `hash_tree_root` back to the anchor.

### Phase 4: Testing and debugging (week 19 - week 21)

- End-to-end and cross-verification tests: independently recompute `hash_tree_root` from returned witnesses to confirm proof correctness.
- Benchmark response size and server-side cost of multiproof vs. per-field proofs and full-object fetches, validating the efficiency claim.
- Test edge cases: empty results, invalid anchors/paths/filters, syncing node (503), and unfinalized blocks.
- Documentation of the query grammar, endpoint usage, and examples.

## Possible challenges

- Execution blocks are not implemented in Erigon right now, and this project depends on them. A delay in their implementation project may delay this project as well.
- There is no formal EIP for SSZ-QL yet, so I have my own proposal for that [here](https://hackmd.io/@SoarinSkySagar/rkwwvAMEzg). The specification may be unstable right now and could keep changing actively as requirements evolve.
- A concrete method is needed for proving the correctness of filtered results.

## Goal of the project

The project is considered successful when Erigon exposes a working SSZ Query Language endpoint that accepts a query, resolves it against an execution block's SSZ objects, and returns the requested values along with a verifiable Merkle proof.

Concretely, success means:

- **Functional endpoint**: the `POST /eth/v0/execution/{block_id}/query` route is implemented in Erigon, accepting the full query grammar (anchors, paths, filters, summaries, and aliases) and returning the specified response envelope (`version`, `finalized`, `data`).
- **Correct proofs**: for both single-leaf proofs (`include_proof=true`) and multiproofs (`multiproof=true`), every response is independently verifiable by recomputing `hash_tree_root` back to the anchor. This removes the need to trust the serving node.
- **Tested and documented**: the implementation is covered by end-to-end and cross-verification tests, and the query grammar and endpoint usage are documented with examples so other clients and applications can adopt them.

The scope of this project is the SSZ-QL endpoint and proof generation within Erigon; it depends on SSZ-encoded execution blocks being available (via the companion Execution Blocks work), and validates against dummy SSZ data until then.

## Collaborators

### Fellows 

* **Sagar Rana** ([@SoarinSkySagar](https://github.com/SoarinSkySagar))

### Mentors

* **Tamaghna Choudhuri** ([@RazorClient](https://github.com/RazorClient))
* **Etan Kissling** ([@etan-status](https://github.com/etan-status))

## Resources

- [Erigon Execution Client](https://github.com/erigontech/erigon)
- [Erigon SSZ Execution Blocks and EIP-7807](./erigon-ssz-execution-blocks-eip-7807.md)
- [SSZ Specs](https://github.com/ethereum/consensus-specs/tree/master/ssz)
- [SSZ-QL specs for EL](https://hackmd.io/@SoarinSkySagar/rkwwvAMEzg)
- [Prysm SSZ Query Language implementation](https://github.com/OffchainLabs/prysm/issues/15587)