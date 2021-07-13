---
title: Converting Applications to Optimistic Ethernet
lang: en-US
---

# {{ $frontmatter.title }}

Optimistic Ethereum contracts are subject to a few limitations, because they need to
be able to run in two separate environments:

- In Optimistic Ethereum itself, for normal operations.
- Inside a virtualized environment on the main Ethereum network in case of a
  [transaction challenge](/docs/protocol/protocol.html#transaction-challenge-contracts).

The issue is that even if the state of the contract is identical, certain 
opcodes such as `CHAINID` and `TIMESTAMP` produce different results. This could
result in a legitimate transaction result being challenged, and the challenge producing
a different result.

Optimistic Ethereum gets around this problem by using a slightly modified Solidity
compiler. Where the standard compiler produces those opcodes, the Optimistic version
produces a call to a different contract that provides consistent information, whether
we are running in L1 or L2. This is the source of the restrictions on contracts:


1. Contracts have to be written in Solidity. We would love to provide compilers for
   Vyper and Yul, but those aren't our priority at the moment.

1. Contract length limits are stricter. The length limit is still 
   [24 kB](https://github.com/ethereum/EIPs/blob/master/EIPS/eip-170.md), but
   the problem opcodes are one byte long. We replace them with a contract call that 
   needs to push parameters into the stack, have the call itself, and then read the
   result. This means that a contract that was close to the limit when compiled with
   normal Solidity might be over the limit with our version of the compiler. 
   [See below for some workarounds to this problem](#working-around-the-contract-length-issue).

1. Both Hardhat and Truffle allow you to run contract tests against their own 
   implementations of the EVM. However, to test contracts that run on Optimistic
   Ethereum you need to run them on a local copy of Optimistic Ethereum, which uses
   [geth](https://geth.ethereum.org/). This requires the use of `Promise` objects,
   and prevents the use of some debugging features that the development environment
   provides. [See below for more details](#running-tests-against-geth).

1. Constructor parameters. Before deploying a new contract, Optimistic Ethereum
   has to check that it does not contain any of the problem opcodes. But when
   one contract creates another part of the code space is used for constructor 
   parameters. If those parameters contain the byte for one of the problem opcodes,
   the creation process fails.


## Working Around the Contract Length Issue

The 24 kB limit is per contract. If you need more you can always move some of your
code to [a library](https://docs.soliditylang.org/en/v0.8.6/contracts.html#libraries),
which means it would be placed in a different contract. 


## Running Tests Against Geth

There are two issues involved in running your tests against a geth instance, rather
than an EVM running inside your development environment:

- Calls take longer, because they require inter-process communication (at least) and
  therefore you might get a [`Promise` 
  object](https://www.w3schools.com/js/js_promise.asp) that isn't resolved yet. You
  have to write your tests to be 
  [asynchronous](https://developer.mozilla.org/en-US/docs/Learn/JavaScript/Asynchronous/Concepts).
- Functions that were added to the development environments to facilitate debugging are
  not available on geth.

### Asynchronous Tests

To cause the test to wait until the actual result is attained, put `await` before the
call. If you are defining functions, define them with the `async` keyword. [You can 
read more about this here](https://www.w3schools.com/js/js_async.asp). 

### Debugging Features

Both [Truffle's Ganache](https://github.com/trufflesuite/ganache-cli#custom-methods) and 
[Hardhat](https://hardhat.org/hardhat-network/#special-testing-debugging-methods) support custom debugging methods such as `evm_snapshot` and `evm_revert`. You cannot 
use these methods in tests for Optimistic Ethereum contracts because they are not available in geth. Nor can you use [Hardhat's `console.log`](https://hardhat.org/tutorial/debugging-with-hardhat-network.html)


## Creating Contracts with Safe Constructor Parameters

# **GOON GOON GOON**