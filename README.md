⸻

WordPress Droplet Stability Bootstrap (DigitalOcean)

This repo contains a cloud-init bootstrap and optional shell script to harden DigitalOcean WordPress 1-Click droplets running on the 1GB Basic (shared CPU) plan.

The goal is simple:
👉 Prevent random crashes caused by out-of-memory (OOM) kills on small droplets.

This setup is based on real-world failure patterns where MySQL and Apache are repeatedly killed under memory pressure.

⸻

What This Solves

On 1GB WordPress droplets, it’s common to see:
	•	MySQL (mysqld) killed by the kernel
	•	Apache workers getting OOM-killed
	•	White screens / “Error establishing database connection”
	•	Droplet becoming unresponsive until a reboot

This bootstrap applies safe, conservative defaults that dramatically improve stability without changing the WordPress stack.

⸻

What It Configures

System-level
	•	Creates a 2GB swapfile at /swapfile
	•	Persists swap in /etc/fstab
	•	Sets vm.swappiness = 10 (prefer swap before killing processes)
	•	Disables packagekit (known to spike memory on small servers)

MySQL (low-memory safe defaults)
	•	innodb_buffer_pool_size = 128M
	•	Caps temp tables and heap usage
	•	Limits max connections
	•	Uses a dedicated override file (/etc/mysql/conf.d/low-mem.cnf)

Apache (prefork MPM)
	•	Limits concurrency to prevent RAM stampedes
	•	Conservative worker counts suitable for 1GB RAM
	•	Restarts Apache to apply limits

Logging
	•	Writes a full execution log to:

/var/log/wp-stability-init.log



⸻

Files

wp-stability.cloud-init.yaml

Primary file.
Paste this into DigitalOcean → Create Droplet → Advanced Options → User Data.

This runs automatically on first boot.

(Optional) wp-stability.sh

A standalone version of the same logic, useful for:
	•	running manually on existing droplets
	•	debugging
	•	reapplying settings after changes

⸻

How to Use (Recommended)

New Droplet (Best Option)
	1.	Create a new WordPress 1-Click droplet
	2.	Choose Basic → 1GB shared plan
	3.	Expand Advanced Options
	4.	Paste the contents of wp-stability.cloud-init.yaml into User Data
	5.	Create the droplet

That’s it. No SSH required.

⸻

How to Verify After Boot

SSH into the droplet and run:

free -h
swapon --show
cat /proc/sys/vm/swappiness

You should see:
	•	~2GB swap enabled
	•	vm.swappiness = 10
	•	Significantly more “available” memory than a default 1-Click droplet

Check the bootstrap log:

sudo tail -n 200 /var/log/wp-stability-init.log


⸻

Why These Defaults

These values are intentionally conservative and based on observed OOM behavior:
	•	128MB InnoDB buffer pool keeps MySQL inside a hard memory fence
	•	Swap > RAM gives the kernel room to breathe
	•	Low Apache worker limits prevent PHP processes from stampeding memory
	•	Swappiness = 10 avoids swap thrashing while preventing panic kills

This is not about performance tuning — it’s about staying alive on small hardware.

⸻

When to Upgrade the Droplet

This setup makes a 1GB droplet stable, not fast.

You should still consider upgrading to 2GB if:
	•	traffic increases
	•	you add heavy plugins (Wordfence, WooCommerce, builders)
	•	you want more headroom and fewer constraints

The same bootstrap works on 2GB droplets — you can later relax limits if needed.

⸻

Assumptions
	•	Ubuntu-based DigitalOcean WordPress 1-Click image
	•	Apache + MySQL (prefork MPM)
	•	systemd
	•	Root access (cloud-init runs as root)

If you later move to Nginx + PHP-FPM, this should be adjusted.

⸻

Philosophy

On small servers, stability beats cleverness.

This bootstrap trades peak throughput for predictability and uptime, which is exactly what you want on 1GB WordPress droplets.

⸻

If you want, next iterations could include:
	•	Nginx + PHP-FPM detection
	•	2GB/4GB profile variants
	•	Basic memory watchdog / alerting
	•	Wordfence-safe defaults

Just say the word.
