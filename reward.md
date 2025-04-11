# Reward Component

## Purpose

Manages the calculation and scheduled distribution of rewards (in the form of MUCO tokens) to participating nodes based on network parameters and node status.

## Key Files/Modules

*   `lib/archethic/reward.ex`: Main interface. Calculates validator rewards (`validation_nodes_reward`), creates reward minting (`new_rewards_mint`) and distribution (`new_node_rewards`) transactions, determines transfer details (`get_transfers`), and manages reward token state.
*   `lib/archethic/reward/supervisor.ex`: Supervises the `Scheduler`, `RewardTokens` table, and `MemTablesLoader`.
*   `lib/archethic/reward/scheduler.ex`: GenServer responsible for periodically triggering the reward distribution process based on a configured cron interval and node election status.
*   `lib/archethic/reward/mem_tables/reward_tokens.ex`: In-memory table (ETS) tracking the addresses of MUCO reward tokens created by `:mint_rewards` transactions.
*   `lib/archethic/reward/mem_tables_loader.ex`: Loads reward token information from `:mint_rewards` transactions into the `RewardTokens` table.

## Core Functionality

*   **Reward Calculation**: Calculates the amount of MUCO tokens to be distributed per node per reward period (`validation_nodes_reward`). This calculation is dynamic, depending on the current UCO price (from `OracleChain`) and the mining protocol version, potentially aiming for a stable USD value reward.
*   **Reward Transaction Creation**: Defines functions to create specific transaction types:
    *   `:mint_rewards`: Creates new MUCO tokens.
    *   `:node_rewards`: Contains `TokenLedger.Transfer` operations to distribute calculated rewards to eligible nodes.
*   **Reward Distribution Scheduling**: The `Scheduler` periodically (cron-based) initiates the reward process:
    *   Checks if the current node is elected to lead the distribution.
    *   Calculates the necessary transfers based on current node status (`P2P`) and available MUCO balance (`TransactionChain`).
    *   Creates the `:node_rewards` transaction.
    *   Submits the transaction for mining.
*   **State Management**: Tracks reward token addresses (`RewardTokens`) loaded from the chain (`MemTablesLoader` via `load_transaction`). Persists the genesis address of the reward chain (`persist_gen_addr`).

## Dependencies

*   **Crypto**: Needed for reward address derivation (`Crypto.reward_public_key`).
*   **TransactionChain**: Used to read `:mint_rewards` transactions (`list_addresses_by_type`), get balances (`Archethic.get_balance`), and define transaction structures (`Transaction`, `TransactionData`, `TokenLedger.Transfer`).
*   **P2P**: `get_transfers` uses `P2P.authorized_and_available_nodes` to get the list of nodes to reward.
*   **OracleChain**: `validation_nodes_reward` calls `OracleChain.get_uco_price` for dynamic reward calculation.
*   **Mining**: Reward calculation depends on `Mining.protocol_version()`. The `Scheduler` submits `:node_rewards` transactions to `Mining`.
*   **Election**: The `Scheduler` likely checks election status (`is_validation_node?`) to determine leadership for initiating reward distribution.
*   **DB**: Indirect dependency via `TransactionChain`.
*   **Utils**: For common utility functions (date calculations, reward occurrence map).
*   **External**: `Crontab` library.

## Interactions with Other Components

*   Reads UCO price from `OracleChain.get_uco_price`.
*   Reads node status from `P2P.authorized_and_available_nodes`.
*   Reads transaction history and balances via `TransactionChain` (`list_addresses_by_type`, `Archethic.get_balance`).
*   Submits `:node_rewards` transactions to `Mining`.
*   Depends on `Election` results for scheduling leadership.
*   `load_transaction` called by `Replication.ingest_transaction` when `:mint_rewards` transactions are stored, updating `RewardTokens` via `MemTablesLoader`. 