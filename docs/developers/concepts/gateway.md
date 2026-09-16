---
title: Gateway
slug: /developers/concepts/gateway
---

## Overview
Gateways are a critical component in based rollups. Their primary purpose is to act as the sequencing entities of the rollup, enabling L1 proposers to provide sequencing services even if they lack the sophistication or infrastructure to do so directly. Outsourcing sequencing to sophisticated third parties keeps the proposer set decentralized and neutral, while providing the necessary performance and reliability required by modern high-throughput rollups.

Because a gateway holds a temporary exclusive right to modify the L2 state, it can provide fast execution preconfirmations to users, _before_ the batch is fully settled on the L1.

:::info
This page describes the **target** gateway design: lookahead assignment, collateral and slashing.
For how gateways are selected and how block production actually works on UniFi today, see
[The based-op Sequencing Stack](based-op-stack.md).
:::

## How UniFi runs gateways today

The mechanics currently in production are covered in detail on
[The based-op Sequencing Stack](based-op-stack.md). In short:

- Gateways are assigned by a **registry** that maps block heights to gateways on a fixed rotation,
  not yet by an L1 proposer lookahead.
- Gateways build blocks as a stream of **fragments**, which is what makes preconfirmations possible.
- A **portal** validates the gateway's block against a fallback execution client before it is used,
  and produces a block itself if the gateway is late or invalid.
- Preconfirmations are **soft**: there is no collateral or slashing behind them yet.

The rest of this page is where that is heading.

## Target design
![Gateway Lookahead](/img/gateway/gateway.png)

L1 proposers establish a schedule of gateways called **lookahead**, which maps L1 slots in a given epoch to gateways. Each gateway holds the write access on the L2 state for a number of L1 slots, and must post these blocks as batches on the L1 via the rollup inbox contract.

### Block production and sync
Subsequent gateways in the lookahead schedule (e.g. Gateway B in the diagram) must obtain the latest L2 state to issue preconfirmations after the slot transition. Other applications, such as explorers, wallets, and indexers, may also need access to the latest L2 state before it is settled on the L1. Relying solely on the L1 state (i.e. waiting for the L2 to sync after the batch has been posted) would result in "gaps" during which no preconfirmations can be issued. This is because Gateway A needs to stop issuing preconfs X seconds before the start of the next L1 slot to ensure that the L2 batch gets settled on the L1, and Gateway B will only receive this updated state Y seconds after the start of the next L1 slot. To solve this issue, gateways share "soft" blocks (i.e. temporary, also known as "unsafe" blocks in the OP stack) to other nodes, before they are settled on the L1.

Note that because these blocks are still not fully settled, there is still a possibility that a different batch will be posted on the L1, thus triggering a reorg on the L2.

This is the problem UniFi's fragment gossip already solves in production — an incoming gateway has been applying its predecessor's fragments all along, so it starts its slot with current state.

### Settlement and slashing
Gateways provide collateral to be able to enter the lookahead. The collateral is subject to slashing conditions e.g. when preconfirmation promises are reneged, but also to incentivise timely settlement of batches on the L1.

For example, if Gateway A fails to settle the required batch on the L1 by end of its slots, the next gateway in the lookahead schedule (Gateway B), will have to re-create and settle the L2 batch transaction(s). By doing so, Gateway B:

- can claim part of the collateral of the previous gateway (i.e. the faulty gateway is slashed). The slash amount should be proportional to the number of preconf blocks which haven't landed
- avoids a L2 reorg and ensures that the L2 advances to where it started issuing preconfs on

In practice, when processing the L2 batch transaction, the inbox contract verifies if the signed blocks are signed by the previous gateway, and if so will slash its collateral and transfer it to the gateway posting the batch instead.

If, following a slashing event, the collateral amount falls below a specified threshold, the gateway will become ineligible for future lookaheads until the collateral is replenished.

### Enforcement
From a user perspective, in the "happy" case, a preconfirmation is issued and the corresponding transaction is eventually included in an L2 batch that settles on the L1. However, if the corresponding transaction is not included and the chain advances without it, the user can use the signed transaction receipt as proof to claim a "refund" amount defined by the protocol.
