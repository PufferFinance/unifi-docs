---
title: TEE Multi Prover
slug: /developers/concepts/tee-multi-prover
---

## Background

The defining characteristic of a rollup is the ability to push L2 state updates to L1. Typically, either fraud proofs or validity proofs are used to ensure the correctness of each state update.

Fraud and validity proofs rely on the strong assumption that the underlying construction and implementation of the proof system are correct. Unfortunately, proof systems are complex pieces of code prone to bugs and errors, and it is impossible to guarantee they are infallible. For this reason, a diverse set of proof systems is essential to safeguard the overall robustness of the system - an idea that has reverberated in multiple places. Different proof systems act as a counter-check against each other: by comparing their outputs against each other we can identify potential discrepancies - similar to differential testing.

We propose implementing an additional prover system by leveraging Trusted Execution Environment (TEE) technology because this approach yields performance without the complexities associated with e.g., ZK circuits.

## Brief Overview

![Brief Overview](/img/tee-multi-prover/overview.png)
*(Note: This design represents the ideal approach and may differ from the existing implementation.)*

This diagram illustrates how the TEE Validity Proof interacts with the L1 rollup contract:
	1.	Whenever the prover relayer needs to update the rollup state on L1, it sends a Proof of Block to the TEE Prover. The Proof of Block contains all the necessary states, enabling the TEE Prover to perform stateless state transitions.
	2.	The TEE Prover executes the blocks based on the Proof of Block and generates a TEE Validity Proof, which is then returned to the prover relayer.
	3.	The prover relayer submits the updated state along with the proofs to the L1 Rollup Contract.

## The Case for Using TEEs

1. TEEs Have Good Performance: TEEs provide reliable, fast, and cost-effective computation. The computational overhead of TEEs is minimal (and can even be negligible, depending on the specific TEE technology). This allows TEEs to perform state transitions with very low latency and high efficiency. For example, based on our tests, Intel TDX introduces less than 1% performance overhead, and the time required to generate a proof is approximately the same as the time needed to execute a block.
2. TEEs can be Used Despite Not Being Infallible: A major concern surrounding TEEs, especially Intel SGX, is that they are frequently compromised. While it is true that vulnerabilities related to TEEs continue to be discovered, it is worth highlighting that these vulnerabilities are not fatal and can't be exploited indefinitely once they are patched - existing exploits are implementation bugs that do not undermine the architectural design of TEEs. Moreover, these vulnerabilities are often difficult, or even impossible, to exploit unless certain conditions are met, e.g., Aepic Leak allows an attacker to access SGX's most precious secrets, but cannot be exploited unless APIC registers are exposed. TEEs are also improving, and more technologies are emerging such as AMD SEV, Intel TDX, and AWS Nitro Enclaves. Lastly, if the vulnerability of one TEE is particularly concerning, a committee consisting of different TEE technologies (e.g., Intel + AMD + AWS) can be employed to increase the security of the overall system - in this scenario, a potential attacker would need to exploit multiple TEE technologies at the same time, making it increasingly difficult to mount an attack.

## On-Chain Proof Verification

During the on-chain proof verification process, the following checks are performed:
	1.	The software of TEE Prover is verified to ensure it aligns with the expected version and has not been tampered with.
	2.	The TEE firmware and SDK of the virtual machine hosting the TEE Prover are confirmed to be up-to-date.
	3.	The initial state of the proof for the state transition matches the current state on-chain.

The attestation report provides the necessary information for points 1 and 2, and we use the [automata-dcap-attestation](https://github.com/automata-network/automata-dcap-attestation) to verify its validity.

A attestation report verification costs approximately 3M gas. To reduce this expense, we use a ProverRegistry to separate the attestation and rollup processes, allowing a single attestation to support multiple state rollups during its validity period.

The process is as follows:
	1.	A secp256k1 key is randomly generated inside the TEE.
	2.	When generating the attestation report, the key is embedded into the attestation report's user data.
	3.	Upon submission of the attestation report to the ProverRegistry, the public key is recorded.
	4.	When the TEE Prover provides a proof for a state transition, it signs the result using the ephemeral key.
	5.	During proof verification, the ProverRegistry uses ecrecover to retrieve the public key and validate it against the recorded key.

With this optimization, the verification cost for a single proof is reduced to no more than 10k gas.


![workflow](/img/tee-multi-prover/workflow.png)