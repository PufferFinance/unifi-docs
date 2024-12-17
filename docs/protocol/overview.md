---
title: Overview
slug: /protocol/overview
---

:::info
UniFi is an Ethereum Based Rollup with enabled preconfirmations.
:::

## UniFi Walkthrough
![Based sequencing](/img/rollup/based-sequencing.png)

### 1️⃣ Transaction Submission
The flow begins with users submitting **rollup transactions** to **UniFi Sequencers**. Sequencers are responsible for transaction aggregation and ordering for the rollup. They play a critical role in offering **preconfirmations** to users when possible.

**Preconfirmations** are essentially "soft promises" from the sequencers that the transactions will be included in the next L1 block. These preconfirmations come with economic guarantees backed by penalties (slashing conditions) to ensure sequencer reliability. If a sequencer fails to include a transaction it has preconfirmed, it risks being slashed.

### 2️⃣ Block Proposal
Once the sequencer has aggregated enough transactions, it bundles them into a **block**. This block is then **proposed** to UniFi's **Rollup Contract** on the Ethereum L1. At this stage, no execution is performed. Ethereum validators simply record the block's ordered transactions on the L1, and only transaction validation is performed on a contract level. It's important to note that Ethereum validators are responsible for the order of inclusion of these **proposed block** transactions in the L1 chain, which is where the "based"-ness of UniFi comes into play.

### 3️⃣ Block Insertion
When a proposed block is successfully included in the Ethereum L1 chain, the **UniFi Client** listens for this event. The client:
   1. Extracts the ordered transaction data from the rollup contract.
   2. Constructs an **L2 block** using this transaction data.
   3. Sends the L2 block to the **L2 Execution Client** (a modified Geth client).

Once the block is executed by the L2 Execution Client, the **L2 state** is updated, and the block is officially added to the L2 chain.

### 4️⃣ Block Proving
The last step is to prove to the L1 that the block has been executed correctly on the L2. This is where **UniFi Provers** come into play. They listen for newly inserted L2 blocks (without a proof) and generate cryptographic proofs to validate their correctness.

To generate a proof, provers create a **Proof of Block (PoB)** statement themselves, which they send over to an **Intel TDX TEE** for attestation. The TEE acts as a trusted computation enclave that uses the PoB to securely re-execute the block's transactions and verify the state transitions. Once the re-execution completes:

- The TEE generates an **attestation** that ensures the L2 block's integrity.
- The attestation and cryptographic proof are submitted to UniFi's L1 contract on Ethereum.
- The L1 contract verifies the proof, ensuring that the state transitions are valid and match the ordered transactions.
- Once verified, the block is considered **proven**, and the L1 state is updated to reflect this.

### Verified block state
In the L2 chain, a block is not considered truly final once it is proven; it must also be **verified**. Verification ensures that a block is not only proven but also part of a fully validated sequence of blocks. This requirement arises because blocks can be proposed and proven asynchronously. For instance, if block N has been proven, but its parent block N-1 has not yet been verified, block N cannot achieve the verified status. The verified state is crucial for the protocol as certain operations can only be performed on verified blocks (e.g., mpt verification of state proofs).

Practically speaking, in order for a block to be considered **verified** it must meet the following conditions:
- The block itself is proven: its cryptographic proof has been submitted and validated by the L1 contract.
- Its parent block is also verified: This guarantees a continuous and valid chain back to the genesis block, which is inherently verified.

![Block states](/img/rollup/block-states.png)
