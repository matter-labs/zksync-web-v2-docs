# ZK Rollup Basics

This page aims to introduce developers to the basic concepts behind zkSync, including topics ranging from the basics of ZK rollups, L1 and L2 chains, and key differences in developing using zkSync over other scaling solutions.

## What are ZK rollups?

ZK rollups ('ZK' standing for zero-knowledge) are a recent development intended to increase the scalability of Ethereum by performing calculations off-chain, rolling many transactions up into a single batch, and sending it to the main Ethereum chain for verification in a single action. In zkSync, this is done via a **SNARK** (succinct non-interactive argument of knowledge), a cryptographic proof that performs the validation of transactions in the batch.

With ZK rollups, funds are locked on a smart contract in the layer 1 blockchain. This allows transactions to be processed without the overhead of all the data typically associated with performing a transaction on the main chain, only requiring a **validity proof** to reach transaction finality. This significantly decreases associated transaction processing times and gas fees.

## L1 and L2: what's the difference?

In decentralised ecosystems, the term **layer 1** (or **L1**) is used to refer to the underlying primary chain architecture, such as the Ethereum network or Bitcoin. Layer 1 blockchains determine protocol rules and transaction finality, and perform the base-level functions of applications built upon them.

**Layer 2** (or **L2**) is used to describe an overlaying application or network that operates on top of the layer 1 chain. These are most often built to provide further scalability solutions by taking on a portion of transaction-based tasks to lighten the impact on the layer-1 chain, quickening transaction times and lowering gas fees.

In the case of zkSync, the main Ethereum blockchain is L1, and the zkSync network is L2.

## How does transaction finality work?

In the context of blockchain technology, **transaction finality** refers to the guarantee that transactions cannot be reverted, altered or mutated.

On Ethereum, as in Bitcoin, finality is probabilistic, i.e. the more blocks that passed since the transaction was processed, the lesser the chance that this transaction will be reverted.

In ZK rollups, once a block has been filled and sealed, its state is committed to the main Ethereum chain. The proving step is then initiated, and a SNARK validity proof is generated for all the block transactions. Once completed, the SNARK is submitted for verification on the L1 smart contract, and after being verified, the transaction state becomes final.

Note that _finality_ from the zkSync perspective happens when the transaction (the SNARK verification) is processed by L1. At this stage, the guarantees are exactly like any other L1 transaction within the same L1 block; the more L1 blocks generated after the initial block is processed, the lesser the chance that this transaction will be reverted.

At the moment, when a user sends a transaction, zkSync waits for the entire block to be filled, meaning finality time may take longer depending on the volume of transactions being submitted via zkSync. As throughput increases, the finality time will subsequently decrease.

## What are operators?

**Operators** are the actors that perform basic ZK rollup functionalities. They are charged with creating blocks, bundling the transactions, performing the calculations and submitting the data to the main Ethereum chain for verification.
