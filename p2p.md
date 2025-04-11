# P2P Component

## Purpose

Manages all peer-to-peer network communication for the Archethic node. This includes discovering other nodes, establishing and managing connections, serializing/deserializing messages, routing requests, and maintaining information about the network topology and peer status.

## Key Files/Modules

*   `lib/archethic/p2p.ex`: Main public API for interacting with the P2P network (connecting, querying nodes, sending messages, performing quorum requests).
*   `lib/archethic/p2p/supervisor.ex` (`Archethic.P2PSupervisor`): Supervises core P2P services like the connection registry, connection supervisor, GeoIP DB process, and the node memory table (`MemTable`).
*   `lib/archethic/p2p/listener_supervisor.ex` (`Archethic.P2PListenerSupervisor`): Supervises incoming connection handling, including the network listener (`Listener`) and the bootstrapping seed manager.
*   `lib/archethic/p2p/node.ex`: Defines the `Node` struct, containing information about a peer (IP, port, public key, status, geo-patch, etc.).
*   `lib/archethic/p2p/mem_table.ex`: Manages an in-memory (ETS) table holding information about all known nodes.
*   `lib/archethic/p2p/mem_table_loader.ex`: Loads initial node data into `MemTable` from configuration and the DB.
*   `lib/archethic/p2p/client.ex` & `lib/archethic/p2p/client/`: Manage outgoing connections to peers.
*   `lib/archethic/p2p/listener.ex` & `lib/archethic/p2p/listener_protocol.ex`: Handle incoming connections and the application-level protocol.
*   `lib/archethic/p2p/message.ex` & `lib/archethic/p2p/message/`: Define the behaviour and specific structures for messages exchanged between nodes (e.g., `GetTransaction`, `NodeList`, `Transaction`).
*   `lib/archethic/p2p/message_envelop.ex`: Defines the wrapper structure for network messages, likely including metadata and signatures.
*   `lib/archethic/p2p/bootstrapping_seeds.ex`: Manages initial node discovery using seed nodes.
*   `lib/archethic/p2p/geo_patch.ex`: Implements geographical bucketing of nodes based on IP address using a GeoIP database (`MaxMindDB`).

## Core Functionality

*   **Node Discovery**: Finds initial peers using `BootstrappingSeeds` and learns about more nodes via peer exchange (`fetch_nodes_list`).
*   **Connection Management**: Establishes and maintains persistent connections (currently TCP) to peers using `Client` and `Listener` modules.
*   **Node State Tracking**: Stores and updates information about known nodes (address, keys, availability, sync status, authorization) in `MemTable`.
*   **Message Handling**: Serializes, signs (`MessageEnvelop`), sends (`send_message`), receives, verifies, and deserializes messages defined in `Message` modules.
*   **Request Routing**: Provides functions for sending requests to specific nodes (`request`), or to a quorum of nodes (`quorum_read`, `quorum_write`) and handling responses based on consistency requirements.
*   **Geographical Awareness**: Uses `GeoPatch` to group nodes geographically, potentially optimizing peer selection.

## Dependencies

*   **Crypto**: Essential for node identity (`P2P.Node.first_public_key`), signing/verifying message envelopes (`MessageEnvelop`).
*   **Networking**: Relies on `Networking.get_node_ip()` during initialization. Receives node IP updates from `Networking.Scheduler`. Uses transport information.
*   **TransactionChain**: Defines and handles many P2P message types (`GetTransaction`, `TransactionList`, etc.) used for fetching chain data. `P2P` calls `TransactionChain` functions upon receiving these requests.
*   **DB**: `MemTableLoader` reads initial P2P node summaries stored by the DB component (`DB.get_last_p2p_summaries`).
*   **Utils**: For common utility functions.
*   **External Libraries**: Likely `MaxMindDB` for GeoIP lookups, potentially `ranch` or similar for TCP listening.

## Interactions with Other Components

*   Provides the communication backbone for most other components.
*   **TransactionChain**: Uses `P2P.request`, `P2P.quorum_read`, `P2P.send_message` to fetch remote transactions/chains and broadcast new ones using specific `Message` types.
*   **Mining**: Uses `P2P.send_message` during distributed workflow (context exchange, stamps) and for `request_chain_lock`.
*   **Election**: Uses `P2P.authorized_and_available_nodes`, `P2P.get_node_info` to get candidate nodes.
*   **Replication**: Uses `P2P.send_message` to send `NotifyLastTransactionAddress`.
*   **BeaconChain**: Uses `P2P.request`/`P2P.quorum_read` to fetch summaries/aggregates/attestations. Provides node lists for P2P sampling.
*   **SelfRepair**: Uses `P2P.request`/`P2P.quorum_read` to fetch missing beacon/transaction data. Updates node availability (`P2P.set_node_globally_available`).
*   **OracleChain**: Potentially uses `P2P.request` if direct API access fails.
*   **Bootstrap**: Uses `P2P.list_bootstrapping_seeds`, `P2P.connect_nodes`, `P2P.get_node_info`, `P2P.get_geo_patch`.
*   **Networking**: `Networking.Scheduler` calls `P2P.MemTable.update_node_ip`.
*   **Telemetry**: `TelemetryPoller` calls `P2P.nodes_connected_count`. 