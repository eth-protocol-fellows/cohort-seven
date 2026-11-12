# State tiering for the Partitioned Binary Tree (EIP-8435)

This project defines the `last_written_block` write-age signal inside the account and storage leaves of the Partitioned Binary Tree.

## Motivation

Ethereum's state keeps growing as users create new accounts and storage slots. The article "[Not All State is Equal](https://ethereum-magicians.org/t/not-all-state-is-equal/25508)" analyzes the state from the genesis block to block 22,431,083 and finds it uneven. Most contracts go inactive right after creation, a majority of storage slots are never touched again after their first write, and a small number of contracts and factories account for most of the state. Around 80% of the full node state has not been accessed in over a year ([EIP-8188 ACD Slides](https://weiihann.github.io/20260521-eip8188-acd/)).

Clients can use a write-age signal to split the state into a "hot" part, written recently, and a "cold" part, not written in a long time. That lets them optimize the commit-critical mutable path: trie updates, state-root computation and other work that repeatedly rewrites state. [EIP-8188](https://eips.ethereum.org/EIPS/eip-8188) defines the signal as a consensus-verified `last_written_block` field on every account and storage slot, and state tiering designs such as [EIP-8295](https://eips.ethereum.org/EIPS/eip-8295) build on it.

EIP-8188 defines the field for the Merkle Patricia Trie (MPT) in RLP, as a fifth element in the account list and a two-element list around each storage slot. That design has two problems:

1. **EIP-8188 was Declined for Inclusion in Hegotá** ([EIP-8081](https://eips.ethereum.org/EIPS/eip-8081)), so the MPT version of the signal will not ship in that fork.
2. **The state tree is expected to change.** [EIP-8297](https://eips.ethereum.org/EIPS/eip-8297) proposes replacing the hexary MPT with a Partitioned Binary Tree (PBT), expected around the I* fork.

The PBT has no RLP. It packs account fields at fixed byte offsets in one 32-byte `BASIC_DATA` leaf, a storage leaf holds only the raw 32-byte word, and there is no `storageRoot`. So EIP-8188's encoding cannot carry over to the new tree. For the tiering signal to exist after the switch, an EIP has to define it for the PBT. That EIP should activate together with the PBT, so the tree has the right layout from the start.

## Project description

[EIP-8435: Last-Written Block in PBT Leaves](https://eips.ethereum.org/EIPS/eip-8435), co-authored with my mentor, puts `last_written_block` into the leaves of the PBT:

* For accounts, the EIP packs `last_written_block` into the existing `BASIC_DATA` leaf, so accounts grow by zero bytes.
* For storage slots, the leaf value grows from 32 to 36 bytes. It holds the unchanged slot value followed by a 4-byte block number.
* The update rules follow EIP-8188, except that a storage write updates only the slot's field, not the account's.
* The field is part of each leaf's committed value, so a client can read it from the leaf and verify it with a normal inclusion proof.

EIP-8435 is standalone. It does not require EIP-8188 and can activate whether or not EIP-8188 ever activates on the MPT. It requires [EIP-7954](https://eips.ethereum.org/EIPS/eip-7954), [EIP-8037](https://eips.ethereum.org/EIPS/eip-8037) and [EIP-8297](https://eips.ethereum.org/EIPS/eip-8297). It was merged into the [EIPs repo](https://github.com/ethereum/EIPs) as a Draft on October 24, 2026.

## Specification

### Account encoding

EIP-8297 packs the `BASIC_DATA` leaf as follows:

| Name        | Offset | Size |
| ----------- | ------ | ---- |
| `version`   | 0      | 1    |
| `reserved`  | 1      | 3    |
| `code_size` | 4      | 4    |
| `nonce`     | 8      | 8    |
| `balance`   | 16     | 16   |

EIP-8435 changes it to:

| Name                 | Offset | Size |
| -------------------- | ------ | ---- |
| `version`            | 0      | 1    |
| `last_written_block` | 1      | 4    |
| `code_size`          | 5      | 3    |
| `nonce`              | 8      | 8    |
| `balance`            | 16     | 16   |

`last_written_block` takes the three reserved bytes plus one byte freed by narrowing `code_size` from four bytes to three. Three bytes still hold up to 16,777,215 bytes, about 256 times the 64 KiB code size limit of [EIP-7954](https://eips.ethereum.org/EIPS/eip-7954). The earlier unified binary tree design, [EIP-7864](https://eips.ethereum.org/EIPS/eip-7864), used the same `code_size` width and offset.

### Storage encoding

Every storage slot leaf, both header slots 0 to 63 and slots in the storage zone, holds a 36-byte value:

| Name                 | Offset | Size |
| -------------------- | ------ | ---- |
| `slot_value`         | 0      | 32   |
| `last_written_block` | 32     | 4    |

The slot value stays at offset 0, so `SLOAD` reads the first 32 bytes as before. A write of zero still deletes the leaf, metadata included. EIP-8188 estimated about 1.5 billion slots in early 2026. Across those slots this adds about 6 GB of raw data, less than the roughly 9 GB EIP-8188 adds with its RLP framing. With BLAKE3, a storage leaf's hash input is at most 103 bytes, which still fits in two 64-byte blocks.

### Update rules

Each state change sets `last_written_block` to the current block number:

* **Storage.** A value change or a new slot sets the slot leaf's field. Deleting a slot removes the leaf and its metadata. A no-op write changes nothing.
* **Accounts.** These events set the account's field: nonzero balance transfers for sender and receiver, nonce increments including [EIP-7702](https://eips.ethereum.org/EIPS/eip-7702) authorizations, nonzero priority fee credits to the fee recipient, nonzero [EIP-4895](https://eips.ethereum.org/EIPS/eip-4895) withdrawals, new accounts, and `SELFDESTRUCT` balance changes under [EIP-6780](https://eips.ethereum.org/EIPS/eip-6780).
* **Reads and reverts.** Reads never update the field. Reverts restore it together with its leaf.
* **No storage-to-account cascade.** In the MPT, every storage write already rewrites the account because the account stores `storageRoot`. The PBT has no `storageRoot`, so a cascade would add an extra `BASIC_DATA` write for every storage write. Each leaf records its own last write instead.

### Retrieval

A client can read the `last_written_block` of any account or storage slot at any time. Reading it needs only the leaf, with no transaction execution, no side index and no state change. The read works the same way from a client's primary database, a cold store, a snapshot or a block witness. `leaf_hash` covers the field, so a standard EIP-8297 inclusion proof also proves it.

### State-creation gas and activation

* EIP-8435 increases `STATE_BYTES_PER_STORAGE_SET` from [EIP-8037](https://eips.ethereum.org/EIPS/eip-8037) by 4 bytes. `STATE_BYTES_PER_NEW_ACCOUNT` does not change.
* EIP-8435 should activate in the same fork as EIP-8297. Activating it later would mean rewriting every `BASIC_DATA` leaf and every storage leaf.
* State that exists at activation has `last_written_block = 0` until its first write.

### Key design decisions

* **Why 4 bytes?** Three bytes (16,777,215) is already below mainnet's block height. Four bytes last until block `2^32`, about 1,600 years at 12-second slots.
* **Why the raw block number?** A client can derive any coarser value a tiering or pricing scheme needs, such as EIP-8295 periods, from the block number, but not the other way round.
* **Why pack into `BASIC_DATA`?** Every account access already opens that leaf. A separate header leaf would add one leaf per account and one extra write per account update.
* **Why widen the storage leaf?** All 32 bytes of a storage word are meaningful, so there is no room to pack the field. A parallel metadata leaf would double the number of storage leaves. Widening keeps the slot and its metadata in one leaf, so clients read, write, delete and prove them together.

## Roadmap

| Phase | Weeks | Deliverables |
| --- | --- | --- |
| 1: Design | 15-16 | Account and storage leaf encoding, update rules, retrieval and the state-creation gas change. EIP draft, PR to the EIPs repo and a discussion thread on Ethereum Magicians. |
| 2: Review and merge | 17-20 | Pass the EIPs repo CI (`eipw`, markdownlint, codespell), go through EIP editor review and merge as a Draft on October 24, 2026. Gather feedback from PBT contributors and the Magicians thread. |

> **Note.** I spent weeks 0-14 on the original version of this proposal, a Reth implementation of EIP-8188 state tiering, before the project changed direction in week 15.

## Possible challenges

* **EIP-8297 is still a Draft.** The PBT layout and its hash function are not final. EIP-8435 has to track any changes to `BASIC_DATA`, the storage leaf or the hashing.
* **Timeline.** EIP-8435 is meant to ship with EIP-8297, which is expected around the I* fork, roughly two years away. Presenting it at ACD only makes sense closer to that point. Until then, the main task is keeping the EIP current and getting feedback.
* **Fixed width.** Unlike EIP-8188's RLP field, a 4-byte field cannot grow on its own. A later fork must widen it before mainnet reaches block `2^32`.
* **Test vectors.** Pinning digests has to wait until EIP-8297 chooses the merkelization hash.
* **Feedback.** EIP-8435 needs input from the people designing the PBT, so that it fits the tree they are building.

## Goal of the project

Success looks like:

* **EIP-8435 merged into the EIPs repo** as a Draft that defines the account and storage encoding, the update rules, retrieval and the state-creation gas change for the PBT. Merged on October 24, 2026.
* **A standalone design** that does not depend on EIP-8188 activating, and that state tiering proposals such as EIP-8295 and EIP-8296 can use.
* **An open discussion** on Ethereum Magicians, with feedback from people working on the PBT.
* **Consideration for inclusion** of EIP-8435 together with EIP-8297, in the longer term.

## Collaborators

### Fellows

[Astrion](https://github.com/astrion-coder)

### Mentors

[Wei Han Ng](https://github.com/weiihann)

## Resources

* [EIP-8435: Last-Written Block in PBT Leaves](https://eips.ethereum.org/EIPS/eip-8435)
* [EIP-8435 PR](https://github.com/ethereum/EIPs/pull/12420)
* [EIP-8435 discussion thread](https://ethereum-magicians.org/t/eip-8435-last-written-block-in-pbt-leaves/29860)
* [EIP-8297: Partitioned Binary Tree](https://eips.ethereum.org/EIPS/eip-8297)
* [EIP-8188: Last-Written Block for Accounts and Slots](https://eips.ethereum.org/EIPS/eip-8188)
* [EIP-8295: State Tiering by Periods](https://eips.ethereum.org/EIPS/eip-8295)
* [EIP-8081: Hardfork Meta - Hegotá](https://eips.ethereum.org/EIPS/eip-8081)
* [EIP-7864: Ethereum state using a unified binary tree](https://eips.ethereum.org/EIPS/eip-7864)
* [Not All State is Equal](https://ethereum-magicians.org/t/not-all-state-is-equal/25508)
* [EIP-8188 ACD Slides](https://weiihann.github.io/20260521-eip8188-acd/)
