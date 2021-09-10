---
title: Contract Overview
lang: en-US
tags:
    - contracts
    - high-level
---

# {{ $frontmatter.title }}

::: tip Work in Progress™
_Our documentation is a rapidly improving work in progress. If you have questions or feel like something is missing feel free to ask in our [Discord server](https://discord.optimism.io) where we (and our awesome community) are actively responding, or [open an issue](https://github.com/ethereum-optimism/community-hub/issues) in the GitHub repo for this site._
:::

## Introduction

<!-- - Welcome!  Give context -- "how to read these docs" -->

Optimistic Ethereum (OE) is a Layer 2 scaling protocol for Ethereum applications.
I.e., it makes transactions cheap. Real cheap.
We aim to make transacting on Ethereum affordable and
accessible to anyone.

This document is intended for anyone looking for a deeper understanding of how the protocol works
'under the hood'.
If you just want to skip straight to integrating your smart contract application with
OE, check out the [Developer Docs](/docs/developers/l2/convert.html).

Optimistic Ethereum is meant to look, feel and behave like Ethereum but cheaper and faster.
For developers building on our OE, we aim to make the transition as seamless as possible.
With very few exceptions,
existing Solidity smart contracts can run on L2 exactly how they run on L1.
Similarly, off-chain code (ie. UIs and wallets), should be able to interact with L2 contract with little more than an updated RPC endpoint.



## System Overview

The smart contracts in the Optimistic Ethereum (OE) protocol can be separated into a few key components. We will discuss each component in more detail below.

- **[Chain:](#chain-contracts)** Contracts on layer-1, which hold the ordering of layer-2 transactions, and commitments to the associated layer-2 state roots.
- **[Transaction result challenges:](#transaction-challenge-contracts)** Contracts on layer-1 which implement the process for challenging a transaction result.
- **[Bridge:](#bridge-contracts)** Contracts which facilitate message passing between layer-1 and layer-2.
- **[Predeploys:](#predeployed-contracts)** A set of essential contracts which are deployed and available in the genesis state of the system. These contracts are similar to Ethereum's precompiles, however they are written in Solidity, and can be found at addresses prefixed with 0x42.
- **[Accounts:](#account-contracts)** Redeployable contracts layer-2 contracts which can represent a user and provide a form of 'account abstraction'.

<!-- Using html instead of markdown so we can put a caption on the image. -->
<figure>
  <img src='../../assets/docs/protocol/oe-arch-rc0.png'>
  <figcaption style="text-align: center; font-size: 12px;">Diagram created with <a href="https://www.diagrams.net/">draw.io</a>. <br>Editable source <a href="https://docs.google.com/document/d/1OObmIhuVyh5GEekqT4dd3bzO58ejSQb_rlnrBmxcNN0/edit#">here</a>.</figcaption>
</figure>

<!--
 - Contracts Reference Sheet (aka glossary)
  - Deployed contracts (with addresses ie. [aave example][aave])
-->

## Chain Contracts

The Chain is composed of a set of contracts running on the Ethereum mainnet. These contracts store ordered
lists of:

1. An _ordered_ list of all transactions applied to the L2 state.
2. The proposed state root which would results from the application of each transaction.
3. Transactions sent from L1 to L2, which are pending inclusion in the ordered list.

<!--
**Planned section outline**
- Delineation between CTC and SCC,
- **high priority**: explain once and for all that challenges roll back state roots, but NOT transactions
- Diagram of "the chains" and what is stored on chain -- ideally illustrates the "roll up" mechanism whereby only roots of batches are SSTOREd
- Sequencing -- what are the properties, what are the implications
- Ring buffer?? (lean deprioritize)
-->

The chain is composed of the following concrete contracts:
<!-- [concrete contracts][stackex]: -->

### [`OVM_CanonicalTransactionChain`](https://github.com/ethereum-optimism/optimism/blob/experimental/packages/contracts/contracts/L1/rollup/OVM_CanonicalTransactionChain.sol) (CTC)

The Canonical Transaction Chain (CTC) contract is an append-only log of transactions which must be applied to the OVM state. It defines the ordering of transactions by writing them to the `CTC:batches` instance of the Chain Storage Container. The CTC also allows any account to `enqueue()` an L2 transaction, which the Sequencer must  eventually append to the rollup state.

### [`OVM_StateCommitmentChain`](hhttps://github.com/ethereum-optimism/optimism/blob/experimental/packages/contracts/contracts/L1/rollup/OVM_StateCommitmentChain.sol) (SCC)

The State Commitment Chain (SCC) contract contains a list of proposed state roots which Proposers assert to be a result of each transaction in the Canonical Transaction Chain (CTC). Elements here have a 1:1 correspondence with transactions in the CTC, and should be the unique state root calculated off-chain by applying the canonical transactions one by one.

### [`OVM_ChainStorageContainer`](https://github.com/ethereum-optimism/optimism/blob/experimental/packages/contracts/contracts/L1/rollup/OVM_ChainStorageContainer.sol)

Provides reusable storage in the form of a "Ring Buffer" data structure, which will overwrite storage slots that are no longer needed. There are three Chain Storage Containers deployed, two are controlled by the CTC, one by the SCC.

## Transaction Challenge Contracts

In the previous section, we mentioned that the Chain includes a list of the _proposed_ state roots
resulting from each transaction. Here we explain a bit more about how these proposals happen, and how
we come to trust them.

In brief: If a proposed state root is not the correct result of executing a transaction, then a Verifier (which is anyone running an OE 'full node') can initiate a transaction result challenge. If the transaction result is successfully proven to be incorrect, the Verifier will receive a reward taken from funds which a Sequencer must put up as a bond.

The challenge system is composed of the following concrete contracts:

### [`OVM_FraudVerifier`](https://github.com/ethereum-optimism/optimism/blob/develop/packages/contracts/contracts/optimistic-ethereum/OVM/verification/OVM_FraudVerifier.sol)
The Fraud Verifier contract coordinates the entire challenge verification process. If the challenge is successful it prunes any state batches from State Commitment Chain which were published after and including the state root in question.

### [`OVM_BondManager`](https://github.com/ethereum-optimism/optimism/blob/develop/packages/contracts/contracts/optimistic-ethereum/OVM/verification/OVM_BondManager.sol)
The Bond Manager contract handles deposits in the form of an ERC20 token from bonded Proposers. It also handles the accounting of gas costs spent by a Verifier during the course of a challenge. In the event of a successful challenge, the faulty Proposer's bond is slashed, and the Verifier's gas costs are refunded.

### [`OVM_StateTransitioner`](https://github.com/ethereum-optimism/optimism/blob/develop/packages/contracts/contracts/optimistic-ethereum/OVM/verification/OVM_StateTransitioner.sol)
The State Transitioner coordinates the execution of a state transition during the evaluation of a challenge. It feeds verified input to the Execution Manager's run(), and controls a State Manager (which is uniquely created for each challenge). Once a challenge has been initialized, this contract is provided with the pre-state root and verifies that the OVM storage slots committed to the State Manager are contained in that state. This contract controls the State Manager and Execution Manager, and uses them to calculate the post-state root by applying the transaction. The Fraud Verifier can then check the correctness of a result by comparing the calculated post-state root with the proposed post-state root.

### [`OVM_StateTransitionerFactory`](https://github.com/ethereum-optimism/optimism/blob/develop/packages/contracts/contracts/optimistic-ethereum/OVM/verification/OVM_StateTransitionerFactory.sol)
Used by the Fraud verifier to create a unique State Transitioner for each challenge.
<!-- - (TODO: are factories even worth including?) -->



## Bridge Contracts

The Bridge contracts implement the functionality required to pass messages between layer 1 and layer 2.  [You can read an overview
here](/docs/developers/bridge/messaging.html)

<!--
**Planned section outline**
- Low-level tools (ovmL1TXORIGIN, state committment access)

### Key concepts
- **Relaying** refers to executing a message sent from the other domain, ie. "this message was relayed
-->

The Bridge is composed of the following concrete contracts:

### [`OVM_L1CrossDomainMessenger`](https://github.com/ethereum-optimism/optimism/blob/experimental/packages/contracts/contracts/L1/messaging/OVM_L1CrossDomainMessenger.sol)
The L1 Cross Domain Messenger (L1xDM) contract sends messages from L1 to L2, and relays messages from L2 onto L1. In the event that a message sent from L1 to L2 is rejected for exceeding the L2 epoch gas limit, it can be resubmitted via this contract's replay function.

### [`OVM_L2CrossDomainMessenger`](https://github.com/ethereum-optimism/optimism/blob/experimental/packages/contracts/contracts/L2/messaging/OVM_L2CrossDomainMessenger.sol)
The L2 Cross Domain Messenger (L2xDM) contract sends messages from L2 to L1, and is the entry point for L2 messages sent via the L1 Cross Domain Messenger.


## The Standard Bridge

One common case of message passing is "transferring" either ERC-20
tokens or ETH between L1 and Optimistic Ethereum. To deposit tokens 
into Optimistic Ethereum, the bridge locks them on L1 and mints equivalent
tokens in Optimistic Ethereum. To withdraw tokens, the bridge burns the 
Optimistic Ethereum tokens and releases the locked L1 tokens. [More details
are here](/docs/developers/bridge/standard-bridge.html)


### [`OVM_L1StandardBridge`](https://github.com/ethereum-optimism/optimism/blob/experimental/packages/contracts/contracts/L1/messaging/OVM_L1StandardBridge.sol)
The L1 part of the Standard Bridge. Responsible for finalising withdrawals from L2 and initiating deposits into L2 of ETH and compliant ERC20s.


### [`OVM_L2StandardBridge`](https://github.com/ethereum-optimism/optimism/blob/experimental/packages/contracts/contracts/L2/messaging/OVM_L2StandardBridge.sol)
The L2 part of the Standard Bridge. Responsible for finalising deposits from L1 and initiating withdrawals from L2 of ETH and compliant ERC20s.


<!--
JM scribbles:
Proving fraud against a crossDomain message with a false account for **l1TxOrigin**:

- All L1 to L2 messages must be passed via `enqueue` in the CTC. This records `msg.sender` in the enqueued tx hash.
- Recall that each transitioner has a specific txHash, and state root.
- During a fraud proof, the verifier must provide the transaction data to `applyTransaction()`, and the transitioner just checks that the hashes match. -->

## Predeployed Contracts

"Predeploys" are a set of essential L2 contracts which are deployed and available in the genesis state of the system. These contracts are similar to Ethereum's precompiles, however they are written in Solidity, and can be found in the OVM at addresses prefixed with 0x42.

Looking up predeploys is available in the Solidity library [`Lib_PredeployAddresses`](https://github.com/ethereum-optimism/optimism/blob/master/packages/contracts/contracts/optimistic-ethereum/libraries/constants/Lib_PredeployAddresses.sol) as well as in the `@eth-optimism/contracts` package as `predeploys` export.

The following concrete contracts are predeployed:

### [`OVM_DeployerWhitelist`](https://github.com/ethereum-optimism/optimism/blob/develop/packages/contracts/contracts/optimistic-ethereum/OVM/predeploys/OVM_DeployerWhitelist.sol)
The Deployer Whitelist is a temporary predeploy used to provide additional safety during the initial phases of our mainnet roll out. It is owned by the Optimism team, and defines accounts which are allowed to deploy contracts on Layer 2. The Execution Manager will only allow an ovmCREATE or ovmCREATE2 operation to proceed if the deployer's address whitelisted.

### [`OVM_ETH`](https://github.com/ethereum-optimism/optimism/blob/develop/packages/contracts/contracts/optimistic-ethereum/OVM/predeploys/OVM_ETH.sol)
The ETH predeploy provides an ERC20 interface for ETH deposited to Layer 2. 

### [`OVM_L1MessageSender`](https://github.com/ethereum-optimism/optimism/blob/develop/packages/contracts/contracts/optimistic-ethereum/OVM/predeploys/OVM_L1MessageSender.sol)
The L1MessageSender is a predeployed contract running on L2.
During the execution of cross domain transaction from L1 to L2, it returns the address of the L1 account (either an EOA or contract) which sent the message to L2 via the Canonical Transaction Chain's `enqueue()` function.
This contract exclusively serves as a getter for the `ovmL1TXORIGIN` operation.
This is necessary because there is no corresponding EVM opcode which the optimistic solidity compiler could replace with a call to the ExecutionManager's `ovmL1TXORIGIN()` function.
That is, if a contract on L2 wants to know which L1 address initiated a call on L2, the way to do it is by calling `OVM_L1MessageSender.ovmL1TXORIGIN()`.

### [`OVM_L2ToL1MessagePasser`](https://github.com/ethereum-optimism/optimism/blob/develop/packages/contracts/contracts/optimistic-ethereum/OVM/predeploys/OVM_L2ToL1MessagePasser.sol)
The L2 to L1 Message Passer is a utility contract which facilitate an L1 proof of the  of a message on L2. The L1 Cross Domain Messenger performs this proof in its _verifyStorageProof function, which verifies the existence of the transaction hash in this  contract's `sentMessages` mapping.

### [`OVM_SequencerEntrypoint`](https://github.com/ethereum-optimism/optimism/blob/develop/packages/contracts/contracts/optimistic-ethereum/OVM/predeploys/OVM_SequencerEntrypoint.sol)
The Sequencer Entrypoint is a predeploy which, despite its name, can in fact be called by  any account. It accepts a more efficient compressed calldata format, which it decompresses and  encodes to the standard EIP155 transaction format. This contract is the implementation referenced by the Proxy Sequencer Entrypoint, thus enabling the Optimism team to upgrade the decompression of calldata from the Sequencer.

### [`OVM_SequencerFeeVault`](https://github.com/ethereum-optimism/optimism/blob/develop/packages/contracts/contracts/optimistic-ethereum/OVM/predeploys/OVM_SequencerFeeVault.sol)
This contract holds fees paid to the sequencer until there is enough to 
justify the transaction cost of sending them to L1 where they are used to
pay for L1 transaction costs (mostly the cost of publishing all L2 transaction
data as CALLDATA on L1).

### [`OVM_L2StandardBridge`](https://github.com/ethereum-optimism/optimism/blob/master/packages/contracts/contracts/optimistic-ethereum/OVM/bridge/tokens/OVM_L2StandardBridge.sol)
The L2 part of the Standard Bridge. Responsible for finalising deposits from L1 and initiating withdrawals from L2 of ETH and compliant ERC20s.
See [Standard Bridge](/docs/developers/bridge/standard-bridge.html) for details.

### [`ERC1820Registry`](https://github.com/ethereum-optimism/optimism/blob/develop/packages/contracts/contracts/optimistic-ethereum/OVM/predeploys/ERC1820Registry.sol)
[ERC1820](https://eips.ethereum.org/EIPS/eip-1820) specifies a registry
service that lets addresses report what interfaces they support and ask 
about other addresses. 

### [`OVM_ExecutionManagerWrapper`](https://github.com/ethereum-optimism/optimism/blob/develop/packages/contracts/contracts/optimistic-ethereum/OVM/predeploys/OVM_ExecutionManagerWrapper.sol)
This is the one contract on L2 that can call another contract without having to
go through virtualization. It is used to call 
[OVM_ExecutionManager](#ovm-executionmanager).


### [`ERC1820Registry`](https://github.com/ethereum-optimism/optimism/blob/develop/packages/contracts/contracts/optimistic-ethereum/OVM/predeploys/ERC1820Registry.sol)
[ERC1820](https://eips.ethereum.org/EIPS/eip-1820) specifies a registry
service that lets addresses report what interfaces they support and ask 
about other addresses. 

### [`Lib_AddressManager`](https://github.com/ethereum-optimism/optimism/blob/develop/packages/contracts/contracts/optimistic-ethereum/libraries/resolver/Lib_AddressManager.sol)
This is a library that stores the mappings between names and their addresses.
It is used by [`OVM_L1CrossDomainMessenger`](#ovm-l1crossdomainmessenger).


## Account Contracts

OVM Account contracts are redeployable contracts layer-2 contracts which can represent a user and provide a form of 'account abstraction'.

<!--
**Planned section outline**
- **TODO:** figure out how much of this needs to go into "integration" section
- explanation of createEOA/nonce opcodes which offer backwards compatibility
- Transaction formats (ethSign vs RLP)
- somewhere this should say it's upgradeable
- Fees (Is this where fee discussion should go? Or with RPC docs?)
 -->

### [`OVM_ProxyEOA`](https://github.com/ethereum-optimism/optimism/blob/develop/packages/contracts/contracts/optimistic-ethereum/OVM/predeploys/OVM_ProxyEOA.sol)
The Proxy EOA contract uses a delegate call to execute the logic in an implementation contract. In combination with the logic implemented in the ECDSA Contract Account, this enables a form of upgradable  'account abstraction' on layer 2.

