# Oracle Chain Component

## Purpose

Manages the process of fetching external data (e.g., cryptocurrency prices like UCO/USD, UCO/EUR) from configured sources, validating it, and recording it onto the blockchain via dedicated `:oracle` and `:oracle_summary` transactions. This makes trusted external data available for use by other components.

## Key Files/Modules

*   `lib/archethic/oracle_chain.ex`: Main interface. Provides functions to validate oracle content (`valid_services_content?`, `valid_summary?`), load oracle transactions (`load_transaction`), and retrieve oracle data (`get_uco_price`, `get_oracle_data`).
*   `lib/archethic/oracle_chain/supervisor.ex`: Supervises:
    *   `MemTable`: In-memory store (ETS) for validated oracle data, keyed by service and date.
    *   `MemTableLoader`: Loads data from oracle transactions into `MemTable`.
    *   `Scheduler`: Periodically fetches external data and creates/submits oracle transactions.
    *   `ServiceCacheSupervisor`: Supervises caching processes for external data fetches.
*   `lib/archethic/oracle_chain/scheduler.ex`: GenServer that periodically:
    *   Polls external data sources (`Services`) based on `polling_interval`.
    *   Creates `:oracle` transactions with fetched data.
    *   Aggregates recent `:oracle` data into `:oracle_summary` transactions based on `summary_interval`.
    *   Submits transactions for mining (likely if elected).
*   `lib/archethic/oracle_chain/services.ex` & `lib/archethic/oracle_chain/services/`: Define interfaces and implementations for fetching data from specific external APIs (e.g., CoinGecko).
*   `lib/archethic/oracle_chain/summary.ex`: Defines the structure and verification logic for `:oracle_summary` data.
*   `lib/archethic/oracle_chain/mem_table.ex`: Implementation of the in-memory oracle data store.
*   `lib/archethic/oracle_chain/mem_table_loader.ex`: Parses `:oracle` and `:oracle_summary` transactions to populate `MemTable`.

## Core Functionality

*   **External Data Fetching**: Periodically fetches data from configured external APIs via modules in `services/`.
*   **Oracle Transaction Creation**: Creates `:oracle` transactions containing fetched data and `:oracle_summary` transactions aggregating recent data.
*   **Scheduling & Submission**: Uses `Scheduler` (cron-based) to manage polling/summarization intervals and submits transactions for mining, likely coordinated by node election status.
*   **Data Validation**: Validates incoming oracle data against external sources (`valid_services_content?`) and internal consistency (`valid_summary?`).
*   **State Management**: Stores validated oracle data in an efficient in-memory table (`MemTable`) loaded from the chain (`MemTableLoader` via `load_transaction`).
*   **Data Provision**: Provides functions (`get_uco_price`, `get_oracle_data`) for other components to easily access on-chain oracle data.

## Dependencies

*   **Crypto**: Used for deriving oracle chain addresses (`Crypto.derive_oracle_address`).
*   **TransactionChain**: Reads oracle transactions (`:oracle`, `:oracle_summary`) to load state (`MemTableLoader`) and define transaction structures.
*   **Mining**: `Scheduler` submits `:oracle` and `:oracle_summary` transactions to `Mining`.
*   **Election**: `Scheduler` likely checks election status (`is_validation_node?`) to coordinate transaction submission.
*   **P2P**: Potentially used by `Scheduler` for fetching/coordination if direct API access fails or for election checks.
*   **DB**: Indirect dependency via `TransactionChain`.
*   **Utils**: For common utility functions (date calculations).
*   **External**: `Jason` (JSON), HTTP client library (e.g., `HTTPoison`), `Crontab`.

## Interactions with Other Components

*   Provides oracle data (e.g., `get_uco_price`) to components like `Reward` (`validation_nodes_reward`).
*   `Scheduler` submits transactions to `Mining`.
*   `Scheduler` depends on `Election` results (`is_validation_node?`).
*   `load_transaction` is called by `Replication.ingest_transaction` when `:oracle` and `:oracle_summary` transactions are stored, updating `MemTable` via `MemTableLoader`.
*   Fetches data from external APIs via `Services` modules. 