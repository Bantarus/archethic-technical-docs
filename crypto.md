# Crypto Component

## Purpose

Provides core cryptographic functionalities used throughout the Archethic node, including hashing, digital signatures (signing and verification), key derivation, address generation, and authenticated encryption/decryption for shared secrets.

## Key Files/Modules

*   `lib/archethic/crypto.ex`: Main public API for all cryptographic operations.
*   `lib/archethic/crypto/supervisor.ex`: Supervises crypto-related processes:
    *   `LibSodiumPort`: A port process for interacting with the native `libsodium` library (primarily for Ed25519).
    *   `KeystoreSupervisor`: Supervises the keystore processes.
*   `lib/archethic/crypto/id.ex`: Manages the identifier bytes prepended to keys and hashes to specify algorithms (e.g., curve type, hash function, key origin).
*   `lib/archethic/crypto/ecdsa.ex`: Interface for ECDSA operations (NIST P-256, secp256k1).
*   `lib/archethic/crypto/ed25519.ex`: Interface for Ed25519 operations (using `LibSodiumPort`).
*   `lib/archethic/crypto/keystore/`: Contains keystore implementations:
    *   `node_keystore.ex`: Manages the node's primary keys.
    *   `shared_secrets_keystore.ex`: Manages keys and nonces related to the Shared Secrets feature (e.g., `storage_nonce`, encryption/decryption keys).
    *   `keystore_supervisor.ex`: Supervisor for the individual keystores.

## Core Functionality

*   **Hashing**: Provides `hash/1` and `hash/2` supporting multiple algorithms identified by a version byte (`versioned_hash`).
*   **Key Management**: Derives deterministic key pairs (`derive_keypair`) from seeds using specified curves and origins. Keys are prepended with identifiers (`key`). Derives specific keys for Beacon (`derive_beacon_keypair`) and Oracle chains (`derive_oracle_keypair`).
*   **Address Generation**: Derives addresses (`derive_address`) from public keys (`prepended_hash`).
*   **Signing/Verification**: Signs data (`sign/2`, `sign_with_origin_key`, `sign_with_daily_nonce_key`) and verifies signatures (`verify/3`, `verify?`). Includes specific functions for verifying Archethic transactions (`verify_transaction/1`) and chains (`verify_chain/1`).
*   **Encryption/Decryption**: Provides AES authenticated encryption (`encrypt/3`, `decrypt/3`) via `SharedSecretsKeystore`.
*   **Keystores**: Manages secure storage and retrieval of cryptographic keys (node keys, shared secret keys, `storage_nonce`).

## Dependencies

*   **SharedSecrets**: Uses the `storage_nonce` provided by `SharedSecretsKeystore` for key derivation. `SharedSecrets` component likely calls `unwrap_secrets`, `decrypt_and_set_storage_nonce`, `encrypt`, `decrypt` functions defined here (operating via `SharedSecretsKeystore`). Uses `SharedSecrets` transaction types during verification.
*   **TransactionChain**: Uses `Transaction`, `ValidationStamp`, `TransactionData` structs as input for verification functions (`verify_transaction`, `verify_chain`).
*   **Utils**: For common utility functions.
*   **External**: Relies on `libsodium` C library (via `LibSodiumPort`) for Ed25519 and potentially other Elixir crypto libraries for ECDSA.

## Interactions with Other Components

*   **TransactionChain**: Calls `verify_signature`, `verify_transaction`, `verify_chain` during transaction processing and validation.
*   **P2P**: Calls `sign` for signing message envelopes (`MessageEnvelop`). Calls `verify?` for verifying received message envelopes.
*   **Election**: Calls `sign_with_daily_nonce_key` and `verify?` for handling the election proof seed.
*   **Mining**: Calls verification functions (`verify_transaction`, `verify_signatures`).
*   **SharedSecrets**: Calls `encrypt`/`decrypt`, `unwrap_secrets`, `storage_nonce`, `decrypt_and_set_storage_nonce` (via `SharedSecretsKeystore`). Calls `derive_keypair` and `derive_address` for origin key management.
*   **BeaconChain**: Calls `derive_beacon_chain_address`, `derive_beacon_keypair`.
*   **OracleChain**: Calls `derive_oracle_address`, `derive_oracle_keypair`.
*   **Contracts**: May call verification functions if contracts handle signatures.
*   **Bootstrap**: Calls `derive_address`, `first_node_public_key`, `previous_node_public_key`.
*   **Reward**: Calls `reward_public_key`, `derive_address`. 