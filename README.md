# 🏠 Homelab Infrastructure

A multi-node homelab built for quantitative market analysis, ML inference, and media serving. This repo documents the architecture, configuration patterns, and lessons learned from building a distributed compute environment on consumer hardware.

> **Note:** This repo contains infrastructure documentation and sanitized configuration templates. The application layer (trading platform, analytics dashboard, ML pipelines) lives in private repos.

---

## Architecture Overview

```
┌─────────────────────────────────────────────────────────────────┐
│                        Home Network                             │
│                    192.168.77.0/24 LAN                          │
│                                                                 │
│  ┌─────────────────┐     ┌──────────────┐    ┌──────────────┐  │
│  │   NAS (Proxmox) │     │ NUC-Compute  │    │ NUC-Compute  │  │
│  │                  │     │   (Desktop)  │    │   (Server)   │  │
│  │  VM 101: Ubuntu  │     │              │    │              │  │
│  │    • App server  │     │  • Dashboard │    │  • Execution │  │
│  │    • API gateway │     │  • Research  │    │  • Scheduling│  │
│  │                  │     │  • 8TB NVMe  │    │  • 4TB NVMe  │  │
│  │  VM 102: OMV     │     └──────┬───────┘    └──────┬───────┘  │
│  │    • ZFS pool    │            │ 1G                │ 10G      │
│  │    • TimescaleDB │            │                   │          │
│  │    • NFS exports │     ┌──────┴───────┐    ┌──────┴───────┐  │
│  └────────┬─────────┘     │   Jetson     │    │    Pi 4B     │  │
│           │ 10G           │  Xavier AGX  │    │   Monitor    │  │
│           │               │              │    │              │  │
│           │               │  • ML / LLM  │    │  • Health    │  │
│           │               │  • 8TB NVMe  │    │    polling   │  │
│           │               └──────┬───────┘    └──────────────┘  │
│           │                      │ 1G                           │
│  ┌────────┴──────────────────────┴──────────────────────────┐   │
│  │              Netgear GS110EMX 10G Switch                 │   │
│  │         Ports 9-10: 10G  |  Ports 1-8: 1G               │   │
│  └──────────────────────────────────────────────────────────┘   │
│                                                                 │
│  Private link: NAS ↔ Jetson (10.10.20.0/24) for bulk ML data   │
└─────────────────────────────────────────────────────────────────┘
```

## Hardware

| Device | CPU | RAM | Storage | Network | Role |
|--------|-----|-----|---------|---------|------|
| NAS (Custom) | — | 32GB | 8×SATA SSD (RAIDZ2) + 4TB NVMe | 10GbE + 1G backup | Proxmox host, storage, database |
| NUC (Visualizer) | Intel i7 | 64GB | 8TB NVMe + 4TB TB3 | 1GbE | Dashboard, research, backtesting |
| NUC (Trader) | Intel i7 | 32GB | 4TB NVMe | 10GbE + 1G backup | Strategy execution, real-time feeds |
| Jetson Xavier AGX | 8-core ARM + 512 CUDA | 32GB | 4TB NVMe + 4TB SATA | 1GbE + 10G P2P | ML training & inference |
| Raspberry Pi 4B | BCM2711 | 4GB | 32GB microSD | 1GbE | Network health monitoring |

## Storage Architecture

### ZFS Pool (NAS — OMV VM)

```
quantpool (RAIDZ2, 8×SATA)
├── market_data_daily/     # Daily OHLCV bars
├── market_data_minute/    # Minute-level bars
├── market_data_tick/      # Tick data (when available)
├── backtests/             # Backtest artifacts (RW)
└── models/                # ML model checkpoints (RW)

Config: compression=lz4, atime=off, xattr=sa, recordsize=128k
```

### NFS Exports

Market data is exported **read-only** to compute nodes. Backtest artifacts and model storage are read-write.

```bash
# On compute nodes (/etc/fstab):
# Market data — READ ONLY (enforced at mount)
<nas-ip>:/export/market_data_daily   /mnt/quantdata/daily    nfs4  defaults,ro,nofail  0 0
<nas-ip>:/export/market_data_minute  /mnt/quantdata/minute   nfs4  defaults,ro,nofail  0 0

# Artifacts — READ WRITE
<nas-ip>:/export/backtests           /mnt/backtests           nfs4  defaults,nofail     0 0
<nas-ip>:/export/models              /mnt/quantdata/models    nfs4  defaults,nofail     0 0
```

**Design principle:** Market data is the source of truth — no compute node can accidentally write to it. Only the data ingestion pipeline on the NAS VM has write access.

