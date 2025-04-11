# Election Component

## Purpose

Manages the selection of nodes for specific network roles, primarily **validators** for new transactions and **storage nodes** for transaction chains. It uses deterministic, verifiable, and constraint-based algorithms to ensure fairness, security, and geographical distribution.

## Key Files/Modules

*   `lib/archethic/election.ex`: Main public API for performing node elections.
    *   `validation_nodes/5`: Elects validator nodes for a given transaction.
    *   `storage_nodes/4`: Elects storage nodes for a given transaction chain (identified by genesis address).
    *   `is_storage_node?/3`: Checks if a given node is elected as a storage node for a chain.
*   `lib/archethic/election/supervisor.ex`: Supervises helper modules (`Constraints`, `HypergeometricDistribution`).
*   `lib/archethic/election/constraints.ex`: Defines the behaviour and common logic for applying election constraints.
*   `lib/archethic/election/constraints/validation_constraints.ex`: Defines constraints specific to validator election (number required, minimum geographical spread).
*   `lib/archethic/election/constraints/storage_constraints.ex`: Defines constraints specific to storage node election.
*   `lib/archethic/election/hypergeometric_distribution.ex`: Provides functions for calculating probabilities related to node selection (e.g., failure probability based on dishonest nodes), using the hypergeometric distribution.

## Core Functionality

*   **Validator Election**: Selects a subset of authorized nodes to validate a specific transaction.
    *   Uses a deterministic sorting mechanism based on rotating node public keys combined with a verifiable `sorting_seed` (derived from the transaction hash and signed using a daily nonce key).
    *   Applies `ValidationConstraints` (number of validators, geographical distribution), prioritizing nodes not already storing the chain and from diverse geographical zones.
*   **Storage Election**: Selects a subset of authorized nodes to store a specific transaction chain.
    *   Uses a deterministic sorting mechanism based on rotating node public keys combined with the chain's genesis address.
    *   Applies `StorageConstraints` (number of storage nodes, geographical distribution).
*   **Verifiability**: The use of a signed `sorting_seed` for validation elections allows any node to verify the election outcome.
*   **Probabilistic Analysis**: Uses `HypergeometricDistribution` to model and calculate probabilities related to the security assumptions of the election process (e.g., likelihood of electing a minimum number of honest nodes).

## Dependencies

*   **Crypto**: Essential for hashing (transaction data for seed), node identities (keys for sorting), and signing (`sign_with_daily_nonce_key`) / verifying (`verify?`) the daily nonce proof for validator election seeds.
*   **P2P**: Relies on node information (`P2P.Node` struct containing `last_public_key`, `geo_patch`) provided by `P2P.authorized_and_available_nodes` as the pool of candidates.
*   **TransactionChain**: Uses the `Transaction` struct (`address`, `type`, content for hashing) as input for validator elections.
*   **SharedSecrets**: Implicitly depends on the daily nonce key (obtained via `Crypto.sign_with_daily_nonce_key` which likely queries `SharedSecrets` or `Crypto.SharedSecretsKeystore`) for signing the validation election proof.
*   **Utils**: For common utility functions (e.g., `key_in_node_list?`).

## Interactions with Other Components

*   **Mining**: Calls `Election.get_validation_nodes` to determine validators for a transaction. Uses `Election.storage_nodes` for chain lock requests.
*   **Replication**: Calls `Election.chain_storage_node?` to determine if the local node should store a transaction.
*   **UTXO**: Calls `Election.chain_storage_node?` to determine if UTXOs should be added to the local in-memory ledger.
*   **SharedSecrets**: `NodeRenewalScheduler` calls `Election.get_validation_constraints` (indirectly via `is_validation_node?`).
*   **SelfRepair**: Calls `Election.storage_nodes` to find peers for data fetching.
*   Relies on `P2P` to provide the pool of candidate nodes (`P2P.authorized_and_available_nodes`).
*   Uses `Crypto` for signing/verification involving the daily nonce key (managed by `SharedSecrets`). 