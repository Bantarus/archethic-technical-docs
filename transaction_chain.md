# Transaction Chain Component

## Purpose

Handles the creation, validation, storage, and processing of transactions within the Archethic network. It provides the primary API for interacting with transaction data, often delegating persistence tasks to the DB component and network interactions to the P2P component.

## Key Files/Modules

*   `lib/archethic/transaction_chain.ex`: Core public API for managing the transaction chain. Delegates many functions to `DB` and `P2P`.
*   `lib/archethic/transaction_chain/supervisor.ex`: Supervises internal processes:
    *   `MemTables.PendingLedger`: Likely manages pending transactions or ledger state changes in memory.
    *   `DBLedger.Supervisor`: Supervises the ledger interaction logic built on top of the DB.
*   `lib/archethic/transaction_chain/transaction.ex`: Defines the main `Transaction` struct and its components (like `ValidationStamp`).
*   `lib/archethic/transaction_chain/transaction_input.ex`: Defines transaction inputs (UTXO model).
*   `lib/archethic/transaction_chain/transaction_summary.ex`: Defines a summarized transaction view.
*   `lib/archethic/transaction_chain/db_ledger.ex`: Abstraction layer for ledger operations interacting with the DB.
*   `lib/archethic/transaction_chain/mem_tables/`: Contains modules for in-memory state management (e.g., `pending_ledger.ex`).
*   `lib/archethic/transaction_chain/transaction/`: Contains sub-modules related to the `Transaction` struct (e.g., `validation_stamp.ex`, `transaction_data.ex`, `ledger_operations.ex`).
*   `lib/archethic/transaction_chain/mem_tables_loader.ex`: Loads data into memory tables during startup.
*   `lib/archethic/transaction_chain/versioned_*.ex`: Modules supporting different versions of transaction structures (e.g., `VersionedTransactionInput`).

## Core Functionality

*   Provides functions to `get` (from DB), `fetch` (from DB or Network), and `list` (from DB) transaction data.
*   Manages the lifecycle of transactions, including validation (partially, delegating ledger validation) and storage.
*   Interacts with the P2P network to request/send transaction information using defined P2P message types.
*   Handles transaction summaries.
*   Manages ledger state, potentially using both persistent storage (`DBLedger`) and in-memory tables (`PendingLedger`).
*   Retrieves chain metadata like genesis address, last address, chain size, etc.

## Dependencies

*   **DB**: Heavily relies on the `Archethic.DB` behaviour for storing and retrieving all persistent transaction and chain data (delegates many functions like `get_transaction`, `list_transactions`, `get_genesis_address`).
*   **P2P**: Essential for fetching data from other nodes (`fetch_transaction`, `fetch_genesis_address`, etc.) and broadcasting new transactions. Uses specific message types defined in `Archethic.P2P.Message` (e.g., `GetTransaction`, `GetTransactionChain`, `TransactionList`, `GetGenesisAddress`, `GenesisAddress`, `GetLastTransactionAddress`, `LastTransactionAddress`).
*   **Crypto**: Required for handling addresses, keys, signatures (`verify_signature`), and hashing within transactions.
*   **Mining**: Relies on `Mining.LedgerValidation` for validating ledger state changes associated with transactions (called within `TransactionValidator`). Also receives the finalized `ValidationStamp` from Mining.
*   **Election**: Likely involved in selecting nodes for fetching data (`fetch_*` functions).
*   **UTXO**: Provides `UTXO.get_balance` for validation checks. Its core logic (inputs, outputs, ledger state) is managed within this component's modules (`transaction_input.ex`, `db_ledger.ex`, `mem_tables/pending_ledger.ex`).
*   **Utils**: For common utility functions.

## Interactions with Other Components

*   Provides the primary interface for most other components (`Mining`, `Contracts`, `Replication`, `OracleChain`, etc.) to access transaction and chain data (e.g., `get_transaction`, `get_last_transaction`, `get_genesis_address`).
*   Receives new transactions to potentially process or validate, likely via `P2P` handlers or internal calls.
*   Initiates P2P requests using specific `Archethic.P2P.Message` types to fetch data.
*   Uses the `DB` component extensively for persistence.
*   Provides transaction data (`Transaction` struct) to `Mining` for validation.
*   Receives the completed `ValidationStamp` from `Mining` and incorporates it into the `Transaction` struct before it's passed to `Replication`.
*   Provides necessary data (like previous transaction) to `Replication` for its validation step. 