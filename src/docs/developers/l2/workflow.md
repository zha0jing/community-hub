---
title: Development Workflow
lang: en-US
---

# {{ $frontmatter.title }}

::: warning NOTICE
This page refers to the design of the current iteration of the Optimistic Ethereum protocol.
Details here are subject to change as the Optimistic Ethereum protocol evolves.
:::

Roughly speaking, these are the steps you need to take to develop for Optimistic
Ethereum:

1. Develop the decentralized application normally.
1. [Create an Optimistic Ethereum development node](/docs/developers/l2/dev-node.html)
1. Run your tests on the Optimistic Ethereum development node you created.
1. Deploy your dapp to the [Optimistic 
   Goerli](/docs/infra/networks.html#optimistic-goerli) network and test it in that
   environment.
1. Upload and verify the contracts' source code on Optimistic Goerli 
   Etherscan
1. [Ask to be added to the Optimistic Ethereum whitelist](https://docs.google.com/forms/d/e/1FAIpQLSdKyXpXY1C4caWD3baQBK1dPjEboOJ9dpj9flc-ursqq8KU0w/viewform)    
1. Once added, deploy your contracts to the 
   [Optimistic Ethereum](/docs/infra/networks.html#optimistic-ethereum) network. Then, upload and 
   verify your contracts' source code on [Optimistic Etherscan](https://optimistic.etherscan.io/verifyContract)