---
title: Signal Service
slug: /developers/concepts/signal-service
---

# Signal Service

Signal Service enables same-slot L1↔L2 message passing in UniFi, allowing atomic operations between Ethereum and the L2 within a single block. This breakthrough capability is uniquely achievable by Based Rollups like UniFi, which have direct control over sequencing and transaction inclusion on both L1 and L2.

## Architecture Overview

### Signal Registry System

Signal Service implements a registry-based approach where signals (messages) are registered on L1 and verified through contract calls rather than traditional block header verification. This eliminates the inherent delay present in standard message passing systems.

### Core Components

- **L1 Signal Service Contract**: Registry where signals are registered and verified
- **L2 Signal Service Contract**: Mirrors L1 signals and manages their delivery status
- **Anchor Transaction**: System transaction that imports signals from L1 to L2
- **L1 Inbox Verification**: Ensures imported signals were legitimately registered

### Signal Flow

```
L1: User Transaction → Signal Registration → Signal Service Contract
                                               ↓
L2: Anchor Transaction ← Signal Import ← L2 Sequencer Selection
    ↓
L2: System Transaction → Mark Signals "Received" → L2 Signal Service
    ↓
L2: User Transactions → Consume Signals → Execute Logic
    ↓
L1: Batch Submission → Inbox Verification → Confirm Signal Validity
```

## Features

### Same-Slot Execution
Execute operations across L1 and L2 within the same Ethereum block, enabling true atomic composability previously impossible in traditional rollup architectures.

### Preserved OP Stack Compatibility
Signal Service complements rather than replaces OP Stack's existing message passing system, ensuring backward compatibility and minimal infrastructure changes.

### Sequencer Autonomy
As a Based Rollup, UniFi's sequencing is controlled by Ethereum validators, providing strong guarantees against censorship while maintaining the flexibility to include signals atomically.

### Fallback Safety
Signals that cannot be processed same-slot automatically fall back to traditional block header verification, ensuring reliability without compromising user experience.

### Enhanced Developer Experience
Developers can build applications that seamlessly interact across layers as if they were operating on a single chain.

## Use Cases

### Financial Applications
- **Atomic Arbitrage**: Execute arbitrage opportunities across L1 and L2 within the same block
- **Cross-Layer Lending**: Collateralize L1 assets for immediate L2 borrowing
- **Synchronized Trading**: Access fresh L1 oracle data for L2 trading decisions

### Infrastructure Services
- **Bridge Enhancements**: Enable instant deposits without waiting periods
- **Oracle Updates**: Provide same-slot data feeds from L1 to L2 applications
- **State Synchronization**: Keep L2 applications synchronized with critical L1 state changes

### User Experience
- **Seamless Onboarding**: Users can deposit and immediately interact with L2 applications
- **Unified Liquidity**: Access combined L1 and L2 liquidity pools atomically
- **Instant Settlement**: Complete complex multi-layer transactions in seconds

## Technical Implementation

### Signal Registration Process

1. **L1 Signal Creation**: User transaction calls Signal Service contract to register a signal with specific data payload
2. **Signal Selection**: L2 sequencer selects which signals to import based on application requirements and gas limits
3. **Anchor Transaction**: First transaction in L2 batch imports selected signals and marks them as "received" in L2 Signal Service contract
4. **Consumption**: Subsequent L2 transactions can safely consume the received signals
5. **Verification**: When batch is submitted to L1, Inbox contract verifies all imported signals were legitimately registered

### Security Considerations

- **Signal Validity**: All signals must be cryptographically verified
- **Sequencer Trust**: While sequencers control signal inclusion, they cannot forge invalid signals
- **Fallback Mechanisms**: Unprocessed signals automatically use traditional verification methods
- **Replay Protection**: Developers must implement replay protection to ensure signals are not consumed more than once

## Getting Started

To integrate Signal Service into your application:

1. Deploy contracts that interface with L1 and L2 Signal Service contracts
2. Implement signal registration logic in your L1 contracts
3. Add signal consumption logic to your L2 contracts
4. Test atomic operations in UniFi's testnet environment

For practical examples of L1↔L2 composability, see [L1 to L2 Composability Examples](../reference/L1-to-L2-composability-examples.md).

---

*References: Signal Service implementation is based on research from [Nethermind's same-slot message passing](https://ethresear.ch/t/same-slot-l1-l2-message-passing/21186) and [OpenZeppelin's Minimal Rollup architecture](https://github.com/OpenZeppelin/minimal-rollup/blob/main/documentation/signal-service.md).*