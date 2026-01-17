# AMD64 Systems Programming — Week 10
## Security Features: SME, SEV, and Hardware Security

**Course**: AMD x86-64 Systems Programming  
**Primary Text**: AMD64 Architecture Programmer's Manual, Volume 2  
**Week Duration**: ~15-20 hours of focused study  

---

# Lecture Notes

## 1. AMD Memory Encryption Overview

### Technology Stack

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    AMD Memory Encryption Technologies                   │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  SME (Secure Memory Encryption):                                       │
│    - Single key for all memory                                         │
│    - Transparent to software                                           │
│    - Protects against physical attacks                                 │
│                                                                         │
│  SEV (Secure Encrypted Virtualization):                                │
│    - Per-VM encryption keys                                            │
│    - Protects guests from hypervisor                                   │
│                                                                         │
│  SEV-ES (Encrypted State):                                             │
│    - Also encrypts guest CPU registers                                 │
│    - Prevents hypervisor from reading register state                   │
│                                                                         │
│  SEV-SNP (Secure Nested Paging):                                       │
│    - Adds memory integrity protection                                  │
│    - Reverse Map Table (RMP) prevents remapping attacks               │
│    - Attestation support                                               │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

> **AMD Manual Reference**: Volume 2, Chapter 16 "Secure Encrypted Virtualization"

---

## 2. SME: Secure Memory Encryption

### How SME Works

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    SME Memory Access Flow                              │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│   CPU Core                                                             │
│      │                                                                  │
│      ▼                                                                  │
│   Cache (unencrypted data)                                             │
│      │                                                                  │
│      ▼                                                                  │
│   Memory Controller                                                     │
│      │                                                                  │
│      ├──── AES Engine ────┐                                            │
│      │     (encrypt)      │                                            │
│      ▼                    ▼                                            │
│   DRAM (encrypted)     DRAM (encrypted)                                │
│                                                                         │
│   C-bit in page table entry:                                           │
│     C-bit = 1: Page is encrypted                                       │
│     C-bit = 0: Page is unencrypted (shared/MMIO)                      │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### Enabling SME

```asm
; Check SME support
mov eax, 0x8000001F
cpuid
test eax, (1 << 0)      ; SME supported
jz no_sme

; EBX[5:0] = C-bit position in page table

; Enable SME
mov ecx, 0xC0010010     ; SYSCFG MSR
rdmsr
or eax, (1 << 23)       ; MEM_ENCRYPT bit
wrmsr
```

### C-bit in Page Tables

```
Standard PTE:
┌────────────────────────────────────────────────────────────────────────┐
│ 63│62      52│51        C │ C-1    12│11      0│                      │
├───┼──────────┼────────────┼──────────┼─────────┤                      │
│XD │ Available│  C-bit     │Phys Addr │  Flags  │                      │
└───┴──────────┴────────────┴──────────┴─────────┘

C-bit position varies by processor (check CPUID 0x8000001F)
Typically bit 47 or bit 51
```

---

## 3. SEV: Per-VM Encryption

### SEV Key Management

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    SEV Key Hierarchy                                   │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│            AMD Secure Processor (PSP)                                  │
│                      │                                                  │
│                      ▼                                                  │
│              ┌──────────────┐                                          │
│              │   CEK        │  (Chip Endorsement Key - factory)       │
│              └──────────────┘                                          │
│                      │                                                  │
│                      ▼                                                  │
│              ┌──────────────┐                                          │
│              │   PEK        │  (Platform Endorsement Key)             │
│              └──────────────┘                                          │
│                      │                                                  │
│                      ▼                                                  │
│              ┌──────────────┐                                          │
│              │   VEK        │  (VM Encryption Key - per guest)        │
│              └──────────────┘                                          │
│                                                                         │
│   Hypervisor cannot access VEK - managed by PSP                       │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### SEV Guest Launch

```c
// Simplified SEV launch flow

// 1. Create guest context (PSP generates VEK)
sev_cmd_launch_start(guest_handle);

// 2. Encrypt guest memory pages
for (each page) {
    sev_cmd_launch_update_data(guest_handle, page);
}

// 3. Finalize launch (get measurement)
sev_cmd_launch_finish(guest_handle, &measurement);

// 4. Guest can now run with encrypted memory
vmrun(vmcb);
```

---

## 4. SEV-ES: Encrypted State

### VMSA (VM Save Area)

