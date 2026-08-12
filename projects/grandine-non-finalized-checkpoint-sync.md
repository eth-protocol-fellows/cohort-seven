# Grandine: Syncing from Non-Finalized Checkpoints

Enabling Grandine to bootstrap and sync from a trusted non-finalized checkpoint during prolonged periods of non-finality, using the least invasive implementation changes possible.

## Motivation

Grandine currently supports checkpoint sync only from finalized checkpoints. Under normal network conditions, this gives a beacon node a safe and efficient way to bootstrap without replaying the chain from genesis.

However, during an extended period of non-finality, the latest finalized checkpoint may become significantly older than the current wall-clock head. A newly started node would still have to begin from that old finalized checkpoint and process the full non-finalized range before catching up.

Supporting startup from a trusted non-finalized checkpoint closer to the current head could improve recovery and bootstrap time during prolonged non-finality.

This is a specialized recovery and testing feature rather than a replacement for normal finalized checkpoint sync. The beacon chain and Grandine’s current architecture are built around finalized checkpoints, so the new path would mainly be useful during exceptional failure scenarios, client testing, and non-finality recovery experiments.

## Project description

This project will investigate and implement support for starting Grandine from an operator-supplied trusted non-finalized checkpoint.

This is intended as a specialized recovery and testing path for prolonged non-finality scenarios. Finalized checkpoint sync will remain the normal and default startup method.

Because Grandine and the beacon chain are designed around finalized checkpoint startup, the implementation may affect several parts of the client, including:

- checkpoint loading and validation;
- startup and state initialization;
- fork-choice initialization;
- forward-sync boundaries;
- P2P finality reporting;
- storage and restart behaviour;
- block ancestry and rollback checks.

The work will follow an implementation-first approach. Rather than treating the current design ideas as fixed, I will begin with a small implementation spike, identify where finalized-checkpoint assumptions prevent startup or syncing, and shape the final design based on those findings.

The goal is to make the least invasive change that achieves the required functionality while preserving the existing finalized checkpoint-sync path.

> This feature introduces an explicit trust assumption: the operator is responsible for selecting a reliable non-finalized checkpoint source.

## Specification

The implementation will begin by extending or reusing Grandine’s existing checkpoint-selection and remote-loading paths.

### Initial implementation spike

The first stage will:

1. Allow the operator to select a checkpoint state or block other than `finalized`.
2. Download the selected beacon state and corresponding block.
3. Validate that the block and state match.
4. Validate that the checkpoint belongs to the expected network.
5. Pass the selected checkpoint through Grandine’s existing startup path.
6. Record every place where the code assumes that the startup anchor is finalized.
7. Make the minimum changes needed to progress through initialization and forward sync.

A possible CLI interface is:

```bash
grandine \
  --checkpoint-sync-url <BEACON_NODE_URL> \
  --checkpoint-sync-state-id justified
```
or:
```bash
grandine \
  --checkpoint-sync-url <BEACON_NODE_URL> \
  --checkpoint-sync-state-id <SLOT>
```
The existing behaviour should remain unchanged when no state ID is provided:
```bash
grandine \
  --checkpoint-sync-url <BEACON_NODE_URL>
```
In that case, Grandine should continue using the latest finalized checkpoint.

### Checkpoint loading

The current loading path uses finalized semantics through logic similar to:
```rust
load_finalized_from_remote(...)
```
and requests:

```rust
BlockId::Finalized
```
The project will investigate generalizing this path so it can load a checkpoint using an explicit state or block identifier.

The downloaded data should be validated for:

- block and state consistency;
- network identity;
- supported checkpoint alignment;
- availability of the required block;
- valid finalized and justified checkpoint references.

### Startup and fork choice

Grandine currently initializes its finalized and justified checkpoints using the startup anchor. This is correct when the anchor is finalized, but may be incorrect when starting from a trusted checkpoint ahead of finality.

The implementation will investigate how to preserve:

```rust
state.finalized_checkpoint()
state.current_justified_checkpoint()
```
while separately enforcing the trusted startup point where required.

