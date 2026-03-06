---
name: perf-engineering
description: >
  Performance engineering guidance for CPU and memory optimization across languages (Rust, C/C++, TypeScript/JavaScript, Go, Python). Use this skill whenever the user asks about optimizing code performance, reducing memory allocations, improving cache locality, SIMD/vectorization, profiling, benchmarking, or any question about making code faster or more memory-efficient. Also trigger when the user mentions: hot loops, allocation pressure, cache misses, false sharing, memory pools, arena allocation, string interning, branch prediction, auto-vectorization, zero-copy, AoS vs SoA, data-oriented design, or profiling tools (perf, flamegraph, Instruments, VTune, cachegrind). Trigger even for indirect performance questions like "why is this slow", "this function is a bottleneck", "how to reduce memory usage", or "should I optimize this".
---

# Performance Engineering: CPU & Memory Optimization

This skill provides structured guidance for performance optimization decisions. It covers the ten core areas below and helps you apply them to real code.

## Philosophy: Measure First, Optimize Second

The single most important rule: **never optimize without profiling data.** Intuition about performance is notoriously unreliable. A function that "looks slow" might account for 0.1% of runtime. Always start with a profiler, identify the actual bottleneck, then apply the appropriate technique from this skill.

The goal is not to make every line of code fast — it's to make the *right* code fast, in the *right* way, with the minimum complexity cost.

## How to Use This Skill

1. **Identify the bottleneck category** — is it CPU-bound (computation, branch mispredicts, poor vectorization) or memory-bound (allocations, cache misses, layout issues)?
2. **Read the relevant reference file(s)** from `references/` for deep guidance
3. **Apply the technique** with the language-specific patterns provided
4. **Verify the improvement** with benchmarks before and after

## Reference Files

Load the appropriate reference based on the optimization area:

- `references/memory-optimization.md` — Stack vs heap, memory pooling, arena allocation, string interning, zero-copy techniques. Read this for anything related to reducing allocations, memory layout, or memory bandwidth.
- `references/cpu-optimization.md` — Cache locality (AoS vs SoA), false sharing, branch prediction, SIMD/vectorization, loop optimization, lazy evaluation. Read this for anything related to making computation faster.
- `references/profiling-guide.md` — When NOT to optimize, profiling tools by language/platform, benchmarking methodology, interpreting results, common pitfalls. Read this FIRST if the user hasn't profiled yet.

## Decision Flowchart

When someone asks "how do I make this faster":

1. Have they profiled? → If no, load `references/profiling-guide.md` and help them profile first
2. Is the bottleneck allocation-heavy? → Load `references/memory-optimization.md`
3. Is the bottleneck compute-heavy? → Load `references/cpu-optimization.md`
4. Is it both? → Load both references
5. Are they designing a new system? → Load both references for upfront architecture guidance

## Quick Reference: When Each Technique Matters

| Technique | When It Helps | When It Doesn't |
|---|---|---|
| Stack allocation | Tight loops, small fixed-size data | Large/dynamic-sized data |
| Arena allocation | Many short-lived objects with shared lifetime | Long-lived objects with varied lifetimes |
| String interning | Repeated string comparisons, deduplication | Unique strings, write-heavy workloads |
| AoS → SoA | Iterating one field across many entities | Accessing all fields of one entity |
| Memory pooling | Frequent alloc/dealloc of same-sized objects | Varied-size allocations |
| SIMD | Uniform operations on contiguous data | Branchy, data-dependent logic |
| Branch prediction | Hot loops with predictable patterns | Cold code, unpredictable data |
| Zero-copy | Large buffers passed between stages | Small data, ownership semantics needed |
| Lazy evaluation | Expensive computation that's often skipped | Always-needed values (adds overhead) |
| False sharing fixes | Multithreaded counters/accumulators | Single-threaded or read-only shared data |

## Keywords
performance, optimization, profiling, benchmark, cache, SIMD, vectorization, allocation, memory pool, arena, false sharing, AoS, SoA, data-oriented design, zero-copy, branch prediction, string interning, flamegraph, perf, hot loop, bottleneck, latency, throughput, memory bandwidth
