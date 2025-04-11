# Telemetry Component

## Purpose

Acts as the central integration point for the node's observability, leveraging the standard Elixir `:telemetry` ecosystem. It defines application-specific metrics based on telemetry events, configures metric reporters (specifically for Prometheus), and uses polling to gather non-event-based metrics.

## Key Files/Modules

*   `lib/archethic/telemetry.ex`: The core module implementing the `Supervisor` behaviour. Its `init/1` function:
    *   Starts `TelemetryMetricsPrometheus.Core`, passing it a list of metric definitions.
    *   Starts `:telemetry_poller` to periodically call specific functions in other components (`Contracts`, `P2P`) and emit their results as telemetry events.
    *   Defines a wide range of metrics (`metrics/0` function) using `Telemetry.Metrics` helpers (`last_value`, `distribution`, `counter`) covering VM stats and application-specific operations (P2P, Mining, Crypto, DB, Election, Replication, etc.).

## Core Functionality

*   **Metric Definition**: Centralizes the definition of application-specific metrics based on expected `:telemetry` events (e.g., `[:archethic, :p2p, :send_message, :stop]`).
*   **Prometheus Export**: Configures and starts the `TelemetryMetricsPrometheus.Core` process, enabling metrics to be scraped by Prometheus.
*   **Polling**: Uses `:telemetry_poller` to periodically gather gauge-like metrics from specific functions in other components.
*   **Supervision**: Supervises the core Prometheus exporter and the poller process.

## Dependencies

*   **Telemetry Ecosystem**: Relies heavily on `:telemetry`, `telemetry_metrics`, `telemetry_metrics_prometheus_core`, and `:telemetry_poller` libraries.
*   **Various Components**: Depends on the structure of `:telemetry` events emitted by nearly all other application components (DB, TransactionChain, Crypto, Election, P2P, Mining, Contracts, Replication, SelfRepair, BeaconChain). It also directly calls functions in `Contracts` (`maximum_calls_in_queue`) and `P2P` (`nodes_connected_count`) via the poller.

## Interactions with Other Components

*   Started directly by `Application`.
*   Attaches to `:telemetry` events emitted globally by other components.
*   Calls functions in `Contracts` and `P2P` via `:telemetry_poller`.
*   The associated `TelemetryMetricsPrometheus.Core` process exposes an HTTP endpoint for external systems (Prometheus).
*   Works in conjunction with `Archethic.Metrics.ETSFlush` (which calls `TelemetryMetricsPrometheus.Core.scrape()`) to ensure correct scraping of distribution metrics. 