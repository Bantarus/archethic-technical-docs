# Metrics Component

## Purpose

Provides infrastructure support for collecting and exposing internal node metrics via the standard Elixir Telemetry ecosystem, specifically targeting Prometheus export.

**Note**: The actual definition and collection of metrics happens primarily within `lib/archethic/telemetry.ex`, not within the `lib/archethic/metrics/` directory itself.

## Key Files/Modules

*   `lib/archethic/metrics/metric_supervisor.ex`: Supervises the `ETSFlush` process.
*   `lib/archethic/metrics/ets_flush.ex`: A GenServer that periodically calls `TelemetryMetricsPrometheus.Core.scrape()` to ensure buffered distribution metrics are flushed for Prometheus scraping.
*   `lib/archethic/telemetry.ex` (**Related Logic**):
    *   Defines numerous metrics using `Telemetry.Metrics` functions (`last_value`, `distribution`, `counter`) based on `:telemetry` events emitted elsewhere.
    *   Starts and configures `TelemetryMetricsPrometheus.Core` to expose these defined metrics.
    *   Uses `:telemetry_poller` to periodically sample specific application states (e.g., contract queue length, connected peers) and emit them as metrics.

## Core Functionality

*   **Prometheus Integration Support**: Ensures Prometheus can correctly scrape all metric types, including distributions, by periodically flushing internal buffers (`ETSFlush`).

## Dependencies

*   **Telemetry Ecosystem**: Relies on `TelemetryMetricsPrometheus.Core` (called by `ETSFlush`) and indirectly on the broader telemetry pipeline set up in `Archethic.Telemetry`.
*   **Utils**: Uses `Utils.configurable_children` in supervisor.

## Interactions with Other Components

*   Started by `Application` supervisor.
*   `ETSFlush` calls `TelemetryMetricsPrometheus.Core.scrape()` which is part of the pipeline configured by `Archethic.Telemetry`.
*   Its primary role is to facilitate the correct operation of the Prometheus exporter configured in `Archethic.Telemetry`. 