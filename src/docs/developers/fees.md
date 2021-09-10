---
title: Gas Costs on L2
lang: en-US
---

# {{ $frontmatter.title }}

::: warning NOTICE
This page documents the current status of the Optimistic Ethereum protocol, and the details here are subject to change based on feedback.
:::

Optimistic Ethereum is a lot cheaper than regular Ethereum but transaction fees do still exist.
Here we'll cover what those fees are for, how to estimate them, and how to present them to your users.

## L2 Transaction Fees

These are fees for an L2 transaction, rather than one that involves communication
between L1 and L2. Every Optimistic Ethereum transaction has two costs:

1. **L2 execution fee**. This fee is `tx.gasLimit * l2gasUsed`, 
   where `l2gasUsed ≤ tx.gasLimit`. This is similar to the L1 cost, but
   **much** lower (unless Optimistic Ethereum congestion is extremely high). [You 
   can see the current cost here](https://public-grafana.optimism.io/d/9hkhMxn7z/public-dashboard?orgId=1&refresh=5m).

   At writing the L2 gas cost in 0.001 gwei, and about 80,000 L2 gas cost the same 
   as one L1 gas.

2. **L1 security fee**. This is the cost of storing the transaction's data on L1.
   This cost is `1.5 * l1GasPrice * (2750 + calldataGas)`, where:

   * `l1GasPrice` is the current L1 gas cost as reported by the `OVM_GasPriceOracle`
     contract
   * `calldataGas` is the standard Ethereum cost, 4 gas for each zero byte, 16
     gas for every byte with a different value


## How is the cost presented to the user?

The L2 execution fee is presented to users in the normal way, with a transaction
gas price and transaction gas limit.

The L1 security fee is deducted automatically from the user's balance on Optimistic 
Ethereum. We are working with wallet developers to display this fee to the user when
submitting transactions.

### Obtaining L1 security fees 

To obtain the L1 security fee to display it to the user, you can use code 
similar to this:

```javascript
// Connect to the gas oracle
import { getContractFactory, predeploys }from '@eth-optimism/contracts'
import { ethers } from 'ethers'
const OVM_GasPriceOracle = getContractFactory('OVM_GasPriceOracle')
  .attach(predeploys.OVM_GasPriceOracle)

// Populate the transaction, get the data, and remove initial 0x
const txData = (await <contract>.populateTransaction.<action>(<parameters>)).data.slice(2)
var txArr = []
for(var i=0; i<txData.length; i+=2)
  txArr.push(parseInt(txData.substring(i,i+2), 16))

// 16 gas for a non-zero. 4 for a zero  
const calldataGas = 4*txArr.length + 12*txArr.filter(x => x != 0).length

const l1FeeInWei = 1.5*OVM_GasPriceOracle.gasPrice()*(2750+calldataGas)
```


## Fees for L1 to L2 transactions

For an L1 to L2 transaction you only pay the L1 cost of submitting the transaction.
You send a transaction to the [`OVM_L1CrossDomainMessenger`](https://github.com/ethereum-optimism/optimism/blob/develop/packages/contracts/contracts/optimistic-ethereum/OVM/bridge/messaging/OVM_L1CrossDomainMessenger.sol)
contract, which then sends a call to the [`OVM_CanonicalTransactionChain`](https://github.com/ethereum-optimism/optimism/blob/develop/packages/contracts/contracts/optimistic-ethereum/OVM/chain/OVM_CanonicalTransactionChain.sol).
This generally isn't *too* expensive, but it mainly depends on L1 congestion.

## Fees for L2 to L1 transactions

Each message from L2 to L1 requires two transactions:

1. An L2 transaction that *initiates* the transaction, which is priced the same way that Sequencer transactions are priced.
1. An L1 transaction that *finalizes* the transaction. This transaction is somewhat expensive because it includes [verifying](https://github.com/ethereum-optimism/optimism/blob/467d6cb6a4a35f2f8c3ea4cfa4babc619bafe7d2/packages/contracts/contracts/optimistic-ethereum/libraries/trie/Lib_MerkleTrie.sol#L73-L93) a [Merkle trie](https://eth.wiki/fundamentals/patricia-tree) inclusion proof.

The total cost of an L2 to L1 transaction is therefore the combined cost of the L2 initialization transaction and the L1 finalization transaction.