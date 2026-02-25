# NFS Configuration Guide

How I set up NFS exports for a multi-node homelab with read-only data governance.

## Design Principle

Market data (or any source-of-truth dataset) should be **read-only** on every node except the one writing it. This prevents accidental corruption from scripts, dev work, or misconfigured jobs.

```
NAS (OMV) ──NFS──> Compute Node 1 (ro)
                ──> Compute Node 2 (ro)
                ──> ML Node (ro)
                ──> Backtest artifacts (rw)
                ──> Model checkpoints (rw)
```

## Server Side (OMV / NFS host)

### 1. Create bind mounts for ZFS datasets

OMV manages NFS shares via bind mounts. In `/etc/fstab`:

```bash
# Bind ZFS datasets to export paths
/pool/market_data_tick/    /export/market_data_tick    none  bind,nofail  0 0
/pool/market_data_minute/  /export/market_data_minute  none  bind,nofail  0 0
/pool/market_data_daily/   /export/market_data_daily   none  bind,nofail  0 0
/pool/backtests/           /export/backtests            none  bind,nofail  0 0
/pool/models/              /export/models               none  bind,nofail  0 0
```

### 2. Set permissions

```bash
# Market data — owned by app user (UID 1000), group users (GID 100)
chown -R 1000:100 /export/market_data_tick /export/market_data_minute /export/market_data_daily
chmod -R 2775 /export/market_data_tick /export/market_data_minute /export/market_data_daily

# Writable shares
chown -R 1000:100 /export/backtests /export/models
chmod -R 2775 /export/backtests /export/models
```

The `2` in `2775` sets the SGID bit, so new files inherit the group. Keeps permissions clean across nodes.

### 3. NFS export config

In OMV, configure NFS shares for your subnet (e.g., `192.168.77.0/24`). All shares use NFSv4.

## Client Side (compute nodes)

### /etc/fstab entries

```bash
# Market data — READ ONLY
<nas-ip>:/export/market_data_tick   /mnt/quantdata/tick     nfs4  defaults,ro,nofail,x-systemd.automount  0 0
<nas-ip>:/export/market_data_minute /mnt/quantdata/minute   nfs4  defaults,ro,nofail,x-systemd.automount  0 0
<nas-ip>:/export/market_data_daily  /mnt/quantdata/daily    nfs4  defaults,ro,nofail,x-systemd.automount  0 0

# Artifacts — READ WRITE
<nas-ip>:/export/backtests          /mnt/backtests           nfs4  defaults,nofail,x-systemd.automount     0 0
<nas-ip>:/export/models             /mnt/quantdata/models    nfs4  defaults,nofail,x-systemd.automount     0 0
```

Key options:
- `ro` — kernel-enforced read-only. Even root can't write.
- `nofail` — boot doesn't hang if NAS is offline
- `x-systemd.automount` — mount on first access, not at boot (faster startup)

### Create mount points and mount

```bash
sudo mkdir -p /mnt/quantdata/{tick,minute,daily} /mnt/backtests /mnt/quantdata/models
sudo mount -a
```

### Verify

```bash
# Should show 'ro' for market data
mount | grep quantdata
# Should fail (read-only filesystem)
touch /mnt/quantdata/daily/test
```

## Docker Integration

When running containers that need NFS data, mount the host paths as volumes:

```yaml
services:
  my-app:
    volumes:
      - /mnt/quantdata:/data:ro       # Market data (enforced RO at both NFS and Docker level)
      - /mnt/backtests:/results        # Artifacts (RW)
```

Double-layered read-only: NFS mount is `ro` AND Docker volume is `:ro`. Belt and suspenders.

## Troubleshooting

```bash
# Check NFS mounts are active
showmount -e <nas-ip>

# Check what's mounted
df -h | grep nfs

# If mount hangs, NAS might be down
# nofail + x-systemd.automount prevents boot issues

# Permission denied? Check UID/GID match across nodes
id                          # On client
ls -ln /export/market_data  # On NAS
```
