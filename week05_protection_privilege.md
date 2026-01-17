# AMD64 Systems Programming — Week 5
## Protection and Privilege

**Course**: AMD x86-64 Systems Programming  
**Primary Text**: AMD64 Architecture Programmer's Manual, Volume 2  
**Week Duration**: ~15-20 hours of focused study  

---

# Lecture Notes

## 1. The Ring Model

### Four Privilege Levels

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    x86-64 Protection Rings                              │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│              ┌─────────────────────────────────────┐                   │
│              │           Ring 3 (User)             │                   │
│              │  Applications, User-space code      │                   │
│              │       ┌─────────────────────┐       │                   │
│              │       │      Ring 2         │       │                   │
│              │       │    (Unused)         │       │                   │
│              │       │  ┌─────────────┐    │       │                   │
│              │       │  │   Ring 1    │    │       │                   │
│              │       │  │  (Unused)   │    │       │                   │
│              │       │  │ ┌───────┐   │    │       │                   │
│              │       │  │ │Ring 0 │   │    │       │                   │
│              │       │  │ │Kernel │   │    │       │                   │
│              │       │  │ └───────┘   │    │       │                   │
│              │       │  └─────────────┘    │       │                   │
│              │       └─────────────────────┘       │                   │
│              └─────────────────────────────────────┘                   │
│                                                                         │
│  Modern OS usage: Only Ring 0 (kernel) and Ring 3 (user)               │
│  Hypervisors: Ring -1 (VMX root mode) below Ring 0                     │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### Current Privilege Level (CPL)

CPL is stored in CS.RPL (bits 0-1 of the Code Segment selector):

```
CPL = 0: Kernel mode (most privileged)
CPL = 1: Unused in practice
CPL = 2: Unused in practice  
CPL = 3: User mode (least privileged)

Access rule: CPL ≤ DPL (Descriptor Privilege Level)
```

> **AMD Manual Reference**: Volume 2, Chapter 4 "Segmented Virtual Memory"

---

## 2. Privilege Level Transitions

### User → Kernel (Increasing Privilege)

```
Method              Mechanism                 When Used
────────────────────────────────────────────────────────────────────
SYSCALL            MSR-based fast path        System calls (Linux)
INT n              IDT lookup                 Legacy syscalls, INT 0x80
Exception          IDT lookup                 Faults (#PF, #GP)
Hardware Interrupt IDT lookup                 Device interrupts
```

### Kernel → User (Decreasing Privilege)

```
Method              Mechanism                 When Used
────────────────────────────────────────────────────────────────────
SYSRET             MSR-based fast return      Syscall return
IRETQ              Stack-based return         Exception/interrupt return
```

---

## 3. Segment-Based Protection

### Segment Descriptor Format (64-bit System Segment)

```
┌────────────────────────────────────────────────────────────────────────┐
│                    System Segment Descriptor (16 bytes)                │
├────────────────────────────────────────────────────────────────────────┤
│  Bytes 0-7 (Low):                                                      │
│  │ 63      56│55     52│51     48│47     40│39     16│15      0│      │
│  ├───────────┼─────────┼─────────┼─────────┼─────────┼─────────┤      │
│  │ Base 31:24│ Flags   │Limit Hi │ Access  │Base15:0 │Limit Lo │      │
│                                                                        │
│  Bytes 8-15 (High):                                                    │
│  │127      96│95       64│                                             │
│  ├───────────┼───────────┤                                             │
│  │ Reserved  │Base 63:32 │                                             │
│                                                                        │
│  Access Byte:                                                          │
│   Bit 7: P (Present)                                                   │
│   Bits 6-5: DPL (Descriptor Privilege Level)                          │
│   Bit 4: S (0 = System, 1 = Code/Data)                                │
│   Bits 3-0: Type                                                       │
│                                                                        │
└────────────────────────────────────────────────────────────────────────┘
```

### Code Segment Types

| Type | Binary | Description |
|------|--------|-------------|
| Execute-Only | 1000 | Code, no read |
| Execute/Read | 1010 | Code, readable |
| Execute-Only Conforming | 1100 | Accessible from lower privilege |
| Execute/Read Conforming | 1110 | Accessible from lower privilege, readable |

### DPL Checks

```
Code Segment Access (JMP, CALL, RET):
  - Non-conforming: CPL must equal DPL
  - Conforming: CPL must be ≥ DPL (numerically)

Data Segment Access:
  - CPL ≤ DPL (can only access same or less privileged)

Gate Access (Call Gate, Interrupt Gate):
  - CPL ≤ Gate.DPL
  - Target.DPL ≤ CPL (for privilege increase)
```

---

## 4. Page-Level Protection

### U/S Bit (User/Supervisor)

```
Page Table Entry U/S bit:
  U/S = 0: Supervisor-only (Ring 0)
  U/S = 1: User-accessible (Ring 0 and Ring 3)

Access Matrix:
┌─────────────────┬──────────────┬──────────────┐
│                 │   U/S = 0    │   U/S = 1    │
├─────────────────┼──────────────┼──────────────┤
│ CPL = 0 (Ring 0)│   Allowed    │   Allowed*   │
│ CPL = 3 (Ring 3)│   #PF        │   Allowed    │
└─────────────────┴──────────────┴──────────────┘

* With SMAP enabled, supervisor access to user pages requires RFLAGS.AC=1
```

### R/W Bit Combined with WP

```
CR0.WP (Write Protect) bit changes kernel behavior:

CR0.WP = 0 (disabled):
  - Kernel can write to read-only pages

CR0.WP = 1 (enabled, default):
  - Kernel respects R/W bit
  - Required for copy-on-write
```

