# Governance Component

## Purpose

Manages on-chain governance, primarily focused on the lifecycle of software update proposals. It handles proposal creation, approval tracking, validation checks (CI/CD), and potentially triggers deployments.

## Key Files/Modules

*   `lib/archethic/governance.ex`: Main interface. Provides functions to list/get code proposals (`list_code_proposals`, `get_code_proposal`) and interact with governance pools (`pool_member?`, `pool_members`). Its `load_transaction` function processes `:code_approval` transactions to potentially trigger testnet deployments.
*   `lib/archethic/governance/supervisor.ex`: Supervises:
    *   `Code.CICD`: Process managing CI/CD steps for code proposals.
    *   `Pools.MemTable`: In-memory store (ETS) for governance pool memberships.
    *   `Pools.MemTableLoader`: Loads pool membership data.
*   `lib/archethic/governance/code.ex`: Core logic for handling code proposals. Defines validation rules, checks approval thresholds (`enough_code_approval?`), interacts with CI/CD (`valid_integration?`), and triggers deployments (`deploy_proposal_testnet`).
*   `lib/archethic/governance/code/proposal.ex`: Defines the `Proposal` struct, likely parsed from `:code_proposal` transaction content.
*   `lib/archethic/governance/pools.ex`: Manages different governance pools (e.g., core developers, security auditors) and checks membership.
*   `lib/archethic/governance/pools/mem_table.ex`: Implementation of the pool membership store.
*   `lib/archethic/governance/pools/mem_table_loader.ex`: Loads pool membership data, likely from configuration or specific transactions.

## Core Functionality

*   **Code Proposal Management**: Tracks software update proposals submitted as `:code_proposal` transactions.
*   **Approval Tracking**: Fetches approval signatures for proposals from the transaction chain.
*   **Pool Membership**: Manages lists of authorized nodes for different governance roles/pools.
*   **CI/CD Integration**: Interacts with a CI/CD process (`Code.CICD`) to validate proposals and check integration status.
*   **Automated Deployment (Testnet)**: Can automatically trigger a testnet deployment (`deploy_proposal_testnet`) for a code proposal if it receives enough approvals from the correct pools and passes integration checks. (Mainnet deployment is mentioned but not implemented in `load_transaction`).
*   **State Loading**: `load_transaction` (called by `Replication`) processes incoming approvals. `Pools.MemTableLoader` loads initial pool state.

## Dependencies

*   **TransactionChain**: Reads `:code_proposal` transactions (`list_transactions_by_type`, `get_transaction`) and fetches approval signatures (`get_signatures_for_pending_transaction`).
*   **Crypto**: Handles node public keys (`Crypto.key`) for pool membership and approvals.
*   **Election**: `load_transaction` uses `Election.chain_storage_nodes` to determine if the current node should trigger actions.
*   **P2P**: Indirectly used via `Election` to get the node list (`P2P.authorized_and_available_nodes`).
*   **Utils**: For common utility functions (`key_in_node_list?`).
*   **DB**: Indirect dependency via `TransactionChain`.

## Interactions with Other Components

*   `load_transaction` is called by `Replication.ingest_transaction` when `:code_approval` transactions are stored.
*   Reads proposal and signature data from `TransactionChain` (`list_transactions_by_type`, `get_transaction`, `get_signatures_for_pending_transaction`).
*   Uses `Election.chain_storage_nodes` results to determine if the node should trigger deployment actions.
*   The `Code.CICD` process likely interacts with external systems (e.g., Git, CI server, deployment tools). 