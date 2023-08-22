# Deneb -- Builder Specification

## Table of Contents

<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->

- [Introduction](#introduction)
- [Containers](#containers)
  - [New containers](#new-containers)
    - [`BlindedBlobsBundle`](#blindedblobsbundle)
    - [`BlindedBlobSidecar`](#blindedblobsidecar)
    - [`SignedBlindedBlobSidecar`](#signedblindedblobsidecar)
    - [`SignedBlindedBlockContents`](#signedblindedblockcontents)
    - [`BlobsBundle`](#blobsbundle)
    - [`ExecutionPayloadAndBlobsBundle`](#executionpayloadandblobsbundle)
  - [Extended containers](#extended-containers)
    - [`BuilderBid`](#builderbid)
    - [`ExecutionPayloadHeader`](#executionpayloadheader)
      - [`BlindedBeaconBlockBody`](#blindedbeaconblockbody)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->

## Introduction

This is the modification of the builder specification accompanying the Deneb upgrade.

## Containers

### New containers

#### `BlindedBlobsBundle`

```python
class BlindedBlobsBundle(Container):
    commitments: List[KZGCommitment, MAX_BLOBS_PER_BLOCK]
    proofs: List[KZGProof, MAX_BLOBS_PER_BLOCK]
    blob_roots: List[Root, MAX_BLOBS_PER_BLOCK]
```

#### `BlindedBlobSidecar`

```python
class BlindedBlobSidecar(Container):
    block_root: Root
    index: uint64
    slot: uint64
    block_parent_root: Root
    proposer_index: uint64
    blob_root: Root
    kzg_commitment: KZGCommitment
    kzg_proof: KZGProof
```

#### `SignedBlindedBlobSidecar`

```python
class SignedBlindedBlobSidecar(Container):
    message: BlindedBlobSidecar
    signature: BLSSignature
```

#### `SignedBlindedBlockContents`

```python
class SignedBlindedBlockContents(Container):
    signed_blinded_block: SignedBlindedBeaconBlock
    signed_blinded_blob_sidecars: List[SignedBlindedBlobSidecar, MAX_BLOBS_PER_BLOCK]
```

#### `BlobsBundle`

Same as [`BlobsBundle`](https://github.com/ethereum/consensus-specs/blob/dev/specs/deneb/validator.md#blobsbundle) in Deneb consensus specs.

#### `ExecutionPayloadAndBlobsBundle`

```python
class ExecutionPayloadAndBlobsBundle(Container):
    execution_payload: ExecutionPayload
    blobs_bundle: BlobsBundle
```

### Extended containers

#### `BuilderBid`

Note: `SignedBuilderBid` is updated indirectly. 

Note: In the case, the `proposer_builder_commitment` is `TOB_ROB_SPLIT` which indicates that the TOB and ROB of a block is built by 2 separate builders. 
The relayer will get seperate bids for the TOB and ROB part of the block. The final payload which the proposer will use will be assembled by the payload assembler. 
The `BuilderBid` will contain the `header` and `blinded_blobs_bundle` post the assembling of the TOB and ROB bids. 
The `value` is the sum of the TOB bid value, and the ROB bid value.
The `pubkey` is the BLS aggregate of the builder pubkeys. 
If the `proposer_builder_commitment` for the validator is `FULL_BLOCK`, then this is akin to the BuilderBids we have in the current MEV Relayer.

```python
class BuilderBid(Container): # if the validator preference is a TOB_ROB_SPLIT, then this bid is built when the assembler assembles the highest value TOB and ROB bid from different builders
    header: ExecutionPayloadHeader # [Modified in Deneb]
    blinded_blobs_bundle: BlindedBlobsBundle  # [New in Deneb]
    value: uint256 # if this bid is built by the TOB_ROB_SPLIT commitment, then this value is the sum of TOB bid value and ROB bid value.
    pubkey: BLSPubkey # if this bid is built by the TOB_ROB_SPLIT commitment, then the pubkey is a BLS aggregate of the builder keys.
```

##### `SignedBuilderBid`

Note: In the case, the `proposer_builder_commitment` is `TOB_ROB_SPLIT`
The `signature` field is the BLS aggregate signature of the builder signatures.

```python
class SignedBuilderBid(Container):
    message: BuilderBid
    signature: BLSSignature # if this bid is built by the TOB_ROB_SPLIT, then this signature is the BLS aggregate of the builder signatures
```


#### `ExecutionPayloadHeader`

See [`ExecutionPayloadHeader`](https://github.com/ethereum/consensus-specs/blob/dev/specs/deneb/beacon-chain.md#executionpayloadheader) in Deneb consensus specs.

##### `BlindedBeaconBlockBody`

Note: `BlindedBeaconBlock` and `SignedBlindedBeaconBlock` types are updated indirectly.

```python
class BlindedBeaconBlockBody(Container):
    randao_reveal: BLSSignature
    eth1_data: Eth1Data
    graffiti: Bytes32
    proposer_slashings: List[ProposerSlashing, MAX_PROPOSER_SLASHINGS]
    attester_slashings: List[AttesterSlashing, MAX_ATTESTER_SLASHINGS]
    attestations: List[Attestation, MAX_ATTESTATIONS]
    deposits: List[Deposit, MAX_DEPOSITS]
    voluntary_exits: List[SignedVoluntaryExit, MAX_VOLUNTARY_EXITS]
    sync_aggregate: SyncAggregate
    execution_payload_header: ExecutionPayloadHeader  # [Modified in Deneb]
    bls_to_execution_changes: List[SignedBLSToExecutionChange, MAX_BLS_TO_EXECUTION_CHANGES]
    blob_kzg_commitments: List[KZGCommitment, MAX_BLOB_COMMITMENTS_PER_BLOCK]  # [New in Deneb]
```

#### `ValidatorRegistrationV2`

The `proposer_builder_commitment` field is added which indicates the type of block the proposer wants from the builder. for e.g: a proposer_builder_commitment
of `FULL_BLOCK` indicates that the proposer wants the highest possible value block built by 1 builder(which is the status quo). Another example, is a proposer_builder_commitment
of `TOB_ROB_SPLIT`. This commitment indicates that the proposer wants a block where the TOB(top of the block) and ROB(rest of the block) is built by 2 seperate builders
to reduce TOB MEV opportunities and make block building more decentralized.
The validity of commitment should be handled by the trusted relay. The relay should reject payloads built by builders which do not validate the commitments indicated
by proposers. The proposer should trust the relay that the block has fulfilled their commitments.

```python
class ValidatorRegistrationV2(Container):
    fee_recipient: ExecutionAddress
    gas_limit: uint64
    timestamp: uint64
    proposer_builder_commitment: string
    pubkey: BLSPubkey
```

#### `SignedValidatorRegistrationV2`

```python
class SignedValidatorRegistrationV2(Container):
    message: ValidatorRegistrationV2
    signature: BLSSignature
```

## Validator registration processing

With the addition of `proposer_builder_commitment`

### `verify_proposer_builder_commitment`

The list of commitments should be stored in the trusted relayer so that builders can access the proposer commitments to build the suitable block
Below the proposer_builder_commitments list should be stored in the relayer.

```python
def verify_proposer_builder_commitment(signed_registration: SignedValidatorRegistrationV2, proposer_builder_commitments: List[string]) -> bool:
    return signed_registration.proposer_builder_commitment in proposer_builder_commitments # the list of commitments should be known in the builder n/w
```

### `process_registration_v2`

A `registration` is considered valid if the following function completes without raising any assertions:

```python
def process_registration_v2(state: BeaconState,
                         registration: SignedValidatorRegistrationV2,
                         registrations: Dict[BLSPubkey, ValidatorRegistrationV2],
                         current_timestamp: uint64):
    registration = registration.message

    # Verify BLS public key corresponds to a registered validator
    validator_pubkeys = [v.pubkey for v in state.validators]
    assert pubkey in validator_pubkeys

    index = ValidatorIndex(validator_pubkeys.index(pubkey))
    validator = state.validators[index]

    # Verify validator registration elibility
    assert is_eligible_for_registration(state, validator)
    
    # Verify timestamp is not too far in the future
    assert registration.timestamp <= current_timestamp + MAX_REGISTRATION_LOOKAHEAD
    
    # Verify whether the proposer builder commitment is valid i.e it matches the known supported commitments.
    assert verify_proposer_builder_commitment(state, validator)

    # Verify timestamp is not less than the timestamp of the previous registration (if it exists)
    if registration.pubkey in registrations:
        prev_registration = registrations[registration.pubkey]
        assert registration.timestamp >= prev_registration.timestamp

    # Verify registration signature
    assert verify_registration_signature(state, registration)
```


#### Constructing the `ExecutionPayloadHeader`

We add the `proposer_builder_commitment` field for validators in the `ValidatorRegistrationV2`. This field indicates the type of block the proposer wants. 
E.g., .a proposer_builder_commitment of `FULL_BLOCK` indicates that the proposer wants the highest possible value block built by one builder(the status quo). 
Another example is a proposer_builder_commitment of `TOB_ROB_SPLIT`. This commitment indicates that the proposer wants a block where the TOB(top of the block) 
and ROB(rest of the block) is built by two separate builders to reduce the chances of one builder dominating TOB MEV opportunities and make block building more decentralised.
The following conditions apply to both `FULL_BLOCK` and `TOB_ROB_SPLIT` commitment
1. The builder MUST provide a bid for an execution payload which fulfils the block validity conditions set in the relay according to `proposer_builder_commitment`.
2. The builder MUST provide a bid for the valid execution payload that can pay the `fee_recipient` in the validator registration for the registered `pubkey` the most.
The following conditions apply to the `FULL_BLOCK` commitment
1. The builder MUST build an execution payload whose `gas_limit` is equal to the `gas_limit` of the latest registration for `pubkey` or as close as possibl.e under the consensus rules.
The following conditions apply to the `TOB_ROB_SPLIT` commitment
1. The builder MUST build ROB and TOB bids whose execution payloads each do not exceed `gas_limit / 2` where `gas_limit` is equal to the `gas_limit` of the latest registration for `pubkey.