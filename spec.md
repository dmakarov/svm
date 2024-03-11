# Solana Virtual Machine specification

# Vision

Several components of the Solana Validator are involved in processing
of a transaction (or a batch of transactions).  Collectively the
components responsible for transaction execution are designated as
Solana Virtual Machine (SVM). SVM packaged as a stand-alone library
can be used in applications outside the Solana Validator.

This document represents specification of the SVM. It covers the API
of using SVM in projects unrelated to Solana Validator and the
internal workings of the SVM, including the descriptions of the inner
data flow, data structures, and algorithms involved in the execution
of transactions. The document’s target audience includes both external
users and the developers of the SVM.

## Applications (or use cases)

The following possible applications for SVM were collected from
potential interested users

- **Transaction execution in Solana Validator**

    This is the primary use case for the SVM. It remains a major
    component of the Solana Validator, but with clear interface and
    isolated from dependencies on other components.

    The SVM is currently viewed as realizing two stages of the
    Transaction Engine Execution pipeline as described in Solana
    Architecture documentation
    [https://docs.solana.com/validator/runtime#execution](https://docs.solana.com/validator/runtime#execution),
    namely ‘load accounts’ and ‘execute’ stages *(confirmation
    needed)*

- **SVM Rollups**

    Rollups that need to execute the block but don’t need the other
    components of the validator can benefit from this as it can reduce
    hardware requirements and decentralize the network. This is
    especially useful for Ephemeral Rollups since the cost of compute
    will be higher as a new rollup is created for every user session
    in applications like gaming.

- **SVM Fraud Proofs for Diet Clients**

    A succinct proof of an invalid state transition by the supermajority (SIMD-65)

- **Validator Sidecar for JSON-RPC**

    The RPC needs to be separated from the validator,
    simulateTransaction requires replaying the transactions and have
    access to necessary account data.

- **SVM-based Avalanche subnet**

    The SVM would need to be isolated to run within a subnet since the
    consensus and networking functionality would rely on Avalanche
    modules.

- **Modified SVM (SVM+)**

    An SVM type with all the current functionality and extended
    instructions for custom use cases. This would form a superset of
    the current SVM.


It’s not clear at this point whether all these use cases can be
satisfied with the same interface to the SVM or different methods of
invocation and/or data structures will be required for invocation of
the SVM in a specific use case. The use-cases need to be expanded in
more descriptive explanations that clearly specify the environment or
context in which the SVM operates for each use-case.

# System Context

In this section SVM is represented as a single entity. We describe its
interfaces to the parts of the Solana Validator external to SVM, even
if at the time of writing they are intertwined with the SVM.

SVM is an integral part of Solana Validator. It is used to process
transaction batches on a single thread. As such it is invoked by a
bank and produces transaction results consumed by the bank.

In the context of Solana Validator the main entity external to SVM is
bank. It creates an SVM, submits transactions for execution and
receives results of transaction execution from SVM. (currently SVM
doesn’t exist as a separate data structure, this document should
define it in the functonal model section).

[SVM Context diagram should be added]

Identify data flows to and from SVM (a diagram might be helpful).

Bank to SVM sends a batch of transactions, etc. (From To Description Data)

In the process of describing the System Context we also define the
Functional Requirements for the SVM.

Non-functional requirements: testability (what else - performance, how to measure?)

ExecutionRecord is only used in bank.rs

## Interfaces

In this section we describe to API of using the SVM both in Solana
Validator and in third-party applications.

Do different use cases change the functional requirements for SVM?
What are our use cases (not clear from the list of high level
applications)? Process a transaction batch is that all?

The interface to SVM is represented by the
`transaction_processor::TransactionBatchProcessor` struct.  To create
a `TransactionBatchProcessor` object the client need to specify the
`slot`, `epoch`, `epoch_schedule`, `fee_structure`, `runtime_config`,
and `loaded_programs_cache`. We'll explain these parameter in detail.

The main entry point to the SVM is the method
`load_and_execute_sanitized_transactions`. In addition
`TransactionBatchProcessor` provides two public utility methods
`filter_executable_program_accounts` and `load_program`, used
externally for testing and internally by SVM.

# Functional Model

In this section we describe the functionality (logic) of the SVM in
terms of its components, relationships among components, and their
interactions.

On a high level the control flow of SVM consists of loading program
accounts, checking and verifying the loaded accounts, creating
invocation context and invoking RBPF on programs implementing the
instructions of a transaction. The SVM needs to have access to account
database, sysvar cache via traits implemented for the corresponding
objects passed to it. The results of transaction execution are
consumed by bank in Solana Validator use case. However, bank structure
should not be part of the SVM.


# Work area
*(stuff from here will be moved up to proper sections when settles down)*

Load and execute transactions.

![class diagram](/diagrams/classdiag.png "Logical structure supporting load_and_execute_transactions")

In bank context `load_and_execute_transactions` is called from
`simulate_transaction` where a single transaction is executed, and
from `load_execute_and_commit_transactions` which receives a batch of
transactions from its caller.

- Input: `TransactionBatch`

 - `TransactionBatch`
   : contains
    - vector of `Result` lock_results -- this vector contains results
      of locking the accounts used in the transactions of the
      batch. When a batch is created the `account_db` is used to lock
      the accounts involved in all the transactions of the batch.
    - a reference to Bank -- need to decouple from Bank
    - a slice of `SanitizedTransaction` wrapped in copy on write
    - a boolean flag `needs_unlock`   -- this field is only used when TransactionBatch is dropped and the locked accounts have to be unlocked.

Of the above we should need only the slice of `SanitizedTransaction`. Let's analyze it next

 - `SanitizedTransaction`
   : contains
    - a SanitizedMessage  -- explain
    - a Hash of the message
    - a boolean flag `is_simple_vote_tx` -- explain
    - a vector of `Signature`  -- explain which signatures are in this vector

`SanitizedMessage` is an enum with two kinds of messages
    - `LegacyMessage`
    - and `LoadedMessage`

Both `LegacyMessage` and `LoadedMessage` consist of
    - `MessageHeader`
    - vector of `Pubkey` of accounts used in the transaction
    - `Hash` of recent block
    - vector of `CompiledInstruction`
    In addition `LoadedMessage` contains a vector of
    `MessageAddressTableLookup` -- list of address table lookups to
    load additional accounts for this transaction.

- Output: `LoadAndExecuteTransactionsOutput`

Multiple results of `load_and_execute_transactions` are aggregated in
the struct `LoadAndExecuteTransactionsOutput`
 - `LoadAndExecuteTransactionsOutput` contains
  - vector of `TransactionLoadResult`
  - vector of `TransactionExecutionResult`
  - vector of indexes of retriable transactions
  - number of executed transactions
  - number of executed non-vote transactions
  - number of successfully executed transactions (no error)
  - signature count (`u64`)
  - error counters (`TransactionErrorMetrics`)

Steps of `load_and_execute_transactions`

1. Collect indexes of retriable transactions in the batch. This is
   done by iterating over the results of locking the accounts of each
   transaction. If the result is a recoverable error, such *account in
   use*, the transaction may be retried at a later time and returned
   in the `LoadAndExecuteTransactionsOutput`.

2. Check transactions, currently two checks
   - check age
     - If the locked results for a transaction are not an error, we:
         - check if the transaction's blockhash is valid for the `max_age` OR
         - verify if the transaction message has a durable nonce that is authorized, if the transaction nonce is not advanceable.
   - check status cache
     - Checks if the transaction has already been processed.

3. Steps of preparation for execution
   - filter executable program accounts and build program accounts map (explain)
   - add builtin programs to program accounts map
   - replenish program cache using the program accounts map (explain)

4. Load accounts (call to `load_accounts` function)
   - For each `SanitizedTransaction` and `TransactionCheckResult`, we:
        - Calculate the number of signatures in transaction and its cost.
        - Call `load_transaction_accounts`
            - The function is interwined with the struct `CompiledInstruction`
            - Load accounts from accounts DB
            - Extract data from accounts
            - Verify if we've reached the maximum account data size
            - Validate the fee payer and the loaded accounts
            - Validate the programs accounts that have been loaded and checks if they are builtin programs.
            - Return `struct LoadedTransaction` containing the accounts (pubkey and data), 
              indices to the excutabe accounts in `TransactionContext` (or `InstructionContext`),
              the transaction rent, and the `struct RentDebit`.
            - Generate a `NonceFull` struct (holds fee subtracted nonce info) when possible, `None` otherwise.
    - Returns `TransactionLoadedResult`, a tuple containing the `LoadTransaction` we obtained from `loaded_transaction_accounts`,
      and a `Option<NonceFull>`.

5. Execute each loaded transactions
   This is done in the method `Bank::execute_loaded_transaction`. The
   method takes a `SanitizedTransaction` as input, LoadedTransaction
   as input, output parameter, `ComputeBudget` and
   `LoadedProgramsForTxBatch` as input, several configuration flags,
   `ExecuteTimings`, and `TransactionErrorMetrics` as output
   parameters. It returns `TransactionExecutionResult` value. This
   method should be completely isolated from Bank.
   1. Compute the sum of transaction account balances. This sum is
      invariant of the transaction execution.
   2. Obtain rent state of each account before the transaction
      execution. This is later used in verifying the account state
      changes (step #7). 
   3. Create a new log_collector.  `LogCollector` is defined in
      solana-program-runtime crate (`InvokeContext` is also there, so
      this will be proper part of SVM).
   4. Obtain last blockhash and lamports per signature. This
      information is read from blockhash_queue maintained in Bank. The
      information is taken in parameters to
      `MessageProcessor::process_message`. How this will be migrated
      to SVM?
   5. Make two local variables that will be used as output parameters
      of `MessageProcessor::process_message`. One will contain the
      number of executed units (the number of compute unites consumed 
      in the transaction). Another is a container of `LoadedProgramsForTxBatch`. 
      The latter is initialized with the slot (another dependency on `Bank`), and
      the clone of environments of `programs_loaded_for_tx_batch`
         - `programs_loaded_for_tx_batch` contains a reference to all the `LoadedProgram`s
            necessary for the transaction. It maintains an `Arc` to the programs in the global
            `LoadedPrograms` data structure.
      6. Call `MessageProcessor::process_message` to execute the
      transaction. `MessageProcessor` is contained in
      solana-program-runtime crate. The result of processing message
      is either `ProcessedMessageInfo` which is an i64 wrapped in a
      struct meaning the change in accounts data length, or a
      `TransactionError`, if any of instructions failed to execute
      correctly.
   7. Verify transaction accounts' `RentState` changes (`verify_changes` function)
      - If the account `RentState` pre-transaction processing is rent exempt or unitiliazed, the verification will pass.
      - If the account `RentState` pre-transaction is rent paying:
         - A transition to a state uninitialized or rent exempt post-transaction is not allowed.
         - If its size has changed or its balance has increased, it cannot remain rent paying.
   8. Extract log messages.
   9. Extract inner instructions (`Vec<Vec<InnerInstruction>>`).
   10. Extract `ExecutionRecord` components from transaction context.
   11. Check balances of accounts to match the sum of balances before
       transaction execution.
   12. Updated loaded transaction accounts to new accounts.
   13. Extract changes in accounts data sizes
   14. Extract return data
   15. Return `TransactionExecutionResult` with wrapping the extracted
       information in `TransactionExecutionDetails`.

6. Prepare the results of loading and executing transactions.

   This includes the following steps for each transactions
   1. dump flattened result to info log for an account which pubkey is
      in the transaction's debug keys (one per transaction, it seems).
   2. Collect logs of the transaction execution for each executed
      transaction, unless Bank's `transaction_log_collector_config` is
      set to `None`.
   3. Finally, increment various statistical counters, and update
      timings passed as a mutable reference to
      `load_and_execute_transactions` in arguments. The counters are
      packed in the struct `LoadAndExecuteTransactionsOutput`. What
      should we do with these counters?

The crate solana-program-runtime should be included in SVM
completely. It contains `ComputeBudget`, `InvokeContext`,
`MessageProcessor`, and other structs.

## Factoring out unnecessary dependencies

`TransactionBatchProcessor` is a new struct factored out of the `Bank` struct
to expose the dependencies between transaction processing and `Bank`.

A callback mechanism, implemented as the
`TransactionProcessingCallback` trait, is used to eliminate the
dependency on reward interval in SVM.  When transaction accounts are
loaded one of the checks performed on the accounts is to identify and
to block transactions that attempt to update stake accounts inside a
reward interval. The SVM should be agnostic to most Solana specific
restrictions, therefore the explicit check is abstracted by a call to
a `check_account_access` method of the new
`TransactionProcessingCallback` trait.

Epoch scheduling is only needed to find the first slot in the epoch,
and doesn't need to be a dependency of transaction batch processing.
The slot could be added as a parameter when
`TransactionBatchProcessor` is created.

Access to `accounts` field of the bank is replaced by the get_account()
method of `TransactionProcessingCallback` trait.

## Differences in the implementation of SVM that affect transaction execution

In this section we describe how the SVM implementation diverged from
the pre-SVM logic of transaction execution. For example, some changes
in the SVM code may affect the order in which errors of failing
transactions are reported. A transaction may fail for more than a
single reason. The error reporting for a failing transaction depends
on the order in which various checks on the transaction are
performed. Splitting the SVM out from the monorepo results in changes
to the order some checks are performed on transactions.  We explain
such and other cases of visible side-effects in this section.
(Currently no such changes have been done).

## Open Questions

1. Many types used to define (transitively) `TransactionBatch` are
   defined in solana-program crate. This crate is part of SDK
   (solana-sdk) and all Solana programs depend on it. Every use of SVM
   crate will have solana-program as a dependency. Assuming we have to
   keep solana-program separate from solana-svm, should we reexport
   solana-program in solana-svm? What is the correct Rust approach?

Questions from Lucas' document (to answer here later)

2. Revise the arguments of `load_and_execute_transactions`. It
   receives a `TransactionBatch`. Are we going to provide our own
   scheme for transactions? What about the other arguments, like
   `BlockHashQueue`, `AccountOverrides`? If defined in runtime, we’ll
   provide the structure in the SVM. If not, we’ll provide a trait.

3. `FeeStructure` needs an interface. Transactions perhaps? What
   happens if users do not follow our transaction scheme?

4. Does `RentState` need an interface? Do we assume all accounts will
   pay rent?

Questions about structure of SVM from Lucas' document (need to review and answer)

5. Parts that should belong to SVM: 1. Inputs: 1. Accounts 2. Transactions 3. Builtin programs 4. A trait for bank 2. Outputs: 1. Processed transactions

   Metrics: `TransactionErrorMetrics`, `ExecuteDetailsTimings`

   Result structs: `TransactionExecutionDetails` and `TransactionExecutionResult` declared in accounts-db should be part of SVM.

   `TransactionCheckResult` is a candidate for moving into runtime.

   Inner structs related to execution: `SanitizedMessage` may be moved to runtime and may belong in SVM. It is declared in solana_program::message and only used in transaction code.


6. Execution context data structures: `TransactionContext`,
   `InstructionContext` is defined in SDK, but is truly part of SVM or
   not?

7. Is the BpfLoader part of the SVM? It is a program on the blockchain, not necessarily part of the VM?


8. What is the purpose of `transaction_debug_keys` in Bank?
