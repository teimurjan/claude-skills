# Memory Optimization Reference

## Table of Contents
1. Stack vs Heap Allocation
2. Memory Pooling
3. Arena (Bump) Allocation
4. String Interning
5. Zero-Copy Techniques
6. Reducing Allocation Pressure (General)

---

## 1. Stack vs Heap Allocation

Stack allocation is essentially free — it's a pointer bump. Heap allocation involves a system allocator (malloc/free or language GC), which is orders of magnitude slower. The optimization is: keep data on the stack when its size is known at compile time and its lifetime is scoped.

### When to apply
- Inner loops creating small, fixed-size temporary structures
- Buffers with known upper bounds (e.g., a 4096-byte read buffer)
- Value types that don't need to outlive the current scope

### When NOT to apply
- Data with dynamic or unbounded size
- Data that must outlive the current function (returned to caller, shared across threads)
- Very large structs on the stack risk stack overflow (platform-dependent, typically >1MB is risky)

### Language patterns

**Rust:**
```rust
// HEAP — unnecessary allocation in a hot loop
for item in items {
    let temp = Box::new(Transform::identity()); // heap alloc every iteration
    process(item, &temp);
}

// STACK — the transform is small and scoped
for item in items {
    let temp = Transform::identity(); // stack, free
    process(item, &temp);
}
```

**C/C++:**
```c
// Use alloca() or VLAs for dynamic-but-bounded stack allocation
// But prefer fixed-size arrays when the bound is truly known
void process_batch(int n) {
    assert(n <= 256);
    double buffer[256]; // stack-allocated, known upper bound
    // ... use buffer[0..n]
}
```

**TypeScript/JavaScript:**
```typescript
// V8 can stack-allocate ("escape analyze") objects that don't escape the function.
// Help it by avoiding patterns that force heap allocation:

// BAD: object escapes via closure
function process(items: number[]) {
    const state = { sum: 0, count: 0 };
    items.forEach(x => { state.sum += x; state.count++; }); // state escapes into closure
    return state;
}

// BETTER: local variables (primitives are always stack/register)
function process(items: number[]) {
    let sum = 0;
    let count = 0;
    for (let i = 0; i < items.length; i++) {
        sum += items[i];
        count++;
    }
    return { sum, count }; // single allocation at return
}
```

**Go:**
```go
// Go's escape analysis decides stack vs heap. Help it:
// - Don't return pointers to local variables
// - Don't store locals in interfaces
// - Use `go build -gcflags='-m'` to see escape decisions

// ESCAPES to heap — pointer returned
func newConfig() *Config {
    c := Config{} // will be heap-allocated because &c escapes
    return &c
}

// STAYS on stack — value returned
func newConfig() Config {
    return Config{} // caller's stack frame holds this
}
```

### Key insight
The stack vs heap decision matters most when it happens thousands or millions of times. A single heap allocation is fine. A heap allocation per pixel in an image diff is catastrophic.

---

## 2. Memory Pooling

Reuse objects instead of allocating and freeing them repeatedly. A pool maintains a set of pre-allocated objects and hands them out on request. When the caller is done, objects are returned to the pool, not freed.

### When to apply
- Frequent allocation/deallocation of same-type, same-size objects
- Connection pools, buffer pools, goroutine-local scratch space
- Game entities, particle systems, network packet buffers

### When NOT to apply
- Varied-size allocations (pools work best for uniform sizes)
- Objects with complex initialization that can't be cheaply reset
- Low-frequency allocations where a pool adds unnecessary complexity

### Patterns

**Rust (typed pool):**
```rust
struct Pool<T> {
    free_list: Vec<T>,
}

impl<T: Default> Pool<T> {
    fn acquire(&mut self) -> T {
        self.free_list.pop().unwrap_or_default()
    }

    fn release(&mut self, mut item: T) {
        // Reset item to default state if needed
        self.free_list.push(item);
    }
}
```

**Go (sync.Pool):**
```go
var bufPool = sync.Pool{
    New: func() interface{} {
        return make([]byte, 0, 4096)
    },
}

func processRequest(data []byte) {
    buf := bufPool.Get().([]byte)
    buf = buf[:0] // reset length, keep capacity
    defer bufPool.Put(buf)
    // use buf...
}
```

