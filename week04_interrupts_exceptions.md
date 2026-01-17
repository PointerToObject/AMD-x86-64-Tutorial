# AMD64 Systems Programming — Week 4
## Interrupts and Exceptions

**Course**: AMD x86-64 Systems Programming  
**Primary Text**: AMD64 Architecture Programmer's Manual, Volume 2  
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

## 1. The Interrupt/Exception Taxonomy

### Classification

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    Interrupts and Exceptions                            │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  INTERRUPTS (Asynchronous)                                             │
│  ├── Hardware Interrupts (External)                                    │
│  │   ├── Maskable (INTR pin, controlled by IF flag)                   │
│  │   └── Non-Maskable (NMI pin, cannot be disabled)                   │
│  └── Software Interrupts                                               │
│      ├── INT n instruction                                             │
│      └── INT3 (breakpoint, single-byte 0xCC)                          │
│                                                                         │
│  EXCEPTIONS (Synchronous)                                              │
│  ├── Faults (restartable, RIP = faulting instruction)                 │
│  │   Examples: #PF, #GP, #DE                                          │
│  ├── Traps (RIP = next instruction after trapping)                    │
│  │   Examples: #DB (debug), #BP (breakpoint)                          │
│  └── Aborts (unrecoverable)                                           │
│      Examples: #DF (double fault), #MC (machine check)                │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

> **AMD Manual Reference**: Volume 2, Chapter 8 "Exceptions and Interrupts"

### Exception Vector Table

| Vector | Mnemonic | Name | Type | Error Code |
|--------|----------|------|------|------------|
| 0 | #DE | Divide Error | Fault | No |
| 1 | #DB | Debug | Fault/Trap | No |
| 2 | NMI | Non-Maskable Interrupt | Interrupt | No |
| 3 | #BP | Breakpoint | Trap | No |
| 4 | #OF | Overflow | Trap | No |
| 5 | #BR | Bound Range | Fault | No |
| 6 | #UD | Invalid Opcode | Fault | No |
| 7 | #NM | Device Not Available | Fault | No |
| 8 | #DF | Double Fault | Abort | Yes (always 0) |
| 10 | #TS | Invalid TSS | Fault | Yes |
| 11 | #NP | Segment Not Present | Fault | Yes |
| 12 | #SS | Stack Fault | Fault | Yes |
| 13 | #GP | General Protection | Fault | Yes |
| 14 | #PF | Page Fault | Fault | Yes |
| 16 | #MF | x87 FPU Error | Fault | No |
| 17 | #AC | Alignment Check | Fault | Yes (always 0) |
| 18 | #MC | Machine Check | Abort | No |
| 19 | #XF | SIMD Exception | Fault | No |
| 32-255 | — | User Defined | — | — |

---

## 2. The Interrupt Descriptor Table (IDT)

### IDT Structure in Long Mode

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    IDT Entry (Gate Descriptor) - 16 bytes               │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  Bytes 0-1:   Offset [15:0]                                            │
│  Bytes 2-3:   Segment Selector                                         │
│  Byte 4:      IST [2:0], Reserved [7:3]                                │
│  Byte 5:      Type [3:0], 0, DPL [1:0], P                             │
│  Bytes 6-7:   Offset [31:16]                                          │
│  Bytes 8-11:  Offset [63:32]                                          │
│  Bytes 12-15: Reserved (must be 0)                                     │
│                                                                         │
│  ┌──────────────────────────────────────────────────────────────────┐  │
│  │ 127                   64 63        48 47      32 31            0 │  │
│  ├──────────────────────────┬───────────┬──────────┬────────────────│  │
│  │       Reserved           │Offset High│ Attr/IST │ Sel │Off Low  │  │
│  └──────────────────────────┴───────────┴──────────┴─────┴─────────┘  │
│                                                                         │
│  Gate Types (in Type field):                                           │
│    0xE = 64-bit Interrupt Gate (IF cleared on entry)                  │
│    0xF = 64-bit Trap Gate (IF unchanged)                              │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### IDTR Register

