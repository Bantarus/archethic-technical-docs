# Nominal Bootstrapping Scenario (Joining Node)

This document outlines the typical code execution flow when a node joins (or rejoins) an already existing Archethic network. It assumes the node is not the very first node initializing the network.

1.  **Application Start (`Application.start/2`)**
    *   The main `Archethic.Application` supervisor starts its child processes according to its defined strategy.
    *   It explicitly starts the `Archethic.Bootstrap` component as a `Task` process, passing necessary configuration (ports, transport, reward address).

2.  **Bootstrap Task Initialization (`Bootstrap.start_link/1`)**
    *   Retrieves node's public IP via `Networking.get_node_ip()`.
    *   Fetches the list of configured seed nodes via `P2P.list_bootstrapping_seeds()`.
    *   Gets the timestamp of the last successful sync via `SelfRepair.last_sync_date()` (will be `nil` if this node identity is new).
    *   Gathers node crypto keys (`Crypto.first_node_public_key`).
    *   Spawns the bootstrap task, calling `Bootstrap.run/7` with the collected info.

3.  **Bootstrap Task Execution (`Bootstrap.run/7`)**

    *   **Seed Node Connection & Discovery:**
        *   Determines node's geo-patch (`P2P.get_geo_patch`).
        *   Connects to seed nodes (`P2P.connect_nodes`).
        *   Discovers closest actual peers via seed nodes (`Bootstrap.Sync.get_closest_nodes_and_renew_seeds`, using P2P messages).
    *   **Determine Bootstrap Action:**
        *   Calls `Bootstrap.should_bootstrap?/5` to check if node identity needs creation/update.
        *   Checks include: `last_sync_date` is `nil`? Node info missing in `P2P.MemTable`? Config changed since `last_sync_date`?.
        *   Assuming `true` (node needs to create/update its `:node` transaction).
    *   **Execute Node Creation/Update (`Bootstrap.start_bootstrap/6`):**
        *   Skips network initialization (`Bootstrap.Sync.should_initialize_network?` is false).
        *   Calls either `Bootstrap.Sync.first_initialization` (if new identity) or `Bootstrap.Sync.update_node` (if config changed).
        *   Both paths use `Bootstrap.TransactionHandler.create_node_transaction` to build the appropriate `:node` transaction.
        *   **Crucially**: This new `:node` transaction must be submitted to the network for validation via the standard `Mining` process (this submission step is implicit).
    *   **Post-Bootstrap Synchronization & Finalization (`Bootstrap.post_bootstrap/1`):**
        *   Checks if sync is needed (`SelfRepair.missed_sync?`). Assuming `true`.
        *   Connects to current discovered nodes (`Bootstrap.Sync.connect_current_node`).
        *   **Calls `SelfRepair.bootstrap_sync(current_nodes)` to perform the main state synchronization** (details below).
        *   *(Waits for `SelfRepair.bootstrap_sync` to complete)*
        *   Connects to all known nodes (`P2P.connect_nodes`).
        *   Persists network constraints genesis address (`Bootstrap.NetworkConstraints.persist_genesis_address`).
        *   If authorized/available (`P2P.authorized_and_available_node?`), performs final mini-sync (`Bootstrap.Sync.sync_current_summary`).
        *   Signals readiness by calling `BeaconChain.add_end_of_node_sync()`.
        *   Starts the ongoing repair scheduler via `SelfRepair.start_scheduler()`.
        *   Bootstrap task finishes.

4.  **Self Repair Sync Execution (`SelfRepair.bootstrap_sync` -> `SelfRepair.Sync.load_missed_transactions`)**
    *   Calculates time range from `last_sync_date` (or network epoch if nil) to now.
    *   Identifies required `BeaconChain` summary times in the range (`BeaconChain.previous_summary_dates`).
    *   Loops through each required summary time:
        *   Fetches `SummaryAggregate` from peers (`P2P.request` -> `GetBeaconSummariesAggregate`).
        *   Processes aggregate, fetching individual `Summary` structs as needed (`P2P.request` -> `GetBeaconSummaries`).
        *   For each `Summary`:
            *   Extracts `TransactionSummary` list.
            *   For each transaction summary:
                *   Fetches the full transaction (`TransactionChain.fetch_transaction`, using P2P `GetTransaction`).
                *   Checks if local node should store it (`Election.chain_storage_node?`).
                *   If yes, calls `Replication.validate_and_store_transaction`:
                    *   Validates transaction (`Replication.TransactionValidator`).
                    *   Stores transaction (`TransactionChain.write_transaction`).
                    *   Ingests transaction state (`Replication.ingest_transaction` -> updates `UTXO`, `SharedSecrets`, `Governance`, etc.).
        *   Updates local network view based on summary data (`SelfRepair.NetworkView.process_p2p_view`).
    *   Updates and persists the `last_sync_date` (`SelfRepair.Sync.store_last_sync_date`).
    *   Returns `:ok` to `Bootstrap.post_bootstrap` upon completion.

The node is now bootstrapped, synchronized, and participating in the network. 