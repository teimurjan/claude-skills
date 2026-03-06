# CPU Optimization Reference

## Table of Contents
1. Cache Locality and Data Layout (AoS vs SoA)
2. Avoiding False Sharing in Multithreaded Code
3. Branch Prediction and Loop Optimization
4. SIMD / Auto-Vectorization
5. Lazy Evaluation and Short-Circuit Patterns

---

## 1. Cache Locality and Data Layout (AoS vs SoA)

Modern CPUs don't fetch individual bytes from RAM — they fetch **cache lines** (typically 64 bytes). If your data layout causes the CPU to load cache lines full of data you won't use, you're wasting memory bandwidth. This is often the single largest performance issue in data-intensive code.

### The core concept

**Array of Structures (AoS):** Each entity is a struct; you have an array of them.
```
[{x, y, z, health, name, sprite}, {x, y, z, health, name, sprite}, ...]
```
When you iterate over positions (x, y, z), each cache line also loads health, name, sprite — wasted bandwidth.

**Structure of Arrays (SoA):** Each field is a separate array.
```
xs:      [x, x, x, ...]
ys:      [y, y, y, ...]
zs:      [z, z, z, ...]
healths: [h, h, h, ...]
```
When you iterate over positions, you get dense, contiguous x values — every byte in the cache line is useful.

### When to use SoA
- Iterating over one or few fields across many entities (physics update, rendering, collision)
- SIMD processing (SoA feeds SIMD naturally — contiguous floats ready for vector ops)
- Very large entity counts (thousands+) where cache misses dominate

### When to keep AoS
- Accessing all fields of one entity at a time (loading/saving, serialization)
- Small entity counts where cache effects are negligible
- Codebases where AoS is the natural domain model and SoA would add excessive complexity

### Hybrid: AoSoA
Group entities into small batches (e.g., 8 or 16) where fields within a batch are SoA, but batches are stored as an array. This gives SoA benefits while keeping related data near each other.

### Patterns

**Rust (SoA with struct-of-vecs):**
```rust
// AoS — natural but cache-unfriendly for field-specific iteration
struct Particle { x: f32, y: f32, z: f32, mass: f32, color: u32 }
let particles: Vec<Particle> = vec![...]; // 20 bytes per particle

// SoA — cache-friendly for position updates
struct Particles {
    x: Vec<f32>,
    y: Vec<f32>,
    z: Vec<f32>,
    mass: Vec<f32>,
    color: Vec<u32>,
}

// Position update now reads only x, y, z — dense, vectorizable
fn update_positions(p: &mut Particles, dt: f32) {
    for i in 0..p.x.len() {
        p.x[i] += p.vx[i] * dt;
        p.y[i] += p.vy[i] * dt;
    }
}
```

**TypeScript (SoA with TypedArrays):**
```typescript
// AoS — each particle is an object, scattered in heap
interface Particle { x: number; y: number; vx: number; vy: number; }
const particles: Particle[] = [...]; // objects scattered across heap

// SoA — contiguous typed arrays, cache-friendly
const count = 10000;
const x  = new Float32Array(count);
const y  = new Float32Array(count);
const vx = new Float32Array(count);
const vy = new Float32Array(count);

function updatePositions(dt: number) {
    for (let i = 0; i < count; i++) {
        x[i] += vx[i] * dt;
        y[i] += vy[i] * dt;
    }
}
```

**C (SoA):**
```c
// SoA layout for image pixel processing
struct ImageChannels {
    uint8_t *r;  // all red values contiguous
    uint8_t *g;  // all green values contiguous
    uint8_t *b;  // all blue values contiguous
    uint8_t *a;  // all alpha values contiguous
    size_t count;
};
// Comparing only alpha channels? Read only the 'a' array — perfect cache usage.
```

### Measuring cache performance
- **Linux**: `perf stat -e cache-misses,cache-references ./your_program`
- **macOS**: Instruments → Counters (PMC) → L1/L2 cache misses
- **valgrind**: `valgrind --tool=cachegrind ./your_program`

---

## 2. Avoiding False Sharing in Multithreaded Code

False sharing occurs when threads on different CPU cores modify different variables that happen to reside on the **same cache line**. The cache coherence protocol forces the cache line to bounce between cores, serializing what should be parallel work.

### Identifying false sharing
Symptoms: multithreaded code that scales poorly (or gets *slower*) with more threads, especially when threads write to adjacent memory locations.

### The fix: pad or separate

**Rust:**
```rust
use std::sync::atomic::{AtomicU64, Ordering};

// BAD: counters are adjacent — likely on the same cache line
struct Counters {
    thread_0: AtomicU64,
    thread_1: AtomicU64,
    thread_2: AtomicU64,
    thread_3: AtomicU64,
}

// GOOD: each counter on its own cache line
#[repr(C)]
struct PaddedCounter {
    value: AtomicU64,
    _pad: [u8; 56], // pad to 64 bytes (cache line size)
}

struct Counters {
    counters: [PaddedCounter; 4],
}
```

**C/C++:**
```c
// Use alignas or manual padding
struct alignas(64) PaddedCounter {
    _Atomic uint64_t value;
};
// Each counter guaranteed on its own cache line
struct PaddedCounter counters[NUM_THREADS];
```

