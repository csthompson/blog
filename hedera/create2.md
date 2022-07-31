# Exploring HIP-329, the CREATE2 Opcode on Hedera

[HIP-329](https://hips.hedera.com/hip/hip-329) introduced the ability to use the CREATE2 opcode on the Hedera Hashgraph. CREATE2, originally introduced to Ethereum with [EIP-1014](https://eips.ethereum.org/EIPS/eip-1014), allows for the deterministic deployment of smart contracts on Ethereum. Before the advent of CREATE2, smart contracts could be deployed to a non-deterministic address based on the sender's nonce and address. The sender would either be an externally owned account (EOA) (smart contract deployment) or a smart contract. In either case, the nonce in the original CREATE opcode introduced non-determinism, meaning there was not a way to reliably perform reproducible deployments to the same smart contract address. CREATE2 changes this by leveraging the sender's address, a user-defined salt, and the bytecode being deployed. Below are the two calculation methods compared:

CREATE:
```
keccak256(senderAddress, nonce)
```
Where the nonce is the current nonce of the sender's address. More about the Ethereum nonce can be seen in [this](https://medium.com/swlh/ethereum-series-understanding-nonce-3858194b39bf) blogpost.

CREATE2:
```
keccak256(0xFF, senderAddress, salt, keccak256(bytecode))
```
Where the salt is user-defined and can be generated using various methods such as tracking the most recent salt for an address, the address of a user, an incrementing value, or the hash of a trading pair.

*Note:* CREATE2 can only be called from a smart contract. It is not possible to use CREATE2 for smart contract deployment from an externally owned account.

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

### Using CREATE2 in a Hedera Smart Contract
CREATE2 can be leveraged in smart contracts to easily create factory patterns (factories that deploy other smart contracts) or to deploy smart contracts for user interaction. We will dive into these use cases shortly, but let's take a look at how CREATE2 can be used in a smart contract.

#### Deployment
CREATE2 has recently been more tightly integrated into Solidity, but used to require the use of inline assembly. Luckily, inline assembly is no longer required. In order to deploy a smart contract, you simply use the `new` keyword, the bytecode of the smart contract you are deploying, the deployment salt, and the constructor arguments for the contract. The bytecode argument is simply the value from an `import` at the top of the smart contract. Below is an example snippet:
```
import "./ContractToDeploy.sol";

...

address deployedContract = address(
    bytes32 deploymentSalt = 0x00;
    new ContractToDeploy{salt: deploymentSalt}(owner_)
);
```
In this example, the smart contract deployment code takes an argument named `owner_` which is the address that the deployed contract should be owned by. However, the arguments behave the same as any other smart contract and can be any combination of arguments. Because each deployed contract in our use case has a unique owner, the deploymentSalt can be a constant since the calculation of the CREATE2 address takes into consideration the deployment code. We will see the deployment code used below to calculate the address.

#### Calculating the address
The power of CREATE2 comes from the deterministic nature of the deployment, meaning the address is easy to compute even before a smart contract gets deployed to the address. Let's look at how the address can be computed using a view function below:
```
import "./ContractToDeploy.sol";

...

function getContractAddress(address owner_) public view returns (address) {
    bytes32 deploymentSalt = 0x00;
    return
        address(
            uint160(
                uint256(
                    keccak256(
                        abi.encodePacked(
                            bytes1(0xff),
                            address(this),
                            deploymentSalt,
                            keccak256(
                                abi.encodePacked(
                                    type(ContractToDeploy).creationCode,
                                    abi.encode(owner_)
                                )
                            )
                        )
                    )
                )
            )
        );
}
```

This looks a bit intimidating, but is simply the original calculation method parameterized and put into a view function. Let's break this code down from the inside out.

We first encode the creation code of the smart contract that is being deployed along with any contructor arguments, and then take the keccak256 hash of the resulting value. This represents the 4th argument in the CREATE2 opcode.
```
keccak256(
    abi.encodePacked(
        type(ContractToDeploy).creationCode,
        abi.encode(owner_)
    )
)
```

We then encode this with the constant `bytes1(0xff)` , the address of the contract we are deploying from, and the deployment salt. Finally, we take the keccak256 value of the encoding.
```
keccak256(
    abi.encodePacked(
        bytes1(0xff),
        address(this),
        deploymentSalt,
        keccak256(
            abi.encodePacked(
                type(ContractToDeploy).creationCode,
                abi.encode(owner_)
            )
        )
    )
)
```

The final part to the calculation is converting the keccak256 value into an EVM address. This is done by casting the output of keccak256 to a uint256, then taking the least-significant 160 bits of the uint256 value, and finally casting this to an address type.
```
address(
    uint160(
        uint256(
            keccak256(
                abi.encodePacked(
                    bytes1(0xff),
                    address(this),
                    deploymentSalt,
                    keccak256(
                        abi.encodePacked(
                            type(ContractToDeploy).creationCode,
                            abi.encode(owner_)
                        )
                    )
                )
            )
        )
    )
);
```
It is important to note that this calculation method can be used in client code as well, leveraging various SDKs' built-in keccak256 functions and ABI packing capabilities.

The calculation method presented here can be leveraged to pre-calculate EVM addresses, even before smart contract code is deployed to the address. For different use cases, all that would need to be changed is the creation code, the arguments to the constructor, and then based on the arguments to the constructor, determining if a constant deployment salt can be leveraged. Let's look at some use cases where CREATE2 can be leveraged.

### CREATE2 Use Cases
CREATE2 can be used in a wide range of use cases, from smart accounts to easing onboarding in web3 applications. These use cases are enabled by the deterministic deployment of smart contracts, and the ability to use an address before the smart contract has been deployed. This concept of being able to use an address or contract before it has been deployed is called _counterfactual instantiation_ and is a boon for scalabilitiy and end-user experience in decentralized applications.

Let's explore the utility of CREATE2 in the context of fundraising. Assume a platform wants to offer users the ability to setup decentralized fundraising goals. Each goal would have its own treasury, deadline, and recipient(s) that would receive the payout of the goal. Deploying smart contracts for users can become quite expensive for a web3 platform, so deploying the fundraising contract at the moment of creation would be cost prohibitive and would not scale well. However, with CREATE2, the fundraising contracts can be utilized without ever deploying the contract itself. 

Assume contract deployment costs $1.00 as noted in this [schedule](https://docs.hedera.com/guides/mainnet/fees). The smart contract address can be pre-computed using a function similar to the one above, with constructor arguments representing the deadline, parameters, and recipient(s) of the payout. This address is guaranteed to only be the smart contract account for the logic presented and the contract with the constructor arguments provided, therefore the funds cannot be used in any way except for the logic specified in the bytecode used in the computation of the CREATE2 address. This creates trust in the logic of the smart contract, without the smart contract ever being deployed. The contract would also have logic to refund the gas fees to the deployer using the funds available in the smart contract account. The platform can specify an threshhold point where the smart contract can be deployed, with the funds in the underlying smart contract account being able to cover the cost of the gas. This removes the need for creating the smart contract at inception, while still being able to fund the smart contract account and trusting the underlying logic, therefore introducing the powerful capability of _counterfactual instantion_. 

### Conclusion
The Hedera Hashgraph provides a powerful decentralized platform, supporting native tokenization, robust smart contracts, and more. Pairing the predictable low fees and scalability of Hedera with the power of the CREATE2 opcode enables users to build incredible user experiences. As you develop on top of the Hedera Smart Contract Service, and you run into a situation where you need to ease the user onboarding process or deploy unique contracts for each user, keep in mind the CREATE2 opcode and counterfactual instantiation. 

