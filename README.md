# Bitcannery Secret Sharing Protocol

![Bitcannery logo](/img/logo.png)

Bitcannery is an open protocol for triggered secret sharing. It was designed to provide a way to transfer blockchain account private keys in case of untimely death of the account owner. The protocol could be used for dead-man-switch applications, data escrow, managed data leaks, last-resort secret backups and disaster recovery device.

Current implementation works on top of Ethereum network. You could check out Registry smart contracts on [Mainnet](https://etherscan.io/address/0x0481D19CDd6a12aa2f85b4a46A31D1d17BA33543) and [Rinkeby test network](https://rinkeby.etherscan.io/address/0x5a6db32a129e9f3ade0c2c9d7ed382b8607ae6f3 )

## Problem space
Bitcannery protocol is designed to solve secret sharing with three major constraints:

1. Only the person intended to receive the message is ever able to read it.
2. Person intended to receive the message is able to read it only after sender loses ability to send the message by themselves.
3. Person intended to receive the message reliably gets the message without any single third party being able to deny the access.

## Protocol
Let’s call the secret sender `Alice`, and the person she’s sending a secret message to `Bob`. Alice wants to send a message to Bob, but have it made accessible to Bob only once she’s not able to reveal it herself. To prevent anyone from getting the message earlier than required, Bitcannery uses the network of anonymous actors — `Keepers`. Every keeper has a keypair with public key available to any would-be Alice.

The protocol works in three stages:
1. Sending the message
2. Keeping the message secret
3. Revealing the message

### Sending the message
To send a message, Alice does two things. First, she authors the message and encrypts it twice: first with Bob’s asymmetric public key (ECC in current implementation), then with random symmetric key (AES in current implementation). Second, she chooses number of Keepers. Symmetric key is divided with [Shamir’s secret sharing algorithm](https://en.wikipedia.org/wiki/Shamir's_Secret_Sharing). Every key part is assigned to one Keeper. Alice encrypts the key part with keeper’s public key, notifies all chosen Keepers and sends twice-encrypted secret and encrypted key parts to secure storage (Ethereum smart contract in current implementation). Original data hashes are stored so decryption results could be checked for all three encryptions.

### Keeping the message secret
Keeping the message secret is based on check-in cycle. Alice specifies time interval she’ll be checking in with the secret. During this time from the last check-in Keepers suppose she has access to her private key, and the secret must be kept off Bob. Every Keeper checks in with the secret during Alice’s check-in interval to see whether she’s been checking in on time.

### Revealing the message
Once the Keeper reveals Alice’s missed check-in, they change the secret’s state to “Receiving key parts”. When the Keeper notice secret’s status change, they decrypt their key part and send it to secret storage. Shamir's Secret Sharing is set up so that less than 100% of key parts are required to restore the key. Then there’s enough key parts, Bob restores symmetric key with Shamir’s secret sharing algorithm, decrypts the message from symmetric encryption, decrypts the result with his private key and finally gets the Alice’s secret message.

## Keeper economics
Keepers must 1) look for Alices’ AES key parts, 2) check in with their secrets timely and 3) provide decrypted key parts for Bobs to gather. To motivate the Keepers to do so, Bitcannery creates market for Keepers’ services. Every Keeper specifies the keeping fee for their duties, and upon accepting the Keeper Alice accepts this fee. Every check-in Alice sends all the Keepers’ fees to the secret, so Keepers could collect their fees on Keepers’ check-in. Keepers get their last keeping fees by supplying decrypted key parts to the secret.

