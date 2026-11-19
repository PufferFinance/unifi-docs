---
title: Bridging
slug: /developers/getting-started/bridging
---

# Bridging

To move ETH between Ethereum and Puffer UniFi, use the [Puffer UniFi Bridge](./bridge-ui.md). It handles deposits and
withdrawals for you.

The rest of this page is for developers who deposit and withdraw from their own code.

Puffer UniFi runs the unmodified OP Stack bridge contracts: `L1StandardBridge`, `L2StandardBridge`, the
`CrossDomainMessenger` pair, `OptimismPortal` and `L2ToL1MessagePasser`, with the same ABIs. Deposits
work exactly as on any OP Stack chain. Withdrawals use the same contracts but finalize differently:
games resolved by Puffer UniFi's TEE provers skip the seven-day challenge period, and Puffer finalizes
withdrawals for you.

For the bridge interfaces themselves, the Optimism guides apply as written:
[Standard Bridge](https://docs.optimism.io/app-developers/guides/bridging/standard-bridge) and
[Sending data between L1 and L2](https://docs.optimism.io/app-developers/guides/bridging/messaging).
Replace their OP Mainnet addresses with the ones below, and read [Withdrawals](#withdrawals-l2--l1)
on this page instead of their timing and prove/finalize sections.

:::caution TODO before publishing
- Supported ERC-20 token pairs (placeholder below).
:::

## Network

| | Puffer UniFi Sepolia |
|---|---|
| L1 | Sepolia (chain ID `11155111`) |
| L2 chain ID | `396776559` |
| L2 block time | 2 s |
| L2 RPC | `https://testnet-sepolia-unifi-rpc.puffer.fi` (HTTPS only, no WebSocket) |
| L2 explorer | [testnet-sepolia-unifi-explorer.puffer.fi](https://testnet-sepolia-unifi-explorer.puffer.fi) |
| L1 explorer | [sepolia.etherscan.io](https://sepolia.etherscan.io) |

### L1 contracts (Sepolia)

| Contract | Address |
|---|---|
| L1StandardBridge | `0x29ada3191032ac82aa69f73fa40eb1d6991a6699` |
| L1CrossDomainMessenger | `0x222ab02e8c86c170faa44803298662e63251272b` |
| OptimismPortal | `0x31d6c448e1f602ae02c30260e090f999e27d643b` |
| DisputeGameFactory | `0xd47c03e49432faee2a80ad5204f9aa777c89aa08` |
| AnchorStateRegistry | `0xdf841d3287631a226040b394c0c4483e9a6df791` |
| OptimismMintableERC20Factory | `0x70082ba1fc1ade4a7024b9ec3cd0d800eeef5fea` |
| SystemConfig | `0xd52f98e739695963f2e324a15e79d893a2c0a016` |
| L1ERC721Bridge | `0xa3b60226a704d833ce0fd0248d970791767e8e2e` |

### L2 contracts

The standard OP Stack predeploys:

| Contract | Address |
|---|---|
| L2StandardBridge | `0x4200000000000000000000000000000000000010` |
| L2CrossDomainMessenger | `0x4200000000000000000000000000000000000007` |
| L2ToL1MessagePasser | `0x4200000000000000000000000000000000000016` |
| OptimismMintableERC20Factory | `0x4200000000000000000000000000000000000012` |

### Tokens

ETH is bridged natively. Puffer UniFi is not on the Superchain Token List, so the Optimism
[verify bridged tokens](https://docs.optimism.io/app-developers/guides/bridging/verify-bridged-tokens)
guide does not apply. Use the token pairs below, and check that an L2 token's `remoteToken()`
returns the L1 token before you bridge to it.

| Token | L1 address | L2 address |
|---|---|---|
| `TODO` | | |

## Deposits (L1 → L2)

Call the L1 contracts exactly as on any OP Stack chain:

- **ETH:** `L1StandardBridge.bridgeETHTo(to, minGasLimit, extraData)` with `msg.value` set to the amount.
  `bridgeETH` sends to yourself, but only from an EOA.
- **ERC-20:** `approve` the `L1StandardBridge` for the amount, then
  `L1StandardBridge.bridgeERC20To(l1Token, l2Token, to, amount, minGasLimit, extraData)`.
- **Arbitrary messages or calls:** `L1CrossDomainMessenger.sendMessage(target, message, minGasLimit)`,
  or `OptimismPortal.depositTransaction` for a raw deposit.

Example with Foundry's `cast`, depositing 0.1 ETH on Puffer UniFi Sepolia:

```bash
cast send 0x29ada3191032ac82aa69f73fa40eb1d6991a6699 \
  "bridgeETHTo(address,uint32,bytes)" $RECIPIENT 200000 0x \
  --value 0.1ether --rpc-url $SEPOLIA_RPC --private-key $KEY
```

A deposit appears on L2 once the sequencer's L1 origin reaches the L1 block that holds it. That takes
about one minute. The deposit's L2 transaction
hash can be computed from the `TransactionDeposited` event, as on any OP Stack chain.

## Withdrawals (L2 → L1)

### 1. Start the withdrawal on L2

Use the same calls as on any OP Stack chain:

- **ETH:** `L2StandardBridge.bridgeETHTo(to, minGasLimit, extraData)` with `msg.value`.
- **ERC-20:** `L2StandardBridge.bridgeERC20To(l2Token, l1Token, to, amount, minGasLimit, extraData)`.
  No approval is needed: the L2 token is burned.
- **Arbitrary messages or calls:** `L2CrossDomainMessenger.sendMessage(target, message, minGasLimit)`.

Keep the transaction hash. The `MessagePassed` event from `L2ToL1MessagePasser` in its receipt carries
the withdrawal hash and every field you need to prove and finalize it.

### 2. Puffer finalizes it on L1

You normally do nothing more. A Puffer relayer proves and finalizes each withdrawal on L1 in a single
transaction, once a TEE-proven dispute game covers the withdrawal's L2 block. A pending withdrawal
makes the sequencer post its batch and create a new game right away, so withdrawals typically
finalize about one and a half to two minutes after the L2 transaction.

To check the status, read `OptimismPortal.finalizedWithdrawals(withdrawalHash)` on L1. It returns `true`
once the withdrawal is finalized. The portal emits `WithdrawalFinalized(withdrawalHash, success)`.

### 3. Finalize it yourself (optional)

The relayer is best-effort, and both portal calls are permissionless. If a withdrawal is not finalized
in a reasonable time, an hour say, you can finalize it yourself with the same two calls as on OP Stack:

1. **Pick a TEE-resolved game.** Use the newest game from `DisputeGameFactory` of the respected type
   (`AnchorStateRegistry.respectedGameType()`, type 1) whose `l2BlockNumber()` is at or above your
   withdrawal's L2 block, and for which `AnchorStateRegistry.checkIsResolvedByTEE(game)` and
   `isGameClaimValid(game)` both return `true`. `DisputeGameFactory.frontierBlockNumber()` is the
   highest L2 block a TEE-resolved game covers.
2. **Prove.** Call `OptimismPortal.proveWithdrawalTransaction(withdrawalTx, disputeGameIndex,
   outputRootProof, withdrawalProof)` against that game. Build the output root proof and the storage
   proof from an L2 node at the game's L2 block, as on any OP Stack chain. The prove transaction must
   land in a later L1 block than the one that created the game.
3. **Finalize.** Call `OptimismPortal.finalizeWithdrawalTransaction(withdrawalTx)`. For a TEE-resolved
   game there is no proof maturity delay, so you can finalize right after proving.

If the game you proved against was **not** TEE-resolved, the standard OP Stack delays apply: seven days
after proving, and the game must also pass its dispute period. Always check
`checkIsResolvedByTEE` before proving.

:::warning Off-the-shelf OP Stack tooling reports the wrong timing
The portal still stores the stock OP Stack values (`proofMaturityDelaySeconds` = seven days,
`disputeGameFinalityDelaySeconds` = 3.5 days); they are skipped for TEE-resolved games only. Helpers
that read them, such as viem's `getTimeToFinalize` and `getWithdrawalStatus`, will report a
seven-day wait for a withdrawal that can be finalized now. Use `finalizedWithdrawals` for status, and
pick the game yourself as described above. Building the proof with viem's `buildProveWithdrawal`
works as on OP Stack, given the right game.
:::

### Failed payouts

A withdrawal is finalized even when its L1 call fails: the portal emits
`WithdrawalFinalized(withdrawalHash, false)`. For a bridge withdrawal or a messenger message, the
message is then recorded in `L1CrossDomainMessenger.failedMessages` and can be retried with
`L1CrossDomainMessenger.relayMessage`, as on OP Stack.

Withdraw to an EOA, or to a contract whose receive path needs a bounded, known amount of gas, and set
`minGasLimit` to cover it. A target that consumes all the gas forwarded to it can leave the message
unrecorded and so impossible to replay.
