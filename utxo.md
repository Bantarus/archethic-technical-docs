# UTXO Component

## Purpose

Manages the **in-memory representation** of the Unspent Transaction Output (UTXO) set for efficient querying and balance calculation. It processes persisted transactions to update this in-memory view.

**Note**: Core UTXO *validation* logic resides in the `Mining` component (`LedgerValidation`), and the initial database *ingestion* trigger happens in the `Replication` component (`Replication.ingest_transaction` calls `UTXO.load_transaction`).

## Key Files/Modules

*   `lib/archethic/utxo.ex`: Provides the `load_transaction/3` function (called by `Replication`) to process a persisted transaction and update the in-memory ledger. Also provides `get_balance/1`.
*   `lib/archethic/utxo/supervisor.ex`: Supervises:
    *   `MemoryLedger`: The GenServer/ETS process managing the in-memory UTXO data.
    *   `LoaderSupervisor`: Manages `Loader` worker processes.
    *   `DBLedgerSupervisor`: *Note: Also supervised by `TransactionChain.Supervisor`. Role here might be related to specific read patterns or loading.*)
*   `lib/archethic/utxo/memory_ledger.ex`: Implements the in-memory UTXO store (ETS), providing functions to get, add, and consume UTXOs, and calculate balances.
*   `lib/archethic/utxo/loader.ex`: Worker process logic that applies calculated changes (additions/consumptions) to the `MemoryLedger`.
*   `lib/archethic/utxo/db_ledger.ex`: Minimal file, likely defines shared types related to DB ledger representation.

## Core Functionality

*   **In-Memory UTXO Management**: Maintains an up-to-date, in-memory view (`MemoryLedger`) of UTXOs relevant to the node (based on storage election).
*   **State Loading (`load_transaction`)**: Processes validated and stored transactions to:
    *   Identify UTXOs created by the transaction (`Transaction.ValidationStamp.LedgerOperations`).
    *   Determine if the current node should store these UTXOs based on `Election`.
    *   Verify that UTXOs haven't already been consumed by checking subsequent chain history (`utxo_consumed?`).
    *   Add valid, relevant UTXOs to `MemoryLedger` via the `Loader`.
    *   Consume the transaction's inputs in `MemoryLedger` via the `Loader`.
*   **Balance Calculation**: Provides `get_balance/1` to efficiently query the UCO and token balance for an address using the `MemoryLedger`.

## Dependencies

*   **TransactionChain**: Crucial for reading transaction history (`get_transaction`, `list_chain_addresses`), getting chain state (`get_last_address`), fetching genesis addresses (`fetch_genesis_address`), and using transaction data structures (`Transaction`, `UnspentOutput`, `VersionedUnspentOutput`, `LedgerOperations`).
*   **Election**: Determines storage node responsibility (`Election.chain_storage_node?`) for filtering which UTXOs to load into memory.
*   **Crypto**: For node identity (`Crypto.first_node_public_key`) and addresses.
*   **P2P**: Indirectly used for fetching genesis addresses (`TransactionChain.fetch_genesis_address`). Provides node list for `Election`.
*   **DB**: Indirect dependency via `TransactionChain`.

## Interactions with Other Components

*   `load_transaction` is called by `Replication.ingest_transaction` after a transaction is successfully stored.
*   Provides balance information (`get_balance`) used by `TransactionChain` (validation), `Mining` (`SmartContractValidation`), and `Contracts`.
*   Uses `Election.chain_storage_node?` results to filter relevant UTXOs.
*   Reads chain data via `TransactionChain` (`get_transaction`, `list_chain_addresses`, etc.).
*   Updates its internal state (`MemoryLedger`) via `Loader` workers.