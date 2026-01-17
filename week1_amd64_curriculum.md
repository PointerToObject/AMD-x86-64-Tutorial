# AMD64 Systems Programming — Week 1
## Introduction & CPU Fundamentals

**Course**: AMD x86-64 Systems Programming  
**Primary Text**: AMD64 Architecture Programmer's Manual (Volumes 1-5)  
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

## 1. Historical Context: Why AMD64 Matters

### The x86 Lineage

The AMD64 architecture (also called x86-64, x64, or Intel 64) is the 64-bit extension of the venerable x86 instruction set architecture. Understanding its lineage is essential:

```
1978: Intel 8086      (16-bit)    ← Original x86
1985: Intel 80386     (32-bit)    ← Protected mode, paging
2003: AMD Opteron     (64-bit)    ← AMD64 architecture born
2004: Intel adopts    (64-bit)    ← Intel EM64T (now Intel 64)
```

**Critical insight**: AMD—not Intel—invented the 64-bit extension to x86. Intel's own 64-bit attempt (IA-64 Itanium) was incompatible with x86 and largely failed in the market. AMD's backwards-compatible approach won.

> **AMD Manual Reference**: Volume 1, Chapter 1 "Overview of the AMD64 Architecture"

### Why This Matters for Systems Programmers

Understanding AMD64 at the manual level gives you:
- **Debugger mastery**: Know exactly what `gdb` or `lldb` is showing you
- **Compiler output literacy**: Read assembly that compilers generate
- **Security research capability**: Exploit development, malware analysis
- **OS/kernel development**: Write code that talks directly to hardware
- **Performance engineering**: Understand why code is fast or slow

---

## 2. Operating Modes: The Three Worlds

AMD64 processors operate in three major modes. This is foundational—every other concept depends on understanding which mode you're in.

### Mode Hierarchy

```
┌─────────────────────────────────────────────────────────────────┐
│                        LONG MODE                                 │
│         (AMD64's native 64-bit mode)                            │
│  ┌───────────────────┬───────────────────────────────────────┐  │
│  │   64-bit Mode     │      Compatibility Mode               │  │
│  │   (native 64-bit  │      (runs 32-bit and 16-bit         │  │
│  │    code)          │       legacy code unchanged)          │  │
│  └───────────────────┴───────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│                      LEGACY MODE                                 │
│         (pre-AMD64 compatibility)                               │
│  ┌───────────────┬────────────────┬──────────────────────────┐  │
│  │ Protected     │  Virtual-8086   │      Real Mode          │  │
│  │ Mode (32-bit) │  Mode          │      (16-bit, no        │  │
│  │               │  (16-bit in    │       protection)        │  │
│  │               │   protected)   │                          │  │
│  └───────────────┴────────────────┴──────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
```

### Mode Details

| Mode | Default Operand Size | Default Address Size | Max Virtual Address |
|------|---------------------|---------------------|---------------------|
| 64-bit Mode | 32 bits | 64 bits | 48 bits (canonical) |
| Compatibility Mode | 16 or 32 bits | 16 or 32 bits | 32 bits |
| Protected Mode | 16 or 32 bits | 16 or 32 bits | 32 bits |
| Real Mode | 16 bits | 20 bits | 20 bits (1 MB) |

> **AMD Manual Reference**: Volume 2, Chapter 1, Section 1.2 "Modes of Operation"

### Entering Long Mode

The transition from legacy to long mode requires specific steps (you'll implement this in a later week's project):

1. Enable PAE (Physical Address Extension) in CR4
2. Load CR3 with a valid PML4 table address
3. Enable long mode by setting EFER.LME
4. Enable paging by setting CR0.PG
5. Jump to 64-bit code segment

**Analogy**: Think of operating modes like different "personalities" the CPU can assume. A 64-bit OS runs in 64-bit mode, but can create a "compatibility bubble" where 32-bit programs think they're on an old 32-bit machine.

---

## 3. Register Architecture: The CPU's Working Memory

Registers are the fastest storage in the system—they're literally part of the CPU core. Mastering registers is non-negotiable for systems programming.

### General-Purpose Registers (GPRs)

AMD64 extends x86's 8 GPRs to 16, and widens them all to 64 bits:

