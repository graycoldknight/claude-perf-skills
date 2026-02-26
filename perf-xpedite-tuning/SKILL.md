---
name: perf-xpedite-tuning
description: Expert guidance on cycle-accurate, probe-based latency profiling using Xpedite. Use when the user asks about xpedite, probe-based profiling, per-transaction profiling, latency distribution, hardware PMU counters, non-sampling profiler, cycle-accurate measurement, XPEDITE_TXN_SCOPE, or profileInfo.py configuration.
---

# perf-xpedite-tuning Skill

This skill provides expert guidance on using Xpedite for cycle-accurate, per-transaction latency profiling in low-latency C++ systems. Xpedite is a non-sampling, probe-based profiler from Morgan Stanley that measures every transaction iteration with nanosecond granularity.

## Overview

Xpedite is fundamentally different from sampling-based profilers (like `perf`) and system-wide analysis tools (like TMA/toplev):

- **Non-sampling**: Probes fire on *every* transaction, not sampled. No bias towards long-running transactions.
- **Probe-based**: Lightweight instrumentation via RAII scope macros that read TSC + PMU counters at entry/exit.
- **Zero overhead when inactive**: Probes are 5-byte NOPs with no runtime cost until attached.
- **Per-transaction latency**: Reports P50/P90/P99 latency distributions and per-probe PMU deltas (instructions, cycles, cache misses, branch mispredictions, iTLB misses).
- **IPC correlation**: Measures instructions-per-cycle for each probe interval to identify execution port pressure.

Typical use case: After TMA identifies a bottleneck bucket (e.g., "Frontend Bound 48%"), use Xpedite to get per-call latency distributions and trace which probe interval correlates with the issue.

## When to Use

Use Xpedite when:

1. **Comparing two code variants** — You've made an optimization and want to measure the latency impact with statistical rigor (P50/P99 distributions vs. mean only).
2. **Investigating latency outliers** — TMA shows a bottleneck, but you need to know which part of your hot loop is stalling (probe-by-probe breakdown).
3. **Validating micro-optimizations** — Atomic ops, branch prediction tuning, cache prefetch hints: these need nanosecond-level precision to measure ROI.
4. **Production diagnosis** — Xpedite can run against live traffic with near-zero overhead (probes are inactive until profiler attaches).

Do NOT use Xpedite for:
- Initial bottleneck identification (use TMA/toplev for that).
- Measuring entire-application throughput (use Google Benchmark for that; Xpedite is per-transaction only).
- Profiling in shared environments without admin (PMU access requires `kernel.perf_event_paranoid=1`).

## Workflow (6 Steps)

### Step 1: Isolate Cores (One-time setup)

To eliminate scheduler jitter, configure CPU isolation on your instance:

```bash
# Add to /etc/default/grub (GRUB_CMDLINE_LINUX_DEFAULT)
# For 2-vCPU instance: isolcpus=1 nohz_full=1 rcu_nocbs=1
# For 4-vCPU instance: isolcpus=1,3 nohz_full=1,3 rcu_nocbs=1,3
sudo update-grub && sudo reboot
```

After reboot, verify:

```bash
cat /proc/cmdline | grep isolcpus
cat /sys/devices/system/cpu/isolated
```

Isolated cores will not run OS scheduler interrupts, reducing noise in latency measurements.

### Step 2: Instrument Code with XPEDITE_TXN_SCOPE

Add probe macros to mark the transaction boundaries you want to measure. The framework provides `TxnBeginProbe` and `TxnEndProbe`:

```cpp
#include <xpedite/framework/Framework.H>
#include <xpedite/framework/Probes.H>

int main() {
    // One-time initialization (thread-safe, awaits profiler attach)
    xpedite::framework::initialize("/tmp/myapp-appinfo.txt",
        {xpedite::framework::AWAIT_PROFILE_BEGIN});

    std::cout << "Attach profiler with: xpedite record -p profiling/profileInfo.py\n";

    // Hot loop
    for (int i = 0; i < iterations; ++i) {
        // Your code here
        auto result = process(input);

        // Prevent compiler from optimizing away the result
        asm volatile("" : : "r,m"(&result) : "memory");
    }

    xpedite::framework::halt();
    return 0;
}
```

The `asm volatile` barrier prevents the compiler from dead-code-eliminating the result. The actual probe macros are defined in `profileInfo.py` (see Step 3).

**Non-Xpedite builds**: Wrap includes in `#ifdef ENABLE_XPEDITE` guards. When Xpedite is disabled, `initialize()` and `halt()` are no-ops.

