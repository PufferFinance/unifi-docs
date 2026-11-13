---
title: Network & RPC Endpoints
slug: /developers/rollup/reference/rpc-endpoints
---

## Puffer UniFi Testnet

You can find access information to Puffer UniFi Testnet RPC below:

| Parameter          | Value                                    |
|--------------------|------------------------------------------|
| Network Name       | `Puffer UniFi Testnet`                          |
| Chain ID           | `2092151908`                             |
| Currency Symbol    | ETH                                      |
| Block Explorer     | https://testnet-unifi-explorer.puffer.fi/|
| Sequencer URL      | https://testnet-unifi-rpc.puffer.fi/      |
| WebSocket URL      | wss://testnet-unifi-rpc.puffer.fi/        |
| Explorer API (contract verification) | https://testnet-unifi-explorer.puffer.fi/api |
| Faucet             | https://testnet-unifi-faucet.puffer.fi/   |
| Contract Addresses | See [here](./contract-addresses.md)      |

Puffer UniFi Testnet settles to the **Hoodi** L1 testnet (chain ID `560048`).

:::info
Transactions are **preconfirmed** by the gateway before they are sealed into a block, so a receipt is
available in well under a second. This changes what some standard RPC methods return during the
preconfirmation window — see
[Preconfirmation RPC Semantics](./preconfirmation-rpc-semantics.md) before you build against this endpoint.
:::

## Puffer UniFi Testnet on Sepolia

A second Puffer UniFi Testnet settles to the **Sepolia** L1 testnet (chain ID `11155111`).

| Parameter          | Value                                    |
|--------------------|------------------------------------------|
| Network Name       | `Puffer UniFi Testnet`                   |
| Chain ID           | `396776559`                              |
| Currency Symbol    | ETH                                      |
| Block Explorer     | https://testnet-sepolia-unifi-explorer.puffer.fi/ |
| RPC URL            | https://testnet-sepolia-unifi-rpc.puffer.fi/ |
| Explorer API (contract verification) | https://testnet-sepolia-unifi-explorer.puffer.fi/api |
| Bridge             | https://testnet-sepolia-unifi-bridge.puffer.fi/ |
| Contract Addresses | See [here](./contract-addresses.md#sepolia-testnet) |

The public RPC serves HTTP only, with no WebSocket. There is no faucet: use a public Sepolia faucet and
[bridge](../getting-started/bridge-ui.md) the ETH.

## Running your own endpoint

The public endpoint above is a follower node, and you can run the same thing yourself — it serves
preconfirmed state, needs no key and no registration. See
[Deploy a Follower Node](../node-operators/follower-node.md).