```asm
; Load IDT
lidt [idt_descriptor]

idt_descriptor:
    dw idt_end - idt_start - 1   ; Limit (size - 1)
    dq idt_start                  ; Base address (64-bit)
```

### IDT Entry Setup (Assembly)

```asm
; Macro to create IDT entry
%macro IDT_ENTRY 2  ; handler_address, attributes
    dw (%1) & 0xFFFF           ; Offset 15:0
    dw 0x0008                   ; Code segment selector
    db 0                        ; IST = 0
    db %2                       ; Attributes
    dw ((%1) >> 16) & 0xFFFF   ; Offset 31:16
    dd (%1) >> 32              ; Offset 63:32
    dd 0                        ; Reserved
%endmacro

; Example: #GP handler
IDT_ENTRY gp_handler, 0x8E     ; 0x8E = Present, Ring 0, Interrupt Gate
```

---

## 3. Interrupt Handling Mechanism

### What the CPU Does Automatically

```
┌─────────────────────────────────────────────────────────────────────────┐
│                Interrupt/Exception Entry (Long Mode)                    │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  1. If privilege change (user→kernel):                                 │
│     - Load RSP from TSS.RSP0                                           │
│     - Push old SS                                                      │
│     - Push old RSP                                                     │
│                                                                         │
│  2. Push RFLAGS                                                        │
│  3. Push CS                                                            │
│  4. Push RIP                                                           │
│  5. If exception has error code: Push error code                       │
│                                                                         │
│  6. If Interrupt Gate: Clear IF (disable interrupts)                   │
│  7. Load CS:RIP from IDT entry                                         │
│                                                                         │
│  Stack after entry (privilege change case):                            │
│                                                                         │
│         +40  │  Old SS        │                                        │
│         +32  │  Old RSP       │                                        │
│         +24  │  Old RFLAGS    │                                        │
│         +16  │  Old CS        │                                        │
│         +8   │  Old RIP       │                                        │
│  RSP ─► +0   │  Error Code    │ (if applicable)                        │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### IRET: Returning from Interrupt

```asm
; IRETQ pops in reverse order:
; 1. RIP
; 2. CS
; 3. RFLAGS
; 4. RSP (if privilege change)
; 5. SS (if privilege change)

; Example handler epilogue
my_handler:
    ; ... handle interrupt ...
    
    ; If error code was pushed, remove it first
    add rsp, 8
    
    iretq
```

---

## 4. The Task State Segment (TSS)

### TSS in Long Mode

In long mode, TSS is minimal—mainly used for:
- RSP values for privilege level changes
- Interrupt Stack Table (IST) pointers
- I/O permission bitmap

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    64-bit TSS Structure                                 │
├─────────────────────────────────────────────────────────────────────────┤
│  Offset   Size   Field                                                 │
│  ────────────────────────────────────────────────────────────────────  │
│  0x00     4      Reserved                                              │
│  0x04     8      RSP0 (Ring 0 stack pointer)                          │
│  0x0C     8      RSP1 (Ring 1 stack pointer)                          │
│  0x14     8      RSP2 (Ring 2 stack pointer)                          │
│  0x1C     8      Reserved                                              │
│  0x24     8      IST1 (Interrupt Stack 1)                             │
│  0x2C     8      IST2                                                  │
│  0x34     8      IST3                                                  │
│  0x3C     8      IST4                                                  │
│  0x44     8      IST5                                                  │
│  0x4C     8      IST6                                                  │
│  0x54     8      IST7                                                  │
│  0x5C     8      Reserved                                              │
│  0x64     2      Reserved                                              │
│  0x66     2      I/O Map Base Address                                  │
│                                                                         │
│  Minimum TSS size: 104 bytes (0x68)                                    │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### Interrupt Stack Table (IST)

IST allows specific interrupts to use dedicated stacks, preventing stack overflow issues (e.g., double fault handler):

```asm
; In IDT entry, IST field (bits 0-2 of byte 4):
; IST = 0: Use normal stack switching
; IST = 1-7: Use TSS.ISTn as new RSP

