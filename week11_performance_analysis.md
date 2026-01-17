# AMD64 Systems Programming — Week 11
## Performance Analysis: PMU, Profiling, and Optimization

**Course**: AMD x86-64 Systems Programming  
**Primary Text**: AMD64 Architecture Programmer's Manual, Volume 2  
**Week Duration**: ~15-20 hours of focused study  

---

# Lecture Notes

## 1. Performance Monitoring Unit (PMU)

### PMU Architecture

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    AMD Performance Monitoring                           │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  Core PMU:                                                             │
│  ├─ 6 General-purpose counters (PerfCtr0-5)                           │
│  ├─ 6 Event select registers (PerfEvtSel0-5)                          │
│  ├─ Fixed-function counters (varies by generation)                    │
│  └─ Global control/status registers                                    │
│                                                                         │
│  Northbridge/Data Fabric PMU:                                          │
│  ├─ Memory controller events                                           │
│  ├─ Interconnect events                                                │
│  └─ L3 cache events                                                    │
│                                                                         │
│  Counting Modes:                                                        │
│  ├─ Event counting (accumulate events)                                │
│  ├─ Sampling (interrupt on overflow)                                  │
│  └─ Precise Event-Based Sampling (PEBS equivalent: IBS)               │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

> **AMD Manual Reference**: Volume 2, Chapter 13 "Performance Monitoring"

### Performance Counter MSRs

```
Core Performance Counters (Zen architecture):
┌──────────────────┬─────────────────────────────────────────────────────┐
│ MSR Address      │ Description                                         │
├──────────────────┼─────────────────────────────────────────────────────┤
│ 0xC001_0200      │ PerfEvtSel0 - Event select for counter 0           │
│ 0xC001_0201      │ PerfCtr0 - Counter 0 value                         │
│ 0xC001_0202      │ PerfEvtSel1                                        │
│ 0xC001_0203      │ PerfCtr1                                           │
│ ...              │ ...                                                 │
│ 0xC001_020A      │ PerfEvtSel5                                        │
│ 0xC001_020B      │ PerfCtr5                                           │
└──────────────────┴─────────────────────────────────────────────────────┘

PerfEvtSel Format:
┌────────────────────────────────────────────────────────────────────────┐
│ 63    32│31  24│23 22│21│20│19│18│17│16│15    8│7       0│            │
├─────────┼──────┼─────┼──┼──┼──┼──┼──┼──┼───────┼─────────┤            │
│ Reserved│ CntMask│Inv│En│  │Int│ Edge│OS│Usr│UnitMask│EventSelect│    │
└────────────────────────────────────────────────────────────────────────┘

Key bits:
  EventSelect[7:0] + [35:32]: Event to count
  UnitMask[15:8]: Event qualification
  Usr (bit 16): Count user-mode events
  OS (bit 17): Count kernel-mode events
  Edge (bit 18): Count edges vs. cycles
  Int (bit 20): Enable interrupt on overflow
  En (bit 22): Enable counter
```

---

## 2. Common Performance Events

### CPU Core Events

```
Event Name              │ EventSelect │ UnitMask │ Description
────────────────────────┼─────────────┼──────────┼─────────────────────────
Retired Instructions    │ 0xC0        │ 0x00     │ Instructions completed
CPU Cycles              │ 0x76        │ 0x00     │ Core clock cycles
Branch Retired          │ 0xC2        │ 0x00     │ All branches retired
Branch Mispredicted     │ 0xC3        │ 0x00     │ Mispredicted branches
L1 D-Cache Miss         │ 0x41        │ 0x01     │ L1 data cache misses
L2 Cache Miss           │ 0x64        │ 0x07     │ L2 cache misses
L3 Cache Miss           │ varies      │ varies   │ (via Data Fabric PMU)
TLB Miss                │ 0x45        │ 0xFF     │ DTLB misses
Stalled Cycles          │ 0x87        │ 0x01     │ Backend stalls
```

### Derived Metrics

```c
// Instructions Per Cycle (IPC)
float ipc = retired_instructions / cpu_cycles;

// Cache Miss Rate  
float l1_miss_rate = l1_misses / l1_accesses;

// Branch Misprediction Rate
float branch_miss_rate = branch_mispred / branch_retired;

// CPI (Cycles Per Instruction) - inverse of IPC
float cpi = cpu_cycles / retired_instructions;
```

---

## 3. Using Performance Counters

### Direct MSR Access (Kernel Mode)

