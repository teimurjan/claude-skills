# Profiling-Driven Optimization Guide

## Table of Contents
1. When NOT to Optimize
2. Profiling Tools by Language/Platform
3. Benchmarking Methodology
4. Interpreting Results
5. Common Pitfalls

---

## 1. When NOT to Optimize

Optimization has costs: code complexity, maintenance burden, reduced readability, and development time. Optimize only when all three conditions are true:

1. **There is a measured performance problem** — not a hunch, not "this looks slow", but actual profiling data showing this code is a bottleneck
2. **The optimization will meaningfully impact user experience or system cost** — shaving 2ms off a function that runs once at startup is not worth the complexity
3. **The simpler approaches have been tried first** — algorithmic improvements (O(n²) → O(n log n)) almost always dwarf micro-optimizations

### The optimization decision tree

```
Is there a performance problem?
├── No → STOP. Don't optimize.
├── Not sure → Profile first, then decide.
└── Yes → What kind?
    ├── Wrong algorithm/data structure → Fix that first (biggest wins)
    ├── Unnecessary work (redundant computation, extra copies) → Eliminate it
    ├── I/O bound (network, disk) → Caching, batching, async, not CPU tricks
    └── Genuinely CPU/memory bound → Apply techniques from this skill
```

### Premature optimization red flags
- Optimizing code that hasn't been benchmarked
- Optimizing code that runs < 1000 times per second (unless latency-critical)
- Choosing a complex data structure "for performance" when a Vec/Array would work
- Inlining everything, unrolling everything, using unsafe everywhere "just in case"
- Spending days on a 5% improvement when a 10x improvement is available via architecture change

### The 90/10 rule
90% of execution time is spent in 10% of the code. Your profiler tells you which 10%. Optimize that. Leave the rest clean and readable.

---

## 2. Profiling Tools by Language/Platform

### Rust
| Tool | What it measures | Command |
|---|---|---|
| `cargo bench` (criterion) | Microbenchmarks with statistical rigor | `cargo bench` |
| `perf` (Linux) | CPU cycles, cache misses, branch misses | `perf record --call-graph dwarf ./target/release/app` |
| `flamegraph` | Visual call stack profiling | `cargo install flamegraph && cargo flamegraph` |
| `DHAT` (valgrind) | Heap allocation profiling | `valgrind --tool=dhat ./target/release/app` |
| `cargo-asm` | Inspect generated assembly | `cargo asm module::function` |
| `heaptrack` | Allocation tracking over time | `heaptrack ./target/release/app` |

**Criterion setup:**
```rust
// benches/my_bench.rs
use criterion::{criterion_group, criterion_main, Criterion, black_box};

fn bench_pixel_diff(c: &mut Criterion) {
    let img_a = generate_test_image(1920, 1080);
    let img_b = generate_test_image(1920, 1080);

    c.bench_function("pixel_diff_1080p", |b| {
        b.iter(|| {
            diff_images(black_box(&img_a), black_box(&img_b))
        })
    });
}

criterion_group!(benches, bench_pixel_diff);
criterion_main!(benches);
```

### TypeScript / JavaScript (Node.js)
| Tool | What it measures | Command |
|---|---|---|
| `node --prof` | V8 CPU profiling | `node --prof app.js && node --prof-process isolate-*.log` |
| `node --inspect` + Chrome DevTools | CPU + memory + allocation timeline | `node --inspect app.js` then open `chrome://inspect` |
| `clinic.js` | Doctor (overall), Flame (CPU), Bubbleprof (async) | `npx clinic doctor -- node app.js` |
| `0x` | Flamegraph from V8 profiler | `npx 0x app.js` |
| `benchmark.js` | Microbenchmarks | See code below |
| `process.memoryUsage()` | Heap snapshots | Call at intervals |

**benchmark.js setup:**
```typescript
import Benchmark from 'benchmark';

const suite = new Benchmark.Suite();
suite
    .add('approach_a', () => { approachA(testData); })
    .add('approach_b', () => { approachB(testData); })
    .on('cycle', (event: any) => console.log(String(event.target)))
    .on('complete', function(this: any) {
        console.log('Fastest is ' + this.filter('fastest').map('name'));
    })
    .run({ async: true });
```

### Go
| Tool | What it measures | Command |
|---|---|---|
| `go test -bench` | Built-in benchmarks | `go test -bench=. -benchmem ./...` |
| `pprof` (CPU) | CPU profiling | `go test -cpuprofile=cpu.prof -bench=.` |
| `pprof` (memory) | Allocation profiling | `go test -memprofile=mem.prof -bench=.` |
| `trace` | Execution trace (goroutines, GC) | `go test -trace=trace.out -bench=.` |
| `GODEBUG=gctrace=1` | GC behavior | `GODEBUG=gctrace=1 ./app` |

**Go benchmark:**
```go
func BenchmarkPixelDiff(b *testing.B) {
    imgA := generateTestImage(1920, 1080)
    imgB := generateTestImage(1920, 1080)
    b.ResetTimer()
    for i := 0; i < b.N; i++ {
        DiffImages(imgA, imgB)
    }
}
// Run: go test -bench=BenchmarkPixelDiff -benchmem -count=5
```

### C / C++
| Tool | What it measures | Command |
|---|---|---|
| `perf` (Linux) | CPU, cache, branches | `perf stat ./app` or `perf record ./app` |
| `Instruments` (macOS) | Time Profiler, Allocations, Leaks | Xcode → Product → Profile |
| `valgrind --tool=cachegrind` | Cache simulation | `valgrind --tool=cachegrind ./app` |
| `valgrind --tool=massif` | Heap profiling over time | `valgrind --tool=massif ./app` |
| `Intel VTune` | Deep microarchitecture analysis | VTune GUI or `vtune -collect hotspots ./app` |
| `google-benchmark` | Microbenchmarks | See Google Benchmark library |

