# AMD64 Systems Programming — Week 6
## System Calls: Complete SYSCALL/SYSRET Analysis

**Course**: AMD x86-64 Systems Programming  
**Primary Text**: AMD64 Architecture Programmer's Manual, Volume 2  
**Week Duration**: ~15-20 hours of focused study  

---

# Lecture Notes

## 1. System Call Evolution

### Historical Methods

```
Method          Era         Mechanism           Overhead
────────────────────────────────────────────────────────────────
Software INT    8086+       INT 0x80            ~300 cycles (IDT lookup)
Call Gate       286+        CALL far            ~100 cycles (descriptor)
SYSENTER/EXIT   Pentium II  MSR-based           ~50 cycles
SYSCALL/SYSRET  K6/Athlon   MSR-based           ~30 cycles (64-bit only)
```

> **AMD Manual Reference**: Volume 2, Chapter 6 "System-Management Instructions"

---

## 2. SYSCALL Architecture

### MSR Configuration

```asm
; Configure SYSCALL target
mov ecx, 0xC0000081        ; IA32_STAR
rdmsr
; Set SYSCALL CS/SS in bits [47:32]
; Set SYSRET CS/SS in bits [63:48]
mov edx, 0x00230010        ; User CS = 0x23, Kernel CS = 0x10
wrmsr

; Set 64-bit syscall handler
mov ecx, 0xC0000082        ; IA32_LSTAR
mov eax, syscall_entry_low
mov edx, syscall_entry_high
wrmsr

; Set flags mask
mov ecx, 0xC0000084        ; IA32_FMASK
mov eax, 0x00000700        ; Clear TF, IF, DF
xor edx, edx
wrmsr

; Enable SYSCALL (EFER.SCE)
mov ecx, 0xC0000080        ; EFER
rdmsr
or eax, 1                  ; Set SCE bit
wrmsr
```

### SYSCALL Execution (Detailed)

```
┌─────────────────────────────────────────────────────────────────────────┐
│                      SYSCALL Microarchitectural Steps                   │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  1. Privilege check: Must be in Long Mode                              │
│                                                                         │
│  2. Save state:                                                        │
│     RCX ← RIP (instruction following SYSCALL)                          │
│     R11 ← RFLAGS                                                       │
│                                                                         │
│  3. Load kernel state:                                                 │
│     RFLAGS ← RFLAGS AND NOT(IA32_FMASK)                               │
│                  AND NOT(RF | VM | reserved)                          │
│     CS.Selector ← STAR[47:32] (kernel CS)                             │
│     CS.Attributes ← Fixed 64-bit code segment                         │
│     SS.Selector ← STAR[47:32] + 8                                     │
│     SS.Attributes ← Fixed 64-bit data segment                         │
│     RIP ← LSTAR (entry point)                                         │
│     CPL ← 0                                                            │
│                                                                         │
│  Note: RSP is NOT modified! Kernel must save/swap immediately         │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 3. Linux System Call Convention

### Register Usage

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    Linux x86-64 System Call ABI                        │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  Input:                                                                 │
│    RAX = System call number                                            │
│    RDI = Argument 1                                                    │
│    RSI = Argument 2                                                    │
│    RDX = Argument 3                                                    │
│    R10 = Argument 4 (NOT RCX - SYSCALL uses RCX)                      │
│    R8  = Argument 5                                                    │
│    R9  = Argument 6                                                    │
│                                                                         │
│  Output:                                                                │
│    RAX = Return value (or negative errno)                              │
│                                                                         │
│  Clobbered by SYSCALL:                                                 │
│    RCX = Contains return RIP                                           │
│    R11 = Contains saved RFLAGS                                         │
│                                                                         │
│  Preserved (by kernel):                                                 │
│    RBX, RBP, R12, R13, R14, R15                                       │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### System Call Example

```asm
; sys_write(fd, buf, count)
; ssize_t write(int fd, const void *buf, size_t count);

section .data
    msg: db "Hello, syscall!", 10

section .text
global _start
_start:
    mov rax, 1              ; __NR_write
    mov rdi, 1              ; fd = stdout
    lea rsi, [rel msg]      ; buf
    mov rdx, 16             ; count
    syscall
    
    ; Check for error
    test rax, rax
    js .error
    
    ; sys_exit(0)
    mov rax, 60
    xor rdi, rdi
    syscall

.error:
    neg rax                 ; Convert to positive errno
    mov rdi, rax
    mov rax, 60
    syscall
```

---

## 4. Kernel Entry Point

### Linux entry_SYSCALL_64 (Simplified)

```asm
; arch/x86/entry/entry_64.S (conceptual)

entry_SYSCALL_64:
    ; Immediately save user RSP and switch to kernel stack
    swapgs                          ; GS now points to kernel per-CPU data
    mov [gs:user_rsp], rsp          ; Save user stack pointer
    mov rsp, [gs:kernel_stack]      ; Load kernel stack
    
    ; Build pt_regs structure on stack
    push [gs:user_rsp]              ; Save user RSP (for pt_regs)
    push r11                        ; RFLAGS (saved by SYSCALL)
    push rcx                        ; RIP (saved by SYSCALL)
    push rax                        ; orig_rax (syscall number)
    
    ; Save remaining registers
    push rdi
    push rsi
    push rdx
    push rcx                        ; Will be overwritten with RIP
    push rax
    push r8
    push r9
    push r10
    push r11                        ; Will be overwritten with RFLAGS
    push rbx
    push rbp
    push r12
    push r13
    push r14
    push r15
    
    ; Call C handler
    mov rdi, rsp                    ; pt_regs pointer
    call do_syscall_64
    
    ; ... return path via SYSRET or IRET ...