; Example: Double fault uses IST1
IDT_ENTRY df_handler, 0x8E, 1   ; IST = 1
```

> **AMD Manual Reference**: Volume 2, Section 8.7 "Task State Segment"

---

## 5. System Calls: SYSCALL/SYSRET

### SYSCALL Mechanism

`SYSCALL` is faster than `INT 0x80` because it avoids IDT lookup:

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    SYSCALL Execution                                    │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  1. RCX ← RIP (save return address)                                    │
│  2. R11 ← RFLAGS (save flags)                                          │
│  3. RFLAGS ← RFLAGS AND NOT(IA32_FMASK) (mask flags)                  │
│  4. CS ← IA32_STAR[47:32] (kernel code segment)                        │
│  5. SS ← IA32_STAR[47:32] + 8 (kernel stack segment)                   │
│  6. RIP ← IA32_LSTAR (kernel entry point)                              │
│  7. CPL ← 0 (ring 0)                                                   │
│                                                                         │
│  Note: RSP is NOT changed by SYSCALL                                   │
│        Kernel must switch to kernel stack manually                     │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### MSRs for SYSCALL

```
MSR Address    Name         Purpose
───────────────────────────────────────────────────────────────
0xC0000081     IA32_STAR    SYSCALL CS/SS (high 32 bits)
0xC0000082     IA32_LSTAR   SYSCALL RIP (64-bit target)
0xC0000083     IA32_CSTAR   SYSCALL RIP (compat mode, unused on most OS)
0xC0000084     IA32_FMASK   RFLAGS mask (bits to clear)
```

### SYSRET Execution

```asm
; SYSRET restores user mode:
; 1. RIP ← RCX
; 2. RFLAGS ← R11 (with some restrictions)
; 3. CS ← IA32_STAR[63:48] + 16
; 4. SS ← IA32_STAR[63:48] + 8
; 5. CPL ← 3
```

### Minimal Syscall Handler

```asm
; System call entry point
syscall_entry:
    ; Save user RSP, load kernel RSP
    mov [gs:user_rsp], rsp
    mov rsp, [gs:kernel_rsp]
    
    ; Save registers
    push rcx                ; User RIP
    push r11                ; User RFLAGS
    
    ; RAX = syscall number
    ; RDI, RSI, RDX, R10, R8, R9 = arguments
    
    ; Dispatch to handler...
    
    ; Restore and return
    pop r11
    pop rcx
    mov rsp, [gs:user_rsp]
    sysretq
```

---

## 6. Hardware Interrupts and the APIC

### Legacy PIC vs. APIC

```
Legacy PIC (8259):           Modern APIC:
- 15 IRQ lines               - 256 interrupt vectors
- Master/Slave cascade       - Per-CPU Local APIC
- Edge-triggered only        - I/O APIC for device routing
- No SMP support             - Full SMP support
```

### Local APIC Registers (Memory-Mapped)

```
Offset    Register              Description
────────────────────────────────────────────────────────────
0x020     Local APIC ID         Unique CPU identifier
0x080     Task Priority (TPR)   Interrupt priority threshold
0x0B0     End of Interrupt      Write to signal EOI
0x0F0     Spurious Vector       Enable APIC, spurious vector
0x300     ICR Low               Inter-processor interrupt
0x310     ICR High              Destination CPU
0x320     LVT Timer             Local timer configuration
0x350     LVT LINT0             Local interrupt 0
0x360     LVT LINT1             Local interrupt 1
0x380     Initial Count         Timer initial value
0x390     Current Count         Timer current value
0x3E0     Divide Config         Timer divider
```

### Sending End-of-Interrupt (EOI)

```asm
; After handling hardware interrupt, send EOI to APIC
mov dword [APIC_BASE + 0x0B0], 0
```

---

## 7. Debugging Exceptions

### Single-Step (#DB)

```asm
; Enable single-step mode
pushfq
or qword [rsp], 0x100    ; Set TF (Trap Flag)
popfq
; Next instruction will trigger #DB
```

### Debug Registers (DR0-DR7)

```
DR0-DR3: Breakpoint addresses (up to 4)
DR6:     Debug status (which breakpoint hit)
DR7:     Debug control (enable/configure breakpoints)

