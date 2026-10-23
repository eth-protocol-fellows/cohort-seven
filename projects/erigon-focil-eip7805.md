# Implementation of FOCIL (EIP-7805) in Erigon

## Motivation

- **FOCIL (Fork Choice Inclusion Lists)**, or **EIP-7805** has been introduced to strengthen Ethereum's censorship resistance. To prevent the deterioration of the network's censorship resistance properties, FOCIL enables validators to impose constraints on builders by force-including transactions in blocks via **Inclusion Lists**. This guarantees the inclusion of timely and valid transactions in a block, preventing MEV attacks. 
- FOCIL has been selected as the headliner feature for **Hegota** hard fork. This includes changes in both the execution and consensus layers. Erigon does not have support for FOCIL yet, and this project's goal is to have Erigon support FOCIL.

## Project description

FOCIL support will be implemented in Erigon in the following areas:

- **Block validation checks**: Implementation of `check_inclusion_list_transactions` equivalent for going through each transaction in the Inclusion List in post-block state.

- **Engine API**: Adding three new Engine API methods, `engine_getInclusionListV1`, `engine_newPayloadV6` and `engine_forkchoiceUpdatedV5`.

- **Building the IL**: Build the Inclusion List from the public mempool, keeping the size within the spec limits and adopting the best transaction selection strategy.

Official spec documents to be referred to during the implementation:

1. [FOCIL for Execution Layer](https://eips.ethereum.org/EIPS/eip-7805#execution-layer)
2. [Block validation check](https://github.com/ethereum/execution-specs/blob/5b33750dbc4e6bfb74c18a990a6669a86f7bc0a6/src/ethereum/forks/amsterdam/fork.py#L1187-L1251)
3. [Bogota fork Engine API methods](https://github.com/ethereum/execution-apis/blob/main/src/engine/bogota.md#methods)
4. [Proof of Concept by me](https://hackmd.io/@SoarinSkySagar/rkfLlFtFfx)

## Specification

### Block Validation Checks

We would need to implement the function with the following signature in `execution/protocol/inclusion_list.go`:

```go
func CheckInclusionListTransactions(
    evm *vm.EVM,
    gp *GasPool,
    signer *types.Signer,
    blockTxs types.Transactions,
    inclusionListTxs [][]byte,
) (bool, error)
```

This is a helper function that takes mainly the block transactions and IL transactions to go through each tx in the IL and validate it if it could have been included in the block if its not. If a given transaction is valid but has not been included, then `state_transition` function is supposed to return status `INCLUSION_LIST_UNSATISFIED`. Main checks for each tx in the IL within this function are supposed to be:

1. If an `inclusionListTxs[i]` is already included in the block, skip the tx.
2. If the block does not have enough gas remaining for inclusion of the `inclusionListTxs[i]`, skip the tx.
3. Validate `inclusionListTxs[i]` against the execution state by checking the nonce and balance of origin. If the tx is invalid, return true from the function. If its valid, return false.

### Engine API Changes

#### `engine_newPayloadV6`

##### New types:

```go
type PayloadStatusV2 struct {
    Status                      EngineStatus      `json:"status" gencodec:"required"`
	ValidationError         *StringifiedError `json:"validationError"`
	LatestValidHash         *common.Hash      `json:"latestValidHash"`
        InclusionListSatisfied  bool              `json:"inclusionListSatisfied"`
	CriticalError           error             `json:"-"`
}
```

Add a new method in `execution/engineapi/engine_api_methods.go`:

```go
func (e *EngineServer) NewPayloadV6(ctx context.Context, payload *engine_types.ExecutionPayload,
      expectedBlobHashes []common.Hash, parentBeaconBlockRoot *common.Hash,
      executionRequests []hexutil.Bytes, inclusionList []hexutil.Bytes,
) (*engine_types.PayloadStatusV2, error) {
      return e.newPayload(ctx, payload, expectedBlobHashes, parentBeaconBlockRoot, executionRequests, inclusionList, clparams.GloasVersion)
}
```

#### `engine_forkchoiceUpdatedV5`

##### New types:

```go
type ForkChoiceUpdatedResponseV2 struct {
	PayloadId       *hexutil.Bytes   `json:"payloadId"` 
	PayloadStatusV2 *PayloadStatusV2 `json:"payloadStatus"`
}
```

Add a new method in `execution/engineapi/engine_api_methods.go`:

```go
func (e *EngineServer) ForkchoiceUpdatedV5(ctx context.Context, forkChoiceState *engine_types.ForkChoiceState, payloadAttributes *engine_types.PayloadAttributes) (*engine_types.ForkChoiceUpdatedResponseV2, error) {
      return e.forkchoiceUpdated(ctx, forkChoiceState, payloadAttributes, clparams.<NewFork>Version)
}
```

#### `engine_getInclusionListV1`

##### New types:
```go
type InclusionListV1 struct {
      Transactions []hexutil.Bytes `json:"transactions" gencodec:"required"`
}
```

Add a new method in `execution/engineapi/engine_api_methods.go`:

```go
func (e *EngineServer) GetInclusionListV1(ctx context.Context, parentHash common.Hash) (*engine_types.InclusionListV1, error) {
      return e.getInclusionList(ctx, parentHash, clparams.GloasVersion)
}
```

**NOTE**: Apart from traditional Engine API methods, Erigon also provides REST SSZ Engine API in `execution/engineapi/sszrest_handler.go` which would also need to be updated with the new endpoints.

### Building the IL

The only spec limitation here is that the maximum size of IL can be 8 KiB. As for the strategy for selection of transactions from the public mempool, it would be decided as the project progresses as this has been left to the discretion of the implementor. Although the most probably option is selecting transactions based on priority fees.

## Roadmap

### Phase 1: Engine API & block validation (week 14 - 18)

- Implement `CheckInclusionListTransactions` function and call on the appropriate post-block state during execution.
- Implement the three new Engine API methods - `engine_newPayloadV6`, `engine_forkchoiceUpdatedV5` and `engine_getInclusionListV1`. For `engine_getInclusionListV1`, return dummy/empty data for now since IL building comes in the next phase.
- Write test cases for each of the above stated functions and methods.

### Phase 2: IL building (week 19 - 21)

- Decide the most appropriate strategy for selection of transactions.
- Implement the IL building mechanism by selecting transactions from the mempool.
- Write test cases for the said strategy to make sure the IL size does not exceed 8 KiB and the list is valid.


### Phase 3: Testing and Completion (week 22+)

- Add end-to-end tests and metrics
- Catch up with spec updates, if any
- Run Kurtosis devnets with other FOCIL supporting clients.
- Update Erigon docs with the new Engine API methods.

## Possible Challenges

- Implementing `engine_getInclusionListV1` would be challenging, as unlike the other two new methods this is an entirely new method without any precedent so all helper functions and logic have to be written from scratch.
- Since IL is not a field in the Execution block, it would require some trial-and-errors to figure out the best way to pass IL from the API method parameters to the state transition function due to Erigon's codebase design.
- Specs are still being updated for FOCIL so work will need to adapt to the changes.

## Goal of the project

FOCIL support in Erigon, fully spec-compliant.

The finished project should provide:
1. The new Engine API methods
2. All EL FOCIL specs matching with the implementation
3. Test cases for all changes
4. Devnet tested with other FOCIL-ready clients
5. Update Erigon docs with the new Engine API methods.

## Collaborators

### Fellows 

* **Sagar Rana** ([@SoarinSkySagar](https://github.com/SoarinSkySagar))

### Mentors

* **Milen, Erigon** ([@taratorio](https://github.com/taratorio))
  
## Resources

- [Erigon Execution Client](https://github.com/erigontech/erigon)
- [Erigon FOCIL issue](https://github.com/erigontech/erigon/issues/24106)
- [Erigon FOCIL PR](https://github.com/erigontech/erigon/pull/24135)
- [FOCIL for Execution Layer](https://eips.ethereum.org/EIPS/eip-7805#execution-layer)
- [Block validation check](https://github.com/ethereum/execution-specs/blob/5b33750dbc4e6bfb74c18a990a6669a86f7bc0a6/src/ethereum/forks/amsterdam/fork.py#L1187-L1251)
- [Bogota fork Engine API methods](https://github.com/ethereum/execution-apis/blob/main/src/engine/bogota.md#methods)
- [Proof of Concept by me](https://hackmd.io/@SoarinSkySagar/rkfLlFtFfx)
- [Prior work on Erigon](https://github.com/erigontech/erigon/pull/17045)