```
┌────────────────────────────────────────────────────────────────────────┐
│  63                              31              15       7      0     │
├────────────────────────────────────────────────────────────────────────┤
│  RAX  ├──────────────────────────┤ EAX  ├────────┤ AX ├──┤AH├──┤AL├   │
│  RBX  ├──────────────────────────┤ EBX  ├────────┤ BX ├──┤BH├──┤BL├   │
│  RCX  ├──────────────────────────┤ ECX  ├────────┤ CX ├──┤CH├──┤CL├   │
│  RDX  ├──────────────────────────┤ EDX  ├────────┤ DX ├──┤DH├──┤DL├   │
│  RSI  ├──────────────────────────┤ ESI  ├────────┤ SI ├───────┤SIL├   │
│  RDI  ├──────────────────────────┤ EDI  ├────────┤ DI ├───────┤DIL├   │
│  RBP  ├──────────────────────────┤ EBP  ├────────┤ BP ├───────┤BPL├   │
│  RSP  ├──────────────────────────┤ ESP  ├────────┤ SP ├───────┤SPL├   │
│  R8   ├──────────────────────────┤ R8D  ├────────┤R8W ├───────┤R8B├   │
│  R9   ├──────────────────────────┤ R9D  ├────────┤R9W ├───────┤R9B├   │
│  R10  ├──────────────────────────┤ R10D ├────────┤R10W├───────┤R10B├  │
│  R11  ├──────────────────────────┤ R11D ├────────┤R11W├───────┤R11B├  │
│  R12  ├──────────────────────────┤ R12D ├────────┤R12W├───────┤R12B├  │
│  R13  ├──────────────────────────┤ R13D ├────────┤R13W├───────┤R13B├  │
│  R14  ├──────────────────────────┤ R14D ├────────┤R14W├───────┤R14B├  │
│  R15  ├──────────────────────────┤ R15D ├────────┤R15W├───────┤R15B├  │
└────────────────────────────────────────────────────────────────────────┘
```

> **AMD Manual Reference**: Volume 1, Chapter 3, Section 3.1 "Registers"

### Critical Zero-Extension Rule

**This is a common source of bugs**: In 64-bit mode, writing to a 32-bit register (like EAX) automatically zeros the upper 32 bits of the corresponding 64-bit register (RAX).

```asm
; Example: Zero-extension behavior
mov rax, 0xFFFFFFFFFFFFFFFF  ; RAX = 0xFFFFFFFFFFFFFFFF
mov eax, 0x00000001          ; RAX = 0x0000000000000001 (upper bits zeroed!)

; But 8-bit and 16-bit writes do NOT zero-extend:
mov rax, 0xFFFFFFFFFFFFFFFF  ; RAX = 0xFFFFFFFFFFFFFFFF
mov ax, 0x0001               ; RAX = 0xFFFFFFFFFFFF0001 (only lower 16 bits changed)
mov al, 0x02                 ; RAX = 0xFFFFFFFFFFFF0002 (only lower 8 bits changed)
```

### Register Conventions (System V AMD64 ABI)

On Linux/macOS/BSD, the System V ABI dictates how registers are used in function calls:

| Register | Purpose | Callee-Saved? |
|----------|---------|---------------|
| RAX | Return value, syscall number | No |
| RBX | General purpose | **Yes** |
| RCX | 4th argument, destroyed by syscall | No |
| RDX | 3rd argument, 2nd return value | No |
| RSI | 2nd argument | No |
| RDI | 1st argument | No |
| RBP | Frame pointer (optional) | **Yes** |
| RSP | Stack pointer | **Yes** |
| R8  | 5th argument | No |
| R9  | 6th argument | No |
| R10 | Temporary, syscall argument | No |
| R11 | Temporary, destroyed by syscall | No |
| R12-R15 | General purpose | **Yes** |

**Callee-saved** means: if a function uses this register, it must restore its original value before returning.

### Special-Purpose Registers

#### Instruction Pointer (RIP)
```
┌────────────────────────────────────────────────────────────────────────┐
│  63                                                               0    │
│  RIP  ├─────────────────────────────────────────────────────────────   │
└────────────────────────────────────────────────────────────────────────┘
```

RIP holds the address of the next instruction to execute. In 64-bit mode, RIP-relative addressing is the default for most data references—this is crucial for position-independent code (PIC).

#### FLAGS Register (RFLAGS)

