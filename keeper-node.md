# Bitcannery Keeper node

## What is Bitcannery

Bitcannery is an open protocol designed to provide an encryption with extendable self-decryption time. Bitcannery time-locks an encrypted message on a blockchain to be auto-decrypted at a set point in time. That point in time can be continuously pushed forward by the sender.

This is useful for timed-out message delivery, dead-man-switch-like applications, data escrow, data leaks, last-resort backups, disaster recovery, etc.

Currently Bitcannery runs on both Ethereum mainnet and Rinkeby test network, but it doesn't have to be bound to Ethereum. Bitcannery system can be generalized for virtually any distributed ledger supporting smart contracts.

## Keepers

To implement trustless secret sharing, Bitcannery uses the network of anonymous actors — `Keepers`. The keys used to encrypt secrets are split into shards (using Shamir's Secret Sharing algorithm) and distributed between Keepers. Keepers keep their shards until the sender pays for keeping. After the sender stops paying, Keepers incentivized to publish their shards to receive the final payout.

Keepers run nodes to discover new secrets and to maintain (keep) existing secrets while collecting keeping fees.

## Keeper node app

The keeper node is implemented as a single binary Node.js command-line application. It requires Node 8+ and Npm 5+. Latest release could be found [here](https://github.com/bitcannery/bitcannery-cli/releases).

The app can be run as Linux daemon as described [here](https://github.com/bitcannery/bitcannery-cli/tree/master/swarm#run-keepers-as-a-daemons).

Keeper node can be run in the cloud. Setting up and running multiple keeper nodes on DigitalOcean instance with Terraform is described [here](https://github.com/bitcannery/bitcannery-cli/tree/master/swarm).

## Keeper node prerequisites and configuration

Keeper node starts running on `./bitcannery-cli keeper run` command. On the first run, cli will ask to import or generate new BIP39 seed phrase for Keeper's Ethereum account. Also, before the first run Keeper must specify keeping fee: `./bitcannery-cli keeper set-fee <X_Gwei>`. `<X_Gwei>` here is a daily fee for keeping single secret's key part.

Alternative way of setting up a Keeper node is restoring it's settings from backup with command `./bitcanery-cli restore`.

Keeper Ethereum account ought to have small amount of ether to send out Ethereum transactions.

Keeper node communicates with Ethereum network through Infura cloud nodes. All smart contract communication is performed with json-rpc http calls and websocket subscription.

Note, that Keeper node can be configured to run on regular geth node instead of Infura. Websocket connection can be replaced with http polling.

On top of Keeper features, cli app implements interface for sending and receiving secrets.

## More information

Full technical description of the protocol and client apps can be found here: https://github.com/bitcannery/bitcannery-docs.
