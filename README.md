<img src="https://s3.alaa.computer/dragonslayer-github-banner.png" alt="guts holding a sword with engraved dragonslayer geyser plugin" width="600">

> It was too big to be called a sword. Massive, thick, heavy, and far too rough. Indeed, it was a heap of raw iron.

# Dragonslayer NATS Geyser Plugin

**Dragonslayer** is a Solana Geyser plugin to stream account, slot, transaction, and block updates via the NATS messaging system. This plugin allows developers to subscribe to real-time Solana blockchain events and integrate them into their applications using the high-performance NATS protocol.

This plugin is designed to work with Solana's [Geyser plugin framework](https://docs.anza.xyz/validator/geyser).

## Table of Contents

- [Introduction](#introduction)
- [Features (Planned)](#features-planned)
- [Architecture](#architecture)
- [Prerequisites](#prerequisites)
- [Installation](#installation)
- [Configuration](#configuration)
- [How it Works](#how-it-works)
- [Message Schemas (Planned)](#message-schemas-planned)
- [Future Development](#future-development)
- [Contributing](#contributing)
- [License](#license)
- [Special Thanks](#special-thanks)

## Introduction

_Dragonslayer_ provides a bridge between the Solana validator's Geyser interface and the NATS messaging system. It captures real-time blockchain event data, such as account modifications, slot status updates, transaction notifications, and block information, and publishes this data to specified NATS subjects (topics). This enables a decoupled architecture where multiple downstream applications can consume Solana events efficiently and reliably.

This plugin aims to offer a flexible and configurable way to stream specific data, similar to how other Geyser plugins operate for different protocols like Kafka or gRPC.

## Features (Planned)
* Stream various Solana events:
    * Account updates
    * Slot status updates
    * Transaction notifications (including vote and failed transactions options)
    * Block updates (meta and full blocks)
    * Entry updates
* Configurable NATS connection parameters (server URLs, credentials, etc.).
* Flexible filtering mechanisms to subscribe only to relevant data (e.g., by program owner, account public keys, commitment levels).
* Support for different NATS message serialization formats (e.g., JSON, Protobuf - to be decided).
* Metrics for monitoring plugin health and data flow (e.g., Prometheus endpoint).
* Durable subscriptions and message delivery guarantees as offered by NATS.

## Architecture

The plugin integrates into the Solana validator process as a shared library.

1.  **Solana Validator**: The validator node captures blockchain events.
2.  **Geyser Plugin Interface**: The plugin framework within the validator forwards these events to registered plugins.
3.  **NATS Geyser Plugin**:
      * Receives event notifications (account updates, slot updates, transaction updates, block updates) from the Geyser interface.
      * Filters events based on the user's configuration.
      * Serializes the event data into a defined message format.
      * Publishes these messages to the configured NATS server(s) on specific subjects.
4.  **NATS Server(s)**: The NATS messaging system routes messages from the plugin to subscribed client applications.
5.  **Consumer Applications**: Applications connect to NATS and subscribe to relevant subjects to receive and process Solana blockchain events.

<!-- end list -->

```
+-------------------+      +-------------------------+      +---------------------+      +-----------------------------+      +-----------------------+
| Solana Validator  |----->| Geyser Plugin Interface |----->| NATS Geyser Plugin  |----->| NATS Server(s) with JetStream|----->| Consumer Applications |
+-------------------+      +-------------------------+      |(Publishes to Streams)|      |(Manages Streams, Persistence, |      |(Durable Subscriptions,|
                                                            +---------------------+      |   ACKs, Replay)             |      |   ACKs, Replay Logic) |
                                                                                         +-----------------------------+      +-----------------------+
```

## Prerequisites

*(This section will be updated as development progresses)*

  * Solana Validator
  * Rust Compiler
  * NATS Server: Accessible NATS server or cluster.
  * Protobuf compiler if Protobuf is chosen for message serialization.

## Installation

*(This section will be updated once build processes and release mechanisms are established)*

### Binary Releases (Planned)

Links to binary releases will be provided here.

### Building from Source (Planned)

```bash
# Placeholder for build commands
git clone https://github.com/alaazorkane/dragonslayer-nats-geyser
cd dragonslayer-nats-geyser
cargo build --release
```

The compiled plugin library (e.g., `libsolana_dragonslayer_nats.so` on Linux or `libsolana_dragonslayer_nats.dylib` on macOS) will need to be placed in a location accessible by the Solana validator.

**Important**: Solana's plugin interface requires the build environment of the Solana validator and this plugin to be identical. This includes the Solana version and Rust compiler version. Loading a plugin targeting different versions can lead to instability or crashes.

## Configuration

The plugin will be configured using a JSON file specified when launching the Solana validator (e.g., via the `--geyser-plugin-config <path_to_config.json>` argument).

Below is a *preliminary example* of what the configuration structure might look like. Actual parameters and their structure are subject to change.

```json
{
  "libpath": "/path/to/libsolana_dragonslayer_nats.so",
  "nats_url": "nats://localhost:4222", // NATS server URL(s)
  "nats_credentials_file": "/path/to/nats/user.creds", // Optional NATS credentials
  "nats_subjects": { // Base NATS subjects (topics) for different event types
    "account_updates": "solana.mainnet.account_update",
    "slot_updates": "solana.mainnet.slot_update",
    "transaction_updates": "solana.mainnet.transaction_update",
    "block_updates": "solana.mainnet.block_update"
  },
  "jetstream": { // Optional: Configuration for NATS JetStream
    "enabled": true, // Set to true to use JetStream features
    "stream_prefix": "solana_events", // Prefix for JetStream stream names
    "publish_async_max_pending": 1024, // JetStream specific: max number of unacknowledged messages the publisher can have in flight
    "retention_policy": "limits", // e.g., limits, interest, workqueue
    "max_age_seconds": 86400 // Example: retain messages for 1 day
    // Other JetStream stream configuration options (max_bytes, max_msgs_per_subject, etc.)
  },
  "shutdown_timeout_ms": 30000, // Graceful shutdown timeout
  "filters": { // Filters to select specific data to stream
    "accounts": {
      "owners": ["owner_pubkey_1", "owner_pubkey_2"],
      "accounts_allowlist": ["account_pubkey_1"]
    },
    "transactions": {
      "include_votes": false,
      "include_failed": false,
      "mentions": ["account_pubkey_to_watch"]
    }
  },
  "prometheus_endpoint": "0.0.0.0:9090" // Optional: for metrics
}
```

Key configuration areas will likely include:

  * **NATS Connection**: Server addresses, security credentials, connection options.
  * **Subject Mapping**: Defining which NATS subjects will receive account, slot, transaction, and block updates.
  * **Data Filtering**: Similar to other plugins, allowing users to specify program IDs, account addresses, or other criteria to filter the streamed data.
  * **Buffering and Retries**: NATS client-side buffering and retry mechanisms.
  * **Message Formatting**: Options for message serialization (e.g., JSON, Protobuf).

## How it Works

1. The Solana validator loads the NATS Geyser plugin.
2. As the validator processes events, it notifies the plugin via the Geyser API.
3. The plugin filters these events based on its configuration.
4. Filtered event data is serialized.
5. If JetStream is enabled: The plugin publishes messages to pre-defined NATS JetStream streams, potentially waiting for publish acknowledgements from the NATS server to ensure messages are persisted.
6. If JetStream is not enabled (core NATS): The plugin publishes messages to standard NATS subjects ("fire and forget").
7. NATS server(s) (with or without JetStream) handle message routing and persistence (if JetStream is used).
8. Consumer applications, connected to NATS (and potentially configured as durable JetStream consumers), receive and process messages. If using JetStream, they will acknowledge messages upon successful processing.

## Message Schemas (Planned)

*(This section will detail the structure of messages published to NATS for each event type: AccountUpdate, SlotUpdate, TransactionNotification, BlockUpdate. The choice between JSON, Protobuf, or other formats will be documented here.)*

Considerations for message schemas:

  * **Clarity and Completeness**: Ensure all relevant information from the Geyser event is available.
  * **Efficiency**: Choose a serialization format that balances readability and performance.
  * **Consistency**: Maintain consistent field names and data types.
  * **Versioning**: Plan for future schema evolution.

## Future Development

*(This section will be updated with a roadmap as the project progresses)*

  * Support for NATS JetStream for enhanced persistence and message delivery guarantees.
  * Advanced filtering capabilities.
  * Schema registry integration (if using formats like Protobuf or Avro).
  * Example client applications in various languages.
  * Comprehensive test suite.

## Contributing

Contributions are welcome! Please refer to `CONTRIBUTING.md` for guidelines on how to contribute to this project, including reporting issues, submitting pull requests, and coding standards.

## License

Check the `LICENSE` and `LICENSING.md` files.

## Special Thanks

Special thanks to the Triton One and OG The solana foundation for two prior arts (yellowstone-grpc and postgresql) on best practices to make geyser plugins!
