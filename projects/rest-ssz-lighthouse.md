# REST-SSZ Engine API for Lighthouse

Add the new [REST-SSZ Engine API transport](https://github.com/ethereum/execution-apis/pull/793) transport to Lighthouse, alongside the existing JSON-RPC transport

## Motivation

The Engine API is the interface between the consensus layer and the execution layer, and the consensus layer drives the execution layer over it by importing blocks, updating the head, producing payloads, and fetching blobs. This link sits on the critical path for block propagation and validation timing, so its throughput and latency directly affect how quickly a node can process and attest to a block. Its current transport is JSON-RPC, which introduces overhead along that path in several ways:

1. JSON-RPC cannot carry raw bytes, so every hash, transaction, and blob is hex-encoded into an ASCII string roughly twice the size of the original data. This inflates every payload exchanged between the layers and saturates the connection between clients under heavy load.

2. The consensus layer holds its data as SSZ, so each call serializes those objects into JSON text and the execution layer deserializes them back into binary on receipt. This repeated encoding and decoding consumes CPU on both sides during the most latency-sensitive moments of block processing.

3. Packaging and parsing these inflated payloads eat into each slot's fixed window, and a block that lands too late is forked out and costs the proposer its reward.

Put together, these inefficiencies force the node to spend additional bandwidth, CPU, and memory to move the same block data. This overhead adds latency to every slot and limits how far the transport can be optimized.

## Project description

The project implements a REST-SSZ Engine API transport for Lighthouse to address the overhead caused by JSON-RPC. It does not replace JSON-RPC. Instead, REST-SSZ becomes the primary transport, and JSON-RPC remains as a fallback for execution clients that have not implemented the server-side REST-SSZ API. The JSON-RPC path stays fully operational, and both transports run on the same port. Since Lighthouse is a consensus layer client, this work covers the client side of the REST-SSZ Engine API.

The implementation will follow two specifications:
- [`REST-SSZ spec, PR #793`](https://github.com/ethereum/execution-apis/pull/793)
- [`new-payload-with-witness endpoint, PR #773`](https://github.com/ethereum/execution-apis/pull/773), which is being brought in line with PR #793 in [this draft](https://github.com/MariusVanDerWijden/execution-apis/pull/1)

### The current Engine API implementation in Lighthouse

```
        ┌───────────────────────────────────────────────────────┐
        │  Other Lighthouse crates (beacon_chain)               │
        └────────────────────────────┬──────────────────────────┘
                                     │ external crates call the execution_layer public interface
        ┌────────────────────────────▼──────────────────────────┐
        │  ExecutionLayer (lib.rs)                              │
        │  public interface + safety checks                     │
        └────────────────────────────┬──────────────────────────┘
                                     │ internal to the execution_layer crate
        ┌────────────────────────────▼──────────────────────────┐
        │  Engine (engines.rs)                                  │
        │  health, payload-id cache, forkchoice replay          │
        └────────────────────────────┬──────────────────────────┘
                                     │ self.api
        ┌────────────────────────────▼──────────────────────────┐
        │  JSON-RPC (http.rs + json_structures.rs)              │
        │  rpc_request chokepoint, JWT, version dispatch        │
        └────────────────────────────┬──────────────────────────┘
                                     │ JSON-RPC over HTTP
        ┌────────────────────────────▼─────────────────────────┐
        │  Execution client                                    │
        └──────────────────────────────────────────────────────┘
```
The `execution_layer` crate isolates the wire format at the bottom of the stack. `ExecutionLayer` is the public interface and runs the local safety checks. It drives one `Engine` for health and caching, which owns the single JSON-RPC transport client where every call goes through the `rpc_request` chokepoint.

### The proposed Engine API implementation in Lighthouse

```
      ┌──────────────────────────────────────────────────────────────────────┐
      │ Other Lighthouse crates (beacon_chain)                               │
      └───────────────────────────────────┬──────────────────────────────────┘
                                          │ unchanged public interface
      ┌──────────────────────────────────────────────────────────────────────┐
      │ ExecutionLayer (lib.rs) + Engine (engines.rs)                        │
      └───────────────────────────────────┬──────────────────────────────────┘
                                          │ Engine.api
      ┌──────────────────────────────────────────────────────────────────────┐
      │ EngineApi seam (engine_api/transport.rs)    [new]                    │
      │ resolves transport once, then dispatches per call                    │
      └───────────────────────────────────┬──────────────────────────────────┘
                                          │
                           ┌──────────────▼─────────────┐
                           │ REST-SSZ resolved          │
                           │ in transport seam?         │
                           └────────┬───────────┬───────┘
                                yes │           │ no
                       ┌────────────┘           └────────────┐
                       │                                     │
      ┌────────────────▼───────────────┐    ┌────────────────▼───────────────┐
      │ REST-SSZ                       │    │ JSON-RPC                       │
      │ (rest.rs + ssz_structures.rs)  │    │ (http.rs + json_structures.rs) │
      │ [new]                          │    │ [reused, unchanged]            │
      │ SSZ body + fork header         │    │                                │
      └────────────────┴───────────────┘    └────────────────┴───────────────┘
                       └──────────────────┬──────────────────┘
                                          │ JSON-RPC or REST-SSZ over HTTP
                                          ▼
      ┌──────────────────────────────────────────────────────────────────────┐
      │ Execution client                                                     │
      └──────────────────────────────────────────────────────────────────────┘
```

Since everything above the transport is wire-format agnostic, REST-SSZ slots in as a new sibling client. The API transport client that is set in `Engine` will be a new seam named `EngineApi` in place of `HttpJsonRpc`. On the first healthy probe the seam resolves the transport once, preferring REST-SSZ and falling back to JSON-RPC when it is unavailable, and that choice stays frozen until the beacon node restarts. REST-SSZ sends SSZ bodies and selects the fork with an `Eth-Execution-Version` header.

## Specification

### Design principles

The work is additive and stays almost entirely within the `execution_layer` crate. Three principles keep it that way:

1. REST-SSZ sits behind a command-line flag. With the flag off, Lighthouse uses JSON-RPC only and with the flag on, Lighthouse chooses the transport once at startup and keeps that choice for the run.
2. The existing JSON-RPC path remains unchanged. REST-SSZ is added alongside it, so JSON-RPC keeps working exactly as before and remains the fallback when an execution client does not support REST-SSZ.
3. The transport choice stays contained inside the `execution_layer` crate, so the crates that consume it call the same public methods without knowing which transport served them. The existing safety checks sit above the transport and are reused unchanged.

### REST-SSZ Endpoints

The version-numbered JSON-RPC methods collapse onto a small set of REST endpoints:

| JSON-RPC method | REST endpoint |
|-----------------|---------------|
| `engine_newPayloadV{1..5}` | `POST /payloads` |
| `engine_forkchoiceUpdatedV{1..4}` | `POST /forkchoice` |
| `engine_getPayloadV{1..6}` | `GET /payloads/{id}` |
| `engine_getPayloadBodiesByHashV{1,2}` | `POST /{fork}/bodies/hash` |
| `engine_getPayloadBodiesByRangeV{1,2}` | `GET /{fork}/bodies?from=...&count=...` |
| `engine_getBlobsV{1..4}` | `POST /blobs/v{1..4}` |
| `engine_getClientVersionV1` | `GET /identity` |
| `engine_exchangeCapabilities` | `GET /capabilities` |

Hot-path calls send SSZ (`application/octet-stream`) bodies over a fork-invariant path and select the fork with an `Eth-Execution-Version` header instead of a method suffix. At the time of drafting the proposal, `eth_*` calls always use JSON-RPC but this may change as the spec is still evolving.

For example, an `engine_newPayloadV5` call becomes a `POST /payloads` with an SSZ body and the fork set in the header:

```console
# Request
$ curl -sX POST localhost:8551/engine/v1/payloads \
    -H "Eth-Execution-Version: amsterdam" \
    -H "Content-Type: application/octet-stream" \
    -H "Authorization: Bearer $JWT" \
    --data-binary @payload.ssz -i
# 584 bytes · SSZ ExecutionPayloadEnvelope

# Response: 200 · VALID
HTTP/2 200
content-type: application/octet-stream

# 41 bytes · SSZ PayloadStatus → VALID
```

### SSZ containers

Each REST-SSZ endpoint carries an SSZ request or response container. Many of these reuse SSZ types that already exist in the consensus code, while the rest are new containers introduced per the [REST-SSZ container spec](https://github.com/ethereum/execution-apis/pull/793/changes#diff-71312f2cbcee21d3ec2b149d19ebb624fd9f511ac6317c8b8568e09ee6edc6c5). Each new container converts to and from an existing internal type, so the verification and fork-choice logic downstream stays shared with the JSON-RPC path. Just as `json_structures.rs` holds the JSON wire types for the JSON-RPC transport, these new containers live in the new `ssz_structures.rs` file shown in the proposed flow chart above.

### Forkchoice serialization and payload id TTL

Under REST-SSZ, the `payload_id` is an opaque, server-assigned token returned by `POST /forkchoice` and marked `Cache-Control: no-store`, rather than a value the consensus layer derives and caches itself as it does over JSON-RPC. The token stays valid until the next forkchoice update replaces it, and the build can be fetched with more than one `GET` in the meantime.

This makes ordering matter, because "the next update" only has meaning if the updates happen in a definite order. For that reason the spec allows at most one `POST /forkchoice` in flight and has the execution client process requests in receive order. With that ordering in place, a superseded token is recovered by simply re-issuing the forkchoice update, and the consensus layer can trust that the token it gets back is the current one.

### Testing

Almost every Engine API test runs against `MockExecutionLayer`, a real `ExecutionLayer` wired over HTTP to an in-process mock engine whose backend works on typed Rust values rather than JSON. That backend is wire-agnostic, so the existing tests carry over to REST-SSZ once the pipeline learns the new transport. The work breaks down as:

- Unit tests for round-trips and conversions of the new SSZ containers.
- A refactor of the mock engine's request dispatcher into a transport-agnostic core, adding REST routes and a structured `/capabilities` responder beside the JSON ones, so the wider beacon-node suites run over REST-SSZ with few changes.
- Conformance checks on the SSZ bytes together with the `Eth-Execution-Version` header and GET path, the part with no JSON-RPC counterpart.
- Integration against real clients that already implement REST-SSZ ([Erigon](https://github.com/erigontech/erigon/pull/21729/), [Nethermind](https://github.com/NethermindEth/nethermind/pull/11887), [Ethrex](https://github.com/lambdaclass/ethrex/pull/6770)).

### Execution Witness

Stateless clients and zkVM provers need a block's execution witness, which today takes a separate call and lags a block behind. A new REST-SSZ endpoint returns the validation result and the SSZ witness together in one request, so those verifiers get both in a single round-trip. It follows the same REST-SSZ conventions as the rest of the transport and is defined in [`PR #773`](https://github.com/ethereum/execution-apis/pull/773), which is being updated to align with #793 in [this draft](https://github.com/MariusVanDerWijden/execution-apis/pull/1).

## Roadmap

**Weeks 5–7: Transport wiring & robustness.** Wire the `EngineApi` seam that resolves the transport once from a `/capabilities` probe and then dispatches each call, and add the robustness the spec calls for: serialized forkchoice updates, recovery of expired payload ids, and per-fork gating on bodies.

**Weeks 8–9: Test apparatus & mock coverage.** `MockExecutionLayer` pipeline to serve REST-SSZ and cover the new behaviors, keeping the existing JSON-RPC tests green.

**Weeks 10–11: Integration testing.** Test end to end against real execution clients that implement REST-SSZ (Erigon, Nethermind, Ethrex) and resolve interoperability issues.

**Weeks 12–13: Research & metrics.** Measure REST-SSZ against JSON-RPC on the devnets (wire size, parse and round-trip latency, and block-import throughput, especially for blob-heavy payloads), quantifying the improvement and flagging any regressions to fix.

**Weeks 14–15: Glamsterdam follow-ups.** Track Lighthouse's Glamsterdam PRs that affect the Engine API (e.g. [`get_blobs_v4` #9438](https://github.com/sigp/lighthouse/pull/9438) and [custody columns #9547](https://github.com/sigp/lighthouse/pull/9547)), and rework the SSZ containers for [EIP-7688](https://eips.ethereum.org/EIPS/eip-7688) progressive containers.

**Weeks 16–17: Execution-witness endpoint.** Implement the REST-SSZ payload-with-witness endpoint in Lighthouse and test it against a supporting execution client. 

**Weeks 18–20: Spec buffer.** Reserved time to absorb revisions to the still-evolving spec.

**Weeks 21+: Report, demo & handoff.** Final EPF report, demo and presentation, and maintainer handoff.

## Possible challenges

- **The spec is still evolving.** Some `execution-apis#793` details (token lifetime, request ordering, size limits) are unsettled. The design stays robust to how they resolve, and the weeks 18–20 buffer absorbs revisions.
- **External dependencies.** The Glamsterdam follow-ups land on their own timelines, and the witness endpoint waits on an execution client that supports it, so these are tracked follow-ups rather than blockers.
- **Cross-client interop.** Several execution clients already implement REST-SSZ (linked under Testing), so integration testing mainly surfaces discrepancies to triage or report upstream.

## Goal of the project

**Minimal goal.** REST-SSZ working end to end against a real execution client on local Kurtosis devnets.

**Main goal.** Full integration testing and a complete REST-SSZ implementation matching the finalized spec, merged into Lighthouse and ready for Glamsterdam.

**Stretch goal.** Use the execution-witness endpoint to explore stateless block building with an execution client and contribute that support to Lighthouse.

## Collaborators

### Fellows

[Sameer](https://github.com/SamAg19)

### Mentors

[Mac Ladson](https://github.com/macladson) - Lighthouse, Sigma Prime.

## Resources

- [`execution-apis#793`](https://github.com/ethereum/execution-apis/pull/793): the REST + SSZ Engine API transport proposal.
- [`execution-apis#773`](https://github.com/ethereum/execution-apis/pull/773): REST + SSZ payload validation with execution witness, the basis for the witness endpoint implemented in this project.
- [Lighthouse](https://github.com/sigp/lighthouse): the work lands in the `beacon_node/execution_layer` crate.