```
With SEV-ES, guest register state is encrypted:
- VMCB state area becomes encrypted VMSA
- Hypervisor cannot read/modify guest registers
- Guest must explicitly share data via GHCB

┌─────────────────────────────────────────────────────────────────────────┐
│                    SEV-ES Communication                                │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│   Guest needs to communicate with hypervisor (e.g., I/O):             │
│                                                                         │
│   1. Guest places request in GHCB (Guest-Hypervisor Comm Block)       │
│   2. Guest executes VMGEXIT (new instruction)                         │
│   3. Hypervisor reads GHCB (unencrypted shared page)                  │
│   4. Hypervisor performs operation                                     │
│   5. Hypervisor writes result to GHCB                                 │
│   6. Control returns to guest                                          │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 5. SEV-SNP: Integrity Protection

### Reverse Map Table (RMP)

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    RMP Protection Model                                │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│   Problem SEV-SNP solves:                                              │
│   - With SEV/SEV-ES, hypervisor can remap guest pages                 │
│   - Could cause guest to use wrong encrypted data                      │
│                                                                         │
│   Solution: RMP (one entry per physical page)                          │
│                                                                         │
│   RMP Entry:                                                           │
│   ┌─────────────────────────────────────────────────────────────────┐  │
│   │ Assigned │ ASID │ Immutable │ Page State │ GPA │ Validated │    │  │
│   └─────────────────────────────────────────────────────────────────┘  │
│                                                                         │
│   Hardware checks:                                                      │
│   - Page assigned to correct guest (ASID match)                       │
│   - GPA in RMP matches GPA in nested page table                       │
│   - Page validated by guest before use                                 │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### Attestation

```
SEV-SNP provides attestation:
1. Guest requests attestation report (ATTESTATION instruction)
2. PSP generates report signed with platform key
3. Report includes:
   - Launch measurement
   - Guest policy
   - Platform info
   - Signature chain to AMD root
4. Remote verifier validates chain to AMD
```

---

## 6. Other Security Features

### Shadow Stacks (CET)

```asm
; Control-flow Enforcement Technology

; Shadow stack setup
mov ecx, 0x6A0           ; IA32_PL0_SSP
mov eax, shadow_stack_addr_low
mov edx, shadow_stack_addr_high
wrmsr

; Enable shadow stacks in CR4
mov rax, cr4
or rax, (1 << 23)        ; CET bit
mov cr4, rax

; Shadow stack operations
incsspq rax              ; Increment shadow stack pointer
rdsspq rax               ; Read shadow stack pointer
```

### IBRS/IBPB (Spectre Mitigations)

```asm
; Indirect Branch Restricted Speculation
mov ecx, 0x48            ; IA32_SPEC_CTRL
rdmsr
or eax, (1 << 0)         ; IBRS bit
wrmsr

; Indirect Branch Predictor Barrier
mov ecx, 0x49            ; IA32_PRED_CMD
mov eax, 1               ; IBPB command
xor edx, edx
wrmsr
```

---

## 7. Security Checklist

```
Hardware Security Features:
□ SME enabled (if physical attack threat model)
□ SEV for VM isolation (cloud/multi-tenant)
□ SEV-ES for register protection
□ SEV-SNP for full integrity (if available)

CPU Mitigations:
□ IBRS/IBPB enabled (Spectre v2)
□ SSBD enabled (Spectre v4)  
□ L1D_FLUSH on VM entry (L1TF)
□ MDS mitigations enabled

Page Table Security:
□ NX/XD enabled
□ SMEP enabled
□ SMAP enabled
□ Protection Keys (if using)
□ CET Shadow Stacks (if using)
```

---

# Key Readings

## Required

| Source | Section | Topic |
|--------|---------|-------|
| AMD Vol 2 | Chapter 16 | SME/SEV |
| AMD SEV-SNP ABI | All | SNP programming |

## Supplementary
- AMD SEV-SNP Whitepaper
- Linux kernel SEV documentation

---

# Exercises & Mini-Projects

## Exercise 1: SME Detection

Write code that detects SME/SEV support and reports C-bit position.

## Exercise 2: Memory Encryption Verification

If available, verify that SME is active by checking SYSCFG MSR.

## Mini-Project: SEV Policy Parser

Write a tool that parses and displays SEV guest policy fields.

---

# Quiz

**Q1**: What is the C-bit and where is it stored?

**Q2**: How does SEV protect a guest from a malicious hypervisor?

**Q3**: What additional protection does SEV-SNP provide over SEV-ES?

**Q4**: What is the RMP and what attacks does it prevent?

**Q5**: How does attestation work in SEV-SNP?

---

# Priority Guide

## CRITICAL
- SME concept and C-bit
- SEV per-VM encryption
- SEV-ES motivation

## IMPORTANT
- SEV-SNP RMP protection
- Attestation basics
- CET shadow stacks

## OPTIONAL
- Full PSP API
- SEV-SNP GHCB protocol
- Live migration with SEV

---

**Next Week**: Performance Analysis—PMU, profiling, and optimization.
