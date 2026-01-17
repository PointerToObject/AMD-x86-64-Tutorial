# AMD64 Systems Programming — Week 9
## Virtualization: AMD-V (SVM)

**Course**: AMD x86-64 Systems Programming  
**Primary Text**: AMD64 Architecture Programmer's Manual, Volume 2, Chapter 15  
**Week Duration**: ~15-20 hours of focused study  

---

# Lecture Notes

## 1. Virtualization Fundamentals

### The Virtualization Problem

```
Without hardware support (software virtualization):
┌─────────────────────────────────────────────────────────────────────────┐
│                                                                         │
│  Guest OS thinks it's in Ring 0, but actually runs in Ring 1 or 3     │
│  Privileged instructions must be trapped and emulated                  │
│  Binary translation needed for non-trapping sensitive instructions    │
│                                                                         │
│  Problems:                                                             │
│  - POPF doesn't trap (can't virtualize cleanly)                       │
│  - SGDT/SIDT reveal host state                                        │
│  - Performance overhead from trapping                                  │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘

With AMD-V (SVM):
┌─────────────────────────────────────────────────────────────────────────┐
│                                                                         │
│  New privilege level: VMX root (hypervisor) vs. VMX non-root (guest)  │
│  Guest runs in true Ring 0, but in non-root mode                      │
│  Hardware traps on configurable events (VM exits)                      │
│  Fast world switches via VMRUN/VMEXIT                                 │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

> **AMD Manual Reference**: Volume 2, Chapter 15 "Secure Virtual Machine"

---

## 2. SVM Architecture

### Mode Overview

```
┌─────────────────────────────────────────────────────────────────────────┐
│                         SVM Execution Model                            │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│                    ┌─────────────────────┐                             │
│                    │     Hypervisor      │                             │
│                    │    (Host Mode)      │                             │
│                    │     Ring 0          │                             │
│                    └──────────┬──────────┘                             │
│                               │                                         │
│            VMRUN             │             VMEXIT                      │
│          (enter guest)       │          (exit to host)                 │
│                               ▼                                         │
│                    ┌─────────────────────┐                             │
│                    │    Virtual Machine  │                             │
│                    │    (Guest Mode)     │                             │
│                    │   Ring 0/3 (guest)  │                             │
│                    └─────────────────────┘                             │
│                                                                         │
│  Guest runs at full speed until:                                       │
│  - Intercept condition triggers VMEXIT                                │
│  - Exception/interrupt requires host handling                          │
│  - Guest executes VMMCALL                                             │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### Enabling SVM

```asm
; Check SVM support
mov eax, 0x80000001
cpuid
test ecx, (1 << 2)      ; SVM bit
jz no_svm

; Check if SVM is disabled in BIOS
mov ecx, 0xC0010114     ; VM_CR MSR
rdmsr
test eax, (1 << 4)      ; SVMDIS bit
jnz svm_disabled_bios

; Enable SVM
mov ecx, 0xC0000080     ; EFER MSR
rdmsr
or eax, (1 << 12)       ; SVME bit
wrmsr
```

---

## 3. Virtual Machine Control Block (VMCB)

### VMCB Structure

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    VMCB (4KB aligned, two areas)                       │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  Control Area (Offset 0x000 - 0x3FF):                                 │
│  ├─ Intercept vectors (which events cause VMEXIT)                     │
│  ├─ IOPM_BASE_PA (I/O permission bitmap)                              │
│  ├─ MSRPM_BASE_PA (MSR permission bitmap)                             │
│  ├─ TSC_OFFSET (offset added to guest TSC)                            │
│  ├─ Guest ASID                                                        │
│  ├─ TLB control                                                       │
│  ├─ Interrupt/exception injection                                      │
│  ├─ EXITCODE, EXITINFO1, EXITINFO2                                    │
│  └─ Nested paging control (if enabled)                                │
│                                                                         │
│  State Save Area (Offset 0x400 - 0xFFF):                              │
│  ├─ Guest segment registers (ES, CS, SS, DS, FS, GS, LDTR, TR)       │
│  ├─ Guest control registers (CR0, CR2, CR3, CR4)                     │
│  ├─ Guest RFLAGS, RIP, RSP                                           │
│  ├─ Guest debug registers                                             │
│  ├─ Guest EFER, PAT                                                   │
│  └─ Guest interrupt state                                             │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### Key VMCB Fields