```
┌────────────────────────────────────────────────────────────────────────┐
│  Bit   │ Name │ Description                                           │
├────────┼──────┼───────────────────────────────────────────────────────┤
│    0   │  CF  │ Carry Flag - unsigned overflow                        │
│    2   │  PF  │ Parity Flag - low byte has even number of 1s          │
│    4   │  AF  │ Auxiliary Carry - BCD operations                      │
│    6   │  ZF  │ Zero Flag - result was zero                           │
│    7   │  SF  │ Sign Flag - result was negative (MSB = 1)             │
│    8   │  TF  │ Trap Flag - single-step debugging                     │
│    9   │  IF  │ Interrupt Enable Flag                                 │
│   10   │  DF  │ Direction Flag - string operation direction           │
│   11   │  OF  │ Overflow Flag - signed overflow                       │
│ 12-13  │ IOPL │ I/O Privilege Level                                   │
│   14   │  NT  │ Nested Task                                           │
│   16   │  RF  │ Resume Flag                                           │
│   17   │  VM  │ Virtual-8086 Mode                                     │
│   18   │  AC  │ Alignment Check                                       │
│   19   │ VIF  │ Virtual Interrupt Flag                                │
│   20   │ VIP  │ Virtual Interrupt Pending                             │
│   21   │  ID  │ CPUID available                                       │
└────────────────────────────────────────────────────────────────────────┘
```

> **AMD Manual Reference**: Volume 1, Chapter 3, Section 3.1.4 "Flags Register"

**Key flags for arithmetic**:
- **ZF (Zero)**: Set if result is zero. Used by JZ/JNZ.
- **SF (Sign)**: Set if result is negative. Used by JS/JNS.
- **CF (Carry)**: Set on unsigned overflow. Used by JC/JNC, JB/JAE.
- **OF (Overflow)**: Set on signed overflow. Used by JO/JNO.

---

## 4. Memory Addressing: How the CPU Finds Data

### The Addressing Mode Formula

In AMD64, memory operands are calculated using this general formula:

```
Effective Address = Base + (Index × Scale) + Displacement
```

Where:
- **Base**: Any GPR (64-bit in long mode)
- **Index**: Any GPR except RSP
- **Scale**: 1, 2, 4, or 8
- **Displacement**: 8-bit, 16-bit, or 32-bit signed constant

### Addressing Mode Examples

```asm
; Direct (displacement only)
mov rax, [0x401000]              ; Load from absolute address

; Register indirect (base only)
mov rax, [rbx]                   ; Load from address in RBX

; Base + displacement
mov rax, [rbp - 8]               ; Load local variable (stack)
mov rax, [rip + 0x1234]          ; RIP-relative (common in 64-bit)

; Base + index
mov rax, [rbx + rcx]             ; Array access: base + index

; Base + (index × scale)
mov rax, [rbx + rcx*8]           ; Array of 8-byte elements

; Full SIB (Scale-Index-Base) + displacement
mov rax, [rbx + rcx*4 + 16]      ; struct array: base + index*size + offset
```

### Memory Reference Diagram

```
                Array of structs example:
                struct item { int a; int b; int c; int d; };  // 16 bytes each
                item array[100];
                
                Access array[i].c:
                
  Base (RBX)                 Index (RCX) × Scale (16)     Displacement (+8)
      │                              │                          │
      ▼                              ▼                          ▼
┌─────────┐   ┌─────────┐   ┌─────────┐   ┌─────────┐   ┌─────────┐
│ array   │ + │    i    │ × │   16    │ + │    8    │ = │ &item.c │
│ address │   │ (index) │   │ (size)  │   │ (offset)│   │ address │
└─────────┘   └─────────┘   └─────────┘   └─────────┘   └─────────┘
```

> **AMD Manual Reference**: Volume 1, Chapter 2, Section 2.2 "Memory Addressing"

### Canonical Addresses

In 64-bit mode, only 48 bits of virtual address are actually used (current implementations). The upper 16 bits must be copies of bit 47 (sign extension):

```
Valid (canonical) addresses:
  0x0000000000000000 - 0x00007FFFFFFFFFFF  (user space, bit 47 = 0)
  0xFFFF800000000000 - 0xFFFFFFFFFFFFFFFF  (kernel space, bit 47 = 1)

Invalid (non-canonical) - will cause #GP exception:
  0x0001000000000000  (bits 48-63 not all same as bit 47)
```

**Analogy**: Think of canonical addresses like properly signed checks. The upper bits are like a signature that must match—if they don't, the CPU refuses to process the address.

---

## 5. Instruction Encoding: The Binary Reality

Understanding encoding helps you read hex dumps, write shellcode, and understand why some instructions are smaller/faster.

### REX Prefix: The 64-bit Enabler

The REX prefix is a single byte (0x40-0x4F) that enables 64-bit features:

