---
title: How to use UniFi Bridge
slug: /developers/getting-started/bridge-ui
---

# Using the UniFi Bridge

In order to use the UniFi Rollup Bridge, you need to have a compatible wallet and some funds on the L1 network.
This guide will help you bridge your funds from the L1 network to the UniFi Rollup network.

## How to bridge (deposit) ETH to UniFi


1. Open the [UniFi Rollup Bridge UI](https://testnet-bridge.unifi.puffer.fi/).
2. Connect your preferred wallet.

   ![Connecting the wallet](/img/rollup/connect-wallet.png)
3. Switch your connected network to L1 (Hoodi).

   - This can be done manually from the top right corner of the page by clicking on your wallet address.

4. Select the token and the amount you want to bridge and press continue.
   ![Token selection](/img/rollup/select-token.png)
5. Review the transaction details and press confirm.
   ![Transaction confirmation](/img/rollup/confirm-transaction.png)
   :::info
   When bridging ERC20 tokens, you will have an additional transaction to approve the bridge contract to spend your tokens.
   :::
6. Accept the transaction in your wallet.
7. Track the transaction in the Transactions tab.
   ![Transaction tab](/img/rollup/transaction-tab.png)
   :::info
   Transaction tab may not be working as expected, however, it does not affect the bridging process. You will see your bridged funds in the L2 network in around just 1 minute.
   :::
8. After a few minutes, the transaction will be confirmed and the status will change to Claimed.
   ![Claimed transaction](/img/rollup/claimed-transaction.png)
9. Once the transaction is confirmed, you can switch to the L2 network (UniFi) and see your bridged funds in your wallet.

## How to bridge (withdraw) ETH from UniFi

1. Make sure that you've selected UniFi Testnet as your source chain
   ![Token Selection L2](/img/rollup/select-token-l2.png)

2. The next steps are the same as when depositing ETH. Once the transaction is confirmed, you can switch to the L1 network (Hoodi) and see your bridged funds in your wallet. The withdrawal will be processed automatically in nearly the same L1 block.