```c
// Control area offsets
#define VMCB_INTERCEPTS_CR      0x000  // CR read/write intercepts
#define VMCB_INTERCEPTS_DR      0x004  // DR read/write intercepts
#define VMCB_INTERCEPTS_EXC     0x008  // Exception intercepts
#define VMCB_INTERCEPTS_CTRL1   0x00C  // INTR, NMI, SMI, INIT, etc.
#define VMCB_INTERCEPTS_CTRL2   0x010  // VMRUN, VMMCALL, etc.
#define VMCB_IOPM_BASE          0x040  // I/O permission bitmap PA
#define VMCB_MSRPM_BASE         0x048  // MSR permission bitmap PA
#define VMCB_TSC_OFFSET         0x050  // TSC offset
#define VMCB_GUEST_ASID         0x058  // Guest Address Space ID
#define VMCB_TLB_CONTROL        0x05C  // TLB flush control
#define VMCB_EXITCODE           0x070  // Exit reason
#define VMCB_EXITINFO1          0x078  // Exit info 1
#define VMCB_EXITINFO2          0x080  // Exit info 2
#define VMCB_EVENTINJ           0x0A8  // Event injection

// State save area offsets
#define VMCB_ES                 0x400
#define VMCB_CS                 0x410
#define VMCB_SS                 0x420
#define VMCB_DS                 0x430
#define VMCB_FS                 0x440
#define VMCB_GS                 0x450
#define VMCB_GDTR               0x460
#define VMCB_LDTR               0x470
#define VMCB_IDTR               0x480
#define VMCB_TR                 0x490
#define VMCB_CR4                0x548
#define VMCB_CR3                0x550
#define VMCB_CR0                0x558
#define VMCB_DR7                0x560
#define VMCB_DR6                0x568
#define VMCB_RFLAGS             0x570
#define VMCB_RIP                0x578
#define VMCB_RSP                0x5D8
#define VMCB_RAX                0x5F8
#define VMCB_CR2                0x640
```

---

## 4. VMRUN and VMEXIT

### VMRUN Instruction

```asm
; Enter guest VM
; RAX = Physical address of VMCB

vmrun_guest:
    ; Save host state (not saved by VMRUN)
    push rbx
    push rcx
    ; ... save all GPRs except RAX
    
    ; Load VMCB address
    mov rax, [vmcb_phys_addr]
    
    ; Enter guest (returns on VMEXIT)
    vmrun
    
    ; We're back - VMEXIT occurred
    ; RAX still contains VMCB address
    
    ; Restore host GPRs
    pop rcx
    pop rbx
    ; ...
    
    ; Check exit reason
    mov rax, [vmcb_phys_addr]
    mov rbx, [rax + VMCB_EXITCODE]
    ; Handle exit...
```

### Common Exit Codes

| Code | Name | Description |
|------|------|-------------|
| 0x40 | VMEXIT_EXCP0 | Division by zero |
| 0x4E | VMEXIT_EXCP14 | Page fault |
| 0x60 | VMEXIT_INTR | Physical interrupt |
| 0x61 | VMEXIT_NMI | NMI |
| 0x72 | VMEXIT_CPUID | CPUID instruction |
| 0x78 | VMEXIT_INVLPG | INVLPG instruction |
| 0x7B | VMEXIT_IOIO | I/O instruction |
| 0x7C | VMEXIT_MSR | RDMSR/WRMSR |
| 0x81 | VMEXIT_VMRUN | Nested VMRUN |

---

## 5. Nested Paging (NPT)