### Combined Protection

```
Final Permission = Segment Check AND Page Check

Example:
  Code Segment: Execute, DPL=3
  Page: U/S=1, R/W=0, XD=0
  CPL: 3 (user mode)
  
  Result: Execute allowed, Write forbidden
```

---

## 5. I/O Privilege Level (IOPL)

### Controlling I/O Instructions

```
RFLAGS.IOPL (bits 12-13):
  - Controls access to IN, OUT, INS, OUTS
  - Controls CLI, STI in legacy mode

Rule: CPL ≤ IOPL to execute these instructions

In 64-bit mode with CPL=3:
  - IOPL ignored for CLI/STI (always cause #GP)
  - I/O Port Bitmap in TSS can override I/O restrictions
```

### I/O Permission Bitmap

```
TSS contains an I/O permission bitmap:
  - 65536 bits (one per I/O port)
  - Bit = 0: Port access allowed
  - Bit = 1: Port access causes #GP
  - Only consulted when CPL > IOPL
```

---

## 6. Security Extensions

### CR4 Protection Bits

| Bit | Name | Function |
|-----|------|----------|
| 2 | TSD | Time Stamp Disable (RDTSC requires CPL=0) |
| 4 | PSE | Page Size Extensions (4MB pages in 32-bit) |
| 5 | PAE | Physical Address Extension |
| 7 | PGE | Page Global Enable |
| 16 | FSGSBASE | Enable RDFSBASE/WRFSBASE from Ring 3 |
| 20 | SMEP | Supervisor Mode Execution Prevention |
| 21 | SMAP | Supervisor Mode Access Prevention |
| 22 | PKE | Protection Keys Enable |
| 23 | CET | Control-flow Enforcement Technology |

### Protection Keys (PKE)

```
User-space memory protection without page table changes:

Each page has a 4-bit protection key (PTE bits 62:59)
PKRU register (per-thread): 32 bits
  - 2 bits per key: Access Disable, Write Disable

Allows fast permission changes without TLB flush
```

### Control-flow Enforcement (CET)

```
Shadow Stack:
  - Separate stack for return addresses only
  - CALL pushes to both stacks
  - RET verifies addresses match
  - Prevents ROP attacks

Indirect Branch Tracking:
  - ENDBR32/ENDBR64 instructions
  - Indirect branches must target ENDBR
  - Prevents JOP attacks
```

---

## 7. Practical: Privilege Escalation Prevention

### Kernel Hardening Checklist

```
□ SMEP enabled (CR4.SMEP = 1)
   - Prevents ret2usr attacks

□ SMAP enabled (CR4.SMAP = 1)  
   - Prevents unintended user memory access

□ NX/XD enabled (EFER.NXE = 1)
   - Stack/heap non-executable

□ CR0.WP = 1
   - Kernel respects read-only pages

□ KASLR enabled
   - Kernel address randomization

□ Stack canaries
   - Detect stack buffer overflows

□ KPTI enabled
   - Kernel page table isolation (Meltdown mitigation)
```

---

# Key Readings

## Required

### AMD64 Architecture Programmer's Manual Volume 2

| Section | Topic | Time |
|---------|-------|------|
| Chapter 4.6-4.8 | Privilege levels, Limit checks | 2 hours |
| Chapter 4.10-4.13 | Type checks, Privilege checks | 2.5 hours |
| Chapter 5.6 | Page protection | 1.5 hours |

---

# Exercises & Mini-Projects

## Exercise 1: Privilege Violation Analysis

For each code path, predict whether #GP or #PF occurs (and why):

1. User code (CPL=3) tries `mov cr3, rax`
2. User code accesses address 0xFFFF800000000000 (kernel mapping)
3. Kernel code (CPL=0) executes code at user address (SMEP on)
4. User code executes `cli`

## Exercise 2: Create Call Gate

Implement a call gate that allows user mode to call a kernel function (legacy but educational).

## Exercise 3: IOPL Demonstration

Write code that demonstrates IOPL checking with port I/O.

## Mini-Project: Privilege Boundary Testing

Create a test kernel that:
1. Creates user/kernel memory regions
2. Tests all combinations of access
3. Verifies SMEP, SMAP, NX behavior
4. Reports protection failures correctly

---

# Quiz

**Q1**: What is CPL and where is it stored?

**Q2**: Can Ring 3 code access a page with U/S=0? Can Ring 0?

**Q3**: What does SMEP prevent? What does SMAP prevent?

**Q4**: How can user mode execute a privileged instruction?

**Q5**: What is the purpose of conforming code segments?

**Q6**: How does Protection Keys differ from standard page protection?

---

# Priority Guide

## CRITICAL
- CPL, DPL, RPL distinction
- U/S bit in page tables
- SMEP/SMAP purpose
- Privilege transition mechanisms

## IMPORTANT
- IOPL and I/O bitmap
- Call gates (conceptual)
- CR0.WP purpose
- Protection keys overview

## OPTIONAL
- Full gate types (task gates, etc.)
- CET implementation details
- Ring 1/2 mechanics

---

# Summary

Protection in AMD64 is enforced at multiple levels:
1. **CPL** determines what code can do
2. **Segment descriptors** (mostly vestigial in 64-bit) define access rights
3. **Page table entries** provide per-page U/S, R/W, NX protection
4. **CR4 bits** (SMEP, SMAP) add defense-in-depth

**Next Week**: System Calls Deep Dive—complete SYSCALL/SYSRET mechanics and Linux implementation.