; Hardware breakpoint on memory access
mov rax, [address_to_watch]
mov dr0, rax
mov rax, dr7
or rax, 0x000F0001      ; Enable DR0, break on write, 4 bytes
mov dr7, rax
```

---

# Key Readings

## Required

### AMD64 Architecture Programmer's Manual Volume 2

| Section | Topic | Time |
|---------|-------|------|
| Chapter 8.1-8.4 | Interrupt/Exception overview | 2 hours |
| Chapter 8.5-8.8 | IDT, Gates, TSS | 2.5 hours |
| Chapter 8.9 | Interrupt priorities | 1 hour |
| Chapter 6.1-6.2 | SYSCALL/SYSRET | 1.5 hours |

## Supplementary

- **Linux kernel source**: `arch/x86/kernel/entry_64.S`
- **OSDev Wiki**: "Interrupt Descriptor Table"
- **Intel SDM Volume 3**: Chapter 6 (different perspective)

---

# Recommended Videos

### 1. OpenSecurityTraining2 - Arch2001
| Video | Topic |
|-------|-------|
| "Interrupts vs Exceptions" | Classification |
| "IDT Setup" | Practical IDT configuration |
| "SYSCALL/SYSRET" | System call mechanism |

### 2. "Writing a Simple Operating System from Scratch" - Nick Blundell
**Format**: PDF/Book with code  
**Chapters 5-6**: Interrupt handling

---

# Exercises & Mini-Projects

## Exercise 1: IDT Entry Encoding

Encode an IDT entry for:
- Handler at 0xFFFF800000001000
- Ring 0 only
- Interrupt gate (clear IF)
- IST = 0

Show all 16 bytes in hex.

## Exercise 2: Exception Analysis

For each scenario, identify the exception vector and error code:

1. Executing at unmapped address
2. Writing to read-only page from user mode
3. Dividing by zero
4. Invalid opcode execution
5. Stack overflow (RSP below stack limit)

## Exercise 3: Syscall Tracer

Write a program that:
1. Uses ptrace to trace another process
2. Intercepts all syscalls
3. Logs syscall numbers and arguments

## Exercise 4: IDT Setup (Mini-Project)

Create a minimal kernel that:
1. Sets up GDT with code/data/TSS segments
2. Sets up IDT with handlers for #DE, #GP, #PF
3. Triggers each exception intentionally
4. Handles and recovers from each

Can use QEMU + bootloader (GRUB multiboot or custom).

## Exercise 5: APIC Timer

Configure the Local APIC timer to fire an interrupt every 10ms. Count interrupts and print a message every second.

---

# Quiz

**Q1**: What's the difference between an Interrupt Gate and a Trap Gate?

**Q2**: What is automatically pushed to the stack when a #GP exception occurs from user mode?

**Q3**: Why does SYSCALL save RIP to RCX instead of pushing it to the stack?

**Q4**: What happens if a #PF occurs while the CPU is already handling a #GP?

**Q5**: What is the IST (Interrupt Stack Table) used for?

**Q6**: How does the CPU determine which ring to switch to when an interrupt occurs?

**Q7**: What must a kernel do BEFORE executing SYSRET?

**Q8**: Why is the Local APIC necessary for SMP systems?

---

# Priority Guide

## CRITICAL
- IDT structure and entry format
- Stack frame on interrupt entry
- Exception types and vectors
- SYSCALL/SYSRET mechanism

## IMPORTANT
- TSS structure and purpose
- IST mechanism
- Error code format
- APIC basics

## OPTIONAL (This Week)
- Full APIC programming
- I/O APIC configuration
- MSI/MSI-X interrupts
- All debug registers

---

# Summary

Interrupts and exceptions are the CPU's mechanism for handling asynchronous events and error conditions. Key insights:

1. **IDT** maps vectors to handlers—16 bytes per entry in long mode
2. **Privilege transitions** involve automatic stack switch via TSS
3. **SYSCALL/SYSRET** is the fast path for user-kernel transitions
4. **TSS.IST** provides dedicated stacks for critical handlers
5. **APIC** enables modern interrupt routing and SMP

**Next Week**: Protection and Privilege—how the CPU enforces security boundaries.
