# Shared Secrets Component

## Purpose

Manages time-sensitive cryptographic secrets and keys used across the network. This primarily involves managing the lifecycle of "daily nonce" keys (used for time-based proofs like election seeds) and organizing "origin keys" associated with different hardware/software sources.

## Key Files/Modules

*   `lib/archethic/shared_secrets.ex`: Main public API. Provides functions to get the daily nonce public key for a given time (`get_daily_nonce_public_key`), manage origin keys (`list_origin_public_keys`, `add_origin_public_key`), and interact with the renewal process.
*   `lib/archethic/shared_secrets/supervisor.ex`: Supervises the core shared secrets processes:
    *   `NetworkLookup`: In-memory table (ETS) mapping time points to daily nonce public keys.
    *   `OriginKeyLookup`: In-memory table (ETS) mapping origin families (`:software`, `:hardware`, etc.) to their associated public keys.
    *   `MemTablesLoader`: Loads data from `:node_shared_secrets` and `:origin` transactions into the lookup tables.
    *   `NodeRenewalScheduler`: Schedules the periodic creation of new `:node_shared_secrets` transactions.
*   `lib/archethic/shared_secrets/node_renewal.ex`: Logic for creating the `:node_shared_secrets` transaction, which updates the daily nonce keys/secrets.
*   `lib/archethic/shared_secrets/node_renewal_scheduler.ex`: Schedules the renewal process based on a cron interval and node election status.
*   `lib/archethic/shared_secrets/mem_tables/network_lookup.ex`: Implementation of the daily nonce lookup table.
*   `lib/archethic/shared_secrets/mem_tables/origin_key_lookup.ex`: Implementation of the origin key lookup table.
*   `lib/archethic/shared_secrets/mem_tables_loader.ex`: Parses transactions to populate the lookup tables.

## Core Functionality

*   **Daily Nonce Management**: Provides access to time-specific public keys (`get_daily_nonce_public_key`) derived from a chain of `:node_shared_secrets` transactions. These keys are crucial for creating verifiable, time-bound cryptographic proofs (e.g., election seeds).
*   **Node Shared Secret Renewal**: Periodically creates new `:node_shared_secrets` transactions (`NodeRenewalScheduler`, `NodeRenewal`), effectively rotating the secrets/keys used for daily nonces. This process is typically initiated by elected nodes.
*   **Origin Key Management**: Organizes and provides lookups for public keys based on their origin family (`OriginKeyLookup`). Derives seeds for these families (`get_origin_family_seed`).
*   **State Loading**: Loads the state of daily nonces and origin keys from the blockchain history (`MemTablesLoader`, called via `load_transaction`).
*   **Persistence**: Stores genesis addresses for relevant chains (`:node_shared_secrets`, `:origin`) in `:persistent_term` for faster lookups (`persist_gen_addr`, `genesis_address`).

## Dependencies

*   **Crypto**: Heavily relies on `Crypto` for key/address derivation (`derive_keypair`, `derive_address`), accessing the `storage_nonce`, encryption/decryption (`Crypto.encrypt`, `Crypto.decrypt`, `Crypto.unwrap_secrets`) within `NodeRenewal`, and getting origin info (`Crypto.key_origin`). Uses `Crypto.SharedSecretsKeystore` functions.
*   **TransactionChain**: Reads `:node_shared_secrets` and `:origin` transactions (`list_addresses_by_type`, `get_genesis_address`). Uses the `Transaction` struct.
*   **Election**: `NodeRenewalScheduler` checks if the current node is an elected validator (`is_validation_node?`) to determine if it should initiate the renewal.
*   **P2P**: `NodeRenewal` potentially uses `P2P.authorized_and_available_nodes` to get the list of nodes to encrypt renewed secrets for.
*   **DB**: Indirect dependency via `TransactionChain`.
*   **Utils**: For common utility functions, including date calculations.
*   **External**: `Crontab` library.

## Interactions with Other Components

*   Provides the daily nonce public key lookup (`get_daily_nonce_public_key`) used by `Crypto` (specifically `Crypto.sign_with_daily_nonce_key`) which is called by `Election`.
*   `NodeRenewal` module calls `Crypto` functions (`encrypt`, `decrypt`, `unwrap_secrets`, `storage_nonce`) likely operating via `Crypto.SharedSecretsKeystore`.
*   `NodeRenewalScheduler` depends on `Election` results (`is_validation_node?`).
*   `MemTablesLoader` (triggered by `load_transaction`) reads transaction history via `TransactionChain` (`list_addresses_by_type`).
*   `load_transaction` is called by `Replication.ingest_transaction` when `:node_shared_secrets` or `:origin` transactions are stored.
*   `NodeRenewalScheduler` submits the generated `:node_shared_secrets` transaction to `Mining`. 