**Go:**
```go
// Go struct padding to avoid false sharing
type PaddedCounter struct {
    value atomic.Int64
    _     [56]byte // pad to cache line
}

type Counters struct {
    c [4]PaddedCounter
}
```

**Java:**
```java
// Use @Contended annotation (JDK internal, enable with --add-opens)
// Or manual padding:
class PaddedCounter {
    volatile long p0, p1, p2, p3, p4, p5, p6; // 7 longs = 56 bytes padding
    volatile long value;
    volatile long q0, q1, q2, q3, q4, q5, q6;
}
```

### Alternative: thread-local accumulation
Instead of sharing counters at all, each thread accumulates locally and merges at the end:
```rust
// Each thread owns its own counter — no sharing at all
let local_count = items_chunk.iter().filter(|x| predicate(x)).count();
// Merge at the end (one atomic add instead of millions)
global_count.fetch_add(local_count as u64, Ordering::Relaxed);
```
This is almost always better than padding when applicable.

### Detecting false sharing
- **Linux**: `perf c2c` — specifically designed to detect false sharing
- **Intel VTune**: Microarchitecture Exploration → contested cache lines
- **Empirically**: if adding 64-byte padding between per-thread variables dramatically improves throughput, you had false sharing

---

## 3. Branch Prediction and Loop Optimization

Modern CPUs predict branch outcomes and speculatively execute the predicted path. Mispredictions cost 10-20 cycles because the pipeline must be flushed. In hot loops processing millions of elements, this matters.

### Techniques to help the branch predictor

**Sort data when branching on it:**
```rust
// If you're filtering by a condition, sorting first makes the branch perfectly predictable
// (all trues come first, then all falses — predictor learns immediately)
data.sort_unstable_by_key(|x| x.category);
for item in &data {
    if item.category == target { // predictable after initial transition
        process(item);
    }
}
```

**Replace branches with arithmetic:**
```c
// BRANCHY: unpredictable if values are random
for (int i = 0; i < n; i++) {
    if (a[i] > b[i]) count++;
}

// BRANCHLESS: conditional move, no misprediction possible
for (int i = 0; i < n; i++) {
    count += (a[i] > b[i]); // boolean-to-int, compiler emits cmov
}
```

**Use lookup tables instead of switch/if chains:**
```typescript
// BRANCHY
function categoryName(code: number): string {
    if (code === 0) return "none";
    if (code === 1) return "low";
    if (code === 2) return "medium";
    if (code === 3) return "high";
    return "unknown";
}

// BRANCHLESS — single array lookup
const CATEGORY_NAMES = ["none", "low", "medium", "high"];
function categoryName(code: number): string {
    return CATEGORY_NAMES[code] ?? "unknown";
}
```

### Loop optimization

**Loop unrolling** — process multiple elements per iteration to reduce branch overhead and enable instruction-level parallelism:
```rust
// Manual unroll (compiler may do this, but sometimes it helps to be explicit)
let mut sum = 0u64;
let chunks = data.chunks_exact(4);
let remainder = chunks.remainder();
for chunk in chunks {
    sum += chunk[0] as u64;
    sum += chunk[1] as u64;
    sum += chunk[2] as u64;
    sum += chunk[3] as u64;
}
for &val in remainder {
    sum += val as u64;
}
```

**Loop interchange** — iterate in the order that matches memory layout:
```c
// BAD: column-major access in row-major array — cache thrashing
for (int j = 0; j < cols; j++)
    for (int i = 0; i < rows; i++)
        sum += matrix[i][j]; // stride = cols * sizeof(element)

// GOOD: row-major access — sequential, cache-friendly
for (int i = 0; i < rows; i++)
    for (int j = 0; j < cols; j++)
        sum += matrix[i][j]; // stride = sizeof(element)
```

**Move invariants out of loops:**
```typescript
// BAD: recomputes config.threshold every iteration
for (let i = 0; i < pixels.length; i++) {
    if (pixels[i] > config.getThreshold()) { ... } // virtual call in hot loop
}

// GOOD: hoist the invariant
const threshold = config.getThreshold();
for (let i = 0; i < pixels.length; i++) {
    if (pixels[i] > threshold) { ... }
}
```

---

## 4. SIMD / Auto-Vectorization

SIMD (Single Instruction, Multiple Data) processes 4, 8, 16, or more values in a single instruction. A 256-bit AVX register can hold 8 floats — one instruction does 8 multiplications. This is a 4-8x speedup on eligible loops.

### Conditions for auto-vectorization
The compiler will auto-vectorize a loop if:
1. Iterations are independent (no loop-carried dependencies)
2. Data is contiguous in memory (arrays, not linked lists)
3. The loop body is simple (no function calls, no complex control flow)
4. Trip count is known or estimable
5. No aliasing between input and output pointers (use `restrict` in C, or separate buffers)

### Helping the compiler vectorize