```

### pt_regs Structure

```c
struct pt_regs {
    unsigned long r15, r14, r13, r12;
    unsigned long rbp, rbx;
    unsigned long r11, r10, r9, r8;
    unsigned long rax, rcx, rdx;
    unsigned long rsi, rdi;
    unsigned long orig_rax;     // Syscall number
    unsigned long rip;          // Return address (from RCX)
    unsigned long cs;           // Code segment
    unsigned long eflags;       // Flags (from R11)
    unsigned long rsp;          // User stack
    unsigned long ss;           // Stack segment
};
```

---

## 5. SYSRET Hazards

### The SYSRET Vulnerability

```
SYSRET checks the return RIP is canonical BEFORE restoring CPL.

Attack scenario:
1. Syscall with RCX pointing to non-canonical address
2. Kernel uses SYSRET
3. #GP occurs at CPL=0 but with user-controlled RCX/R11
4. Attacker controls kernel execution

Mitigation: Use IRETQ for returns to non-canonical addresses
```

### Safe Return Logic

```asm
syscall_return:
    ; Check if return RIP is canonical
    mov rcx, [rsp + PT_REGS_RIP]
    shl rcx, 16
    sar rcx, 16
    cmp rcx, [rsp + PT_REGS_RIP]
    jne .use_iret               ; Non-canonical, use IRET
    
    ; Additional checks...
    
.use_sysret:
    ; Restore registers
    pop r15
    pop r14
    ; ... etc ...
    
    mov rsp, [gs:user_rsp]
    swapgs
    sysretq

.use_iret:
    ; Slower but safe path
    ; ... set up IRET frame ...
    iretq
```

---

## 6. System Call Tracing

### ptrace Interface

```c
// Tracing system calls
#include <sys/ptrace.h>

// Stop on syscall entry/exit
ptrace(PTRACE_SYSCALL, pid, NULL, NULL);

// Read syscall number
long syscall = ptrace(PTRACE_PEEKUSER, pid, 
                      offsetof(struct user_regs_struct, orig_rax), NULL);

// Modify return value
ptrace(PTRACE_POKEUSER, pid,
       offsetof(struct user_regs_struct, rax), new_value);
```

### seccomp-bpf

```c
// Filter syscalls with BPF
struct sock_filter filter[] = {
    BPF_STMT(BPF_LD | BPF_W | BPF_ABS, 
             offsetof(struct seccomp_data, nr)),
    BPF_JUMP(BPF_JMP | BPF_JEQ | BPF_K, __NR_write, 0, 1),
    BPF_STMT(BPF_RET | BPF_K, SECCOMP_RET_ALLOW),
    BPF_STMT(BPF_RET | BPF_K, SECCOMP_RET_KILL),
};
```

---

## 7. vDSO: Virtual Dynamic Shared Object

### Concept

```
vDSO maps kernel code into user space for fast syscalls that don't need
true privilege (time, getcpu, etc.)

User process address space:
┌─────────────────────┐
│      Stack          │
├─────────────────────┤
│      ...            │
├─────────────────────┤
│      vDSO           │ ← Kernel-provided, mapped read-only
│  __vdso_gettimeofday│
│  __vdso_clock_gettime│
├─────────────────────┤
│      vvar           │ ← Kernel data (time, etc.)
├─────────────────────┤
│      ...            │
└─────────────────────┘
```

### Using vDSO

```c
// glibc automatically uses vDSO
// Direct usage:
#include <sys/auxv.h>

void *vdso = (void *)getauxval(AT_SYSINFO_EHDR);
// Parse ELF to find __vdso_clock_gettime
```

---

# Key Readings

## Required

| Source | Section | Topic |
|--------|---------|-------|
| AMD Vol 2 | Chapter 6.1 | SYSCALL/SYSRET |
| Linux source | arch/x86/entry/entry_64.S | Entry code |
| Linux source | arch/x86/entry/syscall_64.c | Dispatch |

---

# Exercises & Mini-Projects

## Exercise 1: Raw System Calls

Implement a syscall wrapper in pure assembly without using libc.

## Exercise 2: System Call Table

Dump the first 20 entries of the system call table by examining kernel memory (requires root or kernel module).

## Exercise 3: SYSCALL Timer

Measure the overhead of SYSCALL vs. INT 0x80 vs. vDSO.

## Mini-Project: Simple System Call Tracer

Build a ptrace-based tool that logs all syscalls from a target process with arguments and return values.

---

# Quiz

**Q1**: Why does Linux use R10 for the 4th argument instead of RCX?

**Q2**: What registers are clobbered by the SYSCALL instruction itself?

**Q3**: What is the SYSRET vulnerability and how is it mitigated?

**Q4**: What does SWAPGS do and why is it necessary?

**Q5**: Why might the kernel use IRETQ instead of SYSRET for returning to user space?

---

# Priority Guide

## CRITICAL
- SYSCALL register convention
- MSR configuration for SYSCALL
- Kernel entry/exit sequence
- SYSRET hazards

## IMPORTANT
- pt_regs structure
- vDSO mechanism
- ptrace interface

## OPTIONAL
- Complete entry_64.S analysis
- seccomp implementation
- SYSENTER (32-bit legacy)

---

**Next Week**: Caching and Memory Types—MTRR, PAT, and cache coherency.
