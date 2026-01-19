# WordPress Droplet Stability Bootstrap (DigitalOcean)

This repo contains a cloud-init bootstrap and optional shell script to harden DigitalOcean WordPress 1-Click droplets running on the 1GB Basic (shared CPU) plan.

The goal is simple: prevent random crashes caused by out-of-memory (OOM) kills on small droplets.

This setup is based on real-world failure patterns where MySQL and Apache are repeatedly killed under memory pressure.

---

## What This Solves

On 1GB WordPress droplets, it’s common to see:

- MySQL (mysqld) killed by the kernel
- Apache workers getting OOM-killed
- White screens or “Error establishing database connection”
- Droplet becoming unresponsive until a reboot

This bootstrap applies safe, conservative defaults that dramatically improve stability without changing the WordPress stack.

---

## What It Configures

### System-level
- Creates a 2GB swapfile at /swapfile
- Persists swap in /etc/fstab
- Sets vm.swappiness = 10
- Disables packagekit to prevent background memory spikes

### MySQL (low-memory safe defaults)
- innodb_buffer_pool_size = 128M
- Caps temp tables and heap usage
- Limits max connections
- Uses a dedicated override file at /etc/mysql/conf.d/low-mem.cnf

### Apache (prefork MPM)
- Limits concurrency to prevent RAM stampedes
- Conservative worker counts suitable for 1GB RAM
- Restarts Apache to apply limits

### Logging
- Writes a full execution log to:
  /var/log/wp-stability-init.log

---

## Files

### wp-stability.cloud-init.yaml
Primary file.

Paste this into:
DigitalOcean → Create Droplet → Advanced Options → User Data

Runs automatically on first boot.

### wp-stability.sh (optional)
Standalone version of the same logic, useful for:
- running manually on existing droplets
- debugging
- reapplying settings after changes

---

## How to Use (Recommended)

### New Droplet

1. Create a new WordPress 1-Click droplet
2. Choose Basic → 1GB shared plan
3. Expand Advanced Options
4. Paste the contents of wp-stability.cloud-init.yaml into User Data
5. Create the droplet

No SSH required.

---

## How to Verify After Boot

SSH into the droplet and run:

    free -h
    swapon --show
    cat /proc/sys/vm/swappiness

You should see:
- Approximately 2GB swap enabled
- vm.swappiness set to 10
- Significantly more available memory than a default 1-Click droplet

Check the bootstrap log:

    sudo tail -n 200 /var/log/wp-stability-init.log

---

## Why These Defaults

These values are intentionally conservative and based on observed OOM behavior:

- 128MB InnoDB buffer pool keeps MySQL inside a hard memory fence
- Swap larger than RAM gives the kernel room to breathe
- Low Apache worker limits prevent PHP processes from stampeding memory
- Swappiness of 10 avoids swap thrashing while preventing panic kills

This setup prioritizes stability over raw performance.

---

## When to Upgrade the Droplet

This configuration makes a 1GB droplet stable, not fast.

Consider upgrading to 2GB if:
- traffic increases
- you add heavy plugins (Wordfence, WooCommerce, page builders)
- you want more headroom and fewer constraints

The same bootstrap works on larger droplets.

---

## Assumptions

- Ubuntu-based DigitalOcean WordPress 1-Click image
- Apache + MySQL
- systemd
- Root access (cloud-init runs as root)

If you move to Nginx + PHP-FPM, this should be adjusted.

---

## Philosophy

On small servers, stability beats cleverness.

This bootstrap trades peak throughput for predictability and uptime, which is exactly what you want on 1GB WordPress droplets.