Possible solutions may include a local irreversible checkpoint or a generalized anchor representation, but these will only be introduced if the implementation demonstrates that they are necessary.

### Sync behaviour

Forward sync must not attempt to operate below a trusted startup point for which the node does not have usable pre-state data. The project will investigate whether the current forward-sync lower bound needs to use a local startup anchor rather than the older finalized checkpoint.

Historical back-sync will not be a priority in the initial implementation unless it is required for correct startup or forward syncing.

### P2P and API semantics

Grandine must not advertise the trusted non-finalized checkpoint as protocol-finalized.

P2P status messages should continue reporting the real:

```rust
finalized_root
finalized_epoch
```
from the on-chain finalized checkpoint.

Beacon API identifiers such as:
```rust
StateId::Finalized
BlockId::Finalized
```
must also retain their existing protocol meaning.

### Storage and restrat

The implementation will investigate whether the trusted checkpoint and any additional metadata must be persisted separately. If restart support requires persistence, the node should restore:

- the trusted startup point;
- the real finalized checkpoint;
- the real justified checkpoint;
without incorrectly storing the trusted checkpoint as finalized history.

### Testing

Tests will be added alongside each implementation stage. The initial testing scope includes:

- finalized checkpoint-sync regression tests;
- non-finalized block and state loading;
- block/state mismatch rejection;
- wrong-network checkpoint rejection;
- startup from a trusted non-finalized checkpoint;
- correct finalized and justified checkpoint handling;
- correct P2P status reporting;
- forward-sync boundary behaviour;
- ancestry checks where required;
- restart behaviour where supported.

