# DB Component

## Purpose

Manages database interactions, providing an abstraction layer for storing and retrieving node data. It utilizes a custom embedded implementation by default, using files for persistent storage (chains, summaries) and likely in-memory structures for fast lookups and indexing.

## Key Files/Modules

*   `lib/archethic/db.ex`: Defines the `Archethic.DB` behaviour (interface) for all database operations. Uses `Knigge` to delegate calls to a configured implementation.
*   `lib/archethic/db/supervisor.ex`: Supervises the chosen database implementation process.
*   `lib/archethic/db/embedded_impl.ex`: The default database implementation (`Archethic.DB.EmbeddedImpl`). It implements the `Archethic.DB` behaviour.
    *   Relies on helper modules within `lib/archethic/db/embedded_impl/` (like `ChainReader`, `ChainWriter`, `ChainIndex`) for specific tasks.
    *   Manages data persistence primarily through files located in a path determined by `Utils.mut_dir()`.
    *   Handles transactions, beacon chain summaries/aggregates, statistics, P2P summaries, and bootstrap information.
*   `lib/archethic/db/embedded_impl/`: Contains helper modules for the embedded implementation:
    *   `chain_index.ex`: Likely manages in-memory indices for fast lookups.
    *   `chain_reader.ex`: Handles reading data from files.
    *   `chain_writer.ex`: Handles writing data to files.
    *   `bootstrap_info.ex`, `p2p_view.ex`, `stats_info.ex`: Likely manage storage/retrieval for their respective data types.

## Core Functionality

*   Provides functions (defined as callbacks in `db.ex`) to get, write, and check the existence of transactions.
*   Manages storage and retrieval of Beacon Chain Summaries and Aggregates.
*   Allows querying transaction chains with pagination and filtering options.
*   Indexes transactions by type and address for efficient querying.
*   Tracks chain metadata like size, last address, and public keys.
*   Stores node statistics (TPS, burned fees) and P2P availability summaries.
*   Persists bootstrap node information.

## Dependencies

*   **Crypto**: Uses types for addresses and keys (`Archethic.Crypto.prepended_hash`, `Archethic.Crypto.key`) in function specs and potentially internally.
*   **BeaconChain**: Defines and uses `Archethic.BeaconChain.Summary` and `Archethic.BeaconChain.SummaryAggregate` structs for storage.
*   **TransactionChain**: Defines and uses the `Archethic.TransactionChain.Transaction` struct for storage.
*   **Utils**: Uses `Archethic.Utils.mut_dir()` for determining storage path.

## Interactions with Other Components

*   Provides the persistence layer accessed by multiple components via the functions defined in the `Archethic.DB` behaviour.
*   **TransactionChain**: Heavily delegates read/write operations (e.g., `get_transaction`, `list_transactions`, `get_last_chain_address`) to `Archethic.DB`.
*   **BeaconChain**: Calls `write_beacon_summary`, `get_beacon_summary`, `get_beacon_summaries_aggregate`.
*   **P2P**: `MemTableLoader` calls `get_last_p2p_summaries`.
*   **Bootstrap**: Calls `get_bootstrap_info`, `set_bootstrap_info`.
*   Other components requiring persistent data likely interact via these defined behaviours. 