```
┌───────────────────────────────────────────────────────────────────┐
│  7   6   5   4 │  3  │  2  │  1  │  0  │                         │
├────────────────┼─────┼─────┼─────┼─────┤                         │
│  0   1   0   0 │  W  │  R  │  X  │  B  │                         │
│    (0x4)       │     │     │     │     │                         │
└────────────────┴─────┴─────┴─────┴─────┘                         
                   │     │     │     │
                   │     │     │     └── Extends ModRM.r/m or SIB base
                   │     │     └──────── Extends SIB index
                   │     └────────────── Extends ModRM.reg
                   └──────────────────── 1 = 64-bit operand size
```

**REX.W**: When set, operand size is 64 bits (otherwise default is 32 bits in 64-bit mode)
**REX.R/X/B**: Extend register fields to access R8-R15

### Instruction Format Overview

```
┌─────────┬─────────┬─────────┬─────────┬─────────┬─────────┬─────────┐
│ Legacy  │   REX   │ Opcode  │  ModR/M │   SIB   │  Disp   │  Immed  │
│Prefixes │ Prefix  │ (1-3B)  │  (opt)  │  (opt)  │  (opt)  │  (opt)  │
│ (0-4B)  │ (0-1B)  │         │  (1B)   │  (1B)   │ (1-4B)  │ (1-8B)  │
└─────────┴─────────┴─────────┴─────────┴─────────┴─────────┴─────────┘
```

### Encoding Example: mov rax, [rbx + rcx*8 + 0x10]

Let's decode this instruction byte by byte:

```
48 8B 44 CB 10

48 = REX.W prefix (0100 1000)
     ││││ └┴┴┴─ B=0, X=0, R=0
     │└┴┴───── 0100 (REX identifier)
     └──────── W=1 (64-bit operand)

8B = MOV r64, r/m64 opcode

44 = ModR/M byte (0100 0100)
     ││ │││ └┴┴─ R/M = 100 (SIB follows)
     ││ └┴┴───── Reg = 000 (RAX)
     └┴──────── Mod = 01 (8-bit displacement)

CB = SIB byte (1100 1011)
     ││ │││ └┴┴─ Base = 011 (RBX)
     ││ └┴┴───── Index = 001 (RCX)
     └┴──────── Scale = 11 (×8)

10 = Displacement (0x10 = 16)
```

> **AMD Manual Reference**: Volume 3, Chapter 1 "Instruction Encoding"

---

## 6. The Stack: A LIFO Data Structure in Hardware

### Stack Mechanics

The stack grows **downward** in memory (toward lower addresses). RSP always points to the **top of the stack** (the last pushed value).

```
High addresses
     │
     ▼
┌────────────────────┐
│   Previous Frame   │  
├────────────────────┤ ◄── Old RSP before CALL
│   Return Address   │  ← Pushed by CALL instruction
├────────────────────┤
│   Saved RBP        │  ← Pushed by function prologue
├────────────────────┤ ◄── RBP (frame pointer)
│   Local var 1      │  [rbp - 8]
├────────────────────┤
│   Local var 2      │  [rbp - 16]
├────────────────────┤
│   ...              │
├────────────────────┤ ◄── RSP (stack pointer)
│   (unused)         │
│                    │
     │
     ▼
Low addresses
```

### PUSH and POP Operations

```asm
; PUSH operation (two steps combined):
push rax
; Equivalent to:
;   sub rsp, 8       ; Allocate space
;   mov [rsp], rax   ; Store value

; POP operation:
pop rbx
; Equivalent to:
;   mov rbx, [rsp]   ; Load value
;   add rsp, 8       ; Deallocate space
```

### Red Zone (System V ABI)

In the System V ABI, the 128 bytes below RSP are the "red zone"—a scratch area that leaf functions can use without adjusting RSP:

```
                    ┌────────────────────┐
                    │                    │
        RSP ──────► ├────────────────────┤
                    │                    │
                    │     Red Zone       │  128 bytes that won't be
                    │     (128 bytes)    │  clobbered by signals or
                    │                    │  interrupts
                    ├────────────────────┤
        RSP-128 ──► │                    │
```

> **AMD Manual Reference**: Volume 1, Chapter 3, Section 3.4.1 "Stack Operations"

---

## 7. Basic Instruction Categories

### Data Movement

