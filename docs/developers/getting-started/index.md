---
title: Introducing UniFi Rollup 🐡
slug: /developers/getting-started/index
---

# What is Puffer UniFi Rollup?

Puffer UniFi is an Ethereum Layer 2 (L2) based rollup designed to provide scalability to Ethereum while at the same time addressing the fragmentation problem faced by Ethereum. 
Specifically, Puffer UniFi rollup will solve 4 big problems faced by Ethereum today:
- Realizing atomic composability allowing for instant transaction between L1 (Ethereum) and L2
- Liquidity fragmentation in L2s
- Enable decentralize sequencing in L2s
- Increase utility of Ethereum Token

### How is "Based Rollup" different from traditional rollups?

Based rollups derive their transaction sequencing directly from Ethereum validators on the L1 network. 
This means that L1 validators determine the order of rollup transactions within the blocks they propose.
By shifting sequencing responsibilities to L1 validators, UniFi eliminates the need for a centralized sequencer,
giving users access to Ethereum's decentralized validator set for a more secure and neutral transaction layer.

### Key Features of UniFi Rollup

- **Credibly Neutral Sequencing**: Transactions are sequenced by Ethereum's L1 validators, ensuring reliability and neutrality.
- **Fast Transactions**: Users enjoy 100ms pre-confirmations, enhancing the overall experience.
- **Value Flow Back to Ethereum**: Sequencing fees flow back to block proposers on Ethereum, supporting the sustainability and value of the L1 network.
- **Fast Withdrawl from L2 to L1 (Ethereum)**: In the initial launch of UniFi, it will allow for fast withdrawal from UniFi to Ethereum L1 within 1 hours. As UniFi achieves true atomic composability, this withdrawal time is expected to reduce to as low as 8 seconds.
  
### Why Choose UniFi Rollup?

UniFi aims to drive long-term sustainability and value into Ethereum's base layer by allowing transaction
sequencing fees to benefit L1 validators. Users benefit from fast, seamless, and cost-efficient transactions,
making it an ideal solution for those seeking scalability without sacrificing decentralization.

### Get Started with UniFi Rollup

To begin using UniFi, start by [acquiring Testnet tokens](/acquire-testnet-tokens).

By leveraging UniFi's Ethereum-based rollup, users can enjoy an efficient, decentralized, and unified experience while contributing to the long-term health of the Ethereum ecosystem.

# Building on UniFi
UniFi is mostly EVM equivalent, which means that developers can use familiar EVM tools and frameworks,
such as Hardhat, Truffle, and Remix, to interact with and develop on UniFi.
This compatibility significantly reduces the learning curve, allowing developers to leverage existing skills and knowledge.

:::info
For the specific differences between UniFi and EVM behaviour, see the [Opcodes](../reference/opcodes.md) page.
:::
