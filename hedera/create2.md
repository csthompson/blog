# Exploring HIP-329, the CREATE2 Opcode on Hedera

[HIP-329](https://hips.hedera.com/hip/hip-329) introduced the ability to use the CREATE2 opcode on the Hedera Hashgraph. CREATE2, originally introduced to Ethereum with [EIP-1014](https://eips.ethereum.org/EIPS/eip-1014), allows for the deterministic deployment of smart contracts on Ethereum. Before the advent of CREATE2, smart contracts would be deployed to a non-deterministic address based on the sender's nonce and address. The sender would either be a user's address (smart contract deployment) or a smart contract. In either case, the nonce in the original CREATE opcode introduced non-determinism, meaning there was not a way to reliably perform reproducible deployments to the same smart contract address. CREATE2 changes this by leveraging the sender's address, a user-defined salt, and the bytecode being deployed. Below are the two calculation methods compared:

CREATE:
```
keccak256(senderAddress, nonce)
```
Where the nonce is the current nonce of the sender's address. More about the Ethereum nonce can be seen in [this](https://medium.com/swlh/ethereum-series-understanding-nonce-3858194b39bf) blogpost.

CREATE2:
```
keccak256(0xFF, senderAddress, salt, bytecode)
```
Where the salt is user-defined and can be generated using various methods such as tracking the most recent salt for an address, the address of a user, an incrementing value, or the hash of a trading pair.

*Note:* CREATE2 can only be called from a smart contract. It is not possible to use CREATE2 for smart contract deployment from a user address.

## How HIP-329 is Implemented

The Hedera Hashgraph is intrinsically different from other EVM-based blockchains in many ways. Most notably:

1. It is not a blockchain
2. It is not inherently EVM-based

Rather, the Hedera Hashgraph is a hashgraph, a novel structure based on a directed acyclical graph (DAG) that lends itself to a powerful consensus protocol called Hashgraph consensus. On the second point, the Hedera Hashgraph provides the ability to reliably execute EVM-compatible smart contracts, but this is only one of its many services. Behind the scenes, Hedera's smart contract service (HSCS) leverages the same consensus engine it exposes through the [Hedera Consensus Service](https://hedera.com/consensus-service), and is enabled through a custom implementation of [Hyperledger Besu](https://www.hyperledger.org/use/besu). The custom implementation of Hyperledger Besu enables bespoke interactions with the rest of the Hedera services such as smart contracts being able to interact with the native Hedera Token Service (HTS) as seen [here](https://docs.hedera.com/guides/core-concepts/smart-contracts/hedera-services-integration-with-smart-contracts).

### Accounts
These key differences mean that there is some magic that goes into making the CREATE2 opcode work. The first main difference is to account for, well, accounts. Accounts on the Hedera Hashgraph are designed to be human readable, semantically relevant, and comprehensible. 

On EVM-based chains, addresses look like `0xb794f5ea0ba39494ce839613fffba74279579268`. This is fantastic for a computer to understand, but is not entirely readable for a human. Hedera's accounts instead look like `0.0.34401272`, which follow a pattern of:
```
<shard>.<realm>.<account>
```
More about this format can be seen [here](https://docs.hedera.com/guides/core-concepts/accounts)

This format has meaning for the end-user, and can easily be represented and understood. This however poses a challenge for the CREATE2 opcode where the calculation generates an EVM (Solidity) address. To enable calling smart contracts deployed using CREATE2, Hedera uses the concept of an account "alias". This concept is also used for [HIP-32](https://hips.hedera.com/hip/hip-32) for the auto-creation of an account where the receiving account can be in the format of `<shard>.<realm>.<public key>`. For CREATE2, the account will follow the format `<shard>.<realm>.<evm address>`. An example of an account for a smart contract deployed with CREATE2 will look like the following:
```
0.0.0xb794f5ea0ba39494ce839613fffba74279579268
```