### Python
| Tool | What it measures | Command |
|---|---|---|
| `cProfile` | Function-level CPU time | `python -m cProfile -s cumtime app.py` |
| `py-spy` | Sampling profiler (low overhead) | `py-spy top --pid <PID>` |
| `memory_profiler` | Line-by-line memory | `python -m memory_profiler app.py` |
| `tracemalloc` | Built-in allocation tracking | `tracemalloc.start()` in code |
| `scalene` | CPU + memory + GPU, line-level | `scalene app.py` |

---

## 3. Benchmarking Methodology

### Rules for reliable benchmarks

1. **Use statistical methods** — never report a single run. Use criterion (Rust), benchmark.js, or go test -count=10. Report mean, median, and standard deviation.

2. **Warm up the system** — JIT compilers (V8, JVM) need warm-up iterations. Criterion handles this automatically. For manual benchmarks, discard the first N runs.

3. **Prevent dead code elimination** — compilers remove code whose results are unused.
   - Rust: `criterion::black_box(result)`
   - C: `benchmark::DoNotOptimize(result)` (Google Benchmark) or `volatile`
   - JS: assign to a module-level variable or use the result

4. **Isolate what you're measuring** — setup (data generation, file loading) should happen outside the timed region.

5. **Control the environment:**
   - Disable CPU frequency scaling: `sudo cpupower frequency-set -g performance`
   - Close other applications
   - Pin to a specific CPU core: `taskset -c 0 ./app`
   - Run multiple times and check for consistency

6. **Benchmark representative data** — a benchmark on 100 items doesn't predict behavior on 10 million items. Test at the scale you care about.

7. **Compare apples to apples** — same input data, same machine, same compiler flags.

### Benchmark size calibration
- **Microbenchmark**: one function, one input size, nanosecond-level (criterion, benchmark.js)
- **Component benchmark**: one module/subsystem, realistic input, millisecond-level
- **End-to-end benchmark**: full pipeline, production-like input, second-level
- Each level catches different issues. A function that's fast in microbenchmarks might cause GC pressure in the full pipeline.

---

## 4. Interpreting Results

### Reading a flamegraph
- Width = time (or samples). Wide bars are where time is spent.
- Height = call stack depth. Read bottom-up (caller → callee).
- Look for: wide bars deep in the stack (hot inner functions), flat tops (leaf functions doing the work)
- Actionable: if 40% of time is in `malloc`, you have an allocation problem. If 30% is in `memcpy`, you might need zero-copy.

### Key perf stat counters
```
$ perf stat ./app
  1,234,567,890  cycles
    345,678,901  instructions          # 0.28 IPC (low — memory bound or stalling)
     12,345,678  cache-misses           # 8.5% of cache refs (high — layout problem)
      3,456,789  branch-misses          # 2.1% (moderate — consider branchless)
```

- **IPC < 1.0**: likely memory-bound (waiting for cache/RAM)
- **IPC > 2.0**: compute-bound, well-utilized
- **Cache miss rate > 5%**: data layout problem — consider SoA, prefetching, or smaller working set
- **Branch miss rate > 2%**: consider branchless alternatives or data sorting

### V8 / Node.js profiling output
```
$ node --prof-process isolate-*.log
 [JavaScript]:
   ticks  total  nonlib   name
    892   23.4%   28.1%  LazyCompile: *processPixels app.js:42
    456   12.0%   14.3%  LazyCompile: *diffRow app.js:78
```
Focus on the highest-tick functions. The `*` prefix means the function was optimized by TurboFan; no `*` means it wasn't — that's often a quick win (figure out why it's not being optimized).

---

## 5. Common Pitfalls

### Benchmarking pitfalls
- **Dead code elimination**: the compiler removes your benchmark because you don't use the result → use `black_box` / `DoNotOptimize`
- **Constant folding**: the compiler precomputes the result at compile time → use runtime-generated input
- **JIT warmup**: first iterations are interpreted, not compiled → warm up before measuring
- **GC pauses**: in GC'd languages, a pause in the middle of your benchmark skews results → run enough iterations for GC effects to amortize, or force GC before measuring
- **Frequency scaling**: turbo boost gives inconsistent results → lock frequency

### Optimization pitfalls
- **Optimizing cold code**: profiler says this function is 0.3% of runtime → leave it alone
- **Micro-optimizing before algorithmic fix**: you're doing O(n²) comparisons → no amount of SIMD fixes that; fix the algorithm first
- **Assuming transfer across languages/runtimes**: what's fast in C might not be fast in JS (V8's optimizer has different strengths). Always benchmark in your target runtime.
- **Cargo-culting**: "I read that SoA is faster" without measuring whether your access pattern benefits from it. Measure first.
- **Ignoring the allocator**: switching from system malloc to jemalloc/mimalloc/tcmalloc can give 10-30% improvements for allocation-heavy workloads with zero code changes.

### The allocator shortcut
Before any custom memory optimization, try a better allocator:
- **Rust**: `#[global_allocator] static GLOBAL: mimalloc::MiMalloc = mimalloc::MiMalloc;`
- **C/C++**: `LD_PRELOAD=/usr/lib/x86_64-linux-gnu/libjemalloc.so ./app`
- **Node.js**: built on V8's allocator, not easily swappable, but `Buffer.allocUnsafe()` skips zeroing

If this alone helps significantly, your bottleneck is allocation and you should apply the techniques from `memory-optimization.md`.