```asm
; MOV - Copy data
mov rax, rbx           ; Register to register
mov rax, [rbx]         ; Memory to register
mov [rbx], rax         ; Register to memory
mov rax, 0x1234        ; Immediate to register

; LEA - Load Effective Address (computes address, doesn't dereference)
lea rax, [rbx + rcx*4] ; RAX = RBX + RCX*4 (no memory access!)

; MOVZX/MOVSX - Move with zero/sign extension
movzx eax, byte [rbx]  ; Zero-extend byte to 32 bits
movsx rax, word [rbx]  ; Sign-extend word to 64 bits

; XCHG - Exchange
xchg rax, rbx          ; Swap RAX and RBX (atomic if memory operand)
```

### Arithmetic

```asm
; ADD/SUB
add rax, rbx           ; RAX = RAX + RBX
sub rax, 10            ; RAX = RAX - 10

; INC/DEC (don't affect CF - important!)
inc rax                ; RAX = RAX + 1
dec rbx                ; RBX = RBX - 1

; MUL/IMUL - Unsigned/Signed multiplication
mul rbx                ; RDX:RAX = RAX * RBX (128-bit result)
imul rax, rbx          ; RAX = RAX * RBX (truncated)
imul rax, rbx, 10      ; RAX = RBX * 10

; DIV/IDIV - Unsigned/Signed division
div rbx                ; RAX = RDX:RAX / RBX, RDX = remainder
idiv rbx               ; Signed version

; NEG - Two's complement negation
neg rax                ; RAX = -RAX
```

### Logical/Bitwise

```asm
; AND/OR/XOR/NOT
and rax, 0xFF          ; Mask lower byte
or  rax, 0x80          ; Set bit 7
xor rax, rax           ; Fast zero (clears RAX, sets ZF)
not rax                ; Bitwise complement

; Shifts
shl rax, 4             ; Shift left (multiply by 16)
shr rax, 2             ; Shift right logical (unsigned divide by 4)
sar rax, 1             ; Shift right arithmetic (signed divide by 2)
rol rax, 8             ; Rotate left
ror rax, 8             ; Rotate right
```

### Control Flow

```asm
; Unconditional jump
jmp label              ; Jump to label

; Conditional jumps (based on FLAGS)
cmp rax, rbx           ; Compare (sets flags like SUB but discards result)
je  equal              ; Jump if ZF=1 (equal)
jne not_equal          ; Jump if ZF=0 (not equal)
jl  less               ; Jump if SF≠OF (signed less)
jg  greater            ; Jump if ZF=0 and SF=OF (signed greater)
jb  below              ; Jump if CF=1 (unsigned below)
ja  above              ; Jump if CF=0 and ZF=0 (unsigned above)

; Function calls
call function          ; Push return address, jump to function
ret                    ; Pop return address, jump to it
```

> **AMD Manual Reference**: Volume 3, Chapters 3-5 for complete instruction details

---

## 8. Your First Assembly Program

### Hello World (Linux, NASM syntax)

```asm
; hello.asm - Hello World for Linux x86-64
; Assemble: nasm -f elf64 hello.asm
; Link:     ld hello.o -o hello

section .data
    message: db "Hello, AMD64!", 10    ; 10 = newline
    msglen:  equ $ - message           ; Calculate length

section .text
    global _start

_start:
    ; sys_write(fd, buf, count)
    mov rax, 1          ; syscall number for write
    mov rdi, 1          ; fd = 1 (stdout)
    mov rsi, message    ; buf = address of message
    mov rdx, msglen     ; count = message length
    syscall             ; invoke kernel

    ; sys_exit(status)
    mov rax, 60         ; syscall number for exit
    mov rdi, 0          ; status = 0
    syscall             ; invoke kernel
```

### Understanding the System Call Interface

Linux x86-64 system call convention:
- **RAX**: System call number
- **RDI, RSI, RDX, R10, R8, R9**: Arguments 1-6
- **RAX**: Return value (negative = error)
- **RCX, R11**: Destroyed by syscall instruction

```
┌────────────────────────────────────────────────────────────────┐
│                    syscall Instruction Flow                    │
├────────────────────────────────────────────────────────────────┤
│  1. RCX ← RIP (save return address)                           │
│  2. R11 ← RFLAGS (save flags)                                 │
│  3. RIP ← IA32_LSTAR MSR (kernel entry point)                 │
│  4. RFLAGS masked with IA32_FMASK                             │
│  5. CPL ← 0 (switch to ring 0)                                │
│  6. Kernel executes system call handler                       │
│  7. sysret returns to user mode                               │
└────────────────────────────────────────────────────────────────┘
```

---

# Key Readings

## Required (Primary Focus)

