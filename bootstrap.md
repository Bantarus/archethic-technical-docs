# Bootstrap Component

## Purpose

Orchestrates the initial startup sequence of the node. It handles network initialization (if the node is the first in the network), creates or updates the node's own identity transaction (`:node` type), connects to seed nodes, and triggers the main synchronization process via the Self Repair component.

## Key Files/Modules

*   `lib/archethic/bootstrap.ex`: Main module implementing the bootstrap logic as a `Task`. Orchestrates the `run/7` sequence.
*   `lib/archethic/bootstrap/network_init.ex`: Contains logic executed only by the *first node* to create and self-validate genesis transactions for all core network chains (node identity, shared secrets, governance, reward, oracle, network constraints).
*   `lib/archethic/bootstrap/sync.ex`: Provides helper functions for determining bootstrap actions (initialize network?, update node?), finding/connecting to seed nodes, and creating/updating the current node's identity transaction chain.
*   `lib/archethic/bootstrap/transaction_handler.ex`: Helpers to create `:node` transactions and self-validate/replicate genesis transactions during network initialization.
*   `lib/archethic/bootstrap/network_constraints.ex`: Handles the genesis transaction for network-wide constraints.

## Core Functionality

*   **Startup Orchestration**: Manages the sequence of actions when a node starts.
*   **Network Initialization**: If identified as the first node, creates and self-stores genesis transactions for all essential network services (`NetworkInit`).
*   **Node Identity Management**: Creates or updates the node's own `:node` transaction chain to reflect its current configuration (IP, port, keys, reward address) (`Sync`, `TransactionHandler`).
*   **Seed Node Connection**: Connects to configured bootstrapping seed nodes to get an initial network view (`P2P`, `Sync`).
*   **Synchronization Trigger**: Determines if synchronization is needed (`SelfRepair.missed_sync?`) and triggers the main sync process (`SelfRepair.bootstrap_sync`).
*   **Post-Sync Finalization**: Connects to the full set of known nodes and signals readiness to the `BeaconChain` (`BeaconChain.add_end_of_node_sync`).
*   **Scheduler Startup**: Starts the `SelfRepair.Scheduler` for ongoing synchronization.

## Dependencies

*   **Networking**: Gets the node's public IP (`Networking.get_node_ip`).
*   **P2P**: Gets seed nodes (`P2P.list_bootstrapping_seeds`), connects to nodes (`P2P.connect_nodes`), gets node info (`P2P.get_node_info`), gets geo patch (`P2P.get_geo_patch`).
*   **SelfRepair**: Manages the `last_sync_date` (`SelfRepair.last_sync_date`, `SelfRepair.put_last_sync_date`), checks if sync is needed (`SelfRepair.missed_sync?`), triggers `SelfRepair.bootstrap_sync`, starts the `SelfRepair.Scheduler`.
*   **Crypto**: Gets node keys (`Crypto.first_node_public_key`, `Crypto.previous_node_public_key`), derives addresses (`Crypto.derive_address`).
*   **TransactionChain**: Reads previous node transactions (`TransactionChain.get_last_transaction`). Creates new `:node` transactions via `Transaction.new`.
*   **Replication**: Used implicitly by `TransactionHandler.validate_and_replicate_tx` for self-validating genesis transactions.
*   **BeaconChain**: Signals completion (`BeaconChain.add_end_of_node_sync`).
*   **DB**: Indirect dependency via other components.
*   **Genesis Chains**: `NetworkInit` interacts with `SharedSecrets`, `Governance`, `Reward`, `OracleChain`, and `NetworkConstraints` to create their initial transactions using their respective `new_..._transaction` functions.

## Interactions with Other Components

*   Started directly by `Application`.
*   Calls `Networking.get_node_ip`.
*   Calls various `P2P` functions for seed nodes and connection management.
*   Critically relies on `SelfRepair` (getting/setting `last_sync_date`, triggering `bootstrap_sync`, starting `Scheduler`).
*   Reads node state via `TransactionChain.get_last_transaction`.
*   `NetworkInit` calls factory functions (e.g., `SharedSecrets.new_node_shared_secrets_transaction`, `Reward.new_rewards_mint`) to create genesis transactions.
*   `TransactionHandler` likely calls `Replication` functions for genesis blocks.
*   Calls `BeaconChain.add_end_of_node_sync`.
*   Calls `NetworkConstraints.persist_genesis_address`. 