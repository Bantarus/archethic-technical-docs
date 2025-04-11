# Beacon Chain Component

## Purpose

Manages a coordination and data aggregation layer operating in parallel to the main transaction chains. It divides the network into 256 subsets and periodically aggregates information within these subsets into time-based "Slots" and longer-term "Summaries" and "Summary Aggregates". It seems crucial for network monitoring (P2P availability, sync status), coordinating storage replication, and potentially providing a source of network time or randomness.

## Key Files/Modules

*   `lib/archethic/beacon_chain.ex`: Main public API for interacting with beacon data (getting timings, retrieving summaries/aggregates, loading slots, getting attestations).
*   `lib/archethic/beacon_chain/supervisor.ex`: Supervises core beacon processes:
    *   `SlotTimer`, `SummaryTimer`: GenServers managing the timing and triggering of slot/summary creation.
    *   `SubsetSupervisor`: Supervises processes managing individual network subsets.
    *   `Update`: Handles applying state updates derived from beacon data.
*   `lib/archethic/beacon_chain/slot.ex`: Defines the `Slot` struct, which aggregates data collected during a time interval within a subset (e.g., `P2PView`, `EndOfNodeSync`, `TransactionAttestation`).
*   `lib/archethic/beacon_chain/summary.ex`: Defines the `Summary` struct, aggregating data from multiple slots within a subset over a longer period.
*   `lib/archethic/beacon_chain/summary_aggregate.ex`: Defines the `SummaryAggregate` struct, combining summaries from all subsets for a specific time.
*   `lib/archethic/beacon_chain/subset.ex` & `lib/archethic/beacon_chain/subset/`: Manage the state and operations for individual network subsets, including caching incoming slot data (`SummaryCache`) and sampling peer status (`P2PSampling`).
*   `lib/archethic/beacon_chain/replication_attestation.ex`: Defines the structure for tracking confirmations of transaction storage.
*   `lib/archethic/beacon_chain/network_coordinates.ex`: Implements Vivaldi network coordinate calculations for nodes.
*   `lib/archethic/beacon_chain/update.ex`: Applies updates based on collected beacon data (e.g., updating node availability in P2P).

## Core Functionality

*   **Time Synchronization**: Provides timed intervals (`SlotTimer`, `SummaryTimer`) that likely drive periodic network activities.
*   **Network Monitoring**: Collects and aggregates data about node status within subsets:
    *   P2P availability and network view (`P2PView` in `Slot`).
    *   Node synchronization completion (`EndOfNodeSync` in `Slot`).
    *   Network coordinates (`NetworkCoordinates`).
*   **Data Aggregation**: Creates periodic `Slot` data, aggregates these into `Summary` structs per subset, and further aggregates these into network-wide `SummaryAggregate` structs.
*   **Storage Coordination**: Tracks confirmation of transaction storage via `ReplicationAttestation` included in slots.
*   **Subset Management**: Divides the network into 256 subsets (likely based on address/key prefix) and manages data aggregation within each.
*   **Persistence**: Stores/Retrieves `Summary` and `SummaryAggregate` data using the `DB` component.
*   **P2P Interaction**: Fetches missing beacon data (`Summary`, `SummaryAggregate`, `ReplicationAttestation`) from peers.

## Dependencies

*   **DB**: Stores (`DB.write_beacon_summary`, `DB.write_beacon_summaries_aggregate`) and retrieves (`DB.get_beacon_summary`, `DB.get_beacon_summaries_aggregate`) summary and aggregate data.
*   **Crypto**: Used for deriving beacon-specific transaction addresses (`Crypto.derive_beacon_chain_address`) and node identities.
*   **P2P**: Used to fetch beacon data (`GetBeaconSummaries`, `GetBeaconSummariesAggregate`, `GetCurrentReplicationAttestations`) via `P2P.request`/`P2P.quorum_read`. Uses `P2P.Node` list for sampling.
*   **TransactionChain**: Uses `TransactionSummary` struct within beacon summaries. Uses `ReplicationAttestation` which relates to transaction storage.
*   **Election**: Potentially used to elect nodes responsible for beacon tasks (less explicit than in Mining).
*   **Utils**: For common utility functions.
*   **External**: Potentially `crontab` for timer scheduling.

## Interactions with Other Components

*   Provides timing signals (`SlotTimer`, `SummaryTimer`) that may trigger actions in other components (e.g., `SelfRepair.Scheduler`, `Reward.Scheduler`).
*   Its `Update` module likely calls `P2P.set_node_globally_available` or similar to update the P2P layer's view of node status.
*   Receives `EndOfNodeSync` events from `SelfRepair` / `Bootstrap`.
*   Receives `ReplicationAttestation` data from `Replication` (likely).
*   Receives P2P view/sampling data likely originating from `P2P` interactions.
*   Uses `DB` for persistence of summaries/aggregates.
*   Uses `P2P` to synchronize beacon data with peers.
*   Provides `previous_summary_time` and `next_summary_date` used by `SelfRepair` to determine sync boundaries. 