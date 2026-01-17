# AMD64 Systems Programming — Week 7
## Caching and Memory Types

**Course**: AMD x86-64 Systems Programming  
**Primary Text**: AMD64 Architecture Programmer's Manual, Volume 2  
**Week Duration**: ~15-20 hours of focused study  

---

# Lecture Notes

## 1. Cache Hierarchy

### Modern AMD Cache Structure

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    Typical AMD Zen Cache Hierarchy                      │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  Per-Core:                                                             │
│  ┌──────────────┬──────────────┐                                       │
│  │   L1 I-Cache │   L1 D-Cache │  32KB each, 8-way, ~4 cycle latency  │
│  └──────────────┴──────────────┘                                       │
│           │              │                                              │
│           └──────┬───────┘                                             │
│                  ▼                                                      │
│         ┌──────────────┐                                               │
│         │   L2 Cache   │  512KB-1MB, 8-way, ~12 cycle latency         │
│         └──────────────┘                                               │
│                  │                                                      │
│  ───────────────────────────────────────────────────────────           │
│                                                                         │
│  Shared (CCX/CCD):                                                     │
│         ┌──────────────────────────────────────────┐                   │
│         │              L3 Cache                     │                   │
│         │   32-96MB, 16-way, ~40 cycle latency     │                   │
│         │   (Victim cache / Shared across cores)   │                   │
│         └──────────────────────────────────────────┘                   │
│                                                                         │
│  ───────────────────────────────────────────────────────────           │
│                                                                         │
│                  ┌──────────────┐                                       │
│                  │  Main Memory │  ~100+ cycle latency                 │
│                  │    (DRAM)    │                                       │
│                  └──────────────┘                                       │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### Cache Line Structure

```
Cache Line (64 bytes on x86-64):
┌─────────────────────────────────────────────────────────────────────────┐
│  Tag  │  Set Index  │  Offset  │                                       │
│ (var) │   (var)     │ (6 bits) │                                       │
└───────┴─────────────┴──────────┘

Example: 32KB, 8-way cache, 64B lines:
  - 512 sets (32KB / 8 ways / 64B = 64 sets... wait, 32KB/64B = 512 lines)
  - Actually: 512 lines / 8 ways = 64 sets
  - Set index: 6 bits
  - Offset: 6 bits
  - Tag: remaining bits
```

> **AMD Manual Reference**: Volume 2, Chapter 7 "Memory System"

---

## 2. Memory Types (MTRR and PAT)

### Memory Type Register (MTRR)

```
Memory Types:
┌──────┬────────────────┬─────────────────────────────────────────────────┐
│ Type │ Name           │ Behavior                                        │
├──────┼────────────────┼─────────────────────────────────────────────────┤
│  0   │ UC (Uncacheable)│ No caching, strict ordering                    │
│  1   │ WC (Write Comb.)│ No caching, writes may combine                 │
│  4   │ WT (Write-Through)│ Reads cached, writes go to memory           │
│  5   │ WP (Write-Protect)│ Reads cached, writes cause #GP              │
│  6   │ WB (Write-Back) │ Full caching, best performance                 │
└──────┴────────────────┴─────────────────────────────────────────────────┘
```

### MTRR MSRs

```
Fixed MTRRs (cover first 1MB):
  MSR 0x250: MTRR_FIX64K_00000   64KB ranges 0x00000-0x7FFFF
  MSR 0x258: MTRR_FIX16K_80000   16KB ranges 0x80000-0x9FFFF
  MSR 0x259: MTRR_FIX16K_A0000   16KB ranges 0xA0000-0xBFFFF
  MSR 0x268-0x26F: MTRR_FIX4K_*  4KB ranges 0xC0000-0xFFFFF

Variable MTRRs (arbitrary ranges):
  MSR 0x200+2n: MTRR_PHYSBASE_n  Base address + type
  MSR 0x201+2n: MTRR_PHYSMASK_n  Mask + valid bit
```

### Page Attribute Table (PAT)

```
PAT allows per-page memory type specification:

PAT MSR (0x277) contains 8 entries:
┌─────┬─────┬─────┬─────┬─────┬─────┬─────┬─────┐
│ PA7 │ PA6 │ PA5 │ PA4 │ PA3 │ PA2 │ PA1 │ PA0 │
└─────┴─────┴─────┴─────┴─────┴─────┴─────┴─────┘
  56    48    40    32    24    16     8     0

Page Table Entry selects PAT entry via:
  Index = (PAT bit << 2) | (PCD << 1) | PWT

Default PAT:
  PA0 = WB, PA1 = WT, PA2 = UC-, PA3 = UC
  PA4 = WB, PA5 = WT, PA6 = UC-, PA7 = UC
```

### Combining MTRR and PAT

```
Final memory type = MTRR type ∩ PAT type

Precedence (effective):
  UC > UC- > WC > WT > WP > WB

Example:
  MTRR says: WB
  PAT says: WC
  Result: WC (more restrictive than WB for caching)
```

---

## 3. Cache Coherency

### MESI Protocol

