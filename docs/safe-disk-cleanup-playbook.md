# Safe Disk Cleanup Playbook

A practical runbook to free disk space on a Linux VPS safely without breaking running services (Docker, systemd-journald, Snap, APT).

## Table of contents
- [A) Scope / What this is for](#a-scope--what-this-is-for)
- [B) Quick checklist: what usually eats disk](#b-quick-checklist-what-usually-eats-disk)
- [C) Diagnostics (no deletion)](#c-diagnostics-no-deletion)
- [D) Safe cleanup steps](#d-safe-cleanup-steps)
- [E) Final control checklist](#e-final-control-checklist)
- [F) Dangerous commands (avoid unless you fully understand impact)](#f-dangerous-commands-avoid-unless-you-fully-understand-impact)
- [G) Prevention / Keeping disk healthy](#g-prevention--keeping-disk-healthy)
- [H) One-shot diagnostics mini script](#h-one-shot-diagnostics-mini-script)
- [I) Healthy target metrics](#i-healthy-target-metrics)

## A) Scope / What this is for
Use this when a Linux VPS (Ubuntu/Debian) is running low on disk and you need safe, reversible cleanup steps. This is intended for hosts running Docker containers and systemd-journald, and may also have Snap and APT caches.

## B) Quick checklist: what usually eats disk ✅
- Docker images, layers, and build cache.
- Journald logs under `/var/log/journal`.
- APT package caches and old packages.
- Snap disabled revisions.
- Large logs in `/var/log` (app logs, nginx, database logs).

## C) Diagnostics (no deletion)
Run these first to locate the biggest consumers.

```bash
# Filesystem usage

df -h
lsblk -f

# Top-level usage (stay on the same filesystem)

du -xhd1 /
du -xhd1 /var
du -xhd1 /var/lib

# Find large files in /var

find /var -type f -size +200M -print

# Docker disk usage (if docker is installed)

if command -v docker >/dev/null 2>&1; then
  docker system df -v
else
  echo "docker not installed"
fi

# Journald disk usage

journalctl --disk-usage
```

## D) Safe cleanup steps
Perform these in order, verifying after each phase.

### 1) Docker images and build cache 🧹
```bash
# Remove dangling images

docker image prune -f

# Remove build cache

docker builder prune -f
```

✅ Verification
```bash
df -h
if command -v docker >/dev/null 2>&1; then
  docker system df
else
  echo "docker not installed"
fi
journalctl --disk-usage
```

### 2) Journald vacuum
```bash
# Keep logs to a maximum size

journalctl --vacuum-size=200M

# Or keep only the last 7 days

journalctl --vacuum-time=7d
```

⚠️ Do not truncate `/var/log/journal` (it is a directory). Use `journalctl` vacuum instead.

✅ Verification
```bash
df -h
if command -v docker >/dev/null 2>&1; then
  docker system df
else
  echo "docker not installed"
fi
journalctl --disk-usage
```

### 3) APT cleanup
```bash
apt-get clean
apt-get autoclean
apt-get autoremove --purge -y
```

✅ Verification
```bash
df -h
if command -v docker >/dev/null 2>&1; then
  docker system df
else
  echo "docker not installed"
fi
journalctl --disk-usage
```

### 4) Snap disabled revisions
```bash
# Remove disabled snap revisions safely

snap list --all | awk '/disabled/{print $1, $3}' | while read -r snapname revision; do
  snap remove "$snapname" --revision="$revision"
done
```

✅ Verification
```bash
df -h
if command -v docker >/dev/null 2>&1; then
  docker system df
else
  echo "docker not installed"
fi
journalctl --disk-usage
```

### 5) /var/log review and targeted truncation
```bash
# Identify large logs

du -sh /var/log/* | sort -h

# Example: truncate a specific large log file

truncate -s 0 /var/log/your-app.log

# Force logrotate if needed

logrotate -f /etc/logrotate.conf
```

⚠️ Only truncate specific large log files you identify. Avoid deleting log directories used by services.

✅ Verification
```bash
df -h
if command -v docker >/dev/null 2>&1; then
  docker system df
else
  echo "docker not installed"
fi
journalctl --disk-usage
```

## E) Final control checklist ✅
- Services are still running (Docker containers, databases, proxies).
- Disk usage is below target thresholds.
- No critical logs or data were removed.
- Journald usage is stable after vacuum.

## F) Dangerous commands (avoid unless you fully understand impact) ⚠️
These can remove data you might need to keep.

```bash
# Removes all images, including in-use tags

docker system prune -a

# Removes unused volumes (can include data)

docker system prune --volumes

# Nukes all docker data on the host

rm -rf /var/lib/docker
```

## G) Prevention / Keeping disk healthy
- Configure journald limits in `/etc/systemd/journald.conf`:
  - `SystemMaxUse=200M`
  - `RuntimeMaxUse=50M`
- Then restart journald:

```bash
systemctl restart systemd-journald
```

- Optional: monitor Docker JSON log size limits per container (`/var/lib/docker/containers/*/*-json.log`).

## H) One-shot diagnostics mini script
```bash
#!/usr/bin/env bash
set -euo pipefail

echo "## df -h"
df -h

echo "## lsblk -f"
lsblk -f

echo "## du -xhd1 /"
du -xhd1 /

echo "## du -xhd1 /var"
du -xhd1 /var

echo "## du -xhd1 /var/lib"
du -xhd1 /var/lib

echo "## find /var -size +200M"
find /var -type f -size +200M -print

echo "## docker system df -v"
if command -v docker >/dev/null 2>&1; then
  docker system df -v
else
  echo "docker not installed"
fi

echo "## journalctl --disk-usage"
journalctl --disk-usage
```

## I) Healthy target metrics 📌
- Root filesystem (`/`) under 80% usage.
- Journald disk usage between 50–200 MB.
- Docker shows minimal or no dangling images and build cache.
- No single log file growing unexpectedly.
