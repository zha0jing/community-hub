---
title: Porting Existing Apps
lang: en-US
---

# {{ $frontmatter.title }}

::: warning NOTICE
This page refers to the design of the current iteration of the Optimistic Ethereum protocol.
Details here are subject to change as the Optimistic Ethereum protocol evolves.
:::


## Differences from L1 Ethereum

### Opcode differences

The L1 verification challenge mechanism needs to be able to simulate every possible opcode that
an L2 contract may have. The requires a few minor differences in Opcode support between standard EVM
and the Optimistic Virtual Machine (OVM)

| EVM Opcode  | Solidity equivalent | OVM Behavior |
| - | - | - |
| COINBASE	 | `block.coinbase`   | Returns 0 |
| DIFFICULTY | `block.difficulty` | Returns 0 |
| BLOCKHASH	 | `blockhash`        |	Returns 0 |
| GASPRICE   | `tx.gasprice`      | Returns 0 |
| SELFDESTRUCT |                  | No operation |
| BASEFEE    | `block.basefee`    | Not supported for now (Optimistic Ethereum is Berlin, not London, at present) |
| NUMBER     | `block.number`     | L2 block number |
| TIMESTAMP  | `block.timestamp`  | Timestamp of latest verified L1 block |
| L1MESSAGESENDER | `assembly { solidityVariableName := verbatim_0i_1o("0x4A")}` | The address of the message sender on L1 |
| L1BLOCKNUMBER | | |


# NEED THE OPCODE FOR L1BLOCKNUMBER, which will probably appear [here eventually](https://github.com/ethereum-optimism/optimism/blob/experimental/l2geth/core/vm/opcodes.go)

### Tests need to run on geth

Both Hardhat and Truffle allow you to run contract tests against their own implementations of the EVM.
However, to test contracts that run on Optimistic Ethereum you need to run them on a local copy of Optimistic Ethereum (which is built on top of [geth](https://geth.ethereum.org/)).

There are two issues involved in running your tests against a geth instance, 
rather than an EVM running inside your development environment:

1. Tests will take longer. For development purposes, Geth is quite a bit slower than the [Hardhat's](https://hardhat.org) EVM or Truffle's [ganache](https://github.com/trufflesuite/ganache-cli). You will likely have to make more liberal use of [asynchronous](https://developer.mozilla.org/en-US/docs/Learn/JavaScript/Asynchronous/Concepts) functions within your tests.
2. Both [Truffle](https://github.com/trufflesuite/ganache-cli#custom-methods) and [Hardhat](https://hardhat.org/hardhat-network/#special-testing-debugging-methods) support custom debugging methods such as `evm_snapshot` and `evm_revert`. You cannot use these methods in tests for Optimistic Ethereum contracts because they are not available in geth. Nor can you use [Hardhat's `console.log`](https://hardhat.org/tutorial/debugging-with-hardhat-network.html).
