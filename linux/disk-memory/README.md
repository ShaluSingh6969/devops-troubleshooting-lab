# Linux Performance Troubleshooting

This section covers Linux performance diagnosis with emphasis on CPU,
memory, system load, and disk I/O.

The goal is to distinguish between different resource bottlenecks instead
of assuming that high system load means high CPU usage.

## Core Principle

Linux load average does not equal CPU utilization.

Load may increase because tasks are:

- runnable and waiting for CPU
- blocked in uninterruptible sleep, often due to I/O

Therefore multiple system signals must be correlated.

---

## Baseline Commands

| Command | Purpose |
|---|---|
| `uptime` | Show load averages |
| `nproc` | Show logical CPU count |
| `free -h` | Show memory and swap usage |
| `vmstat 1` | Show run queue, memory, I/O and CPU |
| `mpstat -P ALL 1` | Show CPU utilization per processor |
| `pidstat 1` | Show per-process CPU usage |
| `pidstat -r 1` | Show per-process memory statistics |
| `iostat -xz 1` | Show detailed disk I/O statistics |

---

## Important `vmstat` Fields

| Field | Meaning |
|---|---|
| `r` | Runnable processes waiting for CPU |
| `b` | Blocked processes |
| `si` | Swap-in |
| `so` | Swap-out |
| `us` | User CPU |
| `sy` | System/kernel CPU |
| `id` | Idle CPU |
| `wa` | I/O wait |
| `st` | Steal time |

---

## CPU Saturation

Typical indicators:

- high load average
- high runnable queue
- low CPU idle
- high user/system CPU
- low I/O wait

Useful commands:

```bash
uptime
vmstat 1
mpstat -P ALL 1
pidstat 1
ps -eo pid,ppid,stat,pri,ni,%cpu,%mem,cmd --sort=-%cpu