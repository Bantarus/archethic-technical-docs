# Self Repair Component

## Purpose

Ensures the node stays synchronized with the network state by periodically detecting and repairing inconsistencies or missing data. It handles both initial synchronization during bootstrap and ongoing scheduled repairs.

## Key Files/Modules

*   `lib/archethic/self_repair.ex`: Main interface. Provides functions to trigger initial sync (`bootstrap_sync`), check if sync is needed (`missed_sync?`), start the repair scheduler (`start_scheduler`), update specific chains (`update_last_address`), trigger repairs (`resync`), and replicate individual transactions (`replicate_transaction`).
*   `lib/archethic/self_repair/supervisor.ex`: Supervises:
    *   `Scheduler`: Periodically triggers the main synchronization process.
    *   `NotifierSupervisor`: Dynamically starts `Notifier` processes when node availability changes.
    *   `RepairRegistry`: Used for locking during chain resynchronization.
    *   `NetworkView`: Maintains the local understanding of the network state.
*   `lib/archethic/self_repair/scheduler.ex`: GenServer that periodically triggers `Sync.synchronize` based on a cron interval.
*   `lib/archethic/self_repair/sync.ex`: Core synchronization logic. Fetches missed beacon summaries/aggregates (`load_missed_transactions`), compares local vs network state, identifies missing data, and triggers repair/replication actions. Persists the `last_sync_date`.
*   `lib/archethic/self_repair/repair_worker.ex`: Asynchronous worker process (managed by `RepairRegistry`) that handles fetching and replicating missing transactions for specific chains.
*   `lib/archethic/self_repair/notifier.ex`: Handles changes in the set of available nodes, potentially updating P2P status.
*   `lib/archethic/self_repair/network_view.ex`: Manages the node's view of the network state, likely updated based on BeaconChain data.
*   `lib/archethic/self_repair/network_chain.ex`: Handles synchronization logic specific to network-level chains (Governance, SharedSecrets, etc.).

## Core Functionality

*   **Bootstrap Synchronization (`bootstrap_sync`)**: Fetches and processes all necessary data (beacon summaries, transactions) missed since the node was last online, ensuring it catches up upon startup.
*   **Scheduled Synchronization (`Scheduler`, `Sync.synchronize`)**: Periodically compares the local state (using `NetworkView` and `last_sync_date`) against the perceived network state (likely from `BeaconChain`), identifies discrepancies, and triggers necessary repair actions.
*   **Data Fetching**: Retrieves missing beacon data and transactions from peers via `P2P`.
*   **Repair Triggering**: Initiates repair processes (`RepairWorker`, `replicate_transaction`) for missing data.
*   **State Integration**: Uses the `Replication` component to validate, store, and ingest repaired data.
*   **Chain Locking**: Uses `RepairRegistry` to prevent concurrent repairs on the same chain.
*   **Availability Monitoring**: Detects changes in node availability after sync cycles and triggers `Notifier`.
*   **Persistence**: Manages the `last_sync_date` marker.

## Dependencies

*   **BeaconChain**: Provides reference points (`BeaconChain.previous_summary_time`, `BeaconChain.next_summary_date`). `Sync` likely fetches `Summary` and `SummaryAggregate` via P2P calls triggered by `BeaconChain`'s interface. Provides `BeaconChain.add_end_of_node_sync`.
*   **TransactionChain**: Used to fetch transactions (`TransactionChain.fetch_transaction`), get chain state (`get_last_address`), fetch addresses (`TransactionChain.fetch_next_chain_addresses`), register addresses (`TransactionChain.register_last_address`), and get genesis addresses.
*   **Replication**: Calls `Replication.validate_and_store_transaction` to ingest repaired data.
*   **P2P**: Essential for fetching missing data (`GetTransaction`, `GetBeaconSummaries`, etc.) via `P2P.request`/`P2P.quorum_read`. Uses `P2P.Node` list. `Notifier` potentially updates node status via `P2P` functions.
*   **Crypto**: For node identity (`Crypto.first_node_public_key`) and addresses.
*   **Election**: Uses `Election.storage_nodes` to select appropriate peers for data fetching.
*   **DB**: Indirect dependency via `TransactionChain` and `BeaconChain`.
*   **Utils**: For common utility functions (`key_in_node_list?`).
*   **External**: Likely `Crontab` for the scheduler.

## Interactions with Other Components

*   Called by `Bootstrap.run` (`bootstrap_sync`).
*   Periodically triggers synchronization via `Scheduler`.
*   Reads time/sync markers from `BeaconChain` (`previous_summary_time`, `next_summary_date`).
*   Fetches data via `P2P` requests (likely using message types defined by `TransactionChain` and `BeaconChain`).
*   Writes repaired data via `Replication.validate_and_store_transaction`.
*   Uses `Election.storage_nodes` to select peers for fetching.
*   `Notifier` or `Sync` may update node status in `P2P` (`set_node_globally_available`, etc.).
*   Calls `BeaconChain.add_end_of_node_sync` upon completion of sync cycles. 