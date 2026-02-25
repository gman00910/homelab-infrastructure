# ZFS Quick Reference

Configuration notes for a RAIDZ2 pool backing a homelab data warehouse.

## Pool Creation

```bash
# RAIDZ2 with 8 drives — survives 2 simultaneous disk failures
zpool create quantpool raidz2 \
  /dev/disk/by-id/ata-... \
  /dev/disk/by-id/ata-... \
  /dev/disk/by-id/ata-... \
  /dev/disk/by-id/ata-... \
  /dev/disk/by-id/ata-... \
  /dev/disk/by-id/ata-... \
  /dev/disk/by-id/ata-... \
  /dev/disk/by-id/ata-...
```

Always use `/dev/disk/by-id/` paths, never `/dev/sdX`. Drive letters change across reboots.

## Pool-Level Settings

```bash
zfs set compression=lz4 quantpool        # ~2x savings on structured data, negligible CPU
zfs set atime=off quantpool              # Don't update access times (big perf win for DBs)
zfs set xattr=sa quantpool               # Store extended attrs in inodes (faster)
zfs set recordsize=128k quantpool         # Good default; tune per-dataset if needed
```

## Dataset Layout

```bash
zfs create quantpool/market_data_daily
zfs create quantpool/market_data_minute
zfs create quantpool/market_data_tick
zfs create quantpool/backtests
zfs create quantpool/models
zfs create quantpool/docker-data          # Container persistent storage
```

### Per-Dataset Tuning (optional)

```bash
# TimescaleDB benefits from 8K records matching PG page size
zfs set recordsize=8k quantpool/docker-data

# Tick data is append-heavy, large records are fine
zfs set recordsize=1m quantpool/market_data_tick
```

## L2ARC (NVMe Read Cache)

```bash
# Add NVMe as L2ARC (read cache) — helps with random reads on time-series queries
zpool add quantpool cache /dev/disk/by-id/nvme-...
```

L2ARC warms up over time. Don't expect instant improvement — it learns your access patterns.

## Snapshots

```bash
# Manual snapshot before risky operations
zfs snapshot quantpool/market_data_daily@before-migration

# List snapshots
zfs list -t snapshot

# Rollback (destroys changes since snapshot)
zfs rollback quantpool/market_data_daily@before-migration

# Auto-snapshot (if using zfs-auto-snapshot)
# Keeps: 4 frequent (15m), 24 hourly, 7 daily, 4 weekly, 12 monthly
```

## Monitoring

```bash
# Pool health
zpool status quantpool

# Space usage per dataset
zfs list -o name,used,avail,compressratio

# Compression savings
zfs get compressratio quantpool

# Scrub (data integrity check — run monthly)
zpool scrub quantpool
zpool status quantpool  # Watch scrub progress
```

## Proxmox Integration

When running ZFS under Proxmox:
- Pass the SATA controller through to the OMV VM, or
- Create the ZFS pool on the Proxmox host and pass datasets to VMs via bind mounts

I chose the OMV VM approach — OMV manages the pool and exports via NFS/SMB. Proxmox just sees the VM.
