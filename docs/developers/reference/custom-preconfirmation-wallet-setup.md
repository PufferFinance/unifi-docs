---
title: Custom Wallet Setup for Pre-confirmations
slug: /developers/reference/custom-preconfirmation-wallet-setup
---

# Custom Wallet Setup for Pre-confirmations

This guide explains how to set up a custom wallet that supports pre-confirmations to experience instant transaction confirmations on UniFi Rollup. We recommend using the modified version of Rabby wallet provided in the [based-op repository](https://github.com/gattaca-com/based-op?tab=readme-ov-file#wallets), which supports pre-confirmations. Follow these steps to build and install it:

### 1. Build the Modified Rabby Extension

First, clone the repository and build the extension. Note that you need to have `yarn` installed to build the extension.
```sh
git clone https://github.com/gattaca-com/based-op.git
cd based-op/rabby
yarn && yarn build:pro && yarn build:pro:mv2
```

### 2. Install the Extension

The compiled extension will be available in the following directories:
- **Google Chrome**: `rabby/dist`
- **Mozilla Firefox**: `rabby/dist-mv2`

#### Chrome Installation
1. Open Chrome and navigate to `chrome://extensions/`
2. Enable "Developer mode" in the top right
3. Click "Load unpacked" and select the `rabby/dist` directory

#### Firefox Installation
1. Open Firefox and navigate to `about:debugging`
2. Click "This Firefox"
3. Click "Load Temporary Add-on"
4. Select any file in the `rabby/dist-mv2` directory

For detailed browser-specific instructions, see:
- [Chrome Extension Installation](https://developer.chrome.com/docs/extensions/get-started/tutorial/hello-world#load-unpacked)
- [Firefox Extension Installation](https://extensionworkshop.com/documentation/develop/temporary-installation-in-firefox/)

## Connecting to UniFi Testnet

> [!WARNING]
> You must have the modified Rabby extension installed before proceeding.

1. Open your Rabby extension
2. Import, create a new wallet, or use your existing one
3. Click on "More"
4. Scroll down to the "Settings" section
5. Click on "Add Custom Network"
6. Fill the form with the following values:
   - **Network Name**: `UniFi Testnet`
   - **RPC URL**: `https://testnet-unifi-rpc.puffer.fi/`
   - **Chain ID**: `2092151908`
   - **Symbol**: `ETH`
   - **Block Explorer URL**: `https://testnet-unifi-explorer.puffer.fi/`
7. **Important**: Check the "This network supports preconfirmations" option
8. Click on "Confirm"

## Experiencing Pre-confirmations

Once you have the modified Rabby wallet connected to UniFi Testnet:

1. Open your Rabby extension
2. Click on "Send"
3. Select "UniFi Testnet" from the chain dropdown
4. Fill out the transaction details
5. Click on "Send"
6. Sign the transaction
7. Watch as your transaction receives instant confirmation!

## Why This Works

The modified Rabby wallet includes:
- Reduced polling intervals for transaction receipts
- Support for pre-confirmation detection
- Enhanced UI feedback for instant confirmations

This allows you to experience the true speed of UniFi's pre-confirmation system, where transactions are confirmed in milliseconds rather than waiting for block finalization.