```asm
; Count retired instructions in user mode
setup_counter:
    ; Select event: Retired Instructions (0xC0), User mode only
    mov ecx, 0xC001_0200        ; PerfEvtSel0
    mov eax, 0x0041_00C0        ; En=1, Usr=1, EventSelect=0xC0
    xor edx, edx
    wrmsr
    
    ; Clear counter
    mov ecx, 0xC001_0201        ; PerfCtr0
    xor eax, eax
    xor edx, edx
    wrmsr
    ret

read_counter:
    mov ecx, 0xC001_0201        ; PerfCtr0
    rdmsr
    ; Result in EDX:EAX
    shl rdx, 32
    or rax, rdx
    ret
```

### Linux perf_event Interface

```c
#include <linux/perf_event.h>
#include <sys/syscall.h>

struct perf_event_attr pe = {
    .type = PERF_TYPE_HARDWARE,
    .config = PERF_COUNT_HW_INSTRUCTIONS,
    .disabled = 1,
    .exclude_kernel = 1,
    .exclude_hv = 1,
};

int fd = syscall(__NR_perf_event_open, &pe, 0, -1, -1, 0);

ioctl(fd, PERF_EVENT_IOC_RESET, 0);
ioctl(fd, PERF_EVENT_IOC_ENABLE, 0);

// ... code to measure ...

ioctl(fd, PERF_EVENT_IOC_DISABLE, 0);

long long count;
read(fd, &count, sizeof(count));
printf("Instructions: %lld\n", count);
```

---

## 4. Instruction-Based Sampling (IBS)

### IBS Overview

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    AMD Instruction-Based Sampling                       │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  IBS provides precise attribution of performance data to instructions  │
│                                                                         │
│  IBS Fetch Sampling:                                                   │
│  ├─ Samples instruction fetch operations                              │
│  ├─ Reports: fetch latency, cache hit/miss, TLB hit/miss             │
│  └─ Precise RIP of fetched instruction                                │
│                                                                         │
│  IBS Op Sampling:                                                      │
│  ├─ Samples retired micro-ops                                         │
│  ├─ Reports: execution latency, cache behavior, branch info          │
│  ├─ Data address for loads/stores                                     │
│  └─ Precise RIP of completed operation                                │
│                                                                         │
│  Unlike traditional sampling, IBS has no skid (precise RIP)           │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### IBS MSRs

```
IBS Fetch MSRs:
  0xC001_1030  IBS_FETCH_CTL     - Control/status
  0xC001_1031  IBS_FETCH_LINADDR - Linear address
  0xC001_1032  IBS_FETCH_PHYSADDR - Physical address

IBS Op MSRs:
  0xC001_1033  IBS_OP_CTL        - Control/status
  0xC001_1034  IBS_OP_RIP        - Instruction RIP
  0xC001_1035  IBS_OP_DATA       - Operation data
  0xC001_1036  IBS_OP_DATA2      - Additional data
  0xC001_1037  IBS_OP_DATA3      - Branch/return info
  0xC001_1038  IBS_DC_LINADDR    - Data cache linear addr
  0xC001_1039  IBS_DC_PHYSADDR   - Data cache physical addr
```

---

## 5. Profiling Tools

### perf (Linux)

```bash
# Basic profiling
perf stat ./myprogram

# Detailed hardware events
perf stat -e cycles,instructions,cache-misses,branch-misses ./myprogram

# Record and report
perf record -g ./myprogram
perf report

# Live monitoring
perf top

# AMD-specific events
perf stat -e cpu/event=0xc0,umask=0x00/ ./myprogram

# IBS sampling (if supported)
perf record -e ibs_op// ./myprogram
```

### AMD μProf

```
AMD μProf capabilities:
- CPU profiling with IBS support
- Power profiling
- Memory access analysis
- Thread concurrency analysis
- System-wide or per-process
- GUI and command-line interfaces
```

### VTune (Intel, but works on AMD)

```bash
# Basic hotspot analysis
vtune -collect hotspots ./myprogram

# Memory access analysis  
vtune -collect memory-access ./myprogram
```

---

## 6. RDTSC and RDTSCP

### Time Stamp Counter

```asm
; RDTSC - Read Time Stamp Counter
; Returns 64-bit cycle count in EDX:EAX
; WARNING: May not be serializing

rdtsc
shl rdx, 32
or rax, rdx         ; RAX = cycle count

; RDTSCP - Read TSC and Processor ID
; Also returns processor ID in ECX
; Has implicit LFENCE (waits for prior instructions)

rdtscp
shl rdx, 32
or rax, rdx         ; RAX = cycle count
; ECX = IA32_TSC_AUX (often contains CPU ID)
```

