# AMD64 Systems Programming — Week 8
## SIMD Programming: SSE, AVX, and AVX-512

**Course**: AMD x86-64 Systems Programming  
**Primary Text**: AMD64 Architecture Programmer's Manual, Volumes 1, 4, 5  
**Week Duration**: ~15-20 hours of focused study  

---

# Lecture Notes

## 1. SIMD Overview

### Register Evolution

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    SIMD Register Generations                            │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  MMX (1997):     MM0-MM7      64-bit   (aliased on x87 FPU stack)     │
│  SSE (1999):     XMM0-XMM7    128-bit  (new registers)                 │
│  SSE (x86-64):   XMM0-XMM15   128-bit  (8 more registers)              │
│  AVX (2011):     YMM0-YMM15   256-bit  (extended XMM)                  │
│  AVX-512 (2017): ZMM0-ZMM31   512-bit  (extended YMM, +16 registers)  │
│                                                                         │
│  Register aliasing:                                                     │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │ ZMM0 [511:0]                                                    │   │
│  │ ├───────────────────────────────────────────────────────────────┤   │
│  │ │ YMM0 [255:0]                                                  │   │
│  │ │ ├───────────────────────────────────────────────────────────┤ │   │
│  │ │ │ XMM0 [127:0]                                              │ │   │
│  │ │ └───────────────────────────────────────────────────────────┘ │   │
│  │ └───────────────────────────────────────────────────────────────┘   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### Data Types

```
128-bit XMM can hold:
  - 16 × 8-bit integers (bytes)
  - 8 × 16-bit integers (words)
  - 4 × 32-bit integers (dwords) or floats
  - 2 × 64-bit integers (qwords) or doubles
  - 1 × 128-bit integer

256-bit YMM doubles all of the above
512-bit ZMM quadruples all of the above
```

> **AMD Manual Reference**: Volume 1, Chapter 4 "128-Bit Media Programming"

---

## 2. Feature Detection

### CPUID Checks

```asm
; Check SSE support
mov eax, 1
cpuid
test edx, (1 << 25)    ; SSE
test edx, (1 << 26)    ; SSE2
test ecx, (1 << 0)     ; SSE3
test ecx, (1 << 9)     ; SSSE3
test ecx, (1 << 19)    ; SSE4.1
test ecx, (1 << 20)    ; SSE4.2

; Check AVX support
test ecx, (1 << 28)    ; AVX
test ecx, (1 << 27)    ; OSXSAVE (OS enabled XSAVE)

; For AVX, also need to check XGETBV
xor ecx, ecx
xgetbv                  ; Get XCR0 into EDX:EAX
and eax, 0x06           ; Check XMM and YMM state enabled
cmp eax, 0x06
jne avx_not_enabled

; Check AVX2 (extended features)
mov eax, 7
xor ecx, ecx
cpuid
test ebx, (1 << 5)      ; AVX2

; Check AVX-512
test ebx, (1 << 16)     ; AVX-512F (Foundation)
test ebx, (1 << 17)     ; AVX-512DQ
test ebx, (1 << 28)     ; AVX-512CD
test ebx, (1 << 30)     ; AVX-512BW
test ebx, (1 << 31)     ; AVX-512VL
```

---

## 3. SSE Programming

### Basic Operations

```asm
section .data
    align 16
    vec_a: dd 1.0, 2.0, 3.0, 4.0      ; 4 floats
    vec_b: dd 5.0, 6.0, 7.0, 8.0
    result: dd 0.0, 0.0, 0.0, 0.0

section .text
    ; Load vectors
    movaps xmm0, [vec_a]        ; Load aligned packed singles
    movaps xmm1, [vec_b]
    
    ; Arithmetic
    addps xmm0, xmm1            ; xmm0 = xmm0 + xmm1 (parallel)
    mulps xmm0, xmm1            ; xmm0 = xmm0 * xmm1
    
    ; Store result
    movaps [result], xmm0       ; Store aligned
```

### Scalar vs. Packed

```asm
; Packed (parallel): All elements processed
addps xmm0, xmm1       ; 4 floats in parallel
addpd xmm0, xmm1       ; 2 doubles in parallel

; Scalar: Only lowest element processed
addss xmm0, xmm1       ; Single float
addsd xmm0, xmm1       ; Single double
```

