# Networking Component

## Purpose

Provides low-level networking utilities focused on establishing the node's network presence. This includes determining the public IP address, attempting automatic port forwarding (UPnP/NAT-PMP), and validating IP addresses.

## Key Files/Modules

*   `lib/archethic/networking.ex`: Main interface. Provides functions to validate IPs (`validate_ip`), get the node's public IP (`get_node_ip`), and attempt port forwarding (`try_open_port`).
*   `lib/archethic/networking/supervisor.ex`: Supervises the `Scheduler`.
*   `lib/archethic/networking/scheduler.ex`: Periodically triggers checks to update the node's public IP address in the P2P layer if it changes.
*   `lib/archethic/networking/ip_lookup.ex` & `lib/archethic/networking/ip_lookup/`: Implements logic to discover the node's public IP address, potentially using multiple external services (e.g., ipify) or local interface checks.
*   `lib/archethic/networking/port_forwarding.ex`: Implements logic to attempt automatic port forwarding using UPnP or NAT-PMP protocols on the local network gateway.

## Core Functionality

*   **Public IP Discovery**: Determines the node's external IP address (`get_node_ip`, `IPLookup`).
*   **Port Forwarding**: Attempts to automatically configure port forwarding rules on the local network gateway (`try_open_port`, `PortForwarding`).
*   **IP Validation**: Checks if an IP address falls within private/local ranges (`validate_ip`, `valid_ip?`).
*   **Scheduled Updates**: Periodically re-checks the public IP and updates the `P2P` component if changes are detected (`Scheduler`).

## Dependencies

*   **P2P**: The `Scheduler` updates the node's IP address stored in `P2P.MemTable` (`P2P.MemTable.update_node_ip`).
*   **OS/System**: Relies on underlying network stack for socket operations (used by UPnP/NAT-PMP) and potentially interface information.
*   **External Services**: `IPLookup` typically uses external web services to discover the public IP. `PortForwarding` interacts with UPnP/NAT-PMP services on the local network.

## Interactions with Other Components

*   Called by `Application.start/2` to check/open P2P and Web ports (`try_open_port`).
*   Called by `Bootstrap.start_link/1` to get the initial node IP (`get_node_ip`).
*   `Scheduler` periodically calls `P2P.MemTable.update_node_ip`.