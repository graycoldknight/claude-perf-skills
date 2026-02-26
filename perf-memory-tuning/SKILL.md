---
name: perf-memory-tuning
description: Expert guidance on optimizing memory access patterns, cache utilization, and data locality for high-performance C++. Use when the user asks about cache misses, memory bandwidth, data locality, false sharing, TLB misses, huge pages, prefetching, DTLB, L1/L2/L3 cache, or backend-bound memory bottlenecks.
---

# perf-memory-tuning Skill

This skill provides expert guidance on optimizing memory access patterns and cache utilization, based on Chapter 8 of "Performance Analysis and Tuning on Modern CPUs".

## Overview
Modern CPUs are often bottlenecked by the "Memory Wall" — the gap between CPU speed and memory latency. This skill helps you bridge that gap.

## Key Techniques

### 1. Data Locality
- **Spatial Locality:** Group data that is accessed together. Use flat arrays or `std::vector` instead of linked structures.
- **Temporal Locality:** Reuse data while it's still in the cache.

### 2. Cache-Friendly Structures
- Use **Structure of Arrays (SoA)** instead of Array of Structures (AoS) when only a subset of fields is accessed in a loop.
- Align data to cache line boundaries (64 bytes) to avoid "false sharing" or split-line accesses.

### 3. Reducing TLB Misses
- Use **Huge Pages** for large data structures to reduce the number of Page Table entries.
- Minimize the number of unique pages accessed in a hot loop.

### 4. Software Prefetching
- Use `__builtin_prefetch` to bring data into the cache before it is needed.
- *Caution:* Over-prefetching can increase memory bus pressure and degrade performance.

## Applying to Your Project

1. **Profile cache behavior**: Run `perf stat -e L1-dcache-load-misses,LLC-load-misses,dTLB-load-misses -- ./your_benchmark` to get baseline miss rates.
2. **Check hot-path data structures**: Flat arrays with O(1) index access (e.g., `std::array<T, N>` indexed by tag/key) are ideal for hot-path lookups. Hash maps (`std::unordered_map`) incur pointer chasing and poor locality.
3. **Ensure contiguous input buffers**: For parsing workloads, keep input data in a single contiguous allocation that is pre-faulted (touched once before the hot loop).
4. **Measure L1D miss ratio**: Compute `L1D Misses / Instructions` — if this exceeds ~1%, data locality improvements will yield measurable gains.

## Resources
- [Optimizing Memory Accesses (Chapter 8)](https://github.com/dendibakh/perf-book/blob/master/chapters/8-Optimizing-Memory-Accesses/8-1%20Optimizing%20Memory%20Accesses.md)