### AMD64 Architecture Programmer's Manual Volume 1: Application Programming
**Download**: Search "AMD64 Architecture Programmer's Manual" on AMD's developer site or archive.org

| Section | Pages | Topic | Time |
|---------|-------|-------|------|
| Chapter 1 | ~10 pages | Overview of AMD64 Architecture | 1 hour |
| Chapter 2, Sections 2.1-2.2 | ~10 pages | Memory Model, Addressing | 1.5 hours |
| Chapter 3, Sections 3.1-3.2 | ~15 pages | General-Purpose Registers, Flags | 2 hours |

### AMD64 Architecture Programmer's Manual Volume 2: System Programming

| Section | Pages | Topic | Time |
|---------|-------|-------|------|
| Chapter 1, Sections 1.1-1.3 | ~15 pages | System Programming Overview, Modes | 2 hours |
| Chapter 2, Sections 2.1-2.2 | ~8 pages | x86 and AMD64 Differences | 1 hour |
| Chapter 3, Section 3.1 | ~10 pages | System Registers Overview | 1.5 hours |

### AMD64 Architecture Programmer's Manual Volume 3: General-Purpose Instructions

| Section | Pages | Topic | Time |
|---------|-------|-------|------|
| Chapter 1 | ~20 pages | Instruction Encoding | 2 hours |
| Skim instruction reference | Variable | MOV, ADD, SUB, CMP, JMP, JCC family | 1.5 hours |

## Supplementary

- **System V AMD64 ABI Specification**: Understand calling conventions
  - Focus on: Section 3.2 "Function Calling Sequence"
  - URL: Search "System V Application Binary Interface AMD64"

---

# Recommended Videos

## Essential Viewing

### 1. OpenSecurityTraining2: Architecture 1001 - x86-64 Assembly
**Platform**: YouTube / p.ost2.fyi  
**Instructor**: Xeno Kovah  
**Total Length**: ~15 hours  

**Week 1 Relevant Sections**:
| Video | Topic | Timecode | Priority |
|-------|-------|----------|----------|
| "Why Learn Assembly?" | Motivation and context | Full (~15 min) | High |
| "Class Introduction" | Course overview | Full (~10 min) | High |
| "x86-64 General Purpose Registers" | Register architecture | Full (~25 min) | **Critical** |
| "Register Conventions" | ABI and conventions | Full (~20 min) | **Critical** |
| "The Stack" | Stack mechanics | Full (~20 min) | **Critical** |
| "New Instructions: Push & Pop" | Stack operations | Full (~15 min) | High |
| "New Instructions: Add/Sub/Call/Ret/Mov" | Basic instructions | Full (~25 min) | **Critical** |

**Access**: Free at https://p.ost2.fyi (registration required)

### 2. "Modern x64 Assembly" Series
**Platform**: YouTube  
**Creator**: Creel  

**Week 1 Relevant Videos**:
| Video | Timecode | Topic |
|-------|----------|-------|
| Part 1: Beginning Assembly Programming | Full | Setup and first program |
| Part 2: 16 Bit Registers | 0:00-10:00 | Historical context |
| Part 3: 32 and 64 bit Registers | Full | Register extensions |
| Part 5: MOV and LEA | Full | Data movement |
| Part 9: The rFlags Register | Full | Flags in depth |

### 3. Conference Talks

**"What Has My Compiler Done for Me Lately?"**  
- Speaker: Matt Godbolt  
- Event: CppCon  
- Relevance: Shows how C compiles to x86-64, excellent for understanding instruction mapping  
- Focus: First 30 minutes for Week 1 context

---

# Exercises & Mini-Projects

## Exercise Set 1: Environment Setup (Day 1)

### E1.1: Install Tools
```bash
# Ubuntu/Debian
sudo apt update
sudo apt install nasm gcc gdb binutils

# Verify
nasm --version
gcc --version
gdb --version
objdump --version
```

### E1.2: Create Your First Program
Create `first.asm`:
```asm
section .text
    global _start
_start:
    mov rax, 60     ; exit syscall
    mov rdi, 42     ; exit code
    syscall
```

Build and run:
```bash
nasm -f elf64 first.asm -o first.o
ld first.o -o first
./first
echo $?  # Should print 42
```

### E1.3: Examine with objdump
```bash
objdump -d first
objdump -d -M intel first  # Intel syntax
```

**Questions**:
1. What is the hex encoding of `mov rax, 60`?
2. What does the `_start` symbol resolve to (what address)?
3. How many bytes is the entire program?

---

## Exercise Set 2: Register Exploration (Day 2-3)

