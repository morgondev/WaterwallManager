# WaterWall Tunnel Manager

A comprehensive bash script for deploying and managing [WaterWall](https://github.com/radkesvat/WaterWall) tunnels on Ubuntu/Debian servers.

## Quick Install

```bash
bash <(curl -fsSL https://raw.githubusercontent.com/morgondev/waterwall/main/waterwall.sh)
```

> **Read before running:** this is a root-level server management script. It downloads a WaterWall release from GitHub, writes `/root/waterwall/core.json` and `/root/waterwall/config.json`, creates/restarts a `waterwall.service` systemd unit, and may install packages with `apt`.

## Safety Notes

- Review the script before piping it directly into `bash`, especially on production VPS instances.
- The tunnel modes use TUN devices, raw sockets, routing/firewall changes, and systemd service management. Run it only on a server dedicated to this tunnel or one you are prepared to reconfigure.
- The **Optimize Server** option is intentionally invasive: it rewrites `/etc/sysctl.conf`, appends limits/profile settings, loads kernel modules, installs `ethtool`, and creates `waterwall-tune.service`.
- The script auto-selects the WaterWall binary by CPU architecture and AVX2/ARM feature detection, but it currently trusts GitHub release assets and does not verify checksums or signatures.
- Public IP auto-detection uses local interface addresses from `hostname -I`; confirm the detected address if your VPS has private/NAT interfaces.

## Features

### Tunnel Modes

| Mode | Description | Use Case |
|------|-------------|----------|
| **BitSwap** | Layer 3 raw socket tunnel with TCP flag manipulation, XOR obfuscation, and MUX multiplexing. Iran initiates connection to Kharej. | General purpose, high-performance tunneling |
| **Reverse Reality** | TLS-based reverse tunnel using Reality protocol. Kharej initiates connection to Iran with domain-fronting handshake. | When Iran outbound is restricted |
| **PacketTunnel (Classic)** | Simple Layer 3 packet tunnel with protocol swap. Lightweight with per-port TCP forwarding. | Basic tunneling with minimal overhead |

### Service Management

- **Restart / Status** - Control the WaterWall systemd service
- **Test Tunnel** - Ping test through the tunnel (10.10.0.2)
- **Change Ports** - Modify listening/connecting ports without reinstalling (supports BitSwap, Reverse Reality, and Classic configs)
- **iPerf3 Speed Test** - Built-in speed test between tunnel endpoints (auto-installs iperf3)
- **MTU Discovery** - Automatically finds optimal MTU using binary search, updates `core.json`
- **Uninstall** - Clean removal with option to keep the binary

### Server Optimization

One-click server tuning for tunnel performance:

- **BBR + fq** - Enables BBR congestion control with `fq` qdisc (optimal pairing for BBR)
- **TCP Tuning** - Enlarged buffers (33MB max), fast open, MTU probing, disabled slow start after idle
- **Network Stack** - Increased backlog, netdev budget, conntrack table (1M entries)
- **Tunnel Interface** - Disables GRO/GSO/TSO offloading on tunnel interfaces, increases txqueuelen
- **System Limits** - Sets nofile to 1M, removes stack/core/nproc limits
- **Persistent** - Creates `waterwall-tune.service` for post-boot interface optimization
- **Versioned** - Tracks optimization version, auto-upgrades old versions without prompting

### Update Core

Checks GitHub for the latest WaterWall release and updates the binary in-place. Automatically detects CPU architecture (x86_64/ARM64) and AVX2 support to download the correct build.

## How It Works

### Architecture

```
                    Internet
                       |
    [Iran Server] ---- Raw Socket Tunnel ---- [Kharej Server]
         |            (BitSwap/Classic)              |
    TcpListener                               TcpConnector
    (User ports)                              (Xray/Service)
         |                                          |
    10.10.0.1  <--- Virtual Network --->  10.10.0.2
```

### BitSwap Pipeline

```
User Traffic (Layer 4):
  TcpListener --> HeaderClient --> MuxClient --> TcpConnector(10.10.0.2)

Packet Tunnel (Layer 3):
  TunDevice --> IpOverrider --> PacketSplitStream --> ObfuscatorClient (XOR)
                                                 --> IpManipulator (TCP flag swap)
                                                 --> RawSocket
```

**What makes BitSwap work:**
- **IpOverrider** - Translates between real IPs and virtual tunnel IPs (10.10.0.x)
- **PacketSplitStream** - Splits packet flow into upload/download branches
- **ObfuscatorClient/Server** - XOR encryption on packet payload
- **IpManipulator** - Swaps TCP flags (PSH/CWR, PSH/RST) to disguise traffic patterns
- **MuxClient/Server** - Multiplexes multiple user connections into fewer tunnel connections
- **HeaderClient/Server** - Preserves original port information across the tunnel

## Configuration Files

| File | Location | Purpose |
|------|----------|---------|
| `core.json` | `/root/waterwall/core.json` | WaterWall core settings (MTU, workers, logging) |
| `config.json` | `/root/waterwall/config.json` | Tunnel node configuration |
| `waterwall.service` | `/etc/systemd/system/` | Systemd service unit |
| `waterwall-tune.service` | `/etc/systemd/system/` | Post-boot interface tuning (after optimization) |

## Requirements

- **OS:** Ubuntu or Debian
- **Access:** Root
- **Dependencies:** `curl`, `jq`, `unzip`, `iptables` (auto-installed)

## Supported Architectures

| Architecture | Standard Build | Old CPU Build |
|---|---|---|
| x86_64 / amd64 | AVX2 supported | No AVX2 |
| aarch64 / arm64 | SHA2+AES features | Older ARM |

CPU capability is **auto-detected** - no manual selection needed.

## Support

Telegram Channel: [@morgondev](https://t.me/morgondev)

## License

This script is a management wrapper for [WaterWall](https://github.com/radkesvat/WaterWall) by radkesvat.