### Proper Benchmarking Pattern

```asm
; Serialize before measurement
cpuid               ; Full serialization (clobbers EAX-EDX)
rdtsc
mov r8, rax
shl rdx, 32
or r8, rdx          ; r8 = start time

; ... code to measure ...

rdtscp              ; Partial serialization (waits for prior ops)
shl rdx, 32
or rax, rdx
sub rax, r8         ; RAX = elapsed cycles

; Note: CPUID before, RDTSCP after is recommended pattern
```

### TSC Considerations

```
TSC behavior varies:
- Invariant TSC: Constant rate regardless of P-state (check CPUID)
- Non-stop TSC: Continues in sleep states
- TSC may differ between cores (check synchronization)

For wall-clock time, prefer:
- clock_gettime(CLOCK_MONOTONIC)
- QueryPerformanceCounter (Windows)
```

---

## 7. Optimization Strategies

### Identifying Bottlenecks

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    Performance Bottleneck Categories                   │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  Frontend Bound (instruction fetch/decode):                            │
│  ├─ I-cache misses                                                     │
│  ├─ Branch mispredictions                                              │
│  └─ Instruction decode bottlenecks                                     │
│                                                                         │
│  Backend Bound (execution):                                            │
│  ├─ Memory Bound:                                                      │
│  │   ├─ L1/L2/L3 cache misses                                         │
│  │   ├─ TLB misses                                                     │
│  │   └─ Memory bandwidth saturation                                   │
│  └─ Core Bound:                                                        │
│      ├─ Execution unit contention                                      │
│      ├─ Long latency operations (div, sqrt)                           │
│      └─ Dependency chains                                              │
│                                                                         │
│  Retiring (good! doing useful work):                                   │
│  └─ High retiring % means efficient code                              │
│                                                                         │
│  Bad Speculation:                                                      │
│  ├─ Branch misprediction recovery                                     │
│  └─ Machine clears                                                     │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### Common Optimizations

```c
// 1. Cache-friendly access patterns
// BAD: Column-major access of row-major array
for (j = 0; j < N; j++)
    for (i = 0; i < N; i++)
        sum += arr[i][j];  // Stride = N * sizeof(element)

// GOOD: Row-major access
for (i = 0; i < N; i++)
    for (j = 0; j < N; j++)
        sum += arr[i][j];  // Sequential access

// 2. Avoid false sharing (pad to cache line)
struct __attribute__((aligned(64))) {
    long counter;
    char padding[56];
} per_cpu_data[NUM_CPUS];

// 3. Reduce branch mispredictions
// Use conditional moves when possible
// Sort data if it improves branch patterns

// 4. Prefetching
__builtin_prefetch(&data[i + 64], 0, 3);
```

---

# Key Readings

## Required

| Source | Section | Topic |
|--------|---------|-------|
| AMD Vol 2 | Chapter 13 | Performance Monitoring |
| AMD PPR | PMC section | Event codes |

## Supplementary
- Agner Fog's optimization manuals
- "Systems Performance" by Brendan Gregg
- perf wiki

---

# Exercises & Mini-Projects

## Exercise 1: IPC Measurement

Write a program that measures its own IPC using perf_event syscall.

## Exercise 2: Cache Miss Analysis

Create a program with controllable cache behavior and measure L1/L2/L3 miss rates.

## Exercise 3: Branch Prediction

Benchmark sorted vs. unsorted array processing to demonstrate branch prediction impact.

## Mini-Project: Profiling Report

Profile a real application (e.g., compression, sorting) and write a detailed performance analysis report with optimization recommendations.

---

# Quiz

**Q1**: What is the difference between RDTSC and RDTSCP?

**Q2**: What does IPC measure and what is a "good" IPC?

**Q3**: How does IBS differ from traditional PMC sampling?

**Q4**: Name three categories of performance bottlenecks.

**Q5**: Why must you serialize around RDTSC for accurate measurements?

---

# Priority Guide

## CRITICAL
- PMC MSR structure
- Common performance events
- Using perf tool
- RDTSC/RDTSCP usage

## IMPORTANT
- IBS concepts
- Bottleneck categories
- Cache optimization patterns

## OPTIONAL
- Full event list per architecture
- Advanced IBS analysis
- Custom perf event definition

---

**Next Week**: Capstone Project—build a complete systems programming project.