### Step 3: Configure profileInfo.py

The profiler's behavior is defined in `profiling/profileInfo.py`:

```python
from xpedite import TxnBeginProbe, TxnEndProbe
from xpedite import Metric, Event

appName = 'myapp'                               # Application identifier
appHost = '127.0.0.1'                           # RPC listen address (localhost for local testing)
appInfo = '/tmp/myapp-appinfo.txt'              # File the app writes on startup
homeDir = 'profiling/reports'                   # Where Jupyter notebooks are generated

probes = [
    TxnBeginProbe('Begin', sysName='TxnBegin'),
    TxnEndProbe('End',     sysName='TxnEnd'),
]

# PMU counters for Sapphire Rapids (SPR, AWS c7i)
# Requires: sudo sysctl -w kernel.perf_event_paranoid=1
pmc = [
    Event('Instructions',      'INST_RETIRED.ANY'),
    Event('Cycles',            'CPU_CLK_UNHALTED.THREAD'),
    Event('L1D Misses',        'MEM_LOAD_RETIRED.L1_MISS'),
    Event('LLC Misses',        'MEM_LOAD_RETIRED.L3_MISS'),
    Event('Branch Mispredict', 'BR_MISP_RETIRED.ALL_BRANCHES'),
    Event('iTLB Misses',       'ITLB_MISSES.WALK_COMPLETED'),
]
```

**Key fields**:
- `appInfo`: Path the running binary writes to announce its RPC port. Framework creates this at startup.
- `homeDir`: Output directory for Jupyter notebooks (must exist before profiling).
- `probes`: Named transaction Begin/End markers. The profiler will record all cycles and PMU events between them.
- `pmc`: Raw PMU event names for SPR. Event names vary per CPU microarchitecture; see Intel SDM for your CPU. Use `perf list` to list available events on your system.

### Step 4: Build with Xpedite

Clone and build Xpedite, then link it into your project:

```bash
# Clone Xpedite
git clone https://github.com/Nasdaq/Xpedite.git third_party/Xpedite

# Build libxpedite.a and Python bindings
cd third_party/Xpedite && ./install.sh
cd ../..
```

In your CMakeLists.txt, add Xpedite when enabled:

```cmake
option(ENABLE_XPEDITE "Enable Xpedite profiling" OFF)

if(ENABLE_XPEDITE)
    add_definitions(-DENABLE_XPEDITE)
    include_directories(third_party/Xpedite/install/include)
    link_directories(third_party/Xpedite/install/lib)

    add_executable(xpedite_runner profiling/xpedite_runner.cpp)
    target_link_libraries(xpedite_runner
        your_library
        xpedite-pic
        pthread dl rt
    )
endif()
```

Build with Xpedite enabled:

```bash
mkdir -p build && cd build
cmake -DCMAKE_BUILD_TYPE=Release -DENABLE_XPEDITE=ON ..
make xpedite_runner
```

