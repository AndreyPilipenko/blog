---
title: "Top commands to diagnose Linux system load"
date: 2026-01-10
draft: false
tags: ["linux", "sre", "debug"]
---

Top commands to diagnose Linux system load

```
iotop -o — shows processes that are currently performing disk I/O
journalctl --disk-usage — displays how much disk space is used by systemd logs
journalctl --vacuum-size=500M — truncates systemd logs to 500 MB
ps aux --sort=-%cpu | head — top processes by CPU usage
ps aux --sort=-%mem | head — top processes by RAM usage
systemctl list-units --failed — services that failed or did not start

Additional useful commands

iotop -a — shows total disk read/write per process since startup
vmstat -SM 5 — memory, swap, and CPU statistics updated every 5 seconds
df -hT — disk usage by filesystem with filesystem type
glances — real-time overview of CPU, memory, disk, network, and processes
```


