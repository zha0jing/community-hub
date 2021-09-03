---
title: Tooling for Hardhat
lang: en-US
---

# {{ $frontmatter.title }}

To use Optimistic Ethreum from Hardhat, simply connect either to 
[a local development node](dev-node.html), [the Goerli test network, or the
production network](http://localhost:8080/docs/infra/networks.html).

## Setup 

Add an `optimistic` network to your exported config:

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

## Usage

Simply add `--network optimistic` whenever you want to test, deploy, or compile an Optimistic Ethereum app.
For instance:

```sh
npx hardhat compile --network optimistic
```
