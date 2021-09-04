---
title: Tooling for Development Environments
lang: en-US
---

# {{ $frontmatter.title }}

Optimistic networks appear identical to normal Ethereum, so whatever development
environment you use, you just need to configure new networks. These networks can be:

* [A local development node](dev-node.html)
* [The Goerli test network](/docs/infra/networks.html#optimistic-goerli)
* [The production network](/docs/infra/networks.html#optimistic-ethereum)

## Hardhat

In Hardhat the configuration file to edit is `hardhat.config.js`. To 
use a local development node, add this network definition:

```javascript
...
module.exports = {
   ...
   networks: {
   ...
      optimistic: {
         url: 'http://127.0.0.1:8545', // this is the default port
         accounts: { mnemonic: 'test test test test test test test test test test test junk' }

      }
   }
   ...
}
```

And then run your commands with `--network optimistic`.

## Truffle

In Truffle the configuration file to edit is `truffle-config.js`. To 
use an external server (rather than [Ganache](https://www.trufflesuite.com/ganache)), 

1. Uncomment this line:
   ```javascript
   const HDWalletProvider = require('@truffle/hdwallet-provider');
   ```

1. Add this definition to the networks:
   ```javascript
   module.exports = {
      ...
      networks: {
         ...
         optimistic: {
            provider: function() {
                return new HDWalletProvider(
                  "test test test test test test test test test test test junk",
                  "http://127.0.0.1:8545", 0, 1)             
            },
            network_id: "*"
         }
      }
      ...
   }
   ```

1. In the command line add the `@truffle/hdwallet-provider` package:

   ```sh
   yarn add @truffle/hdwallet-provider
   ```


## Remix

To use [Remix](https://remix.ethereum.org/):

1. [Connect your wallet](/docs/users/metamask.html) to the appropriate Optimistic network.
1. Click the Ethereum icon in the left sidebar.
1. Select the environment **Injected Web3**.