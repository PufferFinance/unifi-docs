---
title: UniFi Architecture
slug: /developers/concepts/architecture
---

UniFi is an Ethereum Based Rollup with enabled preconfirmations, instant withdrawals, and L1↔L2 synchronous composability. Particularly preconfirmations are enabled by integrating Gateways, instant withdrawals are enabled by leveraging TEE proofs, and synchronous composability is enabled by leveraging Signal Service.

## Preconfirmations

Preconfirmations in UniFi are enabled by **Gateways**, specialized sequencing entities that provide execution preconfirmations on behalf of L1 proposers.

### What are Gateways?

Gateways are high-performance sequencers that handle transaction processing and block production for based rollups. They act as intermediaries between users and L1 proposers, allowing proposers to outsource sophisticated sequencing operations without requiring specialized infrastructure.

### Enabling Preconfirmations:

Gateways enable fast preconfirmations by:

1. **Immediate Transaction Processing**: Accept and validate transactions continuously without waiting for L1 block times
2. **Fragment Broadcasting**: Share partial block data in real-time, allowing users to receive confirmation before final block settlement
3. **Exclusive Write Access**: Maintain temporary exclusive write access to L2 state during assigned slots, enabling confident preconfirmation issuance

This design allows users to receive execution confirmations within milliseconds rather than waiting for L1 block finalization, significantly improving user experience while maintaining security through the underlying based rollup architecture.

For a deep dive into the architecture of a Gateway, you can refer to the [Gattaca's Gateway Documentation](https://gattaca-com.github.io/based-op/).

## Instant Withdrawals

Instant withdrawals in UniFi enable users to withdraw funds from L2 to L1 immediately without waiting for the standard 7-day dispute period. This is achieved through **TEE (Trusted Execution Environment) proofs** that provide cryptographic guarantees of execution correctness.

### How Instant Withdrawals Work

Traditional optimistic rollups require a challenge period where withdrawals can be disputed. UniFi bypasses this delay by using TEE-generated validity proofs that cryptographically prove the correctness of withdrawal transactions.

### Instant Withdrawal Process

![Instant Withdrawal Flow](/img/rollup/instant_withdrawal.png)

1. **User Initiates Withdrawal**: User submits withdrawal transaction on L2
2. **Standard Processing**: Transaction processed in L2 Execution Engine and included in a block
3. **TEE Proof Generation**: TDX Execution Engine re-executes the block and generates attestation proofs
4. **Proof Submission**: L2 Output Submitter collects proofs and calls `atomicWithdrawal` on L1
5. **Verification & Release**: DisputeGameFactory verifies proofs and triggers fund release via OptimismPortal

### Key Benefits

- **No Waiting Period**: Withdrawals complete in tens of seconds instead of 7 days
- **Fallback Safety**: System can revert to standard dispute resolution if TEE verification fails

### Security Guarantees

The instant withdrawal system maintains security through:
- **Execution Integrity**: TEE hardware ensures correct transaction execution
- **Attestation Verification**: On-chain verification of TEE software and firmware integrity  
- **Multi-Prover Validation**: Multiple TEE clients provide redundant verification

This design enables near-instant L2 to L1 transfers while preserving the security properties of the underlying rollup architecture.

## L1↔L2 Synchronous Composability

UniFi leverages **Signal Service** to enable same-slot L1→L2 message passing, achieving synchronous composability across layers. As a Based Rollup, UniFi is uniquely positioned to offer strong guarantees for cross-layer interactions while preserving existing OP Stack functionality.

### What is Signal Service?

Signal Service enables same-slot L1→L2 message passing by implementing a signal registry system that uses **contract calls** for verification instead of relying on block headers with inherent delays. This complements OP Stack's existing message passing capabilities rather than replacing them.

### Why Signal Service for L1→L2 Only?

UniFi uses Signal Service specifically for L1→L2 same-slot messaging because:

1. **Preserve OP Stack**: We avoid extensive modifications to OP Stack's existing message passing system, which continues to function for non-same-slot operations
2. **L2→L1 Already Solved**: OP Stack's existing L2→L1 message passing combined with TEE proofs already enables same-slot L2→L1 operations through instant withdrawals

### How It Works

1. **Signal Registration**: Users register signals (messages) on L1 using contract calls to the Signal Service registry
2. **Anchor Transaction**: L2 sequencer includes a system transaction at the beginning of each batch that imports L1 signals and marks them as "received" on L2
3. **Same-Slot Processing**: L2 transactions in the same batch can safely consume the imported signals
4. **L1 Verification**: When the batch is submitted to L1, the Inbox contract verifies that all imported signals were indeed registered on the Signal Service

### Key Benefits for Based Rollups

- **Atomic Guarantees**: Enables true atomic composability between L1 and L2 operations within a single slot
- **Enhanced UX**: Users can deposit on L1 and immediately use funds on L2, or perform complex multi-layer operations atomically

### Use Cases

- **Same-Slot Deposits**: Deposit ETH on L1 and immediately spend on L2 within the same block
- **Atomic Cross-Layer Operations**: Execute deposit + L2 operations + withdrawal atomically
- **L1 State Access**: Read fresh L1 data (like Chainlink prices) without waiting for block imports

For detailed implementation and usage examples, see the [Signal Service documentation](signal-service.md).
