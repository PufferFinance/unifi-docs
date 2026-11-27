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
games resolved by Puffer UniFi's TEE provers skip the seven-day challenge period, and withdrawals
submitted through the Puffer withdrawal service are proven and finalized for you, usually in the same
L1 transaction that posts the batch holding them.

For the bridge interfaces themselves, the Optimism guides apply as written:
[Standard Bridge](https://docs.optimism.io/app-developers/guides/bridging/standard-bridge) and
[Sending data between L1 and L2](https://docs.optimism.io/app-developers/guides/bridging/messaging).
Replace their OP Mainnet addresses with the ones below, and read [Withdrawals](#withdrawals-l2--l1)
on this page instead of their timing and prove/finalize sections.

:::caution TODO before publishing
- Withdrawal service URL (placeholder below): the service is not reachable from outside the operator's network yet.
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
| ETH | native | native |

No ERC-20 pairs are published for the Sepolia testnet yet; the bridge indexes ETH only.

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

A withdrawal starts on L2 and is then proven and finalized on L1. There are two ways to do it:

- **Puffer relays it (recommended).** Start the withdrawal through Puffer's `WithdrawalHelper`
  contract on L2 and register it with the Puffer withdrawal service. Puffer proves and finalizes it on
  L1 for a flat fee, normally inside the L1 transaction that posts the batch holding your withdrawal,
  about one and a half to two minutes after the L2 transaction.
- **You finalize it yourself.** Use the standard OP Stack calls on L2, then prove and finalize the
  withdrawal on L1 yourself. Puffer does **not** relay these withdrawals.

Both paths use the same bridge contracts and pay out to the same recipient. Only the first is relayed
for you.

### Withdraw through the Puffer withdrawal service

| | Puffer UniFi Sepolia |
|---|---|
| Withdrawal service (JSON-RPC, `sc` namespace) | `TODO` |
| WithdrawalHelper (L2) | `0x23779ea36d29b7b40f2db06750aa8678779ab074` |
| WithdrawalRelayer (L1) | `0xc4a073220587dca0abfb11ce995ed2437d5b4bd9` |

The service is a JSON-RPC endpoint. A refusal is a JSON-RPC error whose `data` field is a short code
(listed under each step); the `message` says what to change.

#### 1. Read the fee table

```bash
curl -s $SERVICE -H 'content-type: application/json' \
  -d '{"jsonrpc":"2.0","id":1,"method":"sc_fees","params":[]}'
```

```json
{
  "helper": "0x…",
  "eth_fee_wei": "0x2386f26fc10000",
  "token_fees": { "0x<l2 token>": "0x…" },
  "max_min_gas": 200000,
  "max_data_bytes": 131072
}
```

- `eth_fee_wei` is the fee when you pay in ETH. `token_fees` lists the L2 tokens you may pay the fee
  in instead, with the fee in that token's units; a token that is not listed can still be withdrawn,
  but its fee must be paid in ETH.
- The fee is flat per request, whatever the amount. Pass the value from the table as `fee` in the
  helper call: a lower fee is refused with `fee_too_low`, a higher one is accepted and not refunded.
  After the table changes, requests built on the previous table are still accepted for a grace period.
- `max_min_gas` is the highest `minGasLimit` the service relays. `max_data_bytes` bounds the
  withdrawal's L1 message; a plain bridge withdrawal is always well inside it.

#### 2. Build the withdrawal transaction

Call the `WithdrawalHelper` from the account that owns the funds. It pays the fee to Puffer's fee
collector, then starts exactly one `L2StandardBridge.withdrawTo` for you.

- **ETH:** `withdrawETH(to, amount, fee, minGasLimit, requestTag)` with `msg.value` exactly
  `amount + fee`.
- **ERC-20, fee in ETH:** `approve` the helper for `amount`, then
  `withdrawERC20(l2Token, to, amount, fee, false, minGasLimit, requestTag)` with `msg.value`
  exactly `fee`.
- **ERC-20, fee in the token:** `approve` the helper for `amount + fee`, then
  `withdrawERC20(l2Token, to, amount, fee, true, minGasLimit, requestTag)` with `msg.value` zero.
  Only tokens listed in `token_fees` can pay the fee this way. Fee-on-transfer and rebasing tokens
  are rejected by the helper.

`minGasLimit` is the gas forwarded to the L1 payout, as on OP Stack; it must not exceed
`max_min_gas`. `requestTag` is any `bytes32` for your own bookkeeping; the helper emits it in its
`FeePaid` event and the service stores it with the request. The helper only supports tokens bridged by
the standard bridge.

Example with Foundry's `cast`, withdrawing 0.1 ETH with a 0.01 ETH fee, signed but not sent:

```bash
cast mktx $HELPER \
  "withdrawETH(address,uint256,uint256,uint32,bytes32)" \
  $RECIPIENT 100000000000000000 10000000000000000 200000 0x$(openssl rand -hex 32) \
  --value 110000000000000000 --rpc-url $UNIFI_RPC --private-key $KEY
```

#### 3. Register it with the service

Either hand the service the signed transaction, or send it yourself and hand the service its hash.

- **Signed transaction** (backends, scripts): do not broadcast it yourself.

  ```json
  {"jsonrpc":"2.0","id":1,"method":"sc_submitRequest",
   "params":[{"flow":"L2_to_L1","signed_tx":"0x02f8…"}]}
  ```

  The service checks the transaction and broadcasts it on L2. If the L2 node refuses it for good
  (wrong nonce, insufficient funds, underpriced), the request is closed as `l2_rejected` and you
  submit a corrected transaction.

- **Transaction hash** (browser wallets, which sign and send in one step): send the helper call
  through the wallet, then register its hash.

  ```json
  {"jsonrpc":"2.0","id":1,"method":"sc_submitRequest",
   "params":[{"flow":"L2_to_L1","l2_tx_hash":"0x…"}]}
  ```

  The service reads the transaction from the L2 node and applies the same checks. Until the node has
  it the answer is `tx_unknown`: retry after a second. A transaction that already mined is accepted
  from its receipt; one that reverted is refused with `tx_reverted`. Registering a hash costs nothing,
  and anyone may do it, because the fee was paid on L2 by the helper call.

Give exactly one of the two fields. The answer is the request, with an `id` to track it by;
submitting the same transaction again, in either form, returns the same request.

Refusal codes: `bad_params` (neither or both fields), `bad_tx` (not a well-formed signed
transaction), `wrong_chain`, `not_helper` (not a call to the helper's two functions), `bad_call`
(a helper call that does not start exactly one withdrawal), `min_gas_too_high`, `fee_too_low`,
`fee_token_unsupported`, `preflight_reverted` (the call would revert on L2: wrong `msg.value`,
missing approval, insufficient balance), `sender_pending_cap` (too many open requests from one
sender; wait for them to finalize), `tx_unknown`, `tx_reverted`.

#### 4. Track it

```json
{"jsonrpc":"2.0","id":1,"method":"sc_getRequest","params":["<id>"]}
{"jsonrpc":"2.0","id":1,"method":"sc_getRequestByTx","params":["0x<l2 tx hash>"]}
```

The request carries a few independent fields; read them together:

| Field | Values | Meaning |
|---|---|---|
| `l2_tx` | `pending`, `submitted`, `confirmed`, `reverted`, `replaced`, `l2_rejected` | the helper transaction on L2 |
| `withdrawal_state` | `awaiting_coupled`, `async_pending`, `async_sent`, `finalized`, `stalled` | the relay on L1 |
| `finalized` | `{by, success, l1_tx}` | set once the portal shows the withdrawal finalized; `by` is `coupled` (in the batch transaction), `async` (a later relay transaction) or `external` (someone else) |
| `delivery` | `unknown`, `delivered`, `replay_required`, `not_relayed` | whether the L1 payout itself succeeded |
| `status` | `pending`, `processing`, `finalized`, `failed` | the summary |

The normal sequence is `confirmed` → `awaiting_coupled` → `finalized{by: coupled}` with
`delivery: delivered`, one to two minutes after the L2 transaction. When the batch transaction could
not carry the withdrawal, the request becomes `async_pending` and a relay transaction finalizes it a
few minutes later (`by: async`). On L1 you can also read
`OptimismPortal.finalizedWithdrawals(withdrawalHash)`; the withdrawal hash is in the request's
`withdrawal` field and in the `MessagePassed` event of your L2 receipt.

#### 5. If it is not delivered

The relay is best-effort. Every state below is recoverable by you, and nothing in the service holds
your funds:

- **`l2_tx` is `reverted`, `replaced` or `l2_rejected`.** Nothing happened on L1 and, for a reverted
  or rejected transaction, no fee was paid. Fix the cause and submit a new transaction. (`replaced`
  means another transaction from your account took the nonce.)
- **`withdrawal_state` is `stalled`, or stays `async_pending` for longer than an hour.** Puffer's relay
  gave up or is lagging. Prove and finalize the withdrawal yourself with the steps under
  [Withdraw with the standard OP Stack calls](#withdraw-with-the-standard-op-stack-calls); the helper
  started an ordinary bridge withdrawal, so they apply unchanged. If Puffer's prove already landed
  (`OptimismPortal.provenWithdrawals(withdrawalHash, <WithdrawalRelayer>)` is set), skip the prove and
  call `OptimismPortal.finalizeWithdrawalTransactionExternalProof(withdrawalTx, <WithdrawalRelayer>)`,
  which is open to anyone. The service notices and marks the request `finalized{by: external}`.
- **`finalized` with `success: false`, or `delivery: replay_required`.** The withdrawal is finalized
  but the L1 payout failed (the recipient reverted or `minGasLimit` was too low for it). Replay it
  with `L1CrossDomainMessenger.relayMessage`, as in [Failed payouts](#failed-payouts).

The fee is charged by the helper on L2 when the transaction mines and is not refunded.

### Withdraw with the standard OP Stack calls

Start the withdrawal on L2 exactly as on any OP Stack chain:

- **ETH:** `L2StandardBridge.bridgeETHTo(to, minGasLimit, extraData)` with `msg.value`.
- **ERC-20:** `L2StandardBridge.bridgeERC20To(l2Token, l1Token, to, amount, minGasLimit, extraData)`.
  No approval is needed: the L2 token is burned.
- **Arbitrary messages or calls:** `L2CrossDomainMessenger.sendMessage(target, message, minGasLimit)`.

Keep the transaction hash. The `MessagePassed` event from `L2ToL1MessagePasser` in its receipt carries
the withdrawal hash and every field you need to prove and finalize it.

Puffer does not relay these withdrawals. Prove and finalize them on L1 yourself, with the same two
portal calls as on OP Stack:

1. **Pick a TEE-resolved game.** Use the newest game from `DisputeGameFactory` of the respected type
   (`AnchorStateRegistry.respectedGameType()`, type 1) whose `l2BlockNumber()` is at or above your
   withdrawal's L2 block, and for which `AnchorStateRegistry.checkIsResolvedByTEE(game)` and
   `isGameClaimValid(game)` both return `true`. `DisputeGameFactory.frontierBlockNumber()` is the
   highest L2 block a TEE-resolved game covers; a pending requested withdrawal makes the sequencer
   create a game right away, and otherwise a new game appears at least every 30 minutes.
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
