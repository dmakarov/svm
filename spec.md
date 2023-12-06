# Solana Virtual Machine specification

# Vision

Several components of the Solana Validator are involved in processing of a transaction (or a batch of transactions). These components can be factored out to decouple the transaction processing components from other components of the Solana Validator with the goal of improving the software architecture of the system as a whole by better modularization and well-defined interfaces between the Solana Validator components. The refactoring is non-functional in the sense that the protocol currently implemented by the Solana Validator is uneffected by changes to the transaction processing code.

Collectively the components responsible for transaction execution are designated as Solana Virtual Machine (SVM). SVM packaged as a stand-alone library can be used in applications outside the Solana Validator.

This document represents specification of the SVM. It covers the API of using SVM in projects unrelated to Solana Validator and the internal workings of the SVM, including the descriptions of the inner data flow, data structures, and algorithms involved in the execution of transactions. The document’s target audience includes both external users and the developers of the SVM.

## Applications (or use cases)

The following possible applications for SVM were collected from potential interested users

- **Transaction execution in Solana Validator**
    
    This is the primary use case for the SVM. It remains a major component of the Solana Validator, but with clear interface and isolated from dependencies on other components.
    
    The SVM is currently viewed as realizing two stages of the Transaction Engine Execution pipeline as described in Solana Architecture documentation [https://docs.solana.com/validator/runtime#execution](https://docs.solana.com/validator/runtime#execution), namely ‘load accounts’ and ‘execute’ stages *(confirmation needed)*
    
- **SVM Rollups**
    
    Rollups that need to execute the block but don’t need the other components of the validator can benefit from this as it can reduce hardware requirements and decentralize the network. This is especially useful for Ephemeral Rollups since the cost of compute will be higher as a new rollup is created for every user session in applications like gaming.
    
- **SVM Fraud Proofs for Diet Clients**
    
    A succinct proof of an invalid state transition by the supermajority (SIMD-65)
    
- **Validator Sidecar for JSON-RPC**
    
    The RPC needs to be separated from the validator, simulateTransaction requires replaying the transactions and have access to necessary account data.
    
- **SVM-based Avalanche subne**t
    
    The SVM would need to be isolated to run within a subnet since the consensus and networking functionality would rely on Avalanche modules.
    
- **Modified SVM (SVM+)**
    
    An SVM type with all the current functionality and extended instructions for custom use cases. This would form a superset of the current SVM.
    

It’s not clear at this point whether all these use cases can be satisfied with the same interface to the SVM or different methods of invocation and/or data structures will be required for invocation of the SVM in a specific use case. The use-cases need to be expanded in more descriptive explanations that clearly specify the environment or context in which the SVM operates for each use-case.

# System Context

In this section SVM is represented as a single entity. We describe its interfaces to the parts of the Solana Validator external to SVM, even if at the time of writing they are intertwined with the SVM.

SVM is an integral part of Solana Validator. It is used to process transaction batches on a single thread. As such it is invoked by a bank and produces transaction results consumed by the bank.

In the context of Solana Validator the main entity external to SVM is bank. It creates an SVM, submits transactions for execution and receives results of transaction execution from SVM. (currently SVM doesn’t exist as a separate data structure, this document should define it in the functonal model section).

[SVM Context diagram should be added]

Identify data flows to and from SVM (a diagram might be helpful).

Bank to SVM sends a batch of transactions, etc. (From To Description Data)

In the process of describing the System Context we also define the Functional Requirements for the SVM.

Non-functional requirements: testability (what else - performance, how to measure?)

ExecutionRecord is only used in bank.rs

## Interfaces

In this section we describe to API of using the SVM both in Solana Validator and in third-party applications.

Do different use cases change the functional requirements for SVM? What are our use cases (not clear from the list of high level applications)? Process a transaction batch is that all?

Currently, the single entry point for the SVM is the function `load_and_execute_transactions` defined in bank.rs (may become a method of SVM struct).

Questions from Lucas document (to answer here later)

Revise the arguments of `load_and_execute_transactions`. It receives a `TransactionBatch`. Are we going to provide our own scheme for transactions? What about the other arguments, like `BlockHashQueue`, `AccountOverrides`? If defined in runtime, we’ll provide the structure in the SVM. If not, we’ll provide a trait.

`FeeStructure` needs an interface. Transactions perhaps? What happens if users do not follow our transaction scheme?

Does `RentState` need an interface? Do we assume all accounts will pay rent?

# Functional Model

In this section we describe the functionality (logic) of the SVM in terms of its components, relationships among components, and their interactions.

On a high level the control flow of SVM consists of loading program accounts, checking and verifying the loaded accounts, creating invocation context and invoking RBPF on programs implementing the instructions of a transaction. The SVM needs to have access to account database, sysvar cache via traits implemented for the corresponding objects passed to it. The results of transaction execution are consumed by bank in Solana Validator use case. However, bank structure should not be part of the SVM.

Questions about structure of SVM fron Lucas document (need to review and answer)

Parts that should belong to SVM: 1. Inputs: 1. Accounts 2. Transactions 3. Builtin programs 4. A trait for bank 2. Outputs: 1. Processed transactions

Metrics: `TransactionErrorMetrics`, `ExecuteDetailsTimings`

Result structs: `TransactionExecutionDetails` and `TransactionExecutionResult` declared in accounts-db should be part of SVM.

`TransactionCheckResult` is a candidate for moving into runtime.

Inner structs related to execution: `SanitizedMessage` may be moved to runtime and may belong in SVM. It is declared in solana_program::message and only used in transaction code.

Program-runtime is all part of SVM. Nothing should be moved out of there.

Execution context data structures: `TransactionContext`, `InstructionContext` is defined in SDK, but is truly part of SVM or not?

Is the BpfLoader part of the SVM? It is a program on the blockchain, not necessarily part of the VM?