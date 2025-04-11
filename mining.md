# Mining Component

## Purpose

Orchestrates the transaction validation and consensus process (referred to as "mining" although not necessarily PoW-based for consensus itself). It ensures transactions are valid according to network rules, ledger state, and smart contract logic before they are finalized with a `ValidationStamp`.

## Key Files/Modules

*   `lib/archethic/mining.ex`: Main entry point API. Initiates validation workflows (`start/5`), gets validators (`get_validation_nodes`), verifies elections (`valid_election?`), and requests chain locks (`request_chain_lock`).
*   `lib/archethic/mining/supervisor.ex`: Supervises the core mining infrastructure:
    *   `WorkflowRegistry`: Tracks active validation workflows.
    *   `WorkerSupervisor`: Dynamically starts workflow processes.
    *   `ChainLockSupervisor`: Manages partitioned `ChainLock` processes for preventing concurrent validation attempts.
*   `lib/archethic/mining/distributed_workflow.ex`: Implements the complex logic for multi-node validation. It likely involves steps like welcome node coordinating, validators performing checks, exchanging context, cross-validating results, aggregating signatures/stamps, and finalizing the `ValidationStamp`.
*   `lib/archethic/mining/standalone_workflow.ex`: Implements a simpler validation process for single-validator scenarios.
*   `lib/archethic/mining/ledger_validation.ex`: Performs critical validation against the ledger state (UTXO checks, double-spend prevention, balance calculation). Generates the necessary `LedgerOperations` for the `ValidationStamp`.
*   `lib/archethic/mining/pending_transaction_validation.ex`: Performs initial stateless transaction checks (signatures, format, PoW check).
*   `lib/archethic/mining/smart_contract_validation.ex`: Handles the specific validation logic for smart contract execution, including running contract code and verifying state transitions.
*   `lib/archethic/mining/chain_lock.ex`: Implements the process logic for locking transaction chains during validation.
*   `lib/archethic/mining/fee.ex`: Calculates transaction fees.
*   `lib/archethic/mining/proof_of_work.ex`: Implements PoW checks, likely used for transaction admission control/anti-spam.
*   `lib/archethic/mining/error.ex`: Defines specific error types for validation failures.

## Core Functionality

*   **Workflow Orchestration**: Manages the end-to-end validation process for transactions, choosing between `StandaloneWorkflow` and `DistributedWorkflow`.
*   **Consensus (Distributed Workflow)**: Coordinates multiple elected validators to reach agreement on a transaction's validity and the resulting ledger changes. Involves context sharing, cross-validation, and aggregation of signatures (`CrossValidationStamp`) into the final `ValidationStamp`.
*   **Ledger Validation**: Verifies transaction inputs against the current UTXO set, ensures no double spends, calculates resulting outputs and state changes (`LedgerValidation`).
*   **Stateless Validation**: Performs initial checks on transaction structure, signatures, and potentially proof-of-work (`PendingTransactionValidation`).
*   **Smart Contract Validation**: Executes contract code and validates resulting state changes (`SmartContractValidation`).
*   **Chain Locking**: Prevents concurrent processing of transactions affecting the same chain using a distributed locking mechanism (`request_chain_lock`, `ChainLock`).
*   **Fee Calculation**: Determines the appropriate fee for a transaction (`Fee`).

## Dependencies

*   **TransactionChain**: Central dependency for transaction data (`Transaction` struct), access to previous transactions (`get_last_transaction`), ledger state (`DBLedger`, `PendingLedger`), and transaction structure definitions (`ValidationStamp`, `CrossValidationStamp`, `LedgerOperations`).
*   **Election**: Uses `Election.get_validation_nodes` to determine validators and `Election.storage_nodes` for chain locking.
*   **P2P**: Used extensively for communication in the `DistributedWorkflow` (exchanging context, stamps via `P2P.send_message`) and for sending `RequestChainLock` messages via `P2P.send_message`.
*   **Crypto**: Required for verifying signatures (`verify_transaction`, `verify_signatures`), hashing data (`Crypto.hash`), and node identities.
*   **DB**: Indirect dependency via `TransactionChain` for ledger state and potentially `Contracts` for contract state.
*   **UTXO**: `LedgerValidation` implicitly uses UTXO concepts. `SmartContractValidation` may query `UTXO.get_balance`.
*   **Contracts**: `SmartContractValidation` calls `Contracts.execute_trigger` to execute contract code.
*   **Utils**: For common utility functions.

## Interactions with Other Components

*   Receives transactions to validate, likely from `P2P` (via a handler handling `Transaction` messages) or `Contracts` (if a contract execution creates a new transaction), or `Reward`/`OracleChain`/`SharedSecrets` (submitting scheduled txns).
*   Calls `Election.get_validation_nodes` and `Election.storage_nodes`.
*   Communicates heavily with peers via `P2P` (`DistributedWorkflow`, `request_chain_lock`).
*   Interacts extensively with `TransactionChain` for transaction data and ledger state.
*   Calls `Contracts.execute_trigger` for smart contract validation.
*   Produces the final `ValidationStamp` which is incorporated into the `Transaction` struct by `TransactionChain` before being passed to `Replication`.
*   Potentially passes validated transaction data to `Replication` to initiate the storage process. 