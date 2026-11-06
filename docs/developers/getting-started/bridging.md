---
title: Bridging
slug: /developers/getting-started/bridging
---

# Bridging

To move ETH between Ethereum and UniFi, use the UniFi bridge (link: `TODO`). It handles deposits and
withdrawals for you.

The rest of this page is for developers who deposit and withdraw from their own code.

UniFi runs the unmodified OP Stack bridge contracts: `L1StandardBridge`, `L2StandardBridge`, the
`CrossDomainMessenger` pair, `OptimismPortal` and `L2ToL1MessagePasser`, with the same ABIs. Deposits
work exactly as on any OP Stack chain. Withdrawals use the same contracts but finalize differently:
games resolved by UniFi's TEE provers skip the seven-day challenge period, and Puffer finalizes
withdrawals submitted through its withdrawal service.

For the bridge interfaces themselves, the Optimism guides apply as written:
[Standard Bridge](https://docs.optimism.io/app-developers/guides/bridging/standard-bridge) and
[Sending data between L1 and L2](https://docs.optimism.io/app-developers/guides/bridging/messaging).
Replace their OP Mainnet addresses with the ones below, and read [Withdrawals](#withdrawals-l2--l1)
on this page instead of their timing and prove/finalize sections.

:::caution TODO before publishing
- Bridge UI link, public L2 RPC and explorer URLs (placeholders).
- Withdrawal service URL and `WithdrawalHelper` address, once the service is live on UniFi Sepolia.
- Supported ERC-20 token pairs (placeholder below).
:::

## Network

| | UniFi Sepolia |
|---|---|
| L1 | Sepolia (chain ID `11155111`) |
| L2 chain ID | `396776558` |
| L2 block time | 2 s |
| L2 RPC | `TODO` |
| L2 explorer | `TODO` |

### L1 contracts (Sepolia)

| Contract | Address |
|---|---|
| L1StandardBridge | `0x4f2e667d13b6969558547d8829c8a90f3148492d` |
| L1CrossDomainMessenger | `0xc42591636f0d84406a39c023c5bcd42be8b148e3` |
| OptimismPortal | `0x8ddfabcac49206e8488a22f4f3803f19a4e572e0` |
| DisputeGameFactory | `0xeb8461c8d057f961aeae5c0b26e7361713a93f65` |
| AnchorStateRegistry | `0xe80154907202b137ebc1e5058a4270526ee98a24` |
| OptimismMintableERC20Factory | `0xc9902f90249e25f12f67beec0a8f2fce8693826d` |
| L1ERC721Bridge | `0x18a63571d0a86fd6dc7c53c8fffc737fab5e9288` |

### L2 contracts

The standard OP Stack predeploys:

| Contract | Address |
|---|---|
| L2StandardBridge | `0x4200000000000000000000000000000000000010` |
| L2CrossDomainMessenger | `0x4200000000000000000000000000000000000007` |
| L2ToL1MessagePasser | `0x4200000000000000000000000000000000000016` |
| OptimismMintableERC20Factory | `0x4200000000000000000000000000000000000012` |

### Tokens

ETH is bridged natively. UniFi is not on the Superchain Token List, so the Optimism
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

Example with Foundry's `cast`, depositing 0.1 ETH on UniFi Sepolia:

```bash
cast send 0x4f2e667d13b6969558547d8829c8a90f3148492d \
  "bridgeETHTo(address,uint32,bytes)" $RECIPIENT 200000 0x \
  --value 0.1ether --rpc-url $SEPOLIA_RPC --private-key $KEY
```

A deposit appears on L2 once the sequencer's L1 origin reaches the L1 block that holds it. That takes
about one minute. The deposit's L2 transaction
hash can be computed from the `TransactionDeposited` event, as on any OP Stack chain.

## Withdrawals (L2 → L1)

A withdrawal starts on L2 and is then proven and finalized on L1. There are two ways to do it:

- **Puffer relays it (recommended).** Start the withdrawal through Puffer's `WithdrawalHelper`
  contract and submit the signed transaction to the Puffer withdrawal service. Puffer proves and
  finalizes it on L1 for a flat fee, usually within a few minutes.
- **You finalize it yourself.** Use the standard OP Stack calls on L2, then prove and finalize the
  withdrawal on L1 yourself. Puffer does **not** relay these withdrawals.

Both paths use the same L1 contracts and pay out to the same recipient. Only the first is relayed
for you.

### Withdraw through the Puffer withdrawal service

| | UniFi Sepolia |
|---|---|
| Withdrawal service URL | `TODO` |
| WithdrawalHelper (L2) | `TODO` |

1. **Read the current fee.** `GET /fees` returns the helper address, the ETH fee in wei
   (`eth_fee_wei`), the fees accepted in tokens (`token_fees`), and the highest `minGasLimit`
   accepted (`max_min_gas`).
2. **Sign the withdrawal transaction.** Build an L2 transaction that calls the helper, sign it, and
   do not broadcast it yourself:
   - **ETH:** `withdrawETH(to, amount, fee, minGasLimit, requestTag)` with `msg.value` equal to
     `amount + fee`.
   - **ERC-20:** first `approve` the helper for `amount` (plus the fee, if paying in the token), then
     `withdrawERC20(l2Token, to, amount, fee, feeInToken, minGasLimit, requestTag)`. With
     `feeInToken` false, `msg.value` is the ETH fee; with it true, `msg.value` is zero.

   `requestTag` is any `bytes32` you choose, for your own bookkeeping; it is emitted in the helper's
   `FeePaid` event. `minGasLimit` must not exceed `max_min_gas`.
3. **Submit it.** `POST /requests` with the body
   `{"flow": "L2_to_L1", "signed_tx": "0x…"}`. The service checks the transaction, broadcasts it
   on L2, and answers with a `request_id`. Submitting the same transaction again returns the same
   request.
4. **Track it.** `GET /requests/{request_id}`, or `GET /requests?l2_tx_hash=0x…`, returns its
   `status`: `pending`, `processing`, `finalized` or `failed`. A finalized request also carries the
   `withdrawal_hash` and the L1 transaction that finalized it.

On L1 you can also read `OptimismPortal.finalizedWithdrawals(withdrawalHash)`, which returns `true`
once the withdrawal is finalized.

### Withdraw with the standard OP Stack calls

Start the withdrawal on L2 exactly as on any OP Stack chain:

- **ETH:** `L2StandardBridge.bridgeETHTo(to, minGasLimit, extraData)` with `msg.value`.
- **ERC-20:** `L2StandardBridge.bridgeERC20To(l2Token, l1Token, to, amount, minGasLimit, extraData)`.
  No approval is needed: the L2 token is burned.
- **Arbitrary messages or calls:** `L2CrossDomainMessenger.sendMessage(target, message, minGasLimit)`.

Keep the transaction hash. The `MessagePassed` event from `L2ToL1MessagePasser` in its receipt carries
the withdrawal hash and every field you need to prove and finalize it.

Puffer does not relay these withdrawals. Prove and finalize them on L1 yourself:

1. **Pick a TEE-resolved game.** Use the newest game from `DisputeGameFactory` of the respected type
   (`AnchorStateRegistry.respectedGameType()`, type 1) whose `l2BlockNumber()` is at or above your
   withdrawal's L2 block, and for which `AnchorStateRegistry.checkIsResolvedByTEE(game)` and
   `isGameClaimValid(game)` both return `true`. `DisputeGameFactory.frontierBlockNumber()` is the
   highest L2 block a TEE-resolved game covers; a new game is created at least every 30 minutes.
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
