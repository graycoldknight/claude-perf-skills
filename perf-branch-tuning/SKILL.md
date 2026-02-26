---
name: perf-branch-tuning
description: Expert guidance on reducing Bad Speculation by optimizing branch prediction and using branchless programming techniques. Use when the user asks about branch misprediction, bad speculation, branchless code, CMOV, conditional moves, lookup tables for branches, or optimizing if-else chains in hot paths.
---

# perf-branch-tuning Skill

This skill provides expert guidance on reducing "Bad Speculation" by optimizing branch prediction, based on Chapter 10 of "Performance Analysis and Tuning on Modern CPUs".

## Overview
Branches (if-statements, loops, switches) can cause the CPU pipeline to stall if the branch predictor fails. In low-latency code, minimizing unpredictable branches is critical.

## Key Techniques

### 1. Branchless Programming
- Use **arithmetic operations** to replace conditional logic.
    - *Example:* `val = (a > b) ? a : b;` can sometimes be faster as `val = a - ((a - b) & (b - a >> 31));` or using `std::max`.
- Use **bitmasks** to apply conditional updates.

### 2. Lookup Tables
- Replace complex `switch` statements or `if-else` chains with a precomputed lookup table.
- This is particularly effective for protocol parsing (e.g., mapping ASCII characters to numeric values).

### 3. Loop Unrolling
- Manually or via compiler hints (`#pragma unroll`) reduce the number of branch iterations.
- This reduces the overhead of the loop counter and the end-of-loop branch.

### 4. Predication
- Use CMOV (conditional move) instructions. Modern compilers do this automatically for simple conditionals, but complex ones might need manual refactoring to assist the compiler.

## Applying to Your Project

1. **Identify hot branches**: Use `perf annotate` or `perf record -b` (Last Branch Record) to find branches with high mispredict rates in your hot path.
2. **Look for unpredictable patterns**: `if`/`switch` statements in parsing loops, validation checks on variable data, and state machines with data-dependent transitions are common culprits.
3. **Measure before and after**: Use `perf stat -e BR_MISP_RETIRED.ALL_BRANCHES` to confirm mispredict reduction, and benchmark to verify latency improvement.
4. **Check compiler output**: Use `objdump -d` or Compiler Explorer (godbolt.org) to verify the compiler emits CMOV or branchless sequences where expected.

## Resources
- [Optimizing Branch Prediction (Chapter 10)](https://github.com/dendibakh/perf-book/blob/master/chapters/10-Optimizing-Branch-Prediction/10-0%20Optimizing%20bad%20speculation.md)
