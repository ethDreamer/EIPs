---
eip: <to be assigned>
title: Deposit Contract Snapshot Interface
description: Standardizing the format (and endpoint?) for transmitting a snapshot of the deposit Merkle tree
author: Mark Mackey (@ethDreamer)
discussions-to: https://ethereum-magicians.org/t/URL-TO-BE-DECIDED-ONCE-EIP-NUMBER-ASSIGNED
status: Draft
type: Standards Track
category: Interface
created: 2021-01-29
---

## Abstract
This EIP defines a standard format for transmitting the deposit contract Merkle tree in a compressed form during weak subjectivity sync. This allows newly syncing consensus clients to reconstruct the deposit tree much faster than downloading all historical deposits. The format proposed also allows clients to prune deposits that are no longer needed to participate fully in consensus (see [Deposit Finalization Flow](#deposit-finalization-flow)).

## Motivation
Most client implementations require beacon nodes to download and store every deposit log since the launch of the deposit contract in order to reconstruct the deposit Merkle tree. This approach requires nodes to store far more deposits than necessary to fully participate in consensus. It also needlessly increases the time it takes for new nodes to fully sync, which is especially noticable during weak subjectivity sync. Furthermore, if [EIP-4444](./eip-4444.md) is adopted, it will not always be possible to download all historical deposit logs from full nodes.

This EIP solves these issues by defining a standard format for transmitting the deposit contract Merkle tree in a compressed form to newly syncing nodes during weak subjectivity sync.

<!---
During deposit processing, the beacon chain requires deposits to be submitted along with a Merkle path to the deposit root. This is required exactly once for each deposit. Once a deposit has been processed by the beacon chain and the state has been finalized, many of the hashes along the path to the deposit root will never be required again to construct a Merkle proof on chain.

When new nodes begin syncing with the network, most implementations require them to download every deposit log since the launch of the deposit contract. This increases the time it takes new nodes to fully sync. In addition, if [EIP-4444](./eip-4444.md) is adopted, it will not always be possible to download all historical deposit logs from full nodes.

Reconstructing and storing the entire deposit Merkle tree is also space inefficient. 

During deposit processing, the beacon chain requires deposits to be submitted along with a Merkle path to the deposit root. This is required exactly once for each deposit. Once a deposit has been processed by the beacon chain and the state has been finalized, many of the hashes along the path to the deposit root will never be required again to construct a Merkle proof on chain.

======
The motivation section should describe the "why" of this EIP. What problem does it solve? Why should someone want to implement this standard? What benefit does it provide to the Ethereum ecosystem? What use cases does this EIP address?
--->

## Specification
Clients MAY continue to implement the Merkle tree however they choose. However, when transmitting the deposit Merkle tree to newly syncing nodes, clients MUST use the following format (and beacon API endpoint?):

```
DepositTreeSnapshot {
    finalized: Vector[Hash32],
    deposits:  uint64,
    eth1_block_hash: Hash32,
}
```

Where the hashes in the `finalized` vector are defined in the [Deposit Finalization Flow](#deposit-finalization-flow) section below, `deposits` is the number of deposits stored in the snapshot, and `eth1_block_hash` is the hash of the eth1 block containing the highest index deposit stored in the snapshot.

#### Deposit Finalization Flow

During deposit processing, the beacon chain requires deposits to be submitted along with a Merkle path to the deposit root. This is required exactly once for each deposit. When a deposit has been processed by the beacon chain and the [deposit finalization conditions](#deposit-finalization-conditions) have been met, many of the hashes along the path to the deposit root will never be required again to construct Merkle proofs on chain. These unnecessary hashes MAY be pruned to save space. The image below illustrates the evolution of the deposit Merkle tree under this process alongside the corresponding `DepositTreeSnapshot` as new deposits are added and older deposits become finalized:

![Deposit Tree Evolution](../assets/eip-draft_deposit_contract_snapshot_interface/deposit_tree_evolution.png)

<!---
The key words “MUST”, “MUST NOT”, “REQUIRED”, “SHALL”, “SHALL NOT”, “SHOULD”, “SHOULD NOT”, “RECOMMENDED”, “MAY”, and “OPTIONAL” in this document are to be interpreted as described in RFC 2119.

The technical specification should describe the syntax and semantics of any new feature. The specification should be detailed enough to allow competing, interoperable implementations for any of the current Ethereum platforms (go-ethereum, parity, cpp-ethereum, ethereumj, ethereumjs, and [others](https://github.com/ethereum/wiki/wiki/Clients)).
--->

## Rationale

The format in this specification was chosen to achieve several goals simultaneously:

1. Enable reconstruction of the deposit contract Merkle tree under the adoption of [EIP-4444](./eip-4444.md)
2. Avoid requiring consensus nodes to retain more deposits than necessary to fully participate in consensus
3. Simplicity of implementation (see [Reference Implementation](#reference-implementation) section)
4. Increase speed of weak subjectivity sync
5. Compatibility with existing implementations of this mechanism (see [Existing Implementations](#existing-implementations))

#### Why not Reconstruct the Tree Directly from the Deposit Contract?

The deposit contract can only provide the tree at the head of the chain. Because the beacon chain's view of the deposit contract lags behind the eth1 chain by `ETH1_FOLLOW_DISTANCE`, there are almost always deposits which haven't yet been included in the chain that need proofs constructed from an earlier version of the tree than exists at the head.

#### Why not Reconstruct the Tree from a Deposit in the Beacon Chain?

In principle, a node could scan backwards through the chain starting from the weak subjectivity checkpoint to locate a suitable [`Deposit`](https://github.com/ethereum/consensus-specs/blob/v1.1.9/specs/phase0/beacon-chain.md#deposit), and then extract the rightmost branch of the tree from that. The node would also need to extract the `eth1_block_hash` from which to start syncing new deposits from the `Eth1Data` in the corresponding `BeaconState`. This approach is less desirable for a few reasons:

* More difficult to implement due to the edge cases involved in finding a suitable deposit to anchor to (the rightmost branch of the latest not-yet-included deposit is required)
* This would make backfilling the blocks a requirement for reconstructing the deposit tree and therefore a requirement for block production
* This is inherantly slower than getting this information from the weak subjectivity checkpoint

<!---
The rationale fleshes out the specification by describing what motivated the design and why particular design decisions were made. It should describe alternate designs that were considered and related work, e.g. how the feature is supported in other languages.
--->

## Backwards Compatibility

This proposal is fully backwards compatible.

<!---
All EIPs that introduce backwards incompatibilities must include a section describing these incompatibilities and their severity. The EIP must explain how the author proposes to deal with these incompatibilities. EIP submissions without a sufficient backwards compatibility treatise may be rejected outright.
--->

## Test Cases

TBD

<!---
Test cases for an implementation are mandatory for EIPs that are affecting consensus changes.  If the test suite is too large to reasonably be included inline, then consider adding it as one or more files in `../assets/eip-####/`.
--->

## Reference Implementation
This implementation lacks error checking and is optimized for readability over efficiency. If `tree` is a `DepositTree`, then the `DepositTreeSnapshot` can be obtained by calling `tree.get_snapshot()` and a new instance of the tree can be recovered from the snapshot by calling `DepositTree.from_snapshot()`. See the [Deposit Finalization Conditions](#deposit-finalization-conditions) section for discussion on when the tree can be pruned by calling `tree.finalize()`.

Generating proofs for deposits against an earlier version of the tree is relatively fast in this implementation; just create a copy of the finalized tree with `copy = DepositTree.from_snapshot(tree.get_snapshot())` and then append the remaining deposits to the desired count with `copy.push_leaf(deposit)`. Proofs can then be obtained with `copy.get_proof(index)`.
```python:../assets/eip-draft_deposit_contract_snapshot_interface/deposit_snapshot.py

```

<!---
An optional section that contains a reference/example implementation that people can use to assist in understanding or implementing this specification.  If the implementation is too large to reasonably be included inline, then consider adding it as one or more files in `../assets/eip-####/`.
--->

#### Existing Implementations
* [lighthouse](https://github.com/sigp/lighthouse/pull/2915)
* [nimbus](https://github.com/status-im/nimbus-eth2/commit/372c9b798c005102c7777a16aaa7c822267330fa)

## Security Considerations

#### Relying on Weak Subjectivity Sync

The upcoming switch to PoS will require newly synced nodes to rely on valid weak subjectivity checkpoints because of long-range attacks. This proposal relies on the weak subjectivity assumption that clients will not bootstrap with an invalid WS checkpoint.

#### Deposit Finalization Conditions

Care must be taken not to send a snapshot which includes deposits that haven't been fully included in the finalized checkpoint. Let `state` be the [`BeaconState`](https://github.com/ethereum/consensus-specs/blob/v1.1.9/specs/phase0/beacon-chain.md#beaconstate) at a given block in the chain. Under normal operation, the [`Eth1Data`](https://github.com/ethereum/consensus-specs/blob/v1.1.9/specs/phase0/beacon-chain.md#eth1data) stored in `state.eth1_data` is replaced every `EPOCHS_PER_ETH1_VOTING_PERIOD` epochs. Thus, finalization of the deposit tree proceeds in increments of `state.eth1_data`. Let `eth1data` be some `Eth1Data`. Both of the following conditions MUST be met to consider `eth1data` finalized:

1. A finalized checkpoint exists where the corresponding `state` has `state.eth1_data == eth1data`
2. A finalized checkpoint exists where the corresponding `state` has `state.eth1_deposit_index >= eth1_data.deposit_count`

When these conditions are met, the tree can be pruned in the [reference implementation](#reference-implementation) by calling `tree.finalize(eth1_data)`

#### Deposit Queue Exceeds EIP-4444 Pruning Period

The proposed design could fail if the deposit queue becomes so large that deposits cannot be processed within the [EIP-4444 Pruning Period](./eip-4444.md) (currently set to 1 year). The beacon chain can process `MAX_DEPOSITS/SECONDS_PER_SLOT` deposits/second without skipped slots. Even under extreme conditions where 25% of slots are skipped, the deposit queue would need to be >31.5 million to hit this limit. This is more than 8x the total supply of ether assuming each deposit is a full validator. The minimum deposit is 1 ETH so an attacker would need to burn >30 Million ETH to create these conditions.

<!---
All EIPs must contain a section that discusses the security implications/considerations relevant to the proposed change. Include information that might be important for security discussions, surfaces risks and can be used throughout the life cycle of the proposal. E.g. include security-relevant design decisions, concerns, important discussions, implementation-specific guidance and pitfalls, an outline of threats and risks and how they are being addressed. EIP submissions missing the "Security Considerations" section will be rejected. An EIP cannot proceed to status "Final" without a Security Considerations discussion deemed sufficient by the reviewers.
--->

## Copyright
Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).

