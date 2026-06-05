# dApp Ideas - architechture (mvp focused)
> personal experiments tracker; e2ee with uncompromised ux

> [!TIP]
> APP_ID = `keccak256("com.dapp.ideas/alpha")` = \
> `0x8a8fbc3f4562a759a7ff2a7bbe961df6a719fd660c4c5dd6e691a4a795b73382`

The primary design philosophy revolves around keeping the ux familiar to web2 senses, while relying on web3 niceties to offer newer experience and market access to the [varied] users.

## Affordances

We strive to offer multiple ux affordances, abstracting web3 complexities and other complex computer stuff (note that over time we can and will expose all these parts either as enterprise configurations or open source software or both).

### Key Management

Sia operates using a BIP-39 seed phrase and the immutable APP_ID. We use the same master seed to derive keys for various purposes (encryption keys, multichain addresses, etc.)

Additionally, we use AES-GCM symmetric encryption to save an encrypted version of the master seed, encrypted by the user's auth method.

### Easy Auth

The basic auth offered is a simple username and password. The username is also generated server side from a curated wordlist

### Conflict Resolution

### Caching

### Sharing Module

## MVP Scope


### Larger Vision


## Requirements

## Components

### Client

### Backend

### Data model

## Cost Estimation
