# AMD64 Systems Programming — Week 3
## Instruction Set Deep Dive: Encoding, Decoding, and Performance

**Course**: AMD x86-64 Systems Programming  
**Primary Text**: AMD64 Architecture Programmer's Manual, Volumes 3-5  
**Week Duration**: ~15-20 hours of focused study  

---

# Table of Contents

1. [Lecture Notes](#lecture-notes)
2. [Key Readings](#key-readings)
3. [Recommended Videos](#recommended-videos)
4. [Exercises & Mini-Projects](#exercises--mini-projects)
5. [Quiz](#quiz)
6. [Priority Guide](#priority-guide)

---

# Lecture Notes

## 1. Instruction Encoding Architecture

### The Variable-Length Challenge

x86-64 instructions range from 1 to 15 bytes. This creates complexity but enables density:

```
┌──────────────────────────────────────────────────────────────────────────┐
│                    x86-64 Instruction Format                             │
├──────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  ┌─────────┬──────┬─────────┬────────┬─────┬──────────┬───────────────┐ │
│  │ Legacy  │ REX  │ Opcode  │ ModR/M │ SIB │ Displace │  Immediate   │ │
│  │Prefixes │      │ (1-3B)  │ (0-1B) │(0-1)│  (0-4B)  │   (0-8B)     │ │
│  │ (0-4B)  │(0-1B)│         │        │     │          │              │ │
│  └─────────┴──────┴─────────┴────────┴─────┴──────────┴───────────────┘ │
│                                                                          │
│  Maximum instruction length: 15 bytes                                    │
│  Minimum instruction length: 1 byte (e.g., NOP = 0x90)                  │
│                                                                          │
└──────────────────────────────────────────────────────────────────────────┘
```

> **AMD Manual Reference**: Volume 3, Chapter 1 "Instruction Encoding"

### Legacy Prefixes

```
Group 1 (Lock/Repeat):
  F0 = LOCK       (atomic memory operations)
  F2 = REPNE/REPNZ (repeat while not equal)
  F3 = REP/REPE   (repeat while equal)

Group 2 (Segment Override):
  2E = CS         (code segment - branch hint: not taken)
  36 = SS         (stack segment)
  3E = DS         (data segment - branch hint: taken)
  26 = ES         (extra segment)
  64 = FS         (FS segment - thread local storage)
  65 = GS         (GS segment - kernel per-CPU)

Group 3 (Operand Size):
  66 = Operand size override (32→16 bit in 64-bit mode)

Group 4 (Address Size):
  67 = Address size override (64→32 bit addressing)
```

### REX Prefix Deep Dive

```
┌────────────────────────────────────────────────────────────────────────┐
│                        REX Prefix Byte                                 │
├────────────────────────────────────────────────────────────────────────┤
│   7    6    5    4  │  3   │  2   │  1   │  0   │                     │
│  ─────────────────  │──────│──────│──────│──────│                     │
│   0    1    0    0  │  W   │  R   │  X   │  B   │                     │
│      (fixed)        │      │      │      │      │                     │
├─────────────────────┴──────┴──────┴──────┴──────┤                     │
│                                                  │                     │
│  W = 1: 64-bit operand size                     │                     │
│  W = 0: Default operand size (usually 32-bit)   │                     │
│                                                  │                     │
│  R: Extends ModR/M reg field (0→R8-R15)         │                     │
│  X: Extends SIB index field (0→R8-R15)          │                     │
│  B: Extends ModR/M r/m or SIB base (0→R8-R15)   │                     │
│                                                  │                     │
│  REX range: 0x40-0x4F (any REX prefix)          │                     │
│                                                  │                     │
└──────────────────────────────────────────────────┘
```

**When REX is mandatory:**
- Accessing R8-R15, SPL, BPL, SIL, DIL
- 64-bit operand size (REX.W=1)

**When REX is optional but present:**
- Can be 0x40 (no bits set) just to access new byte registers

### VEX and EVEX Prefixes (AVX/AVX-512)

Modern SIMD uses VEX (2-3 bytes) or EVEX (4 bytes) instead of REX:

```
VEX 2-byte (0xC5):
┌────────┬────────────────────────────────────────┐
│  C5    │  R vvvv L pp                           │
└────────┴────────────────────────────────────────┘

VEX 3-byte (0xC4):
┌────────┬────────────────────────┬──────────────┐
│  C4    │  R X B mmmmm           │  W vvvv L pp │
└────────┴────────────────────────┴──────────────┘

EVEX 4-byte (0x62):
┌────────┬─────────┬─────────┬─────────┐
│  62    │ R X B R'│ W vvvv  │ z L'L b │
│        │ 00 mm   │ 1 pp    │ V' aaa  │
└────────┴─────────┴─────────┴─────────┘
```

---

## 2. ModR/M and SIB Bytes

### ModR/M Decoding

```
┌────────────────────────────────────────────────────────────────────────┐
│                        ModR/M Byte                                     │
├────────────────────────────────────────────────────────────────────────┤
│   7    6  │  5    4    3  │  2    1    0  │                           │
│  ─────────│───────────────│───────────────│                           │
│    Mod    │     Reg       │     R/M       │                           │
│  (2 bits) │   (3 bits)    │   (3 bits)    │                           │
├───────────┴───────────────┴───────────────┤                           │
│                                            │                           │
│  Mod = 00: [r/m] or [disp32] or SIB       │                           │
│  Mod = 01: [r/m + disp8]                  │                           │
│  Mod = 10: [r/m + disp32]                 │                           │
│  Mod = 11: r/m is register (no memory)    │                           │
│                                            │                           │
│  Reg: Register operand or opcode extension│                           │
│  R/M: Register or memory operand          │                           │
│                                            │                           │
│  Special: R/M=100 with Mod≠11 → SIB follows│                          │
│  Special: R/M=101 with Mod=00 → RIP+disp32│                           │
│                                            │                           │
└────────────────────────────────────────────┘
```

### Register Encoding Table

```
Reg/R/M │ Without REX.R/B │ With REX.R/B
────────┼─────────────────┼──────────────
  000   │  RAX/EAX/AX/AL  │  R8/R8D/R8W/R8B
  001   │  RCX/ECX/CX/CL  │  R9
  010   │  RDX/EDX/DX/DL  │  R10
  011   │  RBX/EBX/BX/BL  │  R11
  100   │  RSP/ESP/SP/AH* │  R12
  101   │  RBP/EBP/BP/CH* │  R13
  110   │  RSI/ESI/SI/DH* │  R14
  111   │  RDI/EDI/DI/BH* │  R15

* AH/CH/DH/BH only accessible without REX prefix
```

### SIB Byte (Scale-Index-Base)

```
┌────────────────────────────────────────────────────────────────────────┐
│                          SIB Byte                                      │
├────────────────────────────────────────────────────────────────────────┤
│   7    6  │  5    4    3  │  2    1    0  │                           │
│  ─────────│───────────────│───────────────│                           │
│   Scale   │    Index      │    Base       │                           │
│  (2 bits) │   (3 bits)    │   (3 bits)    │                           │
├───────────┴───────────────┴───────────────┤                           │
│                                            │                           │
│  Scale: 00=×1, 01=×2, 10=×4, 11=×8        │                           │
│  Index: Register to scale (RSP=none)      │                           │
│  Base: Base register (RBP special cases)  │                           │
│                                            │                           │
│  Effective Address = Base + (Index×Scale) + Displacement              │
│                                            │                           │
└────────────────────────────────────────────┘
```

---

## 3. Complete Decoding Example

### Decode: `48 8B 84 C8 00 01 00 00`

```
Step 1: Identify REX prefix
  48 = 0100 1000
       ││││ └┴┴┴─ B=0, X=0, R=0
       │└┴┴───── 0100 (REX identifier)
       └──────── W=1 (64-bit operand size)

Step 2: Opcode
  8B = MOV r64, r/m64

Step 3: ModR/M
  84 = 1000 0100
       ││ │││ └┴┴─ R/M = 100 (SIB follows)
       ││ └┴┴───── Reg = 000 (RAX, with REX.R=0)
       └┴──────── Mod = 10 (disp32)

Step 4: SIB
  C8 = 1100 1000
       ││ │││ └┴┴─ Base = 000 (RAX, with REX.B=0)
       ││ └┴┴───── Index = 001 (RCX, with REX.X=0)
       └┴──────── Scale = 11 (×8)

Step 5: Displacement
  00 01 00 00 = 0x00000100 (little-endian) = 256

Result: mov rax, [rax + rcx*8 + 256]
```

---

## 4. Instruction Categories and Performance

### Data Movement Instructions

| Instruction | Latency | Throughput | Notes |
|-------------|---------|------------|-------|
| MOV r, r | 0-1 | 4/cycle | Often eliminated (register renaming) |
| MOV r, m | 4-5 | 2/cycle | L1 cache hit |
| MOV m, r | 4-5 | 1/cycle | Store buffer |
| MOVZX r, m | 4-5 | 2/cycle | Zero-extend |
| MOVSX r, m | 4-5 | 2/cycle | Sign-extend |
| LEA r, m | 1 | 2/cycle | Address calculation only |
| XCHG r, r | 1-2 | 1/cycle | Implicit LOCK if memory |
| CMOVcc r, m | 5-6 | 1/cycle | Conditional move |

### Arithmetic Instructions

| Instruction | Latency | Throughput | Notes |
|-------------|---------|------------|-------|
| ADD/SUB r, r | 1 | 4/cycle | Fast path |
| ADD/SUB r, m | 5 | 2/cycle | Memory operand |
| INC/DEC r | 1 | 4/cycle | Partial flags update |
| IMUL r, r | 3 | 1/cycle | Modern CPUs |
| IMUL r, r, i | 3 | 1/cycle | Three-operand form |
| DIV r64 | 35-90 | - | Very slow |
| IDIV r64 | 35-90 | - | Very slow |

> **AMD Manual Reference**: Volume 4 "128-Bit and 256-Bit Media Instructions"

### Control Flow Instructions

| Instruction | Latency | Notes |
|-------------|---------|-------|
| JMP rel | 0-1 | Fused with prediction |
| Jcc rel | 0-1 | Branch misprediction: ~15 cycles |
| CALL rel | 2-3 | Push + jump |
| RET | 1-2 | Predicted via Return Stack Buffer |
| LOOP | 5-7 | Avoid; use DEC+JNZ instead |

### String Instructions

```asm
; String instructions use RSI (source), RDI (dest), RCX (count)
; Direction flag (DF): 0=forward, 1=backward

rep movsb        ; Copy RCX bytes from [RSI] to [RDI]
rep stosb        ; Fill RCX bytes at [RDI] with AL
rep cmpsb        ; Compare RCX bytes [RSI] vs [RDI]
repne scasb      ; Scan for AL in [RDI], max RCX bytes
```

**Performance note**: Modern CPUs have "fast string" microcode for `rep movsb/stosb` that can match or exceed SSE/AVX copies for large blocks.

---

## 5. Instruction Selection for Performance

### Idioms and Their Encodings

```asm
; Zeroing a register
xor eax, eax      ; 2 bytes, breaks dependency, sets flags
xor rax, rax      ; 3 bytes (needs REX), same effect
mov eax, 0        ; 5 bytes, avoid
mov rax, 0        ; 7 bytes, avoid

; Testing for zero
test rax, rax     ; 3 bytes, preferred
cmp rax, 0        ; 4 bytes, avoid
and rax, rax      ; 3 bytes, modifies flags same as test

; Copying register
mov rax, rbx      ; Normal copy
lea rax, [rbx]    ; Same result, sometimes useful

; Multiply by constants
imul rax, rbx, 10 ; General case
lea rax, [rbx + rbx*4] ; ×5
shl rax, 1        ; then ×2 = ×10 (two instructions)
lea rax, [rbx*8 + rbx] ; ×9

; Sign extension
movsx rax, ecx    ; Explicit sign-extend
cdqe              ; Sign-extend EAX to RAX (1 byte)
movsxd rax, ecx   ; Same as movsx for dword→qword
```

### Memory Access Patterns

```asm
; Prefer:
mov rax, [rbx]           ; Simple addressing
mov rax, [rbx + 8]       ; Base + small displacement
mov rax, [rbx + rcx*8]   ; Scaled index for arrays

; Avoid when possible:
mov rax, [rbx + rcx*8 + 0x12345678]  ; Complex + large disp
```

### Branch Optimization

```asm
; Predictable branches are fast
; Unpredictable branches (~50% taken) hurt performance

; Use conditional moves for small, unpredictable selections:
cmp rax, rbx
cmovl rax, rbx       ; rax = min(rax, rbx), no branch

; Use branches for larger code blocks or when prediction is good
```

---

## 6. CPUID: Feature Detection

### Using CPUID

```asm
; CPUID input: EAX (and sometimes ECX)
; CPUID output: EAX, EBX, ECX, EDX

; Basic CPUID
mov eax, 0           ; Function 0: Vendor string
cpuid
; EBX:EDX:ECX = "GenuineIntel" or "AuthenticAMD"

mov eax, 1           ; Function 1: Features
cpuid
; ECX, EDX = feature flags

; Extended CPUID
mov eax, 0x80000000  ; Max extended function
cpuid

mov eax, 0x80000001  ; Extended features (AMD)
cpuid
```

### Key Feature Bits (Function 1)

```
EDX Feature Flags:
  Bit 0:  FPU        Bit 23: MMX
  Bit 4:  TSC        Bit 25: SSE
  Bit 5:  MSR        Bit 26: SSE2
  Bit 9:  APIC       Bit 28: HTT

ECX Feature Flags:
  Bit 0:  SSE3       Bit 19: SSE4.1
  Bit 9:  SSSE3      Bit 20: SSE4.2
  Bit 13: CX16       Bit 28: AVX
  Bit 25: AES-NI

Extended ECX (0x80000001):
  Bit 5:  LZCNT      Bit 11: XOP (AMD)
  Bit 6:  SSE4A      Bit 29: LONG MODE
```

> **AMD Manual Reference**: Volume 3, Appendix D "CPUID Specification"

---

## 7. Atomic Operations and Memory Ordering

### LOCK Prefix

```asm
; LOCK makes read-modify-write atomic
lock add [counter], 1     ; Atomic increment
lock cmpxchg [ptr], rbx   ; Compare-and-swap
lock xadd [ptr], rax      ; Fetch-and-add

; Implicit LOCK:
xchg [mem], rax           ; Always atomic (LOCK implied)
```

### Memory Barriers

```asm
; Full fence (all loads and stores complete)
mfence

; Load fence (all loads complete)
lfence

; Store fence (all stores complete)
sfence

; LOCK instructions also act as full fences
```

### x86-64 Memory Model

x86-64 has a relatively strong memory model (TSO - Total Store Order):

- Loads are not reordered with other loads
- Stores are not reordered with other stores
- Stores are not reordered with earlier loads
- **BUT**: Loads may be reordered with earlier stores to different locations
- LOCK'd instructions have total order

---

# Key Readings

## Required

### AMD64 Architecture Programmer's Manual Volume 3

| Section | Topic | Time |
|---------|-------|------|
| Chapter 1 (all) | Instruction Encoding | 3 hours |
| Chapter 2 | Instruction Prefixes | 1.5 hours |
| Selected instructions | MOV, ADD, IMUL, DIV, Jcc, CALL, RET | 2 hours |

### AMD64 Architecture Programmer's Manual Volume 1

| Section | Topic | Time |
|---------|-------|------|
| Chapter 3.5-3.6 | Data Types, Operand Addressing | 1.5 hours |

## Supplementary

- **Agner Fog's Instruction Tables**: Latency and throughput for all CPUs
- **uops.info**: Detailed microarchitectural data
- **Intel Optimization Reference Manual**: Chapters 2-3

---

# Recommended Videos

### 1. Matt Godbolt - "What Has My Compiler Done for Me Lately?"
**Platform**: YouTube (CppCon)  
**Length**: ~60 minutes  
**Focus**: Compiler output analysis, instruction selection

### 2. OpenSecurityTraining2 - Arch1001 Instruction Videos
| Video | Topic |
|-------|-------|
| "Your First Assembly Instruction: NOP" | Encoding basics |
| "Boolean Logic" | Logical operations |
| "Bit Manipulation" | Shifts, rotates |

### 3. "Performance Analysis and Tuning on Modern CPUs" - Denis Bakhvalov
**Format**: Blog series with video content  
**Focus**: Modern CPU performance, instruction analysis

---

# Exercises & Mini-Projects

## Exercise 1: Manual Instruction Encoding

Encode these instructions by hand (show all bytes):

1. `mov rax, rbx`
2. `mov eax, [rbx + rcx*4 + 0x10]`
3. `add r12, [r13 + r14*8]`
4. `push r15`
5. `imul rax, rbx, 0x1234`

Verify with: `echo "instruction" | as -o /dev/stdout | objdump -d`

## Exercise 2: Instruction Decoding

Decode these byte sequences:

1. `48 89 C3`
2. `48 8D 04 CD 00 00 00 00`
3. `4C 03 24 C5 00 01 00 00`
4. `F0 48 0F C1 07`
5. `48 0F AF C1`

## Exercise 3: Compiler Output Analysis

Write C functions and examine compiler output on Godbolt (godbolt.org):

```c
int multiply_by_10(int x);      // See LEA tricks
int abs_value(int x);           // See branchless abs
int min(int a, int b);          // See CMOV
int popcount(unsigned x);       // See POPCNT or fallback
```

## Exercise 4: CPUID Feature Detector

Write an assembly program that:
1. Gets vendor string
2. Checks for specific features (SSE4.2, AVX, AVX2)
3. Prints results

## Exercise 5: Atomic Counter (Mini-Project)

Implement a thread-safe counter in assembly using LOCK prefix:

```asm
; atomic_increment(uint64_t *counter)
; atomic_decrement(uint64_t *counter)
; atomic_compare_swap(uint64_t *ptr, uint64_t expected, uint64_t desired)
```

Test with a C program using pthreads.

---

# Quiz

**Q1**: What is the maximum length of an x86-64 instruction? What happens if you try to create a longer one?

**Q2**: Decode: `4D 8B 54 85 F8`. What instruction is this?

**Q3**: Why does `xor eax, eax` also zero the upper 32 bits of RAX?

**Q4**: What's the difference between `test rax, rax` and `cmp rax, 0`? Which is preferred and why?

**Q5**: When is a SIB byte required?

**Q6**: What does the LOCK prefix do? Which instructions can use it?

**Q7**: Why is `div` so much slower than `imul`?

**Q8**: What CPUID function do you use to check for AVX2 support?

---

# Priority Guide

## CRITICAL
- ModR/M byte decoding
- REX prefix purpose and encoding
- Register encoding table
- Basic instruction latencies
- CPUID feature detection

## IMPORTANT
- SIB byte decoding
- VEX prefix awareness
- Memory ordering basics
- LOCK prefix usage

## OPTIONAL (This Week)
- EVEX encoding details
- Complete instruction timing tables
- All CPUID leaves
- String instruction microcode

---

# Summary

This week you've learned how x86-64 instructions are encoded at the byte level. Key insights:

1. **Variable-length encoding** enables code density but complicates decoding
2. **REX prefix** unlocks 64-bit features and extended registers
3. **ModR/M + SIB** encode complex addressing modes
4. **Instruction selection** significantly impacts performance
5. **CPUID** enables runtime feature detection

**Next Week**: Interrupts and Exceptions—how the CPU handles asynchronous events and errors.