**Note**: Upstream Xpedite may need patches for newer CPUs (Sapphire Rapids, Emerald Rapids, etc.). Check the [Xpedite GitHub issues](https://github.com/Nasdaq/Xpedite/issues) for PMU event compatibility and TSC frequency detection fixes.

### Step 5: Two-Terminal Attach

**Terminal 1** (on the target machine):
```bash
taskset -c 1 ./build/xpedite_runner --iterations=10000000
```

This launches the profiling binary on isolated core 1, initializes the Xpedite framework, and waits for profiler attachment. Output:
```
Xpedite framework initialized. Attach profiler with:
  xpedite record -p profiling/profileInfo.py
Running 10000000 iterations...
```

**Terminal 2** (same machine, separate SSH session):
```bash
third_party/Xpedite/scripts/bin/xpedite record -p profiling/profileInfo.py
```

The profiler attaches via RPC, enables hardware PMU counters on the running process, and begins collecting samples. Each probe interval records:
- TSC (timestamp) at entry/exit.
- PMU counter deltas (instructions, cycles, cache misses, branch mispredicts, iTLB misses).

Press **Ctrl+C** in Terminal 2 to stop profiling. The profiler generates a Jupyter notebook in `profiling/reports/` with the collected samples.

### Step 6: View Results via SSH Port-Forward

On your **local machine**:
```bash
ssh -N -L 8888:localhost:8888 -i <your-key.pem> ubuntu@<ec2-public-ip>
```

Then open http://localhost:8888 in your browser. Jupyter should serve the reports generated by Xpedite. Click on the notebook to see latency distributions, IPC per probe, cache miss rates, etc.

## Code Instrumentation Details

### Probe Macros

Xpedite provides two fundamental probe types:

1. **`TxnBeginProbe(name, sysName='...')`** — Marks the start of a transaction. Records TSC + PMU counters at entry.
2. **`TxnEndProbe(name, sysName='...')`** — Marks the end of a transaction. Records TSC + PMU counters at exit. Computes delta.

Probes are named in `profileInfo.py`'s `probes[]` list. The `sysName` is an internal identifier used by the kernel module (no spaces allowed).

### Framework Initialization

```cpp
xpedite::framework::initialize(appinfo_path, options);
```

- `appinfo_path`: File path where the application writes its RPC endpoint (port).
- `options`: Flags like `AWAIT_PROFILE_BEGIN` (block until profiler connects) or `NO_PROFILE_BEGIN` (continue if no profiler).

### Framework Cleanup

```cpp
xpedite::framework::halt();
```

Must be called before process exit to cleanly detach from the kernel PMU module and write out remaining samples.

### Compiler Barrier

```cpp
asm volatile("" : : "r,m"(&result) : "memory");
```

This inline assembly statement prevents the compiler from:
- Dead-code-eliminating the result (since we don't use it after the loop body).
- Hoisting/moving the call outside the loop.
- Optimizing away dependent code.

The `"memory"` clobber also acts as a memory barrier, preventing compiler reordering across the loop iteration.

## profileInfo.py Reference

### Required Fields

| Field | Type | Description |
|-------|------|-------------|
| `appName` | str | Identifier for the profiled application (used in report titles). |
| `appHost` | str | RPC bind address. `127.0.0.1` for local, `0.0.0.0` for remote attach. |
| `appInfo` | str | Path where the app writes its RPC port at startup (e.g., `/tmp/myapp-appinfo.txt`). |
| `homeDir` | str | Directory where Jupyter notebooks are generated. Must be created beforehand. |
| `probes` | list | List of `TxnBeginProbe` and `TxnEndProbe` objects. |
| `pmc` | list | List of `Event(name, pmu_event_name)` for hardware counters. |

### PMU Events for Sapphire Rapids (SPR)

The following events are commonly used and must be available on the target CPU:

```python
Event('Instructions',      'INST_RETIRED.ANY'),            # Retired instructions
Event('Cycles',            'CPU_CLK_UNHALTED.THREAD'),     # Core cycles (includes stalls)
Event('L1D Misses',        'MEM_LOAD_RETIRED.L1_MISS'),    # L1 data cache misses
Event('L1I Misses',        'L1-icache-load-misses'),       # L1 instruction cache misses
Event('LLC Misses',        'MEM_LOAD_RETIRED.L3_MISS'),    # L3 cache misses
Event('Branch Mispredict', 'BR_MISP_RETIRED.ALL_BRANCHES'),# Branch mispredictions
Event('iTLB Misses',       'ITLB_MISSES.WALK_COMPLETED'),  # Instruction TLB misses
```

CPU-specific event names vary. Use `perf list` to list available events on your system.

## Interpreting Results

### Latency Distribution

Jupyter shows P50, P90, P99 (sometimes P99.9) latency for each probe interval:

- **P50 (median)**: Typical case. This is what you optimize for.
- **P99**: Worst-case outliers. Indicates tail latency (scheduler jitter, TLB misses, cache evictions).

Compare two runs' P50/P99 to measure optimization impact.

### Instructions Per Cycle (IPC)

- **IPC > 2.0**: Good instruction-level parallelism (backend executing multiple ops per cycle).
- **IPC < 1.0**: Stalls (memory latency, data dependencies, backend port contention).

If IPC drops after a code change, suspect memory latency or execution port pressure.

### Cache Misses (L1D, LLC)

- **L1D Misses / Instructions**: Cache miss ratio. High = data locality issue.
- **LLC Misses / L1D Misses**: Ratio of L3 misses to L1 misses. High = data doesn't fit in L3.

### Branch Mispredictions

- **Branch Mispredict / Instructions**: Mispredict ratio. High = unpredictable control flow (see perf-branch-tuning skill).

### iTLB Misses

- Indicates the instruction code footprint is larger than the iTLB. Consider code layout optimizations (PGO, function reordering — see perf-code-layout-tuning skill).

## Resources

- **Xpedite GitHub**: https://github.com/Nasdaq/Xpedite
- **Morgan Stanley blog**: https://www.morganstanley.com/technology/research/xpedite
- **Intel SDM** (for CPU-specific PMU event names): https://www.intel.com/content/www/us/en/developer/articles/technical/intel-sdm.html