### Shuffle and Permute

```asm
; SHUFPS: Shuffle packed singles
; imm8 selects source elements: [d c | b a]
; Bits 1:0 = element from src1 for position 0
; Bits 3:2 = element from src1 for position 1
; Bits 5:4 = element from src2 for position 2
; Bits 7:6 = element from src2 for position 3

shufps xmm0, xmm1, 0b00011011  ; Example shuffle

; PSHUFD: Shuffle packed dwords (integer)
pshufd xmm0, xmm1, 0x1B        ; Reverse order

; UNPCKLPS/UNPCKHPS: Unpack and interleave
unpcklps xmm0, xmm1    ; Interleave low elements
unpckhps xmm0, xmm1    ; Interleave high elements
```

### Horizontal Operations

```asm
; HADDPS: Horizontal add (SSE3)
; xmm0 = [a3, a2, a1, a0]
; xmm1 = [b3, b2, b1, b0]
haddps xmm0, xmm1
; Result: [b3+b2, b1+b0, a3+a2, a1+a0]

; Sum all elements of xmm0:
haddps xmm0, xmm0      ; [a3+a2, a1+a0, a3+a2, a1+a0]
haddps xmm0, xmm0      ; [sum, sum, sum, sum]
```

---

## 4. AVX Programming

### VEX-Encoded Instructions

```asm
; AVX uses 3-operand form (non-destructive)
; SSE: addps xmm0, xmm1    ; xmm0 = xmm0 + xmm1 (overwrites xmm0)
; AVX: vaddps xmm2, xmm0, xmm1  ; xmm2 = xmm0 + xmm1 (preserves xmm0)

vaddps ymm0, ymm1, ymm2    ; 8 floats in parallel
vmulpd ymm0, ymm1, ymm2    ; 4 doubles in parallel

; 128-bit operations with VEX zero upper bits
vaddps xmm0, xmm1, xmm2    ; Also clears ymm0[255:128]
```

### FMA (Fused Multiply-Add)

```asm
; FMA3: d = a*b + c (single rounding, more accurate)
vfmadd132ps ymm0, ymm1, ymm2   ; ymm0 = ymm0 * ymm2 + ymm1
vfmadd213ps ymm0, ymm1, ymm2   ; ymm0 = ymm1 * ymm0 + ymm2
vfmadd231ps ymm0, ymm1, ymm2   ; ymm0 = ymm1 * ymm2 + ymm0

; Also: vfmsub (multiply-subtract), vfnmadd (negate), etc.
```

### Gather

```asm
; Gather: Load non-contiguous elements
; vgatherdps ymm0, [rbx + ymm1*4], ymm2
; ymm1 contains indices, ymm2 is mask
; Loads: mem[rbx + ymm1[0]*4], mem[rbx + ymm1[1]*4], ...

vgatherdps ymm0, [rsi + ymm1*4], ymm2
```

---

## 5. Practical SIMD Patterns

### Vector Dot Product

```asm
; Dot product of 8 floats (AVX)
section .data
    align 32
    vec_a: dd 1.0, 2.0, 3.0, 4.0, 5.0, 6.0, 7.0, 8.0
    vec_b: dd 1.0, 1.0, 1.0, 1.0, 1.0, 1.0, 1.0, 1.0

section .text
dot_product:
    vmovaps ymm0, [vec_a]
    vmovaps ymm1, [vec_b]
    
    vmulps ymm0, ymm0, ymm1      ; Element-wise multiply
    
    ; Horizontal sum
    vhaddps ymm0, ymm0, ymm0     ; [s3+s2, s1+s0, s3+s2, s1+s0, ...]
    vhaddps ymm0, ymm0, ymm0     ; [sum_low, sum_low, sum_hi, sum_hi]
    
    ; Extract and add high and low halves
    vextractf128 xmm1, ymm0, 1  ; Get high 128 bits
    vaddps xmm0, xmm0, xmm1     ; Add high + low
    
    ; xmm0[0] now contains the dot product
    ret
```

### Array Sum (Loop)

