---
title: Converting Applications for Optimistic Ethernet
lang: en-US
---

# {{ $frontmatter.title }}

Optimistic Ethereum contracts are subject to a few limitations, because they need to
be able to run in two separate environments:

1. In Optimistic Ethereum itself, for normal operations.
1. Inside a virtualized environment on the main Ethereum network in case of a
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
1. Both Hardhat and Truffle allow you to run contract tests against their own 
   implementations of the EVM. However, to test contracts that run on Optimistic
   Ethereum you need to run them on a local copy of Optimistic Ethereum, which uses
   [geth](https://geth.ethereum.org/). **GOON GOON GOON**
1. Constructor parameters