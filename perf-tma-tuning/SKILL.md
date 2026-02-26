---
name: perf-tma-tuning
description: Expert guidance on Top-Down Microarchitecture Analysis (TMA) for optimizing C++ applications, tailored for low-latency systems. Use when the user asks about TMA, toplev, frontend bound, backend bound, bad speculation, retiring metrics, perf profiling methodology, or microarchitecture bottleneck analysis.
---

# perf-tma-tuning Skill

This skill provides expert guidance on Top-Down Microarchitecture Analysis (TMA) for optimizing C++ applications, specifically tailored for high-frequency trading (HFT) and low-latency systems.

## Overview
TMA is a methodology to identify the bottleneck in a program by categorizing CPU cycles into four main buckets:
1. **Frontend Bound**: Stalls in fetching and decoding instructions.
2. **Backend Bound**: Stalls in execution units or memory access.
3. **Bad Speculation**: Cycles wasted on mispredicted branches.
4. **Retiring**: Cycles spent on useful work.

## When to Use
Use this skill when:
- You have reached a performance plateau with algorithmic optimizations.
- You see high latency in benchmarks but aren't sure if it's CPU or Memory bound.
- You are running on modern Intel CPUs (e.g., AWS c7i Sapphire Rapids).

## Workflow

### 1. Install pmu-tools

```bash
git clone --depth 1 https://github.com/andikleen/pmu-tools.git third_party/pmu-tools
```

### 2. Collect Baseline TMA Metrics

Run `toplev.py` at Level 2 to identify which bucket dominates:

```bash
# TMA Level 2 (recommended starting point)
sudo python3 third_party/pmu-tools/toplev.py \
    --force-cpu spr -l2 -v --no-desc \
    -- taskset -c 1 ./your_benchmark

# TMA Level 1 (quick overview)
sudo python3 third_party/pmu-tools/toplev.py \
    --force-cpu spr -l1 -v --no-desc \
    -- taskset -c 1 ./your_benchmark
```

**Notes:**
- `--force-cpu spr` is needed on AWS instances where the kernel may not expose the correct CPU model. Replace `spr` with your microarchitecture (e.g., `icl` for Ice Lake, `skl` for Skylake).
- `taskset -c 1` pins execution to an isolated core for stable measurements (see core isolation below).
- Requires `sudo` or `kernel.perf_event_paranoid <= 1`.

### 3. Interpret the Results
- **Backend Bound (Memory)**: Often caused by cache misses (L1, L2, L3) or DTLB misses.
  - *Fix:* Use flatter data structures, improve data locality, or use prefetching.
- **Backend Bound (Core)**: Caused by lack of execution resources (e.g., too many complex divisions) or data dependencies.
  - *Fix:* Reduce instruction count, use SIMD (AVX-512), or break dependency chains.
- **Frontend Bound**: Caused by large code footprint, ITLB misses, or complex instruction decoding.
  - *Fix:* Use PGO (Profile Guided Optimization), `__attribute__((always_inline))`, or simplify hot paths.
- **Bad Speculation**: Caused by unpredictable branches.
  - *Fix:* Use branchless programming, lookup tables, or `std::clamp`/`std::min`/`std::max`.

### 4. Deep Dive
If a specific bucket is high, run `toplev` at a deeper level (Level 3 or 4):
```bash
sudo python3 third_party/pmu-tools/toplev.py \
    --force-cpu spr -l3 -v --no-desc \
    -- taskset -c 1 ./your_benchmark
```

### 5. Core Isolation (Recommended)

For deterministic measurements, isolate CPUs from the OS scheduler:

```bash
# Add to /etc/default/grub (GRUB_CMDLINE_LINUX_DEFAULT)
# For 2-vCPU instance: isolcpus=1 nohz_full=1 rcu_nocbs=1
# For 4-vCPU instance: isolcpus=1,3 nohz_full=1,3 rcu_nocbs=1,3
sudo update-grub && sudo reboot

# Verify after reboot
cat /sys/devices/system/cpu/isolated
```

Then pin your benchmark to an isolated core with `taskset -c 1`.

## Resources
- [Performance Analysis and Tuning on Modern CPUs](https://github.com/dendibakh/perf-book) by Denis Bakhvalov.
- [Intel 64 and IA-32 Architectures Optimization Reference Manual](https://www.intel.com/content/www/us/en/developer/articles/technical/intel-sdm.html).
