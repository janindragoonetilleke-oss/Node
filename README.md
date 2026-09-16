# MASQ Node ![masq-logo](images/masq-logo-sm.png)

<div align="center">

[![ci-matrix](https://github.com/MASQ-Project/Node/workflows/ci-matrix/badge.svg)](https://github.com/MASQ-Project/Node/actions)
[![GitHub Release](https://img.shields.io/github/v/release/MASQ-Project/Node?display_name=release&color=green)](https://github.com/MASQ-Project/Node/releases/latest)
[![Discord](https://badgen.net/badge/icon/discord?icon=discord&label)](https://discord.gg/masq)
[![License: GPL v3](https://img.shields.io/badge/License-GPLv3-blue.svg)](LICENSE)
[![Rust](https://img.shields.io/badge/rust-1.63%2B-orange.svg)](https://www.rust-lang.org)

</div>

**MASQ Node** forms the foundation of the MASQ Network: an open-source decentralized mesh-network (dMN) combining the benefits of VPN and Tor technology to create next-generation privacy software. Users are rewarded with **$MASQ** utility tokens for allocating spare computing resources and bandwidth to support an uncensored, borderless, and privacy-preserving global Web.

---

## Table of Contents
- [Overview](#overview)
- [Key Features](#key-features)
- [Architecture](#architecture)
  - [System Workflow](#system-workflow)
  - [Daemon vs. Node Security Model](#daemon-vs-node-security-model)
  - [Workspace Subprojects](#workspace-subprojects)
  - [Internal Node Subsystems](#internal-node-subsystems)
- [Prerequisites & Supported Platforms](#prerequisites--supported-platforms)
- [Building from Source](#building-from-source)
- [Quick Start Guide](#quick-start-guide)
  - [1. Starting the MASQ Daemon](#1-starting-the-masq-daemon)
  - [2. Setting up Wallets and Password](#2-setting-up-wallets-and-password)
  - [3. Configuring and Starting the Node](#3-configuring-and-starting-the-node)
  - [4. Routing Traffic (Proxy & DNS Subversion)](#4-routing-traffic-proxy--dns-subversion)
  - [5. Clean Shutdown](#5-clean-shutdown)
- [Configuration Reference](#configuration-reference)
  - [Configuration Priority](#configuration-priority)
  - [Configuration Parameters](#configuration-parameters)
  - [Node Descriptor Format](#node-descriptor-format)
  - [Sample `config.toml`](#sample-configtoml)
- [CLI Reference (`masq`)](#cli-reference-masq)
- [Testing](#testing)
- [Troubleshooting & Diagnostics](#troubleshooting--diagnostics)
  - [Port 53 Binding Failures](#port-53-binding-failures)
  - [TLS Alerts & Routing Errors](#tls-alerts--routing-errors)
  - [Router NAT & Port Forwarding](#router-nat--port-forwarding)
- [Component Documentation](#component-documentation)
- [Origin & Attribution](#origin--attribution)
- [License](#license)

---

## Overview

Traditional VPNs suffer from centralized trust, single points of failure, and vulnerability to geographic IP blocking. Tor provides anonymity but relies on volunteer exit nodes and lacks built-in economic incentives for high-speed bandwidth relaying.

**MASQ Node** provides a hybrid decentralized model:
- **Decentralized Mesh Routing:** Data requests are split and forwarded across dynamic multi-hop paths via peer Nodes.
- **Incentivized Relays:** Nodes that route data and provide exit bandwidth earn $MASQ tokens via micro-transactions settled across supported EVM blockchains (Ethereum, Base, and Polygon).
- **Censorship Evasion:** With no central server infrastructure or static IP ranges, traffic resists DNS filtering, deep packet inspection (DPI), and geo-restrictions.

> [!IMPORTANT]
> **Beta Disclaimer:** MASQ Node software is in active development. While traffic payload is encrypted and cannot be deciphered in transit, network metadata analysis on local ISP connections can indicate peer-to-peer mesh traffic. Do not use for high-risk or life-critical privacy activities.

---

## Key Features

- **Multi-Hop CORES Packaging:** Encapsulates traffic inside layered, multi-hop encrypted CORES packets (similar to onion routing). Intermediate relay nodes cannot inspect payloads or endpoints.
- **Zero-Knowledge DNS Resolution:** Intercepts local DNS requests and securely resolves them across the mesh via uncensored exit nodes, neutralizing DNS-level ISP hijacking and poisoning.
- **Multi-Chain EVM Accounting:** Supports real-time balance tracking, debt limits, and settlements across **Ethereum Mainnet**, **Base Mainnet**, **Polygon Mainnet**, and testnets (**Base Sepolia**, **Polygon Amoy**).
- **Dual Privilege Architecture:** Separation into a privileged Daemon (bound to `localhost`) and an unprivileged Node process that drops root permissions immediately after binding low ports.
- **Automated NAT Traversal:** Built-in router port mapping (`automap`) supporting UPnP, NAT-PMP, and PCP.
- **Multiple Interfaces:** Can be managed via graphical UI ([MASQ Browser](https://masqbrowser.com)), command-line interface (`masq`), or headless automated daemon.

---

## Architecture

### System Workflow

The diagram below illustrates how client applications (browsers) route traffic through the local MASQ Node, hop through the decentralized mesh network, and reach destination servers:

```mermaid
flowchart LR
    subgraph LocalMachine ["User Local Machine"]
        Browser["Web Browser / Apps"]
        DNSUtil["dns_utility"]
        Daemon["MASQ Daemon (:5333)"]
        CLI["masq CLI / UI"]
        NodeProc["MASQ Node Process"]
        
        Browser -->|"HTTP/HTTPS Proxy"| NodeProc
        Browser -.->|"DNS Query (:53)"| NodeProc
        CLI -->|"WebSocket (MASQNode-UIv2)"| Daemon
        Daemon -->|"Spawns & Monitors"| NodeProc
        CLI -.->|"Direct Control"| NodeProc
        DNSUtil -->|"Configure System DNS"| Browser
    end

    subgraph MeshNetwork ["MASQ Decentralized Mesh"]
        Hop1["Intermediate Relay Node 1"]
        Hop2["Intermediate Relay Node 2"]
        Exit["Exit Node"]
        
        NodeProc -->|"Encrypted CORES Packets"| Hop1
        Hop1 -->|"Encrypted CORES Packets"| Hop2
        Hop2 -->|"Encrypted CORES Packets"| Exit
    end

    subgraph Internet ["Clearnet & Blockchain"]
        TargetServer["Destination Web Server"]
        RPC["Web3 / Blockchain RPC Node"]
        
        Exit -->|"Decrypted Request"| TargetServer
        NodeProc -.->|"Settlement & Accounts"| RPC
        Exit -.->|"Settlement & Accounts"| RPC
    end
```

### Daemon vs. Node Security Model

To protect against local privilege escalation:
1. **The Daemon (`MASQNode --initialization`)**:
   - Starts at system boot with administrative (root/elevated) privileges.
   - Listens exclusively on `localhost` (default port `5333`) for UI connections (`masq` CLI or MASQ Browser).
   - **Strictly isolated:** Never communicates over the public Internet and cannot process external mesh data.
   - Responsible for initialization, environment validation, and spawning the Node process.
2. **The Node (`MASQNode`)**:
   - Launched by the Daemon.
   - Briefly utilizes elevated privileges to bind necessary low network ports (e.g., DNS port `53`).
   - Immediately drops all elevated privileges to a standard user (`--real-user`) before reading any packets from external network interfaces.
   - Communicates with UIs via WebSocket protocol (`MASQNode-UIv2`).

### Workspace Subprojects

The repository is organized as a Cargo workspace with dedicated crates:

| Subproject | Description | Primary Artifacts |
| :--- | :--- | :--- |
| [`node`](node) | Core MASQ Node and Daemon implementation | `MASQNode`, `MASQNodeW` (Windows), `node_lib` |
| [`masq`](masq) | Official command-line user interface | `masq` (CLI binary) |
| [`dns_utility`](dns_utility) | OS-agnostic utility to inspect, subvert, and revert system DNS | `dns_utility`, `dns_utilityw` |
| [`automap`](automap) | Firewall and router traversal engine (UPnP, NAT-PMP, PCP) | `automap`, `automap_lib` |
| [`ip_country`](ip_country) | Offline IP-to-Country geolocation database engine | `ip_country`, `ip_country_lib` |
| [`masq_lib`](masq_lib) | Core shared types, constants, crypto, and schema definitions | `masq_lib` |
| [`port_exposer`](port_exposer) | Network utility to forward meta-address `0.0.0.0` to loopback | `port_exposer` |
| [`multinode_integration_tests`](multinode_integration_tests) | Docker-based integration test suite simulating multi-node networks | `multinode_integration_tests_lib`, `mock_node` |
| [`test_utilities`](test_utilities) | Shared test fixtures, mock servers, and testing helpers | `test_utilities` |

### Internal Node Subsystems

Inside the `node` crate, functionality is structured into modular actors and layers:
- **`proxy_server`**: Inbound local proxy listening for HTTP/HTTPS requests from client browsers, packing requests into encrypted CORES packages.
- **`hopper`**: Micro-routing engine responsible for unpacking CORES headers, verifying routing cryptograms, and forwarding packets to the next hop or local service.
- **`neighborhood`**: Gossip protocol and peer management engine. Maintains neighborhood tables, measures peer latency/reputation, and selects multi-hop routing paths.
- **`proxy_client`**: Handles exit operations. Translates decrypted CORES packages back into standard TCP/HTTP requests to target servers on the Internet, and repacks responses.
- **`entry_dns`**: Local DNS interceptor that handles queries for domain names and directs lookups through mesh exit nodes.
- **`accountant`**: Financial engine managing micro-transactions, tracking bandwidth debits and credits, checking peer solvency, and submitting blockchain payments.
- **`ui_gateway`**: WebSocket server facilitating command-and-control communication between the Node/Daemon and connected UI clients.

---

## Prerequisites & Supported Platforms

### Supported Operating Systems
- **Linux:** Ubuntu 20.04 / 22.04 LTS or newer (x86_64)
- **macOS:** macOS 11 Big Sur, 12 Monterey, 13+ (Apple Silicon & Intel x86_64)
- **Windows:** Windows 10 / 11 64-bit

> *Note:* 32-bit platforms are not actively supported for building due to upstream toolchain limitations.

### System Dependencies
- **Rust Toolchain:** Stable 1.63.0+ (with `cargo`, `rustfmt`, and `clippy`)
- **C Compiler & Build Tools:** `gcc`, `g++`, `make`, `pkg-config`
- **OpenSSL:** Headers and development libraries (e.g., `libssl-dev` on Debian/Ubuntu)
- **SQLite:** Handled via bundled `rusqlite` dependencies.

On Debian / Ubuntu:
```bash
sudo apt-get update
sudo apt-get install -y build-essential pkg-config libssl-dev git
```

On macOS:
```bash
xcode-select --install
```

---

## Building from Source

1. **Clone the Repository:**
   ```bash
   git clone https://github.com/MASQ-Project/Node.git masq-node
   cd masq-node
   ```

2. **Build the Entire Workspace:**
   To build release binaries for all workspace components (`MASQNode`, `masq`, `dns_utility`, `automap`):
   ```bash
   cargo build --release --manifest-path node/Cargo.toml
   cargo build --release --manifest-path masq/Cargo.toml
   cargo build --release --manifest-path dns_utility/Cargo.toml
   cargo build --release --manifest-path automap/Cargo.toml
   ```

   Alternatively, execute the project-wide CI build script:
   ```bash
   ./ci/all.sh
   ```

3. **Locate Compiled Binaries:**
   - `node/target/release/MASQNode` (MASQ Node & Daemon)
   - `masq/target/release/masq` (CLI client)
   - `dns_utility/target/release/dns_utility` (DNS utility)
   - `automap/target/release/automap` (NAT traversal utility)

### Pre-built Releases
Pre-compiled binaries are available on the [GitHub Releases](https://github.com/MASQ-Project/Node/releases/latest) page, as well as CI build artifacts generated on [GitHub Actions](https://github.com/MASQ-Project/Node/actions).

---

## Quick Start Guide

### 1. Starting the MASQ Daemon

The Daemon must be started with administrative privileges to manage port bindings and initialization:

**Linux / macOS:**
```bash
sudo nohup ./target/release/MASQNode --initialization &
```

**Windows (Administrator Command Prompt):**
```cmd
start /b MASQNode.exe --initialization
```

The Daemon begins listening on `127.0.0.1:5333` for UI and CLI connections.

### 2. Setting up Wallets and Password

In a separate terminal, launch the `masq` interactive CLI:
```bash
./target/release/masq
```

Set your database encryption password:
```text
masq> set-password
```

Create or import your cryptocurrency wallets:
- **Generate new wallets:**
  ```text
  masq> generate-wallets
  ```
  *(Save the generated 24-word BIP-39 mnemonic seed phrase in a safe location!)*
- **Recover existing wallets:**
  ```text
  masq> recover-wallets
  ```

### 3. Configuring and Starting the Node

Configure your node initialization settings:
```text
masq> setup --chain base-mainnet --blockchain-service-url https://mainnet.base.org --ip <YOUR_PUBLIC_IP> --clandestine-port 9342
```

Provide seed neighbors if connecting to an existing network:
```text
masq> setup --neighbors "masq://base-mainnet:ZjPLnb9RrgsRM1D9edqH8jx9DkbPZSWqqFqLnmdKhsk@112.55.78.0:7878"
```

Start the Node service:
```text
masq> start
```

Verify connection status:
```text
masq> connection-status
masq> descriptor
```

### 4. Routing Traffic (Proxy & DNS Subversion)

To browse through your MASQ Node:
1. **Configure Browser HTTP Proxy:**
   Set your browser's HTTP/HTTPS proxy to:
   - Host: `127.0.0.1`
   - Port: Proxy port displayed in node startup logs (or configured in setup).
2. **Subvert System DNS (Optional):**
   Direct all DNS lookups through the MASQ Node's encrypted mesh resolver:
   ```bash
   sudo ./target/release/dns_utility subvert
   ```
   Verify DNS status:
   ```bash
   ./target/release/dns_utility status
   ```

### 5. Clean Shutdown

1. Revert DNS configuration:
   ```bash
   sudo ./target/release/dns_utility revert
   ```
2. Stop the Node from `masq`:
   ```text
   masq> shutdown
   ```
3. Disable any manual proxy settings in your browser or operating system.

---

## Configuration Reference

### Configuration Priority

MASQ Node resolves configuration options in the following order of precedence (highest to lowest):
1. **Interactive / Non-interactive UI Command (`masq setup ...`)**
2. **Shell Environment Variables (`MASQ_*`)**
3. **Configuration File (`config.toml`)**
4. **Compiled Defaults**

### Configuration Parameters

| Parameter | Environment Variable | Config File Key | Default | Description |
| :--- | :--- | :--- | :--- | :--- |
| `--blockchain-service-url` | `MASQ_BLOCKCHAIN_SERVICE_URL` | `blockchain-service-url` | *None* (Required) | JSON-RPC endpoint for Ethereum/Base/Polygon. |
| `--chain` | `MASQ_CHAIN` | `chain` | `base-mainnet` | Blockchain network (`base-mainnet`, `polygon-mainnet`, `eth-mainnet`, `base-sepolia`, `polygon-amoy`). |
| `--ip` | `MASQ_IP` | `ip` | *None* (Required) | External public IPv4 address of the Node. |
| `--clandestine-port` | `MASQ_CLANDESTINE_PORT` | `clandestine-port` | Dynamic | External listening port for Gossip and mesh traffic. |
| `--neighbors` | `MASQ_NEIGHBORS` | `neighbors` | *None* | Comma-separated list of peer node descriptors. |
| `--data-directory` | `MASQ_DATA_DIRECTORY` | `data-directory` | OS App Data | Persistent storage directory for database and configuration. |
| `--config-file` | `MASQ_CONFIG_FILE` | *N/A* | `config.toml` | Path to persistent TOML configuration file. |
| `--db-password` | `MASQ_DB_PASSWORD` | `db-password` | *None* | Encryption password for database and secret keys. |
| `--earning-wallet` | `MASQ_EARNING_WALLET` | `earning-wallet` | *None* | Hex address (`0x...`) to receive routing rewards. |
| `--consuming-private-key` | `MASQ_CONSUMING_PRIVATE_KEY` | `consuming-private-key` | *None* | 64-character private key for paying network fees (testing). |
| `--gas-price` | `MASQ_GAS_PRICE` | `gas-price` | `30000000000` | Gas price (in wei) for on-chain settlement transactions. |
| `--min-hops` | `MASQ_MIN_HOPS` | `min-hops` | `3` | Minimum number of routing hops across mesh. |
| `--neighborhood-mode` | `MASQ_NEIGHBORHOOD_MODE` | `neighborhood-mode` | `standard` | Mode of operation: `standard` (mesh) or `zero-hop` (standalone/testing). |
| `--mapping-protocol` | `MASQ_MAPPING_PROTOCOL` | `mapping-protocol` | `upnp` | Router port mapping protocol (`upnp`, `natpmp`, `pcp`, or `off`). |
| `--dns-servers` | `MASQ_DNS_SERVERS` | `dns-servers` | `1.1.1.1,8.8.8.8` | Upstream DNS servers used by exit nodes. |
| `--log-level` | `MASQ_LOG_LEVEL` | `log-level` | `info` | Logging verbosity (`trace`, `debug`, `info`, `warn`, `error`). |
| `--ui-port` | `MASQ_UI_PORT` | `ui-port` | `5333` | Localhost port for Daemon UI WebSocket connections. |
| `--real-user` | `MASQ_REAL_USER` | `real-user` | Current user | User identity to assume after dropping privileges. |
| `--new-public-key` | `MASQ_NEW_PUBLIC_KEY` | `new-public-key` | `off` | Force regeneration of node identity public key (`on`/`off`). |
| `--scans` | `MASQ_SCANS` | `scans` | `on` | Toggle periodic scans for payables and deadbeat peers. |

### Node Descriptor Format

Node descriptors follow the format:
```text
masq://<chain-identifier>:<base64-public-key>@<ip-address>:<clandestine-port>
```
Example:
```text
masq://base-mainnet:ZjPLnb9RrgsRM1D9edqH8jx9DkbPZSWqqFqLnmdKhsk@112.55.78.0:7878
```

### Sample `config.toml`

Place this file in your data directory (or point to it via `--config-file`):

```toml
chain = "base-mainnet"
blockchain-service-url = "https://mainnet.base.org"
ip = "203.0.113.195"
clandestine-port = 9342
log-level = "info"
min-hops = 3
mapping-protocol = "upnp"
dns-servers = "1.1.1.1,8.8.8.8"
neighbors = "masq://base-mainnet:ZjPLnb9RrgsRM1D9edqH8jx9DkbPZSWqqFqLnmdKhsk@112.55.78.0:7878"
```

---

## CLI Reference (`masq`)

The `masq` utility can be operated interactively (launching `masq` without arguments) or non-interactively (`masq <subcommand> [flags]`):

| Subcommand | Description |
| :--- | :--- |
| `setup` | Pre-configure Daemon initialization parameters before starting the Node. |
| `start` | Instruct the Daemon to launch the Node process. |
| `shutdown` | Gracefully shut down the running Node and/or Daemon. |
| `configuration` | View the current active configuration values. |
| `set-configuration` | Dynamically update runtime configuration parameters on a live Node. |
| `connection-status` | Display status of peer connections, active hops, and network health. |
| `descriptor` | Print the local Node's network descriptor for sharing with neighbors. |
| `exit-location` | View or specify preferred exit node geographic locations. |
| `financials` | Inspect accounts payable, accounts receivable, and token balances. |
| `generate-wallets` | Generate new mnemonic seed and derived earning/consuming wallets. |
| `recover-wallets` | Import existing wallets via BIP-39 mnemonic seed. |
| `wallet-addresses` | Display configured public addresses for earning and consuming wallets. |
| `set-password` | Set the initial database symmetric encryption password. |
| `change-password` | Update existing database encryption password. |
| `check-password` | Verify if the supplied password can decrypt the local database. |
| `get-neighborhood-graph` | Output an ASCII representation of known mesh network topology. |
| `scan` | Trigger an immediate manual scan for payables or peer status. |

---

## Testing

The codebase includes extensive unit, integration, and multi-node network tests:

```bash
# Run unit tests across all workspace crates
cargo test --workspace

# Run tests for a specific crate
cargo test -p node
cargo test -p masq
cargo test -p dns_utility

# Run multi-node integration tests (requires Docker)
./ci/multinode_integration_test.sh

# Run code format and lint checks
./ci/format.sh
./ci/lint.sh
```

> [!NOTE]
> Multi-node integration tests use the Docker network `integration_net` and must be run serially (`--test-threads=1`).

---

## Troubleshooting & Diagnostics

### Port 53 Binding Failures

If you encounter:
```text
thread 'main' panicked at 'Cannot bind socket to V4(0.0.0.0:53): Address already in use'
```
Another local DNS resolver (such as `systemd-resolved` on Linux or `Internet Connection Sharing` on Windows) is occupying port 53.
- **Linux (`systemd-resolved`):** Disable stub listener by adding `DNSStubListener=no` to `/etc/systemd/resolved.conf` and restart `systemd-resolved`.
- **See comprehensive guide:** [`node/docs/PORT_53.md`](node/docs/PORT_53.md).

### TLS Alerts & Routing Errors

Because MASQ Node operates below the application layer, it uses synthetic TLS Alerts during handshake negotiation to notify the browser of connection issues:
- **`internal_error` (Routing Failure):** The local node has not yet discovered enough credible peers to assemble a path satisfying `--min-hops`. Wait 1-2 minutes for Gossip discovery to populate the neighborhood table.
- **`unrecognized_name` (DNS Failure):** The requested hostname could not be resolved across exit nodes. Verify spelling or check if the exit node has unrestricted DNS access.

### Router NAT & Port Forwarding

MASQ nodes require incoming reachability on the configured clandestine port:
- Check if your router supports UPnP or NAT-PMP (enabled by default via `--mapping-protocol upnp`).
- Test router capability using the built-in `automap` test tool:
  ```bash
  cargo run --bin automap
  ```
- If UPnP fails, log into your router administration page and manually forward your clandestine port (e.g., `9342`) to your computer's local IPv4 address.

---

## Component Documentation

For in-depth developer documentation of internal crates and modules, refer to:
- [Blockchain Service Configuration](node/docs/Blockchain-Service.md)
- [Port 53 Resolution Guide](node/docs/PORT_53.md)
- [Daemon & UI IPC Protocol Specification](USER-INTERFACE-INTERFACE.md)
- [`node/src/accountant`](node/src/accountant/README.md)
- [`node/src/entry_dns`](node/src/entry_dns/README.md)
- [`node/src/hopper`](node/src/hopper/README.md)
- [`node/src/neighborhood`](node/src/neighborhood/README.md)
- [`node/src/proxy_client`](node/src/proxy_client/README.md)
- [`node/src/proxy_server`](node/src/proxy_server/README.md)
- [`node/src/ui_gateway`](node/src/ui_gateway/README.md)
- [`dns_utility`](dns_utility/README.md)
- [`multinode_integration_tests`](multinode_integration_tests/tests/README.md)

---

## Origin & Attribution

The MASQ project was originally forked from Substratum's Node project in October 2019 to maintain and advance decentralized mesh technology after Substratum ceased operations. All credit for the foundational idea and architectural design belongs to Substratum Services, Inc., preserved under the open-source GPL-3.0 license.

---

## License

This project is licensed under the **GNU General Public License v3.0** (`GPL-3.0-only`). See the [LICENSE](LICENSE) file for details.

Copyright (c) 2019-2024, MASQ Network and/or its affiliates. All rights reserved.
