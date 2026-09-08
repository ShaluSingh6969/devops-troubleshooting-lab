# Linux Incident — High System Load

## Scenario

A Linux host reports a significantly elevated load average.

The objective is to determine whether the underlying cause is CPU
saturation, memory pressure, or I/O contention.

## Initial Investigation

Check load:

```bash
uptime
```

# Linux Incident — High System Load

## Scenario

A Linux host reports a significantly elevated load average.

The objective is to determine whether the underlying cause is CPU
saturation, memory pressure, or I/O contention.

## Initial Investigation

Check load:

```bash
uptime
```

Check CPU count:

```bash
nproc
```

Check run queue and resource pressure:

```bash
vmstat 1
```

Inspect CPU utilization:

```bash
mpstat -P ALL 1
```

Inspect expensive processes:

```bash
pidstat 1
```

Check memory:

```bash
free -h
pidstat -r 1
```

Check disk I/O:

```bash
iostat -xz 1
```

## CPU Saturation Indicators

- high runnable queue
- low CPU idle
- high user/system CPU
- load near or above CPU capacity

## Memory Pressure Indicators
- low available memory
- increasing swap activity
- process RSS growth
- potential OOM-killer activity

## I/O Pressure Indicators
- blocked processes
- increased I/O wait
- elevated disk latency
- high device utilization

## Root-Cause Principle

A high load average alone does not identify the bottleneck.

The root cause must be determined by correlating CPU, memory, I/O,
and process-level metrics.

## Validation

After removing the generated workload, verify:

```bash
uptime
vmstat 1 5
mpstat -P ALL 1 5
iostat -xz 1 5
```

System metrics should move back toward baseline.

## Lessons Learned
- Load average is not equivalent to CPU percentage.
- A high load may be caused by CPU contention or blocked tasks.
- vmstat provides a fast system-wide diagnostic view.
- pidstat helps move from system symptoms to process-level cause.
- iostat is important for distinguishing CPU problems from storage problems.
- Performance incidents should be diagnosed by correlating multiple signals.

## The two useful concepts

vmstat r → runnable processes
vmstat b → blocked processes
wa       → I/O wait



|Situation|	r|b|CPU idle|wa|First hypothesis|
|CPU saturation|High|Low|Low|Low|CPU contention|
|I/O contention|Often lower|High|Can be high|High|I/O bottleneck|
Healthy/idle	Low	Low	High	Low	No major pressure