**Rust:**
```rust
// This auto-vectorizes well:
fn add_arrays(a: &mut [f32], b: &[f32]) {
    assert_eq!(a.len(), b.len());
    for i in 0..a.len() {
        a[i] += b[i];
    }
}

// Hint alignment for better codegen:
#[repr(align(32))] // AVX alignment
struct AlignedBuffer([f32; 1024]);

// Check if it vectorized: cargo rustc -- --emit=asm
// Or: RUSTFLAGS="-C target-cpu=native" cargo build --release
```

**C/C++:**
```c
// Use restrict to tell the compiler pointers don't alias
void add_arrays(float * restrict a, const float * restrict b, size_t n) {
    for (size_t i = 0; i < n; i++) {
        a[i] += b[i]; // compiler can now vectorize safely
    }
}

// Compile with: gcc -O3 -march=native -ftree-vectorize
// Check: gcc -O3 -march=native -fopt-info-vec-optimized
```

**Explicit SIMD (Rust):**
```rust
// When auto-vectorization isn't enough, use explicit SIMD
#[cfg(target_arch = "x86_64")]
use std::arch::x86_64::*;

unsafe fn sum_f32_avx(data: &[f32]) -> f32 {
    let mut acc = _mm256_setzero_ps();
    for chunk in data.chunks_exact(8) {
        let v = _mm256_loadu_ps(chunk.as_ptr());
        acc = _mm256_add_ps(acc, v);
    }
    // horizontal sum of acc...
    horizontal_sum_256(acc)
}
```

**WASM SIMD (Rust targeting wasm32):**
```rust
#[cfg(target_arch = "wasm32")]
use std::arch::wasm32::*;

fn pixel_diff_simd(a: &[u8], b: &[u8]) -> u32 {
    let mut total = i32x4_splat(0);
    for (ca, cb) in a.chunks_exact(16).zip(b.chunks_exact(16)) {
        let va = v128_load(ca.as_ptr() as *const v128);
        let vb = v128_load(cb.as_ptr() as *const v128);
        // compute absolute differences...
    }
    // extract and sum
}
```

### Auto-vectorization blockers to avoid
- **Function calls in the loop body** (unless inlined)
- **Early exits / breaks** (prevents vectorization)
- **Non-contiguous access** (gather/scatter is slow)
- **Loop-carried dependencies** (each iteration depends on the previous)
- **Pointer aliasing** (compiler assumes the worst without `restrict`)
- **Mixed data types** (widening/narrowing instructions are expensive)

### Verifying vectorization
- **Rust**: `RUSTFLAGS="-C target-cpu=native" cargo build --release`, inspect with `cargo asm`
- **C/C++**: `-fopt-info-vec` (GCC), `-Rpass=loop-vectorize` (Clang)
- **Go**: Go doesn't auto-vectorize well; use assembly or CGo for SIMD-critical paths
- **General**: look for `vmovups`, `vaddps`, `vpmovmskb` (x86 AVX) or `v128.load`, `i32x4.add` (WASM) in the disassembly

---

## 5. Lazy Evaluation and Short-Circuit Patterns

Don't compute values until they're needed. Short-circuit evaluation avoids unnecessary work when an early condition determines the result.

### Lazy evaluation

**Rust:**
```rust
// EAGER: computes expensive_check even if cheap_check fails
let result = expensive_check(data) && cheap_check(data);

// LAZY: short-circuits — if cheap_check fails, expensive_check never runs
let result = cheap_check(data) && expensive_check(data);

// Lazy iterators — nothing computed until consumed
let result: Vec<_> = items.iter()
    .filter(|x| quick_filter(x))        // eliminates 90% of items
    .filter(|x| expensive_filter(x))     // only runs on the 10%
    .map(|x| transform(x))
    .take(10)                            // stop after 10 results
    .collect();
```

**TypeScript:**
```typescript
// Lazy computation with getters
class ExpensiveResult {
    private _value?: number;

    get value(): number {
        if (this._value === undefined) {
            this._value = computeExpensiveThing(); // computed only on first access
        }
        return this._value;
    }
}

// Short-circuit pattern: order checks from cheapest to most expensive
function isMatch(pixel: Uint8Array, target: Uint8Array): boolean {
    // Alpha check first (single byte comparison, eliminates transparent pixels)
    if (pixel[3] !== target[3]) return false;
    // Then red (cheapest remaining)
    if (pixel[0] !== target[0]) return false;
    // Then green
    if (pixel[1] !== target[1]) return false;
    // Blue last
    return pixel[2] === target[2];
}
```

**Go:**
```go
// sync.Once for lazy initialization
var expensiveResult *Result
var once sync.Once

func getResult() *Result {
    once.Do(func() {
        expensiveResult = computeExpensiveThing()
    })
    return expensiveResult
}
```

### Short-circuit ordering principle
When combining multiple conditions with AND/OR:
1. Put the **cheapest** check first
2. Put the check that **rejects the most** items first (highest selectivity)
3. Ideal: a cheap check that rejects 90% of items, saving the expensive check for the remaining 10%

### When lazy evaluation hurts
- When the value is always needed (lazy adds overhead: the check + branch)
- In latency-sensitive paths where you want to precompute during idle time
- When lazy initialization causes unpredictable latency spikes (first access is slow)