**TypeScript (object pool):**
```typescript
class ObjectPool<T> {
    private pool: T[] = [];
    constructor(private factory: () => T, private reset: (obj: T) => void) {}

    acquire(): T {
        return this.pool.pop() ?? this.factory();
    }

    release(obj: T): void {
        this.reset(obj);
        this.pool.push(obj);
    }
}

// Usage: reuse pixel diff result objects
const resultPool = new ObjectPool(
    () => ({ x: 0, y: 0, diff: 0 }),
    (r) => { r.x = 0; r.y = 0; r.diff = 0; }
);
```

### Gotchas
- **Thread safety**: pools must be synchronized in multithreaded contexts (Go's `sync.Pool` handles this; in Rust, use `Mutex<Vec<T>>` or a lock-free pool)
- **Memory leaks**: if the pool grows unboundedly, set a max capacity and actually free excess objects
- **Stale state**: always reset objects on release — leftover state causes subtle bugs

---

## 3. Arena (Bump) Allocation

An arena allocates by bumping a pointer forward in a pre-allocated buffer. Deallocation is all-or-nothing: you free the entire arena at once. This is extremely fast (one pointer increment per allocation) and eliminates individual free() calls.

### When to apply
- Phase-based work: parse a request, process it, discard everything
- AST/IR construction in compilers and parsers
- Per-frame allocation in games or real-time systems
- Any batch of short-lived, heterogeneous allocations

### When NOT to apply
- Long-lived objects with independent lifetimes
- When you need to free individual objects
- When total allocation size is unpredictable and could be very large

### Patterns

**Rust (bumpalo):**
```rust
use bumpalo::Bump;

fn parse_document(input: &str) -> Document {
    let arena = Bump::new();
    // All allocations use the arena — blazing fast
    let tokens = arena.alloc_slice_copy(&tokenize(input));
    let ast = parse_tokens(&arena, tokens);
    // Convert AST to owned Document before arena drops
    ast.to_owned()
    // arena drops here — all memory freed in one shot
}
```

**C:**
```c
typedef struct {
    char *base;
    size_t offset;
    size_t capacity;
} Arena;

void *arena_alloc(Arena *a, size_t size) {
    // Align to 8 bytes
    size_t aligned = (a->offset + 7) & ~7;
    assert(aligned + size <= a->capacity);
    void *ptr = a->base + aligned;
    a->offset = aligned + size;
    return ptr;
}

void arena_reset(Arena *a) {
    a->offset = 0; // "free" everything instantly
}
```

**TypeScript (batch processing):**
```typescript
// In JS/TS, you can't control allocation directly, but you can
// use the arena *pattern*: pre-allocate arrays, reuse them per batch.
class PixelArena {
    private buffer: Uint8Array;
    private offset = 0;

    constructor(capacity: number) {
        this.buffer = new Uint8Array(capacity);
    }

    alloc(size: number): Uint8Array {
        const slice = this.buffer.subarray(this.offset, this.offset + size);
        this.offset += size;
        return slice;
    }

    reset(): void {
        this.offset = 0; // instant "free"
    }
}
```

### Key insight
Arenas shine when allocation lifetimes are grouped. If you can say "everything allocated in this phase can be freed together", use an arena.

---

## 4. String Interning

Store one copy of each unique string and use references (indices, pointers, or integer handles) everywhere else. Comparisons become integer comparisons (O(1)) instead of byte-by-byte (O(n)).

### When to apply
- Symbol tables (compilers, interpreters)
- Property names, enum-like string keys used repeatedly
- Deduplication of repeated strings in data processing
- Any hot path doing frequent string equality checks

### When NOT to apply
- Strings are mostly unique (interning overhead exceeds savings)
- Write-heavy workloads where strings are created but rarely compared
- When string lifetime management is complicated (interned strings often live forever)

### Patterns

**Rust:**
```rust
use std::collections::HashMap;

struct Interner {
    map: HashMap<String, u32>,
    strings: Vec<String>,
}

impl Interner {
    fn intern(&mut self, s: &str) -> u32 {
        if let Some(&id) = self.map.get(s) {
            return id;
        }
        let id = self.strings.len() as u32;
        self.strings.push(s.to_owned());
        self.map.insert(s.to_owned(), id);
        id
    }

    fn resolve(&self, id: u32) -> &str {
        &self.strings[id as usize]
    }
}
// Now compare u32 instead of &str — huge win in hot loops
```

**TypeScript:**
```typescript
class StringInterner {
    private map = new Map<string, number>();
    private strings: string[] = [];

    intern(s: string): number {
        let id = this.map.get(s);
        if (id !== undefined) return id;
        id = this.strings.length;
        this.strings.push(s);
        this.map.set(s, id);
        return id;
    }

    resolve(id: number): string {
        return this.strings[id];
    }
}
```

### Performance note
In V8, short strings (<64 bytes) used as object keys are often already interned. The biggest wins come from explicit interning of longer strings or strings used in arrays/loops rather than as object keys.

---

## 5. Zero-Copy Techniques

Avoid copying data by passing references, slices, or views of existing buffers. The data stays in one place; you just create lightweight handles that point into it.

### When to apply
- Parsing protocols or file formats (reference slices of input buffer)
- Pipeline stages passing large buffers downstream
- Networking: reading from a socket buffer without copying to user space
- Image processing: operating on sub-regions of a pixel buffer

### When NOT to apply
- When you need ownership semantics (the source might be freed)
- When data needs to be mutated independently
- When the overhead of lifetime tracking exceeds the copy cost
- Small data where copying is cheaper than indirection

### Patterns

**Rust (slices and borrowed views):**
```rust
// ZERO-COPY: parse JSON keys as &str references into the input
fn parse_keys<'a>(input: &'a str) -> Vec<&'a str> {
    // Keys borrow from input — no allocation per key
    input.split(',').map(|s| s.trim()).collect()
}

// Bytes: work on slices of the original buffer
fn process_chunk(image: &[u8], x: usize, y: usize, w: usize, h: usize, stride: usize) {
    for row in y..y+h {
        let row_slice = &image[row * stride + x * 4..(row * stride + (x + w) * 4)];
        // operate on row_slice — no copy
    }
}
```

**TypeScript (TypedArray views):**
```typescript
// ZERO-COPY: create views into the same underlying ArrayBuffer
const buffer = new ArrayBuffer(1024 * 1024);
const fullView = new Uint8Array(buffer);
const header = new Uint8Array(buffer, 0, 64);      // same memory, offset 0
const payload = new Uint8Array(buffer, 64, 1024);   // same memory, offset 64

// Subarray also creates a view, not a copy
const row = fullView.subarray(rowStart, rowEnd); // zero-copy
const rowCopy = fullView.slice(rowStart, rowEnd); // THIS copies — avoid in hot paths
```

**Go (slices):**
```go
// Go slices are already views into an underlying array
data := make([]byte, 4096)
header := data[:64]   // zero-copy view
payload := data[64:]  // zero-copy view
// Both share the same backing array
```

### Critical distinction in JavaScript/TypeScript
- `TypedArray.subarray()` — zero-copy view (shares buffer)
- `TypedArray.slice()` — copies data (new buffer)
- `Buffer.slice()` in Node.js — zero-copy (unlike Array.slice!)

---

## 6. Reducing Allocation Pressure (General)

Beyond the specific techniques above, here are general strategies for reducing the GC/allocator burden.

### Pre-size collections
```rust
let mut vec = Vec::with_capacity(expected_count); // one allocation instead of log(n)
```
```typescript
const arr = new Array(expectedCount); // pre-allocate backing store
```
```go
m := make(map[string]int, expectedCount) // pre-size map
s := make([]int, 0, expectedCount)       // pre-size slice
```

### Reuse buffers across iterations
```typescript
// BAD: allocates a new result array every call
function processFrame(pixels: Uint8Array): Uint8Array {
    const result = new Uint8Array(pixels.length);
    // ...
    return result;
}

// GOOD: caller provides the output buffer
function processFrame(pixels: Uint8Array, out: Uint8Array): void {
    // write directly into out
}
```

### Avoid intermediate allocations
```rust
// BAD: creates intermediate Vec
let processed: Vec<_> = items.iter().map(|x| transform(x)).collect();
let result: Vec<_> = processed.iter().filter(|x| x.is_valid()).collect();

// GOOD: chain iterators — no intermediate collection
let result: Vec<_> = items.iter()
    .map(|x| transform(x))
    .filter(|x| x.is_valid())
    .collect(); // single allocation
```

### Flatten nested allocations
```typescript
// BAD: Vec<Vec<u8>> or Array<Array<number>> — N+1 allocations
const rows: number[][] = Array.from({ length: height }, () => new Array(width));

// GOOD: flat buffer + index math — 1 allocation
const pixels = new Uint8Array(width * height * 4);
const index = (x: number, y: number) => (y * width + x) * 4;
```

### Move allocations out of hot loops
If you must allocate, do it once before the loop and reuse inside:
```rust
let mut scratch = vec![0u8; MAX_ROW_SIZE];
for row in 0..height {
    scratch.clear(); // reuse allocation, just reset length
    process_row(row, &mut scratch);
}
```