### E2.1: Register Aliasing
Write a program that demonstrates the zero-extension behavior. Create `zero_ext.asm`:

```asm
section .text
    global _start
_start:
    mov rax, 0xFFFFFFFFFFFFFFFF  ; Set all bits
    mov eax, 1                   ; Write to lower 32 - what happens to upper 32?
    ; At this point, RAX should be 0x0000000000000001
    
    mov rax, 0xFFFFFFFFFFFFFFFF  ; Reset
    mov ax, 1                    ; Write to lower 16 - what happens?
    ; RAX should be 0xFFFFFFFFFFFF0001
    
    mov rax, 0xFFFFFFFFFFFFFFFF
    mov al, 1                    ; Write to lower 8
    ; RAX should be 0xFFFFFFFFFFFFFF01
    
    mov rax, 60
    mov rdi, 0
    syscall
```

**Task**: Use GDB to step through and verify each register state:
```bash
gdb ./zero_ext
(gdb) break _start
(gdb) run
(gdb) stepi
(gdb) info registers rax
```

### E2.2: Register Quiz Program
Write a program that uses all 16 GPRs, storing different values in each, then exits with the sum of R12 + R13 as the exit code.

---

## Exercise Set 3: Memory Addressing (Day 4)

### E3.1: Array Access
Create a program that has an array of 8 quadwords (64-bit values) and sums them:

```asm
section .data
    numbers: dq 10, 20, 30, 40, 50, 60, 70, 80  ; 8 quadwords
    
section .text
    global _start
_start:
    xor rax, rax        ; sum = 0
    xor rcx, rcx        ; index = 0
    
.loop:
    add rax, [numbers + rcx*8]  ; sum += numbers[index]
    inc rcx
    cmp rcx, 8
    jl .loop
    
    ; Exit with sum as exit code (will overflow, that's fine)
    mov rdi, rax
    mov rax, 60
    syscall
```

**Tasks**:
1. Calculate the expected sum
2. Verify with `echo $?`
3. Use objdump to see the addressing mode encoding

### E3.2: Addressing Mode Variations
Rewrite the array sum using different valid addressing modes. Document which generates the shortest machine code.

---

## Exercise Set 4: Stack Operations (Day 5)

### E4.1: Stack Frame Visualization
Write a program that:
1. Pushes several values onto the stack
2. Calls a function that creates a stack frame
3. Uses GDB to visualize the entire stack

```asm
section .text
    global _start

_start:
    push 0x1111111111111111
    push 0x2222222222222222
    push 0x3333333333333333
    call my_function
    
    mov rax, 60
    mov rdi, 0
    syscall

my_function:
    push rbp
    mov rbp, rsp
    sub rsp, 32          ; Allocate 32 bytes for locals
    
    mov qword [rbp-8], 0xAAAAAAAAAAAAAAAA
    mov qword [rbp-16], 0xBBBBBBBBBBBBBBBB
    
    ; GDB breakpoint here
    nop
    
    mov rsp, rbp
    pop rbp
    ret
```

**GDB Commands**:
```bash
(gdb) break my_function
(gdb) run
(gdb) x/20xg $rsp    # Examine 20 quadwords from RSP
(gdb) info frame
```

### E4.2: Draw the Stack
Based on E4.1, draw the exact stack layout showing:
- Each value and its address
- RSP and RBP positions
- Return address location
- Local variable locations

---

## Mini-Project: Fibonacci Calculator (Weekend)

Implement a recursive Fibonacci function in pure assembly. Requirements:
1. Use proper stack frame setup/teardown
2. Follow System V ABI calling convention
3. Handle the base cases (fib(0)=0, fib(1)=1)
4. Exit with fib(10) as exit code (should be 55)

Starter template:
```asm
section .text
    global _start

_start:
    mov rdi, 10         ; Calculate fib(10)
    call fib
    
    mov rdi, rax        ; Exit code = result
    mov rax, 60
    syscall

; int64_t fib(int64_t n)
fib:
    ; Your code here
    ; Remember: RDI = first argument, RAX = return value
    ; Must preserve: RBX, RBP, R12-R15
```

---

# Quiz

## Conceptual Questions

**Q1**: What is the difference between Real Mode, Protected Mode, and Long Mode? Which mode(s) support 64-bit registers?

**Q2**: Explain why `mov eax, 5` clears the upper 32 bits of RAX in 64-bit mode, but `mov ax, 5` does not clear the upper bits.

