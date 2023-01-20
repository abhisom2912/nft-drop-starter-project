---
title: Configure and Run Orchestrator
sidebar_position: 3
---


<details>
<summary><b>Step 0) Run validator node before running orchestrator</b></summary>

Follow [this guide](./becoming-validator) to become a validator.

</details>

<details>
<summary><b>Step 1) Configure the orchestrator</b></summary>

```jsx
mkdir .router-orchestrator
cp network-config/devnet/10001/orchestrator-config.json ~/.router-orchestrator/config.json
cd ~/.router-orchestrator
```

Update the `chainRpc` in the `config.json` file with a valid EVM RPC Endpoints for all the chains.

Orchestrator also requires access to the validator's Cosmos and Ethereum credentials to sign transactions for the corresponding networks.

### Cosmos Keys

There are two ways to provide the credential access - a keyring with encrypted keys, or just a private key in plaintext.

**1. Cosmos Keyring**

Update the `cosmosPrivateKey` to the validator key name (or account address). Please note that the default keyring backend is `file` and it will try to locate the keys on the disk.

The keyring path must be pointing to homedir of the routerd node, in case keys needs to be reused from there.

**2. Cosmos Private Key (Unsafe)**

Simply update the `cosmosPrivateKey` with Validator's Account private key.

To obtain the validator's Cosmos private key, run `routerd keys unsafe-export-eth-key $VALIDATOR_KEY_NAME`.

### Ethereum Keys

To provide the credential access, a private key in plaintext needs to be provided.

**Ethereum Private Key (Unsafe)**

Simply update the `ethPrivateKey` with a new Ethereum Private Key from a new account.

Then ensure that the Ethereum addresss has balance for all the configured EVM chains. 

</details>

<details>
<summary><b>Step 2) Register the Ethereum Address</b></summary>

Submit `set-orchestrator-address` tx to Routerchain with **orchestrator-router-address** and **orchestrator-eth-address.** 

This tx will register the orchestrator addresses on Routerchain

```
routerd tx attestation set-orchestrator-address [orchestrator-router-address] [orchestrator-eth-address]

Example: routerd tx attestation set-orchestrator-address router1emlu0gy7hju5pywvmkhy529f7s24ydtm49pwcl 0x1E5B81378a1D484169aB9b133FFD97003316e840 --from my-node --home ~/.routerd --keyring-backend file --chain-id router-1  --fees 100000000000000route
```

Successful registration can be verified by checking for Validator's mapped Ethereum address on [list of orchestrators](https://devnet.lcd.routerprotocol.com/router-protocol/router-chain/attestation/list_orchestrators).

</details>

<details>
<summary><b>Step 3) Start the Orchestrator</b></summary>

```jsx
cd ~/.router-orchestrator
router-orchestrator start --reset --config ~/.router-orchestrator/config.json
```

This will start the Orchestrator.

</details>