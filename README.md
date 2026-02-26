# claude-perf-skills

Reusable [Claude Code](https://claude.ai/code) performance tuning skills for low-latency C++ on Intel CPUs.

These skills provide expert guidance on microarchitecture-level optimization, based on [Performance Analysis and Tuning on Modern CPUs](https://github.com/dendibakh/perf-book) by Denis Bakhvalov.

## Skills

| Skill | Description |
|-------|-------------|
| **perf-tma-tuning** | Top-Down Microarchitecture Analysis (TMA) — identify bottleneck buckets (Frontend Bound, Backend Bound, Bad Speculation, Retiring) using `toplev.py` |
| **perf-xpedite-tuning** | Cycle-accurate probe-based profiling with Xpedite — per-transaction P50/P99 latency distributions and PMU counter deltas |
| **perf-branch-tuning** | Reduce Bad Speculation — branchless programming, CMOV, lookup tables, loop unrolling |
| **perf-memory-tuning** | Optimize cache utilization — data locality, flat arrays, TLB reduction, prefetching |
| **perf-code-layout-tuning** | Reduce Frontend Bound — I-cache/ITLB optimization, PGO, hot/cold splitting, function reordering |

## Installation

Symlink each skill directory into your Claude Code skills folder:

```bash
# Clone (or add as a submodule to your project)
git clone https://github.com/graycoldknight/claude-perf-skills.git ~/claude-perf-skills

# Symlink all skills
for skill in ~/claude-perf-skills/perf-*/; do
    ln -sf "$skill" ~/.claude/skills/$(basename "$skill")
done
```

Or add as a git submodule in your project:

```bash
cd your-project
git submodule add https://github.com/graycoldknight/claude-perf-skills.git skills/perf
# Then symlink from ~/.claude/skills/ as above
```

## Usage

Skills are triggered automatically in Claude Code when you ask about related topics, or explicitly via `/skill-name`:

```
/perf-tma-tuning        # Get TMA profiling guidance
/perf-xpedite-tuning    # Get Xpedite setup and usage guidance
/perf-branch-tuning     # Get branchless optimization guidance
/perf-memory-tuning     # Get cache/memory optimization guidance
/perf-code-layout-tuning # Get I-cache/PGO optimization guidance
```

## Target Platform

These skills are written with **Intel Sapphire Rapids** (AWS c7i instances) as the primary target, but the techniques and methodology apply broadly to:

- Other Intel microarchitectures (Ice Lake, Alder Lake, Emerald Rapids) — adjust `--force-cpu` flag and PMU event names accordingly
- Any x86-64 system with PMU counter access (`kernel.perf_event_paranoid <= 1`)

PMU event names are CPU-specific. Use `perf list` on your target system to verify available events.