### Two-Level Address Translation

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    Nested Paging Translation                           │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  Guest Virtual ──► Guest Page Tables ──► Guest Physical               │
│       (GVA)           (guest CR3)           (GPA)                      │
│                                              │                          │
│                                              ▼                          │
│                         Nested Page Tables ──► System Physical         │
│                            (nCR3 in VMCB)       (SPA/HPA)             │
│                                                                         │
│  Without NPT: Every GPA access causes VMEXIT                          │
│  With NPT: Hardware walks both levels, minimal VMEXITs                │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### Enabling NPT

```c
// In VMCB control area
vmcb->control.np_enable = 1;
vmcb->control.n_cr3 = nested_page_table_phys;  // Host-controlled
```

---

## 6. Minimal Hypervisor Structure

```c
// Pseudocode for minimal SVM hypervisor

struct vmcb *vmcb;
struct guest_state guest;

void hypervisor_main(void) {
    // 1. Enable SVM
    enable_svm();
    
    // 2. Allocate and initialize VMCB
    vmcb = alloc_vmcb();
    init_vmcb(vmcb, &guest);
    
    // 3. Set up intercepts
    vmcb->control.intercept_cpuid = 1;
    vmcb->control.intercept_msr = 1;
    vmcb->control.intercept_io = 1;
    
    // 4. Set up nested page tables
    vmcb->control.np_enable = 1;
    vmcb->control.n_cr3 = setup_npt();
    
    // 5. Run guest
    while (1) {
        vmrun(vmcb);
        
        switch (vmcb->control.exitcode) {
        case VMEXIT_CPUID:
            handle_cpuid(&guest);
            break;
        case VMEXIT_MSR:
            handle_msr(&guest);
            break;
        case VMEXIT_IOIO:
            handle_io(&guest);
            break;
        // ... other handlers ...
        }
    }
}
```

---

## 7. Security: SEV (Secure Encrypted Virtualization)

### SEV Overview

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    AMD SEV Technology                                   │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  SEV: Memory encryption per-VM (AES-128)                              │
│  SEV-ES: Also encrypts guest register state                           │
│  SEV-SNP: Adds integrity protection and attestation                   │
│                                                                         │
│  Threat model:                                                         │
│  - Protects guest from malicious hypervisor                           │
│  - Protects guest from physical memory attacks                        │
│  - Hardware-based key management                                       │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

---

# Key Readings

## Required

| Source | Section | Topic |
|--------|---------|-------|
| AMD Vol 2 | Chapter 15.1-15.10 | SVM basics |
| AMD Vol 2 | Chapter 15.20-15.25 | Nested paging |
| AMD Vol 2 | Appendix B | VMCB layout |

## Supplementary
- KVM source code: arch/x86/kvm/svm/
- SimpleVisor project (educational hypervisor)

---

# Exercises & Mini-Projects

## Exercise 1: SVM Detection

Write code that detects SVM support and checks BIOS enable status.

## Exercise 2: VMCB Structure

Create header files defining the complete VMCB structure.

## Mini-Project: Minimal Hypervisor

Build a hypervisor that:
1. Enables SVM
2. Runs a simple guest (real mode code)
3. Handles CPUID intercepts
4. Exits cleanly

---

# Quiz

**Q1**: What is the difference between host mode and guest mode in SVM?

**Q2**: What causes a VMEXIT?

**Q3**: What is the purpose of the VMCB?

**Q4**: How does nested paging improve performance?

**Q5**: What does VMRUN do to host register state?

---

# Priority Guide

## CRITICAL
- SVM enable sequence
- VMCB structure (control + state areas)
- VMRUN/VMEXIT flow
- Basic intercepts

## IMPORTANT
- Nested paging concept
- Exit code handling
- MSR/IO interception

## OPTIONAL
- SEV details
- Nested virtualization
- AVIC (interrupt virtualization)

---

**Next Week**: Security Features—SEV, SME, and hardware security mechanisms.