## Implementation
Currently Bitcannery protocol is implemented as two Ethereum smart contracts (single [Registry contract](https://github.com/bitcannery/bitcannery-dapp/blob/master/core/contracts/Registry.sol) and [Secret sharing contracts](https://github.com/bitcannery/bitcannery-dapp/blob/master/core/contracts/CryptoLegacy.sol) for every secret shared by the protocol), [DApp for Alice and Bob](https://github.com/bitcannery/bitcannery-dapp/tree/master/web) and [command-line application for Keepers](https://github.com/bitcannery/bitcannery-cli/).

### Secret Sharing smart contract
Alice’s secret is stored inside this [smart contract](https://github.com/bitcannery/bitcannery-dapp/blob/master/core/contracts/CryptoLegacy.sol). Alice deploys the instance of Secret Sharing smart contract for every secret. Once deployed, Secret Sharing smart contract accepts `CallForKeepers` state and receives proposals from Keepers. Every proposal consists of required Keeping fee and Keeper’s public key, with Keeper’s Ethereum address being saved along with the data. Once the contract gets enough proposals to reliably run the secret, Alice accepts preferred Keepers, saves the twice-encrypted secret and starts check-in cycle by setting contract in `Active` state. Every once in a defined time interval Alice checks in to postpone secret revealing. Keepers check in with this contract to collect their keeping fees and check whether the secret should be revealed. Once the Keeper catches Alice’s missed check-in, smart contract gets into `CallForKeys` state and starts receiving Keepers’ key parts. Keepers decrypt their parts of AES key and send them to the smart contract. Once enough key parts are supplied, anyone can get the twice-encrypted message, restore AES key with Shamir’s secret sharing algorithm and decrypt AES encryption layer. Only someone with Bob’s private key can then decrypt asymmetric encryption layer and get the secret message text itself. Outside the “happy path” scenario, Secret sharing contract could be cancelled by Alice receiving `Cancelled` state.

### Registry smart contract
To notify Keepers about new Secret Sharing contracts, Bitcannery protocol requires every Secret sharing contract to be registered in single [Registry smart contract](https://github.com/bitcannery/bitcannery-dapp/blob/master/core/contracts/Registry.sol). Upon registering, every Secret Sharing smart contract gets `name` — unique human-readable id. Keepers watch for events on this smart contract, and thus are able to discover new Secret Sharing contracts and send their “Keeping proposals” there. From there on, secret keeping process goes inside Secret Sharing smart contracts. Currently deployed version provides a method to create fresh Secret Sharing smart contract and register it in the Registry in single Ethereum transaction. Current Registry contracts are deployed to [0x0481D19CDd6a12aa2f85b4a46A31D1d17BA33543](https://etherscan.io/address/0x0481D19CDd6a12aa2f85b4a46A31D1d17BA33543) on Mainnet and [0x5a6db32a129e9f3ade0c2c9d7ed382b8607ae6f3](https://rinkeby.etherscan.io/address/0x5a6db32a129e9f3ade0c2c9d7ed382b8607ae6f3) on Rinkeby test network.

### DApp
Using the [DApp](https://bitcannery.github.io/bitcannery-dapp) ([code](https://github.com/bitcannery/bitcannery-dapp)) Alice can create new Secret Sharing smart contract and check-in with already existing ones, and Bob can get the secret. Aside from secret management, DApp provides offline ECC keypair generator. DApp communicates with Ethereum blockchain through MetaMask browser extension and is implemented as React web single-page application.

### Command-line application
For Keepers’ convenience Bitcannery provides node.js-based [command-line application](https://github.com/bitcannery/bitcannery-cli), which could be run without much oversight as [linux daemon](https://github.com/bitcannery/bitcannery-cli/tree/master/swarm#run-keepers-as-a-daemons). The application provides an interface for Keeper to set [their keeping fee](https://github.com/bitcannery/bitcannery-cli/blob/master/HOWTO.md#run-as-a-keeper) and check Keeper’s balance. While running in “Keeper’s mode” app polls Registry smart contract through [Infura](https://infura.io/) Ethereum node, gets smart contract events for new Secret Sharing smart contracts, automatically sends Keeping proposals to this new Secret Sharing contracts and checks in with already active Secret Sharing smart contracts, decrypting and sending AES key parts to ones being in `CallForKeepers` state. List of all Keeper’s contracts is kept in local cache. The cache could be backed up and restored on another machine if required.
Aside from Keeper utilities, app provides an alternative interface for Alice and Bob to send and receive secret messages.

## Happy path
1. Alice gets Bob’s public key
2. Alice uses DApp to call Registry smart contract method `deployAndRegisterContract` to create new Secret Sharing smart contract in `CallForKeepers` state and register it in Registry smart contract
3. Keepers get Registry smart contract event about new Secret Sharing smart contract and call it's `submitKeeperProposal` method
4. Alice gets enough Keepers to reliably run secret sharing. She calls uses DApp to activate Secret Sharing smart contract setting it in `Active` state by calling `acceptKeepersAndActivate` or `acceptKeepers` and `activate` methods, depending on the number of keepers accepted. DApp encrypts the secret text with Bob’s public key provided by Alice, During this calls, generates AES key, encrypts the secret text once again with this AES key, divides AES key with Shamir’s secret sharing algorithm, assigns AES key parts to Keepers, encrypts these key parts with Keepers’ public keys and saves them along with twice-encrypted secret text in Secret Sharing smart contract
5. Alice checks in with the Secret Sharing smart contract through DApp, which calls `ownerCheckIn` method and sends Ether for keepers’ fees
6. Keepers check in with the Secret Sharing smart contract through command-like app. App calls `keeperCheckIn` smart contract method
7. Once Alice fails to check-in during check-in period, next Keeper sending check-in request sets Secret Sharing smart contract to `CallForKeys` state
8. Every Keeper checking in with Secret Sharing smart contract through command-line app detects smart contract state change, decrypts their key part and provides it to Secret Sharing smart contract by calling `supplyKey` method
9. Bob uses DApp to locate Secret Sharing smart contract by checking its name in Registry smart contract with `getContractAddress` method. Then DApp collects all supplied AES key parts, restores the AES key with Shamir’s secret sharing algorithm, decrypts AES encryption, and uses private key provided by Bob to decrypt ECC layer of encryption and reveal secret text.
