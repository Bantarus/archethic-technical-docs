# Contracts Component

## Purpose

Manages the deployment, execution, state management, and lifecycle of smart contracts on the Archethic network. Supports both a custom interpreted language and WebAssembly (WASM) contracts.

## Key Files/Modules

*   `lib/archethic/contracts.ex`: Main interface. Primarily provides `execute_trigger/6` to run contract logic based on triggers (e.g., transactions, timers). Handles dispatching between interpreted and WASM contracts.
*   `lib/archethic/contracts/supervisor.ex`: Supervises:
    *   `Loader`: Process for loading contract code/definitions.
    *   `Archethic.ContractRegistry`: Tracks active contract worker processes.
    *   `Archethic.ContractSupervisor`: Dynamically starts/supervises individual `Worker` processes for each active contract.
*   `lib/archethic/contracts/worker.ex`: GenServer process representing a running instance of a single smart contract. Holds the contract's state and executes its logic.
*   `lib/archethic/contracts/loader.ex`: Loads contract code (interpreted or WASM) from transaction data, parses it, and prepares it for execution.
*   `lib/archethic/contracts/interpreter/`: Contains the implementation for the custom interpreted language engine.
*   `lib/archethic/contracts/wasm/`: Contains modules for handling WASM contracts (`WasmContract`, `WasmModule`, `WasmSpec`), including parsing, execution, and interaction with a WASM runtime.
*   `lib/archethic/contracts/contract/`: Defines common result types for contract execution (e.g., `ActionWithTransaction`, `ActionWithoutTransaction`, `Failure`).

## Core Functionality

*   **Contract Loading**: Loads contract definitions (code, manifest) from transaction data (`Loader`).
*   **Contract Execution**: Executes contract logic based on triggers (`execute_trigger`). Supports different trigger types (e.g., `:transaction`, potentially `:timer`). Dispatches execution to the appropriate engine (Interpreter or WASM).
*   **State Management**: Each active contract runs as a `Worker` process holding its current state. Contract execution can modify this state.
*   **WASM Support**: Parses, validates, and executes WASM bytecode using an external WASM runtime. Handles WASM contract upgrades.
*   **Interpreter Support**: Executes contracts written in a custom language using a dedicated interpreter.
*   **Interaction with Ledger**: Contracts can read balances (`UTXO.get_balance`) and potentially other chain state. Their execution can result in new transactions (`ActionWithTransaction`) or state changes affecting the ledger (`LedgerOperations`).
*   **Instance Management**: Manages running contract instances via the `ContractSupervisor` and `ContractRegistry`.

## Dependencies

*   **TransactionChain**: Reads contract deployment/update transactions (`:contract` type). Uses `Transaction`, `TransactionData`, `Recipient` structs. Contract execution may generate new transactions.
*   **UTXO**: Provides balances (`UTXO.get_balance`) and UTXO inputs to contracts. Contract execution modifies UTXO state via `LedgerOperations`.
*   **Crypto**: For contract addresses.
*   **DB**: Contract state (`State`) is likely persisted, potentially via `TransactionChain` or dedicated storage. Contract code itself is stored in transactions.
*   **Mining**: `Mining.SmartContractValidation` calls `Contracts.execute_trigger` during transaction validation.
*   **Utils**: For common utility functions.
*   **External**: Requires a WASM runtime library (e.g., `wasmer`, `wasmex`).

## Interactions with Other Components

*   Called by `Mining.SmartContractValidation` (`execute_trigger`) to execute contract logic during transaction validation.
*   Reads state from `UTXO.get_balance`.
*   Execution results (`ActionWithTransaction`) can lead to the creation of new transactions submitted back to `Mining`.
*   Contract code is loaded via `Loader` when `:contract` type transactions are processed by `Replication.ingest_transaction`.
*   Running contract instances are managed internally via `Worker` processes. 