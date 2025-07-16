---
title: UniFi Architecture
slug: /developers/concepts/architecture
---

UniFi is an Ethereum Based Rollup with enabled preconfirmations and instant withdrawals. Particularly preconfirmations are enabled by integrating Gateways and instant withdrawals are enabled by leveraging TEE proofs.

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