**Q3**: What is a canonical address in AMD64? Give an example of a valid and invalid 64-bit address.

**Q4**: In the addressing mode `[rbx + rcx*8 + 0x20]`, identify the base, index, scale, and displacement.

**Q5**: What is the purpose of the REX prefix? When is it required?

**Q6**: Explain the difference between CF (Carry Flag) and OF (Overflow Flag). Give an example calculation that sets one but not the other.

**Q7**: What happens to RSP when you execute `push rax`? What about `call function`?

**Q8**: According to System V ABI, which registers must a function preserve? Which can it freely clobber?

## Practical Questions

**Q9**: Decode this instruction: `48 89 44 24 F8`
- What instruction is it?
- What are the operands?
- What does it do?

**Q10**: Write assembly (one instruction) to:
a) Zero RAX (using XOR)
b) Copy RBX to RAX
c) Load a quadword from address in RSI into RAX
d) Calculate RDI + RSI*4 and store in RAX (no memory access)

**Q11**: Given this stack state, what value is returned by `ret`?
```
RSP → 0x00007fffffffde00: 0x0000000000401050
      0x00007fffffffde08: 0x0000000000000001
      0x00007fffffffde10: 0x0000000000000002
```

**Q12**: What is wrong with this code?
```asm
my_func:
    push rbx
    mov rbx, rdi
    call other_func
    mov rax, rbx
    ret
```

---

# Priority Guide

## CRITICAL (Must Master)

These concepts are foundational—you cannot proceed without them:

| Topic | Why Critical |
|-------|--------------|
| 16 General-Purpose Registers | Every instruction uses them |
| Zero-extension rule | Common bug source |
| System V ABI calling convention | Required for any real code |
| Stack operations (push/pop/call/ret) | Function calls, debugging |
| Basic addressing modes | Every memory access |
| FLAGS register (ZF, SF, CF, OF) | All conditional logic |

**Time allocation**: 70% of your study time

## IMPORTANT (Should Understand)

| Topic | Why Important |
|-------|--------------|
| Operating modes overview | Context for protection/privilege |
| REX prefix purpose | 64-bit instruction encoding |
| RIP-relative addressing | Modern position-independent code |
| Canonical addresses | Memory layout understanding |

**Time allocation**: 20% of your study time

## OPTIONAL (Awareness Level)

| Topic | When Needed |
|-------|-------------|
| Full instruction encoding details | Shellcode, binary analysis |
| All legacy modes (real, v8086) | OS development, bootsectors |
| Complete FLAGS register | Specific debugging scenarios |
| ModR/M and SIB byte encoding | Manual disassembly |

**Time allocation**: 10% of your study time (or defer to later weeks)

---

# Course Roadmap Preview

| Week | Topic | AMD Manual Focus |
|------|-------|------------------|
| 1 | **Foundations** (current) | Vol 1: Ch 1-3; Vol 2: Ch 1-2 |
| 2 | Memory: Paging, Segments, TLB | Vol 2: Ch 4-5 |
| 3 | Instruction Set Deep Dive | Vol 3-5: Selected instructions |
| 4 | Interrupts and Exceptions | Vol 2: Ch 8 |
| 5 | Protection and Privilege | Vol 2: Ch 4 |
| 6 | System Calls and Syscall/Sysret | Vol 2: Ch 6 |
| 7 | Caching and Memory Types | Vol 2: Ch 7 |
| 8 | SIMD: SSE, AVX | Vol 1: Ch 4-5; Vol 4 |
| 9 | Virtualization (SVM/AMD-V) | Vol 2: Ch 15 |
| 10 | Security Features (SME, SEV) | Vol 2: Ch 16 |
| 11 | Performance Analysis | Vol 2: Ch 13; External resources |
| 12 | Capstone Project | All volumes |

---

# Next Steps

After completing Week 1:

1. **Verify your understanding**: Can you write a simple function in assembly from scratch?
2. **Check your GDB skills**: Can you examine registers, memory, and single-step through code?
3. **Prepare for Week 2**: Download and skim AMD Manual Vol 2, Chapters 4-5 on memory management

**You're ready for Week 2 when you can**:
- List all 16 GPRs and their purposes
- Explain what happens during `call` and `ret`
- Write a program that uses the stack correctly
- Use GDB to debug assembly code
- Decode simple instruction encodings

---

*"The more constraints one imposes, the more one frees one's self of the chains that shackle the spirit."* — Igor Stravinsky

The same applies to understanding hardware constraints—mastering what the CPU actually does frees you from abstract confusion.
