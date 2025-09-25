---
title: L1 to L2 Composability Examples
slug: /developers/reference/L1-to-L2-composability-examples
---

# L2 Reads from L1

This page provides practical examples of how to implement L1↔L2 synchronous composability using UniFi's Signal Service. These examples demonstrate how L2 contracts can read and consume data from L1 within the same slot.

*Note: Example code is illustrative - actual implementation may vary*

## Reading Chainlink Price Feeds

```solidity
// L2 Contract reading fresh L1 Chainlink data
contract L2PriceConsumer {
    ISignalService public signalService;

    function executeTradeWithFreshPrice(int256 ethPrice, uint256 blockNumber) external {
        // Compute signal ID from provided price data
        bytes32 signalId = keccak256(abi.encode("chainlink.eth.price", ethPrice, blockNumber));

        // Verify the signal was received from L1
        require(signalService.isSignalStored(signalId), "Price signal not confirmed");

        // Execute trade logic with same-slot L1 price data
        _executeTrade(ethPrice);
    }
}
```

## Same-Slot Deposit and Trade

```solidity
// L2 DEX contract utilizing same-slot deposits
contract L2DEX {
    ISignalService public signalService;

    function swapWithDeposit(uint256 depositAmount, address tokenOut) external {
        // Verify same-slot deposit signal from L1
        bytes32 depositSignal = keccak256(abi.encode(
            "deposit",
            msg.sender,
            depositAmount,
            block.number
        ));

        require(signalService.isSignalStored(depositSignal), "Deposit not confirmed");

        // Execute swap immediately with deposited funds
        _executeSwap(depositAmount, tokenOut, msg.sender);
    }
}
```

## Atomic Cross-Layer Operations

```solidity
// L2 Contract coordinating atomic L1↔L2 operations
contract AtomicBridge {
    function atomicDepositAndWithdraw(
        uint256 depositAmount,
        uint256 withdrawAmount
    ) external {
        // Verify L1 deposit signal
        bytes32 depositSignal = keccak256(abi.encode(
            "deposit", msg.sender, depositAmount, block.number
        ));
        require(signalService.isSignalStored(depositSignal), "Deposit failed");

        // Process L2 operations
        _processL2Operations(depositAmount);

        // Initiate instant withdrawal back to L1
        _initiateInstantWithdrawal(msg.sender, withdrawAmount);
    }
}
```

## Advanced Use Cases

### Cross-Layer Arbitrage

```solidity
contract ArbitrageBot {
    ISignalService public signalService;

    function executeArbitrage(
        uint256 l1Price,
        uint256 l2Price,
        uint256 amount
    ) external {
        // Verify L1 price signal
        bytes32 priceSignal = keccak256(abi.encode("price.update", l1Price, block.number));
        require(signalService.isSignalStored(priceSignal), "Price not confirmed");

        // Execute arbitrage if profitable
        if (l1Price > l2Price) {
            _buyOnL2SellOnL1(amount, l1Price, l2Price);
        } else if (l2Price > l1Price) {
            _buyOnL1SellOnL2(amount, l1Price, l2Price);
        }
    }
}
```

### Dynamic Collateral Management

```solidity
contract DynamicVault {
    ISignalService public signalService;

    function adjustCollateralRatio(uint256 l1CollateralValue, uint256 blockNumber) external {
        // Compute signal ID from provided collateral data
        bytes32 collateralSignal = keccak256(abi.encode("collateral.value", l1CollateralValue, blockNumber));

        // Verify the signal was received from L1
        require(signalService.isSignalStored(collateralSignal), "Collateral signal not confirmed");

        // Adjust L2 lending parameters based on L1 collateral
        _updateLendingRatio(l1CollateralValue);
    }
}
```

## Best Practices

### Signal Verification

Always verify that signals have been properly received before consuming them:

```solidity
function verifySignal(bytes32 signalId) internal view {
    require(signalService.isSignalStored(signalId), "Signal not received");
}
```

### Replay Protection

Implement replay protection to prevent signal reuse:

```solidity
mapping(bytes32 => bool) private consumedSignals;

function consumeSignalOnce(bytes32 signalId) internal {
    require(!consumedSignals[signalId], "Signal already consumed");
    consumedSignals[signalId] = true;
}
```

### Fallback Mechanisms

Provide fallback options when signals are not available:

```solidity
function processDataWithFallback(uint256 data, uint256 blockNumber) external returns (uint256) {
    bytes32 signalId = keccak256(abi.encode("data.update", data, blockNumber));

    if (signalService.isSignalStored(signalId)) {
        // Use the verified signal data
        return _processData(data);
    } else {
        // Fallback to cached or default value
        return _getFallbackData();
    }
}
```

## Testing

When testing Signal Service integration:

1. **Testnet Environment**: Use UniFi's testnet for realistic testing
2. **Mock Signals**: Create mock signals for unit testing
3. **Edge Cases**: Test signal failures and fallback scenarios
4. **Gas Optimization**: Monitor gas usage for signal operations

For more information on Signal Service architecture and implementation details, see the [Signal Service documentation](../concepts/signal-service.md).