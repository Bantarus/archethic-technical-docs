# Replication Component

## Purpose

Ensures that validated transactions are reliably stored on the designated storage nodes and integrated into the node's local state (UTXO set, caches, other component states). It acts as the final step after consensus/mining.

## Key Files/Modules

*   `lib/archethic/replication.ex`: Main interface. Provides functions to validate transactions upon reception (`validate_transaction`), temporarily pool them (`add_transaction_to_commit_pool`), sync/write chains (`sync_transaction_chain`), validate and store directly (`validate_and_store_transaction`), and ingest into local state (`ingest_transaction`).
*   `lib/archethic/replication/supervisor.ex`: Supervises the `TransactionPool`.
*   `lib/archethic/replication/transaction_pool.ex`: A temporary holding pool (likely ETS) for validated transactions awaiting final write/commit.
*   `lib/archethic/replication/transaction_validator.ex`: Re-validates a transaction against the node's current view of the ledger state before storage.
*   `lib/archethic/replication/transaction_context.ex`: Fetches context (previous transaction, inputs) needed for validation.

## Core Functionality

*   **Final Validation**: Storage nodes perform a final validation (`validate_transaction`, `TransactionValidator`) on transactions received after consensus to ensure consistency with their local state.
*   **Transaction Pooling**: Temporarily holds validated transactions (`TransactionPool`) before they are written to persistent storage.
*   **Chain Synchronization & Storage**: Writes transaction chains to persistent storage (`sync_transaction_chain`, `TransactionChain.write_transaction`). Can handle different storage types (`:chain`, `:io`).
*   **State Ingestion (`ingest_transaction`)**: Integrates a stored transaction into the node's local state:
    *   Updates the UTXO set (`UTXO.load_transaction`).
    *   Loads network-specific transactions into relevant components (`SharedSecrets.load_transaction`, `Governance.load_transaction`, `OracleChain.load_transaction`, `Reward.load_transaction`, `Contracts.load_transaction`).
    *   Updates P2P information (`P2P.authorize_node`, `P2P.set_node_globally_synced`, `P2P.update_node_network_patch` via `NetworkView`).
    *   Notifies other storage nodes about the last transaction address for the chain (`notify_last_transaction_address`).
    *   Generates and sends replication attestations (likely to `BeaconChain`).
*   **Role-Based Logic**: Differentiates actions based on the node's role (chain storage, IO storage) determined via `Election`.
*   **Local Notification**: Notifies other local components of newly stored transactions via `PubSub`.

## Dependencies

*   **TransactionChain**: Essential for reading previous transactions (`fetch_context`), checking existence (`transaction_exists?`), writing transactions (`write_transaction`), getting genesis addresses, and using `Transaction` structs.
*   **Mining**: Uses `Mining.ValidationContext` and `Mining.Error` types. `TransactionValidator` likely reuses logic from `Mining` validators (`LedgerValidation`, `PendingTransactionValidation`, `SmartContractValidation`).
*   **P2P**: Used to fetch transaction context (`fetch_context`) and notify peers (`NotifyLastTransactionAddress` via `P2P.send_message`). Uses `P2P.Node` list and updates node state (`P2P.authorize_node`, etc.).
*   **Election**: Determines storage node responsibility (`Election.chain_storage_node?`, `Election.storage_nodes`).
*   **DB**: Indirect dependency via `TransactionChain`.
*   **UTXO**: Calls `UTXO.load_transaction` to update the in-memory UTXO set after storage.
*   **Crypto**: For addresses and validation checks (`Crypto.verify_chain`).
*   **BeaconChain**: Likely sends `ReplicationAttestation` data to `BeaconChain` after successful storage.
*   **SharedSecrets, Governance, OracleChain, Reward, Contracts**: Calls their respective `load_transaction` functions during ingestion.
*   **PubSub**: For local event notification (`notify_new_transaction`).
*   **Utils**: For common utility functions.

## Interactions with Other Components

*   Receives validated transaction data, likely including the `ValidationStamp`, from the `Mining` component or via `P2P` message handlers.
*   Calls `Election.chain_storage_node?` to determine storage responsibilities.
*   Calls `TransactionChain.write_transaction` for persistence.
*   Calls `UTXO.load_transaction` to update the in-memory UTXO state.
*   Calls `load_transaction` on `SharedSecrets`, `Governance`, `OracleChain`, `Reward`, `Contracts` for network-specific transaction types.
*   Updates node status in `P2P` (`authorize_node`, etc.).
*   Communicates with peers via `P2P` (`notify_last_transaction_address`).
*   Sends `ReplicationAttestation` data to the `BeaconChain` component. 