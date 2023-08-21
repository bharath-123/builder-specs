# Deneb -- Assembler Specification

## Table of Contents

<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->

- [Introduction](#introduction)
- [TOB and ROB validation](#containers)
- [Final bid building](#containers)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->

## Introduction

The assembler is a component which is used in parallel block auctions based proposer builder commitments. It is used to concatenate the txs of multiple
bids and build an aggregated execution payload. This aggregated execution payload is sent to the relayer. The relayer then builds an aggregated bid using the
sent aggregated execution payload.
It is also responsible for validating the txs of the multiple bids for state interference.
It can be implemented as an RPC call in an execution client. Although the execution client would have to be stripped of from the following capabilities:
1. Receiving txs to its mempool. The payloads only need to be built by txs sent by the relayer.
2. Building payloads through the engine api. Payloads should only be built when the bids are sent by the relayer.
3. Receiving txs from peers. 

It should only be responsible for:
1. Validating the TOB and ROB bid txs for state interference
2. Creating an aggregated execution payload from the merged tx lists of the TOB and ROB bid

It should be connected to a beacon node client to stay up to date with the latest block. It should be synced.

## TOB and ROB Validation

Below we define a method which takes in 2 bids. A TOB(Top of block) bid and ROB(Rest of block) bid. We can expand it to n bids later
in the future. 

We also define a few helper methods

TODO - Define state interference methods. We can black box them for now in `check_state_interference`

```python
def is_tx_blob(tx: Transaction) -> bool:
    pass
```

```python
def is_tx_uniswap_swap(tx: Transaction) -> bool:
    pass
```

```python
def is_tx_in_bundle(tx: Transaction) -> bool:
    pass
```

```python
def check_state_interference(tob_bid_execution_payload: ExecutionPayload, rob_bid_execution_payload: ExecutionPaylod):
    """
    Ensure tob bid has only uniswap swap txs at the top. It shouldn't have bundles, blob txs and meta txs.
    Ensure rob bid has no unswap swap txs. We can include bundles and blob txs.
    This method can be treated as a black box and can be evolved as we go along with developing pepc-boost.
    """    
    # TODO - Figure out how to check for bundle
    # TODO- Develop this method as we go along    
    # check that there are no blob bundles for tob_bid
    assert len(tob_bid.blob_bundles) == 0
    
    tob_bid_txs = tob_bid_execution_payload.transctions
    rob_bid_txs = rob_bid_execution_payload.transactions

    for tob_bid_tx in tob_bid_txs:
        assert !is_tx_blob(tob_bid_tx)
        assert is_tx_uniswap_swap(tob_bid_tx)
        assert !is_tx_in_bundle(tob_bid_tx)
    
    for rob_bid_tx in rob_bid_txs:
        assert !is_tx_uniswap_swap(rob_bid_tx)
```

Below we define a method to check if the tob and rob builder bids which are to be sent to the assembler to build an aggregate builder bid are compatible and valid.

```python
def validate_bids(signed_tob_bid: BuilderBid, tob_bid_execution_payload: ExecutionPayload, signed_rob_bid: BuilderBid, rob_bid_execution_payload: ExecutionPayload, validator_registration: ValidatorRegistrationV2) -> bool:
    tob_bid = signed_tob_bid.message
    rob_bid = signed_rob_bid.message
    assert tob_bid.pubkey != rob_bid.pubkey # the bids shouldn't be from the same builder
    assert tob_bid.header.parent_hash == rob_bid.header.parent_hash
    assert tob_bid.header.timestamp == rob_bid.header.timestamp
    # check gas limit
    assert tob_bid.header.gas_used < validator_registration.gas_limit / 2
    assert rob_bid.header.gas_used < validator_registration.gas_limit / 2
    # we can avoid checking blob gas since we are restricting blobs only to ROB and also given that blobs work in a seperate gas fee market
    tob_bid_txs = tob_bid_execution_payload.transctions
    rob_bid_txs = rob_bid_execution_payload.transactions
    no_of_tob_bid_txs = len(tob_bid_txs)
    no_of_rob_bid_txs = len(rob_bid_txs)
    tob_validator_payout = tob_bid_txs[no_of_tob_bid_txs - 1]
    rob_validator_payout = rob_bid_txs[no_of_rob_bid_txs - 1]
    assert tob_validator_payout.value == tob_bid.value
    assert rob_validator_payout.value == rob_bid.value

    check_state_interference(signed_tob_bid, signed_rob_bid)
```

## Final Bid Building

Below we define how to merge the tob_bid and rob_bid to get the transaction list which should be used to apply to the state and create a block out

```python
def merge_txs(tob_bid_execution_payload: ExecutionPayload, rob_bid_execution_payload: ExecutionPayload) -> Transaction[]:
    tob_bid_txs = tob_bid_execution_payload.transactions
    rob_bid_txs = rob_bid_execution_payload.transactions

    return tob_bid_txs + rob_bid_txs
```

We do not specify how to get the final payload from the tx list obtained from the above. That activity is left to the implementor. The implementor of the assembler
has to set the fee recipient of the block to the relayers address. All the MEV rewards should go the relayer. The relayer should be responsible for paying out the validator
and the builder. The builder can add the validator and builder payouts to the end of the TOB and ROB bid. 

Below we define how to merge the bids and get an aggregated builder bid. In this stage, we also have the latest execution payload header which is 
built from the tx list generated from merge_txs. The final bid pubkey is a BLS aggregate of the tob builder pub key and rob builder pub key 

```python
def merge_bids(signed_tob_bid: SignedBuilderBid, signed_rob_bid: SignedBuilderBid, aggregate_payload_header: ExecutionPayloadHeader, final_blinded_blobs_bundle: BlindedBlobsBundle) -> SignedBuilderBid:
    tob_bid = signed_tob_bid.message
    rob_bid = signed_rob_bid.message
    
    new_bid = BuilderBid(
        header=aggregate_payload_header,
        blinded_blobs=final_blinded_blobs_bundle,
        value = tob_bid.value + rob_bid.value,
        pubkey = bls.Aggregate(tob_bid.pubkey, rob_bid.pubkey)
    )
    
    new_signed_bid = SignedBuilderBid(
        message = new_bid,
        signature = bls.Aggregate(signed_tob_bid.signature, signed_rob_bid.signature)
    )

    return new_signed_bid
```

The assembler will return the aggregated payload which is obtained when we apply the tx list we received by merging the tob and rob tx list. We can derive
the payload headers from the final payload. Using this, we can merge the tob and rob bids to generate the final bid. This final bid is an aggregation 
of the tob and rob bid. It contains the aggregated payload header, the total value of both bid, and the aggregated BLS pubkeys and signatures.