The detailed research and possible design directions are available in the [Draft Design Document](https://hackmd.io/@dicethedev/H15xe3M4zl).

## Roadmap

The roadmap is intentionally implementation-driven and may change based on findings and Grandine team feedback.

### Weeks 6-7: Implementation spike
- Add or reuse checkpoint state/block selection.
- Load a selected non-finalized checkpoint.
- Validate the checkpoint block and state.
- Pass the checkpoint through the current startup path.
- Identify finalized-checkpoint assumptions.
- Preserve existing finalized checkpoint-sync behaviour.
- Open the first small PRs for review.

### Weeks 8-9: Minimal startup support
- Make the minimum changes required for initialization.
- Preserve real on-chain finalized and justified checkpoints.
- Investigate whether a separate local startup boundary is required.
- Add focused startup and fork-choice tests.
- Refine the design based on findings and review feedback.

### Weeks 10-11: Sync and protocol behaviour
- Update forward-sync boundaries where necessary.
- Audit P2P finalized-checkpoint reporting.
- Validate ancestry and rollback behaviour.
- Investigate storage and restart requirements.
- Add integration and negative tests.

### Week 12: Documentation and integration
- Document the implemented behaviour and trust assumptions.
- Document unsupported cases and known limitations.
- Update the draft design document with implementation findings.
- Address outstanding review feedback.
- Prepare the implementation for integration where feasible.

### Stretch goals
Depending on implementation complexity and review feedback:

- restart support from a non-finalized checkpoint;
- support for state-root or head identifiers;
- limited historical back-sync;
- local-anchor metrics and logging;
- a debug endpoint exposing startup-anchor information;
- cross-client checkpoint compatibility tests.

## Possible challenges

### Finality assumptions across the codebase

Grandine was designed to start from finalized checkpoints. Finality assumptions may be spread across fork choice, storage, sync, P2P, and startup components.

Some of these assumptions may only become visible during implementation.

### Local trust versus protocol finality

The trusted startup checkpoint may be ahead of the protocol’s actual finalized checkpoint. Grandine must avoid using or reporting the trusted checkpoint as if it were finalized.

### Epoch alignment

Grandine’s current startup flow expects an epoch-start anchor. The first implementation may need to restrict non-finalized checkpoints to epoch-aligned points.

### Storage and restart

A trusted non-finalized checkpoint may need to survive restart without being persisted as finalized chain history.

### Historical back-sync

Back-sync from a non-finalized point may require additional trust and state-availability rules. It may need to remain outside the initial scope.

> [!WARNING]
> A non-finalized checkpoint is an explicitly trusted local input. Unlike a finalized checkpoint, it cannot be established as canonical by protocol finality alone.
>
> Operators using this feature must trust the checkpoint source and understand that selecting an incorrect or conflicting checkpoint may cause the node to follow the wrong chain.
>
> This feature is intended for controlled recovery, testing, and exceptional non-finality scenarios—not as the default checkpoint-sync mode.

### Testing non-finality scenarios

Testing prolonged non-finality and recovery may require custom fixtures, mocked remote endpoints, or dedicated integration environments.

### Scope growth

Implementation findings may reveal that supporting non-finalized startup requires more changes than initially expected. The project will prioritize the smallest safe and reviewable solution.


## Goal of the project

The project will be considered successful if Grandine can start and sync forward from an operator-supplied trusted non-finalized checkpoint without incorrectly treating that checkpoint as protocol-finalized.

The expected result should:

- preserve the existing finalized checkpoint-sync path;
- load and validate a trusted non-finalized checkpoint;
- initialize the node from that checkpoint;
- preserve the real on-chain justified and finalized checkpoints;
- avoid advertising false finality through P2P or Beacon API behaviour;
- use safe sync boundaries;
- include focused unit and integration tests;
- document the feature’s trust assumptions and limitations;
- be delivered through small, reviewable pull requests.

If full support cannot be completed within the fellowship period, the project should still produce a tested implementation spike, clearly documented blockers, and a concrete path for completing the remaining work.

##  Collaborators

### Fellows 

- [Blessing Samuel](https://github.com/dicethedev)

### Mentors 

- [Saulius Grigaitis](https://github.com/sauliusgrigaitis)
- Grandine team members providing implementation and review feedback

## Resources

### Grandine
- [Grandine repository](https://github.com/grandinetech/grandine)
- [Grandine checkpoint-sync documentation](https://docs.grandine.io/checkpoint_sync.html)
- [Grandine PR #661 - Various checkpoint sync centric fixes](https://github.com/grandinetech/grandine/pull/661)
- [Grandine PR #144 - Custom-slot checkpoint sync](https://github.com/grandinetech/grandine/pull/144)
- [Grandine Issue #48 - Teku checkpoint-sync compatibility](https://github.com/grandinetech/grandine/issues/48)
- [Grandine PR #607 - Checkpoint state during sync](https://github.com/grandinetech/grandine/pull/607)
- [Grandine PR #609 - Store checkpoint state every archival epoch interval](https://github.com/grandinetech/grandine/pull/609)

### Design, research and EPF7 updates
- [Draft Design Document](https://hackmd.io/@dicethedev/H15xe3M4zl)
- [EPF7 Week 2 Update](https://hackmd.io/@dicethedev/ryFx3knXMg)
- [EPF7 Week 3 Update](https://hackmd.io/@dicethedev/Skg-ZdMEMx)
- [EPF Week 4 Update](https://hackmd.io/@dicethedev/rJm1y3wEfl)
- [EPF Week 5 Update](https://hackmd.io/@dicethedev/HyyHatiVGe)

### Lighthouse references
- [Lighthouse Issue #7089 - Non-finalized state sync](https://github.com/sigp/lighthouse/issues/7089)
- [Lighthouse PR #8382 - Safe non-finalized checkpoint sync](https://github.com/sigp/lighthouse/pull/8382)
- [Lighthouse checkpoint-sync documentation](https://lighthouse-book.sigmaprime.io/checkpoint-sync.html)

### Ethereum specifications

- [Fork-choice specification](https://ethereum.github.io/consensus-specs/phase0/fork-choice/)
- [Weak subjectivity specification](https://ethereum.github.io/consensus-specs/phase0/weak-subjectivity/)
- [Consensus-layer P2P Status message](https://ethereum.github.io/consensus-specs/phase0/p2p-interface/#status-v1)
- [Beacon API specification](https://ethereum.github.io/beacon-APIs/)