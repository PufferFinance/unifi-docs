# Opcodes

All opcodes are supported in UniFi and are EVM equivalent unless stated otherwise.

# Modified Opcodes

| Opcode | Name      | Solidity Equivalent | Rollup Behaviour                             | Ethereum L1 Behaviour                |
|--------|-----------|---------------------|----------------------------------------------|--------------------------------------|
| 32     | ORIGIN  | tx.origin      | If the transaction is an L1 ⇒ L2 transaction triggered by a smart contract on L1, then tx.origin is set to the aliased address of the address that triggered the L1 ⇒ L2 transaction. Otherwise, this opcode behaves normally. | Get execution origination address |
| 33     | CALLER  | msg.sender      | If the transaction is an L1 ⇒ L2 transaction triggered by a smart contract on L1, and this is the first call frame (rather than an internal transaction from one contract to another), the same address aliasing behavior applies. | Get caller address |
| 41     | COINBASE  | block.coinbase      | Returns the address of the L2 block proposer | Gets the block's beneficiary address |
| 42     | TIMESTAMP | block.timestamp     | Timestamp of the L2 block                    | Timestamp of the L1 block            |
| 43     | NUMBER    | block.number        | L2 block number                              | Gets the L1 block number             |
| 44     | PREVRANDAO    | block.prevrandao        | Returns the PREVRANDAO (the most recent RANDAO) value of L1 at the current L1 origin block | Returns the output of the randomness beacon provided by the beacon chain |
| 49     | BLOBHASH   | blobhash(index)       | N/A | Get versioned hashes |
| 4A     | BLOBBASEFEE | block.blobasefee    | Returns 1 | Returns the value of the blob base-fee of the current block |