### NVMe Tiering

| Tier | Location | Purpose |
|------|----------|---------|
| Hot | NAS 1TB NVMe L2ARC | ZFS read cache for frequent time-series queries |
| Warm | NAS 3TB NVMe | Active time-series data, VM disks |
| Local hot | Each NUC 4-8TB NVMe | Node-local datasets, execution logs, research data |
| Archive | ZFS RAIDZ2 pool | Long-term storage, cold data |

## Proxmox Configuration

### VM Layout

```
Proxmox Host (bare metal)
├── VM 101: Ubuntu Server (app server + API)
│   └── 10 cores, 16GB RAM, 50GB disk
├── VM 102: OpenMediaVault (storage management)
│   └── 6 cores, 16GB RAM, 50GB disk
│   └── ZFS pool passthrough
│   └── Docker: TimescaleDB container
└── Resources: ~6GB RAM for Proxmox itself
    Ballooning enabled on both VMs
```

### TimescaleDB (Docker on OMV)

```yaml
# docker-compose.yml (sanitized)
version: '3.8'
services:
  timescaledb:
    image: timescale/timescaledb-ha:pg16-all
    container_name: timescaledb_service
    restart: unless-stopped
    environment:
      - POSTGRES_USER=${DB_USER}
      - POSTGRES_PASSWORD=${DB_PASSWORD}
      - POSTGRES_DB=${DB_NAME}
    ports:
      - "5432:5432"
    volumes:
      # Map to ZFS dataset for data safety
      - /quantpool/docker-data/timescaledb:/var/lib/postgresql/data
```

**Why TimescaleDB on ZFS?** The database volume maps directly to the ZFS pool, so we get compression, snapshots, and RAIDZ2 redundancy for free. No separate backup strategy needed for the DB — ZFS snapshots handle it.

## Network Design

### 10G Backbone

The NAS and primary compute node (Trader) connect via 10GbE for low-latency data access. The Jetson has a dedicated private 10G point-to-point link to the NAS for bulk ML dataset transfers without saturating the main switch.

### Switch Port Map (Netgear GS110EMX)

| Port | Speed | Device |
|------|-------|--------|
| 1 | 1G | Pi 4B (monitor) |
| 2 | 1G | Jetson Xavier AGX |
| 3 | 1G | GL.iNet KVM |
| 4 | 1G | Peplink router |
| 5 | 1G | NUC-Trader (backup) |
| 6 | 1G | NAS (backup) |
| 7 | 1G | NUC-Visualizer |
| 8 | 1G | Aux / Laptop |
| 9 | 10G | NUC-Trader |
| 10 | 10G | NAS |

### Remote Access

Tailscale mesh network for remote access to all nodes. No port forwarding, no exposed services.

## Docker Compose Templates

See the [`/templates`](./templates/) directory for sanitized Docker Compose files:

- `timescaledb/` — TimescaleDB on ZFS
- `media-server/` — Plex + *arr stack (see [media-server-stack](https://github.com/gman00910/media-server-stack))
- `monitoring/` — Tautulli, Homepage, Watchtower

## Lessons Learned

1. **ZFS + NFS + read-only mounts** is a simple but effective data governance pattern. Prevents "oops I wrote to the wrong path" bugs that corrupt market data.

2. **Proxmox VM separation** (compute vs. storage) lets you snapshot/rollback the app server without touching the database, and vice versa.

3. **Dedicated P2P links** for bulk data transfers (NAS ↔ Jetson) keep your main network clean. A single ML training job can saturate 10G — you don't want that competing with your real-time data feeds.

4. **TimescaleDB on ZFS** gives you time-series performance with filesystem-level redundancy. Continuous aggregates + hypertable chunking handle the transition from minute bars to daily rollups cleanly.

5. **Start with 1G, upgrade to 10G where it hurts.** Most nodes are fine on 1GbE. Only the data-heavy paths (NAS ↔ Trader, NAS ↔ Jetson) justified the 10G upgrade.

## What This Powers

The infrastructure supports a private quantitative trading platform with:
- Real-time market data ingestion and storage (~50K+ FRED observations, multi-timeframe OHLCV)
- Full-stack analytics dashboard (FastAPI + React + TypeScript)
- Backtesting engines with artifact storage
- ML inference pipeline (LLM + FinBERT on Jetson GPU)
- Network-wide health monitoring

The application repos are private, but the infrastructure patterns here are broadly applicable to any homelab running distributed data-intensive workloads.

---

## License

MIT — use whatever's useful to you.
