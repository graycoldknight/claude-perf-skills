---
name: perf-code-layout-tuning
description: Expert guidance on optimizing machine code layout to improve Instruction Cache (I-cache) and ITLB performance. Use when the user asks about frontend bound, I-cache misses, ITLB pressure, PGO, profile-guided optimization, function reordering, code placement, hot/cold splitting, or instruction fetch bottlenecks.
---

# perf-code-layout-tuning Skill

This skill provides expert guidance on optimizing the physical layout of machine code to improve Instruction Cache (I-cache) and Instruction TLB (ITLB) performance, based on Chapter 11 of "Performance Analysis and Tuning on Modern CPUs".

## Overview
Efficient execution requires that the instructions themselves are fetched quickly. Large or fragmented codebases can suffer from "Frontend Bound" stalls due to I-cache misses and ITLB misses.

## Key Techniques

### 1. Basic Block Placement
- Move "cold" code (error handling, rare branches) out of the hot path.
- Keep "hot" basic blocks contiguous to improve fetch efficiency and branch prediction.

### 2. Function Reordering
- Group hot functions together in memory. This reduces the number of pages needed for the hot set of instructions, lowering ITLB pressure.
- Linker scripts or flags like `-ffunction-sections` and `-fdata-sections` combined with a gold or lld linker can assist with this.

### 3. Profile-Guided Optimization (PGO)
- The most effective way to automate code layout.
- **Workflow:**
    1. Build with instrumentation (`-fprofile-generate`).
    2. Run a representative workload (benchmarks).
    3. Rebuild using the profile data (`-fprofile-use`).
- This allows the compiler to make informed decisions about inlining, branch weights, and block placement.

### 4. Function Splitting
- Split a large function into a hot "head" and one or more cold "tails". The compiler often does this automatically with PGO.

### 5. Alignment
- Align hot loop entries and function start addresses to 16, 32, or 64-byte boundaries to avoid fetching unnecessary padding or crossing cache line boundaries.

## Applying to Your Project

- Use `[[likely]]` and `[[unlikely]]` attributes (C++20) or `__builtin_expect` to guide the compiler on branch probability, especially in error-handling paths.
- **Header-only libraries**: If your hot-path code lives in headers (common for templated or performance-critical libraries), ensure hot functions are marked `inline` or `__attribute__((always_inline))` to keep them within the caller's context. For cold code (e.g., YAML/config parsing, error formatting), move implementations to `.cpp` files to avoid polluting the I-cache.
- **Compiled libraries**: Keep non-template shared logic in free functions (not inside class templates) to avoid duplicating code across multiple template instantiations.
- Analyze "Frontend Bound" metrics in TMA. If high (>30%), PGO is likely the highest-impact optimization.

## Resources
- [Machine Code Layout Optimizations (Chapter 11)](https://github.com/dendibakh/perf-book/blob/master/chapters/11-Machine-Code-Layout-Optimizations/11-1%20Machine%20Code%20Layout.md)
