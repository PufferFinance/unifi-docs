---
title: How to use the Puffer UniFi Bridge
slug: /developers/getting-started/bridge-ui
---

# Using the Puffer UniFi Bridge

The [Puffer UniFi Testnet Bridge](https://testnet-unifi-bridge.puffer.fi/) moves ETH between the Hoodi testnet and Puffer UniFi Testnet.
You need an EOA wallet, such as MetaMask or Rabby, and some ETH on the chain you are bridging from.
Contract wallets, such as Safe, can't deposit through the bridge app.
To get testnet ETH, see [Acquire Testnet Tokens](./acquire-testnet-tokens.md).

| Network                | Chain ID     | Bridge role |
| ---------------------- | ------------ | ----------- |
| Ethereum Hoodi Testnet | `560048`     | L1          |
| Puffer UniFi Testnet   | `2092151908` | L2          |

## How to bridge (deposit) ETH to Puffer UniFi

1. Open the [Puffer UniFi Testnet Bridge](https://testnet-unifi-bridge.puffer.fi/).
2. Click **Connect wallet** and choose your wallet.

   ![Connecting the wallet](/img/bridge/connect-wallet.png)

   Once connected, your balance and address appear in the top right corner.

   ![Connected wallet](/img/bridge/wallet-connected.png)
3. Make sure **Ethereum Hoodi Testnet** is the source chain and **Puffer UniFi Testnet** is the destination chain.
   If your wallet is on Hoodi, the bridge fills these in for you. You can also pick a chain from the source chain list.

   ![Chain selection](/img/bridge/select-chain.png)
4. Select **ETH**, enter the amount you want to bridge, and press **Continue**.
   The estimated fee is shown under the amount.

   ![Token and amount selection](/img/bridge/select-token.png)
5. Review the transfer details and the recipient address, then press **Continue**.
   By default, the recipient is your own address. Use **Edit** to change it.

   ![Transaction review](/img/bridge/review-transaction.png)
6. Press **Bridge** and accept the transaction in your wallet.

   ![Transaction confirmation](/img/bridge/confirm-transaction.png)
   :::info
   When bridging ERC20 tokens, you will have an additional transaction to approve the bridge contract to spend your tokens.
   :::
7. Track the transaction in the **Transactions** tab.
   The status shows **Processing** while the transfer is in progress and **Succeed** once it is complete.
   **Failed** means the transfer did not go through.

   ![Transactions tab](/img/bridge/transaction-tab.png)
8. Once the status is **Succeed**, switch your wallet to Puffer UniFi Testnet to see your bridged funds.

## How to bridge (withdraw) ETH from Puffer UniFi

1. Select **Puffer UniFi Testnet** as the source chain and **Ethereum Hoodi Testnet** as the destination chain.
   You can press the swap button (⇅) between the two chains to reverse the direction.

   ![Withdrawal chain selection](/img/bridge/select-token-l2.png)

2. The next steps are the same as for a deposit. Once the status is **Succeed**, switch your wallet to Hoodi to see your funds.

   :::info
   Withdrawals are processed automatically. There is no separate prove or claim step, and no seven-day
   challenge period. Expect the funds to arrive on Hoodi after a few minutes.
   :::
