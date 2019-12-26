# Bitcannery time-locked encryption

Bitcannery is an open protocol for triggered secret sharing. It could be used for dead-man-switch applications, data escrow, managed data leaks, last-resort secret backups and disaster recovery device.
Current implementation works on top of Ethereum network. You could check out Registry smart contracts on Mainnet and Rinkeby test network

## Bitcannery infrastructure and secret sharing

Bitcannery provides the infrastructure for triggered secret sharing with several properties:
1. The secret is encrypted so that only the original sender has access to the secret text before disclosure event
2. The secret is encrypted with asymmetric cryptography (currently it’s ECC), so it could be accessed only with private key
3. Once disclosure event is reached, no single user could prevent secret disclosure

To ensure these properties, Bitcannery relies on the network of anonymous actors — Keepers. No single Keeper has any access to secrets on the network, and no single Keeper could affect secret disclosure after trigger event, nor affect trigger event detection.

Every Bitcannery secret is encrypted twice upon sharing. First layer is ECC encryption with receiver’s public key, and the second layer is AES with one-time key, which is divided between number of Keepers with [Shamir’s Secret sharing algorithm](https://en.wikipedia.org/wiki/Shamir's_Secret_Sharing). Keepers don’t have access to secret text and no single Keeper could jeopardise secret disclosure.

Secret sender has the option to delay secret disclosure. To do so they have to “check in” with secret and signal the Keepers not to disclose their key parts for some time interval. Once this interval elapses without sender signaling another delay, Keepers disclose their key parts. Having enough Keepers’ key parts holder of the private key could decrypt the secret.

The detailed protocol spec can be found [here](https://github.com/bitcannery/bitcannery-docs/blob/master/README.md)

## Time-locked encryption over Bitcannery

Bitcannery infrastructure could be used as trustless time-locked secret vault. In order to do so, Alice needs to share the secret with check-in interval equal to time-lock length, and skip checking in altogether. On check-in interval expiration Keepers will disclose their parts of key, making the original secret available for anyone with the private encryption key. Bitcannery DApp supports secret-specific urls for decryption of disclosed secrets.

The steps required to perform time-locked encryption over Bitcannery:

1. Create one-time Ethereum address for sending the message. This address would need to have Ether for transaction fees
2. Use Bitcannery [DApp](https://bitcannery.github.io/bitcannery-dapp/#/generate-key) to generate the pair of private and public keys
3. Share the secret with Bitcannery [DApp](https://bitcannery.github.io/bitcannery-dapp/#/new-secret). Length of check-in interval is the minimum time before secret disclosure
4. After secret is submitted, it receives Keepers’ proposals. Choose number of Keepers to look over the key and activate the secret
5. Once the secret is activated, Ethereum address from the first step could be forgotten. Anyone with access to the address could postpone or cancel secret’s disclosure
6. If the secret is to be made public after disclosure, prepare the link to decryption. Bitcannery DApp provides a page for disclosed secrets decryption. To read the secret one needs secret’s name and receiver’s private key. [The DApp page](https://bitcannery.github.io/bitcannery-dapp/#/decrypt)supports query string parameters name and key. So for the secret it should look like https://bitcannery.github.io/bitcannery-dapp/#/decrypt?name=secret_name_from_dapp&key=private_key. *secret_name_from_dapp* is the name assigned to secret by Bitcannery (it’s in the DApp), and *private_key* is the private key of the keypair generated on step 2
7. Once check-in interval passes, Keepers start sending their key parts. After enough Keepers have submitted the key parts, anyone with secret’s name and private key would be able to decrypt the secret

From the technical standpoint actions are the same as in [dead-man-switch case](https://github.com/bitcannery/bitcannery-docs/blob/master/README.md#happy-path) with two differences:
1. Sender generates asymmetric keypair by themself
2. Sender doesn’t ever check in to postpone the secret’s disclosure

Couple things to keep in mind while time-locking the secret on Bitcannery:
1. Ethereum address sending the secret to the network should be untrackable one-time only, with Ether coming there through crypto exchange or other anonymizer
2. This Ethereum address is to be forgotten right after secret activation, so no one could postpone or cancel the disclosure
3. Check-in interval is not the exact time of the secret disclosure. It defines the moment after which Keepers would start sending their key parts to the secret. Gathering enough key parts for Shamir’s Secret sharing algorithm takes some time
