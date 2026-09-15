---
title: Preconfirmation RPC Semantics
slug: /developers/reference/preconfirmation-rpc-semantics
---

# Preconfirmation RPC Semantics

**Who this is for:** anyone integrating against a UniFi RPC endpoint.

On UniFi, transactions are **preconfirmed** by the gateway before they are sealed into a block. A
receipt appears within tens of milliseconds; the block seals up to ~1 second later. No method name
or signature changes, which makes the differences from vanilla Ethereum easy to miss.

If you read one thing, read [The two clocks](#the-two-clocks) and
[Logs are not preconfirmed](#known-limitation-logs-are-not-preconfirmed).

## The short version

- **State reads at `latest` reflect preconfirmed transactions.** If you have a receipt, an
  `eth_call` at `latest` sees that transaction.
- **Block reads do not.** `eth_blockNumber` and `eth_getBlockByNumber` describe sealed blocks at
  every tag.
- **A preconfirmed receipt has a zero `blockHash`.** That is not an error — see
  [Telling preconfirmed from sealed](#telling-preconfirmed-from-sealed).
- **Logs are the exception you need to know about.** They are on the receipt, but `eth_getLogs`,
  filters and `logs` subscriptions are sealed-only.

## The two clocks

There are two views of "now", and each RPC method uses one of them.

|                    | What it is                                                            | Used by                                       |
|--------------------|-----------------------------------------------------------------------|-----------------------------------------------|
| **Sealed head**    | the last block that completed consensus                               | every *block-shaped* read                     |
| **Unsealed block** | the block currently being built from preconfirmation fragments        | *state-shaped* reads at `latest` and `pending`|

The unsealed block is one ahead of the sealed head, so **`latest` state can be one block ahead of the
`latest` block.** That is deliberate: keeping the block surface sealed means the ubiquitous
`blockNumber → getBlock(n)` polling pattern keeps working.

**Practical consequence:** do not pair a `latest`/`pending` state read with a block read and expect
them to describe the same block.

## The guarantee

> If `eth_getTransactionReceipt(h)` returns non-null on a node, then on that same node `eth_call`,
> `eth_getBalance`, `eth_getStorageAt`, `eth_getCode` and `eth_getTransactionCount` at tag `latest`
> (and `pending`) reflect that transaction.

This matters because most client libraries default every contract read to `latest`.

Two caveats:

- **Per node.** The guarantee is per-endpoint. Behind a load balancer you can observe a receipt from
  one node and stale state from another. Use a single or session-affine endpoint if you depend on
  read-your-writes.
- **Volatile before seal.** If the unsealed block is discarded, `latest` state steps back to the
  sealed view until the next fragment — the same volatility a preconfirmed receipt has.

## Method reference

`N` = sealed head; the unsealed block is `N+1`.

### State-shaped — preconfirmed at `latest` and `pending`

| Method | `latest` / `pending` | `safe` / `finalized` / number / hash |
|---|---|---|
| `eth_call` | unsealed | sealed |
| `eth_estimateGas` | unsealed | sealed |
| `eth_getBalance` | unsealed | sealed |
| `eth_getCode` | unsealed | sealed |
| `eth_getStorageAt` | unsealed | sealed |
| `eth_getTransactionCount` | unsealed (`pending` = max of pool and unsealed) | sealed |
| `eth_createAccessList` | unsealed | sealed |
| `eth_simulateV1` | unsealed base | sealed |

### Block-shaped — sealed at every tag

| Method | Behaviour |
|---|---|
| `eth_blockNumber` | sealed head `N` |
| `eth_getBlockByNumber` | block `N`, including at `pending` |
| `eth_getBlockTransactionCountByNumber` | from `N` |
| `eth_getTransactionByBlockNumberAndIndex` | from `N` |
| `eth_feeHistory` | up to `N` |

### Receipt- and hash-shaped — preconfirmation-aware

| Method | Behaviour |
|---|---|
| `eth_getTransactionReceipt` | **preconfirmed** — returns as soon as the transaction is in a fragment |
| `eth_getTransactionByHash` | **preconfirmed** |
| `eth_getRawTransactionByHash` | **preconfirmed** |
| `eth_getBlockReceipts("pending")` | **preconfirmed** — the unsealed block's receipts |
| `eth_getBlockReceipts(<number \| hash \| latest>)` | sealed |
| `eth_getProof` | **sealed only**, at every tag |
| any method given a **block hash** | sealed only |

`eth_getProof` is sealed at every tag by design: the unsealed block has no computed state root, so a
proof against it would be meaningless. Proofs lag state by up to one block.

## Telling preconfirmed from sealed

A preconfirmed receipt or transaction carries:

- `blockHash` = `0x0000…0000` (32 zero bytes)
- `blockNumber` = the unsealed block's number, populated and real
- `transactionIndex` = populated and real

**A zero `blockHash` with a non-null `blockNumber` means "preconfirmed, not yet sealed".** Once the
block seals, the same transaction reports its real block hash.

:::warning
Do not treat the zero hash as an error, and do not try to fetch a block by it.
:::

## Known limitation: logs are not preconfirmed

**Logs are available on the receipt. The log *query and subscription* APIs are sealed-only.**

| API | Preconfirmed? |
|---|---|
| `logs` field of `eth_getTransactionReceipt` | **yes** |
| `eth_getLogs` | no — sealed blocks only |
| `eth_newFilter` / `eth_getFilterChanges` / `eth_getFilterLogs` | no |
| `logs` subscriptions (`eth_subscribe`) | no |

A transaction whose receipt you already hold will appear to have **emitted nothing** through those
APIs for up to one block.

**If you react to events:**

- **read the logs off the receipt** — they are there, with correct `blockNumber` and
  `transactionIndex`, and a zero `blockHash` per the convention above; or
- accept a delay of up to one block if you use `eth_getLogs` or a `logs` subscription.

:::danger
Do **not** infer from an empty `eth_getLogs` that a receipted transaction emitted no events. During
the preconfirmation window that inference is wrong, and it is the most likely way this limitation
causes a real bug.
:::

## See also

- [Network & RPC Endpoints](./rpc-endpoints.md)
- [Custom Wallet Setup for Pre-confirmations](./custom-preconfirmation-wallet-setup.md)
