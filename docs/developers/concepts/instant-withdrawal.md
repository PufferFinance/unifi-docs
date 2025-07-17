# TDX Based Atomic Withdrawal

# Description

The TDX Atomic Withdrawal procedure enables atomic, secure withdrawals from L2 to L1 by utilizing TDX execution proofs. This document outlines the implementation scenarios and API specification. 

## **System Components**

The TDX Atomic Withdrawal procedure integrates with the Optimism architecture and consists of these key components:

1. Rollup Node (op-node):
    - Transforms the latest L1 blocks into the L2 block inputs
    - Provides base API for the execution engine
    - Manages the derivation pipeline for L2 blocks
2. Execution Engine (op-geth):
    - Standard execution environment for the L2 chain
    - Accepts transactions, builds blocks, stores the chain
    - Serves RPC requests for transaction and block data
    - Contains key components: TxPool, EVM, Chain, and State
3. TDX Execution Engine (op-geth):
    - Enhanced version of op-geth with TDX capabilities
    - Re-executes batches and generates block hash
    - Produces cryptographic and TDX attestation proofs
    - Exposes API for retrieving execution proofs
4. L2 Output Submitter:
    - Queries for the next assertable block hash
    - Submits it to the L2 Output Oracle
    - Checks the state root consistency
    - Initiates atomic withdrawals based on verified proofs
5. L1 Smart Contracts:
    - OptimismPortal: Handles fund transfers between L2 and L1
    - DisputeGameFactory: Creates dispute games and verifies TDX proofs for withdrawals
    - Attestation Verification: Verifies the attestation proofs
    - Prover Registry: Maintains a registry of authorized provers

## Interaction Flow

![atomic_withdrawal](/img/rollup/instant_withdrawal.png)

## Workflow Description

1. **Initiation**: A withdrawal transaction is initially processed in the standard Execution Engine (op-geth) on L2, which "accepts transactions, builds blocks, stores the chain, and serves RPC requests."
2. **Block Processing and Output Submission**:
    - The L2 Output Submitter queries for the next assertable block hash (which contains withdrawal transactions)
    - It submits this block hash to the L2 Output Oracle
    - It also verifies state root consistency. If verification failed, the left procedure should follow the legacy op procedure.
3. **TDX Verification Flow**:
    - The TDX Execution Engine (op-geth) shown at the bottom of the diagram is critical for withdrawals
    - It performs two key functions:
    - Re-executes the batch and generates the block hash
    - Generates the attestation proof (which would include the TdxExecutionProof)
4. **L1 Contract Interaction**:
    - The attestation proof is submitted to the DisputeGameFactory contract on Ethereum L1
    - The DisputeGameFactory verifies the proof
    - Upon successful verification, it calls the OptimismPortal contract
5. **Fund Release**:
    - The OptimismPortal contract (shown in the top row) completes the actual fund transfer to the user on L1

## Security Considerations

- Execution Integrity: TDX Execution Engine provides cryptographic proofs of correct execution
- Dispute Resolution: The system includes mechanisms to challenge invalid withdrawal claims
- Rollback Protection: L2 state can revert to last valid state in case of proven fraud
- Fund Safety: Withdrawals only proceed with valid execution proofs
- Attestation Verification: Cryptographic attestation ensures the integrity of execution proofs

# Atomic Withdrawal Implementation Scenarios

## Scenario 1: All Transactions in the Current SafeL2 Block Execute Successfully

1. Transaction Verification:
TDX Execution Engine (op-geth) verifies the output of the current SafeL2 block, with results passing validation.
2. Withdrawal Transaction Filtering:
Filter withdrawal transactions from verified transactions.
3. Atomic Withdrawal Execution:
    - Proposer calls the `withdrawal` method on the L1 DisputeGameFactory contract to initiate atomic withdrawals.
    - DisputeGameFactory verifies the TdxExecutionProof and calls the Portal contract to complete fund transfers.
4. Dispute Game Creation:
Proposer calls the L1 DisputeGameFactory contract to create a dispute game for potential future dispute handling.

## Scenario 2: The Current SafeL2 Block Contains Fraudulent Transactions

1. Transaction Verification Failure:
TDX Execution Engine (op-geth) fails to verify the output of the current SafeL2 block, preventing atomic withdrawals.
2. Subsequent Block Verification:
Since subsequent blocks depend on the invalid current block, their outputs will also fail verification and cannot execute atomic withdrawals.
3. Dispute Initiation:
Proposer calls the L1 DisputeGameFactory contract to create a dispute game, initiating the dispute resolution process.
4. L2 State Rollback:
Due to disputed block data, the L2 chain reverts to the most recent block where all transactions are valid, ensuring data consistency.
5. Revalidation:
TDX Execution Engine (op-geth) re-verifies the output of the rolled-back block. If validation passes, the system returns to Scenario 1 flow and continues normal transaction processing.

# TDX Execution Proof API

## API Name

`tdx_getExecutionProof`

Description
This API retrieves execution proofs for specified blocks, returning a `TdxExecutionProof` structure containing execution proof, state root, and block header information.

---

## API Definition

### RPC Method

```json
{
  "jsonrpc": "2.0",
  "method": "tdx_getExecutionProof",
  "params": [<block_number>],
  "id": 1
}

```

### Request Parameters

| Parameter | Type | Required | Description |
| --- | --- | --- | --- |
| `block_number` | `rpc.BlockNumber` | Yes | Block number to query. |

---

### Response Structure

Returns a `TdxExecutionProof` structure containing execution proof and block header information.

Response Format

```json
{
  "proof": "0xE000",
  "stateRoot": "0x...",
  "header": {
    "hash": "0x...",
    "number": 123456,
    "parentHash": "0x...",
    "timestamp": 1700000000
  }
}

```

Response Fields

| Field | Type | Description |
| --- | --- | --- |
| `proof` | `string` | Hexadecimal string of the execution proof. |
| `stateRoot` | `string` | State root of the block. |
| `header` | `object` | Block header information. |
| `header.hash` | `string` | Current block hash. |
| `header.number` | `uint64` | Current block height (number). |
| `header.parentHash` | `string` | Parent block hash. |
| `header.timestamp` | `uint64` | Block timestamp (in seconds). |

---

## Example

Request Example

```json
{
  "jsonrpc": "2.0",
  "method": "tdx_getExecutionProof",
  "params": ["latest"],
  "id": 1
}

```

Response Example

```json
{
  "jsonrpc": "2.0",
  "result": {
    "proof": "0x...",
    "stateRoot": "0xabcd1234...",
    "header": {
      "hash": "0x123456...",
      "number": 123456,
      "parentHash": "0xabcdef...",
      "timestamp": 1700000000
    }
  },
  "id": 1
}

```