```
┌─────────────────────────────────────────────────────────────────────────┐
│                         MESI States                                     │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  Modified (M): Line is dirty, only copy in system                      │
│  Exclusive (E): Line is clean, only copy in system                     │
│  Shared (S): Line is clean, may exist in other caches                  │
│  Invalid (I): Line is not valid                                        │
│                                                                         │
│  State Transitions:                                                     │
│                                                                         │
│      Read hit      ┌───┐                                               │
│    ┌──────────────►│ E │◄──────────────┐                               │
│    │               └─┬─┘               │ Read miss                     │
│    │                 │                 │ (no other cache has it)       │
│    │   Write        │ Snoop read      │                                │
│    │                ▼                  │                                │
│  ┌─┴─┐           ┌───┐             ┌───┐                               │
│  │ M │◄──────────│ S │─────────────│ I │                               │
│  └───┘  Write    └───┘  Snoop inv  └───┘                               │
│    │                ▲                  ▲                                │
│    │                │ Read miss        │                                │
│    │                │ (shared)         │ Write from other              │
│    └────────────────┴──────────────────┘                               │
│       Writeback                                                        │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### Cache Line Bouncing (False Sharing)

```c
// BAD: Variables in same cache line, accessed by different threads
struct {
    volatile int counter1;  // Thread 1
    volatile int counter2;  // Thread 2
} shared;  // Both in same 64-byte line!

// GOOD: Pad to separate cache lines
struct {
    volatile int counter1;
    char padding[60];       // Force counter2 to different line
    volatile int counter2;
} shared;
```

---

## 4. Cache Control Instructions

### Prefetch

```asm
; Hint to load data into cache
prefetcht0 [rsi]      ; Prefetch to all cache levels
prefetcht1 [rsi]      ; Prefetch to L2 and above
prefetcht2 [rsi]      ; Prefetch to L3 and above
prefetchnta [rsi]     ; Prefetch non-temporal (minimize cache pollution)

prefetchw [rsi]       ; Prefetch for write (AMD-specific)
```

### Cache Flush

```asm
; Flush cache line
clflush [rsi]         ; Write back and invalidate (ordered)
clflushopt [rsi]      ; Optimized clflush (weakly ordered)
clwb [rsi]            ; Write back, don't invalidate (persistent memory)

; Memory fences
mfence                ; Full memory fence
sfence                ; Store fence
lfence                ; Load fence (also serializes instructions)

; Write-back entire cache
wbinvd                ; Write back and invalidate all caches (privileged)
invd                  ; Invalidate without writeback (DANGEROUS)
```

### Non-Temporal Operations

```asm
; Bypass cache for streaming writes
movntdq [rdi], xmm0   ; Store 128 bits non-temporal
movntps [rdi], xmm0   ; Store 128 bits non-temporal (float)
movnti [rdi], eax     ; Store 32/64 bits non-temporal

; Non-temporal loads (AMD SSE4a)
movntsd xmm0, [rsi]   ; Load 64 bits non-temporal
movntss xmm0, [rsi]   ; Load 32 bits non-temporal
```

---

## 5. Practical Performance Implications

### Cache-Friendly Code

```c
// BAD: Column-major access of row-major array
for (int j = 0; j < cols; j++)
    for (int i = 0; i < rows; i++)
        sum += array[i][j];  // Stride = row_size, cache thrashing

// GOOD: Row-major access
for (int i = 0; i < rows; i++)
    for (int j = 0; j < cols; j++)
        sum += array[i][j];  // Sequential access, cache friendly
```

### Cache Blocking

```c
// Blocked matrix multiplication for better cache utilization
#define BLOCK 64

for (int ii = 0; ii < N; ii += BLOCK)
    for (int jj = 0; jj < N; jj += BLOCK)
        for (int kk = 0; kk < N; kk += BLOCK)
            for (int i = ii; i < min(ii+BLOCK, N); i++)
                for (int j = jj; j < min(jj+BLOCK, N); j++)
                    for (int k = kk; k < min(kk+BLOCK, N); k++)
                        C[i][j] += A[i][k] * B[k][j];
```

---

## 6. CPUID Cache Information

### Querying Cache Parameters

```asm
; CPUID leaf 0x80000005: L1 cache info (AMD)
mov eax, 0x80000005
cpuid
; ECX[7:0] = L1 D-cache line size
; ECX[15:8] = L1 D-cache lines per tag
; ECX[23:16] = L1 D-cache associativity
; ECX[31:24] = L1 D-cache size in KB

; CPUID leaf 0x80000006: L2/L3 cache info (AMD)
mov eax, 0x80000006
cpuid
; Similar encoding for L2/L3
```

---

# Key Readings

## Required

| Source | Section | Topic |
|--------|---------|-------|
| AMD Vol 2 | Chapter 7.1-7.6 | Memory types, MTRR |
| AMD Vol 2 | Chapter 7.7-7.8 | PAT, Cache control |

## Supplementary
- "What Every Programmer Should Know About Memory" - Ulrich Drepper
- Agner Fog's optimization manuals

---

# Exercises & Mini-Projects

## Exercise 1: Cache Timing Attack

Measure cache hit vs. miss timing to demonstrate side-channel potential.

## Exercise 2: MTRR Dump

Write code to dump all MTRR settings on your system.

## Exercise 3: False Sharing Benchmark

Demonstrate performance impact of false sharing with counters.

## Mini-Project: Cache-Optimized Matrix Multiply

Implement naive vs. blocked matrix multiplication and benchmark.

---

# Quiz

**Q1**: What are the five memory types and their uses?

**Q2**: How does the CPU combine MTRR and PAT types?

**Q3**: What is the MESI protocol?

**Q4**: When would you use CLFLUSH vs. CLWB?

**Q5**: What causes false sharing?

---

# Priority Guide

## CRITICAL
- Memory type overview (WB, UC, WC, WT)
- Cache hierarchy understanding
- Cache line size importance

## IMPORTANT
- MTRR programming
- PAT configuration
- MESI protocol basics

## OPTIONAL
- Full MTRR MSR details
- Cache coherency protocols variations
- NUMA implications

---

**Next Week**: SIMD Programming—SSE, AVX, and AVX-512.