```asm
; Sum array of floats
; RDI = array pointer, RSI = count (multiple of 8)
array_sum_avx:
    vxorps ymm0, ymm0, ymm0     ; Accumulator = 0
    
.loop:
    vaddps ymm0, ymm0, [rdi]
    add rdi, 32
    sub rsi, 8
    jnz .loop
    
    ; Reduce ymm0 to scalar
    vextractf128 xmm1, ymm0, 1
    vaddps xmm0, xmm0, xmm1
    vhaddps xmm0, xmm0, xmm0
    vhaddps xmm0, xmm0, xmm0
    
    ret
```

### Conditional Selection (Blend)

```asm
; Select elements based on mask
; if (a[i] > b[i]) result[i] = a[i] else result[i] = b[i]

vmovaps ymm0, [a]
vmovaps ymm1, [b]
vcmpps ymm2, ymm0, ymm1, 14    ; ymm2 = (a > b) ? all 1s : all 0s
vblendvps ymm3, ymm1, ymm0, ymm2  ; Select based on mask
```

---

## 6. SSE/AVX Transition

### State Management

```asm
; Mixing SSE and AVX causes performance penalty
; Solution: Use vzeroupper before calling SSE code

my_avx_function:
    ; Use AVX...
    vmulps ymm0, ymm1, ymm2
    
    vzeroupper              ; Clear upper 128 bits of all YMM
    call sse_function       ; SSE code runs without penalty
    
    ret

; Or use vzeroall to clear all YMM registers
```

### OS State Save/Restore

```asm
; XSAVE/XRSTOR for context switch
; XCR0 controls which state is saved

; Save state
mov eax, 7              ; Save x87, SSE, AVX state
xor edx, edx
xsave [state_buffer]

; Restore state
mov eax, 7
xor edx, edx
xrstor [state_buffer]
```

---

## 7. Alignment Requirements

### Critical Alignment

```
movaps/vmovaps:    16-byte aligned (SSE), 32-byte aligned (AVX)
movups/vmovups:    Unaligned (slower, but works)

Stack alignment in function calls: 16 bytes (System V ABI)
SIMD data should be aligned to vector width for best performance
```

```asm
section .data
    align 32                    ; Align to 32 bytes for AVX
    my_data: dd 1.0, 2.0, 3.0, 4.0, 5.0, 6.0, 7.0, 8.0

section .text
    ; Aligned load (fast)
    vmovaps ymm0, [my_data]
    
    ; Unaligned load (works, potentially slower)
    vmovups ymm0, [unaligned_ptr]
```

---

# Key Readings

## Required

| Source | Section | Topic |
|--------|---------|-------|
| AMD Vol 1 | Chapter 4 | 128-bit Media Programming |
| AMD Vol 4 | All | 128-bit/256-bit Instructions |
| AMD Vol 5 | All | 64-bit Media Instructions |

## Supplementary
- Intel Intrinsics Guide (online reference)
- Agner Fog's optimization manuals

---

# Exercises & Mini-Projects

## Exercise 1: CPUID SIMD Detection

Write a function that detects and prints all supported SIMD extensions.

## Exercise 2: Vector Operations

Implement in SSE/AVX:
- Vector addition
- Dot product
- Cross product (3D)

## Exercise 3: Image Processing

Implement a simple 3×3 convolution filter using SIMD.

## Mini-Project: SIMD memcpy

Implement an optimized memcpy using AVX with proper alignment handling.

---

# Quiz

**Q1**: What is the difference between packed and scalar SIMD instructions?

**Q2**: Why does mixing SSE and AVX code cause performance penalties?

**Q3**: What does vzeroupper do and when should it be used?

**Q4**: How do you detect AVX support at runtime?

**Q5**: What is FMA and why is it more accurate than separate multiply and add?

---

# Priority Guide

## CRITICAL
- Register architecture (XMM, YMM, ZMM)
- Basic packed arithmetic
- Feature detection via CPUID
- Alignment requirements

## IMPORTANT
- Shuffle/permute operations
- FMA instructions
- SSE/AVX transition handling

## OPTIONAL
- AVX-512 mask registers
- Gather/scatter operations
- Full intrinsics mapping

---

**Next Week**: Virtualization—AMD-V (SVM) and hardware virtualization support.
