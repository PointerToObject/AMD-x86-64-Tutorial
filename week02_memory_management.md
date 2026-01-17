# AMD64 Systems Programming — Week 2
## Memory Management: Paging, Segments, and the TLB

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

## 1. Virtual Memory: The Grand Illusion

### Why Virtual Memory Exists

Every process believes it has exclusive access to a contiguous address space starting at 0. This is a lie—a beautiful, necessary lie maintained by the CPU and OS working together.

**Problems virtual memory solves:**
- **Isolation**: Process A cannot access Process B's memory
- **Abstraction**: Programs don't need to know physical RAM layout
- **Overcommitment**: Total virtual memory can exceed physical RAM
- **Shared memory**: Multiple processes can map the same physical page
- **Memory-mapped I/O**: Hardware registers appear as memory addresses

```
Process A's View          Physical Memory         Process B's View
┌─────────────────┐      ┌─────────────────┐     ┌─────────────────┐
│ 0x00000000      │      │                 │     │ 0x00000000      │
│    Code         │─────►│  Frame 0x1000   │◄────│    Code         │
├─────────────────┤      ├─────────────────┤     ├─────────────────┤
│    Heap         │─────►│  Frame 0x2000   │     │    Heap         │─────►│Frame 0x5000│
├─────────────────┤      ├─────────────────┤     ├─────────────────┤
│    Stack        │─────►│  Frame 0x3000   │     │    Stack        │─────►│Frame 0x6000│
└─────────────────┘      └─────────────────┘     └─────────────────┘
        │                        ▲                       │
        └────────────────────────┴───────────────────────┘
                    Page Tables (per process)
```

> **AMD Manual Reference**: Volume 2, Chapter 5 "Page Translation and Protection"

---

## 2. Segmentation: The Legacy Layer

### Segments in Long Mode

Segmentation was the original x86 memory protection mechanism. In 64-bit long mode, segmentation is largely disabled—but not entirely.

**What's ignored in 64-bit mode:**
- Base address for CS, DS, ES, SS (treated as 0)
- Segment limits (treated as infinite)

**What still works:**
- FS and GS base addresses (used for thread-local storage)
- Segment privilege levels (DPL/RPL/CPL)
- Code segment attributes (L bit for long mode)

```
┌────────────────────────────────────────────────────────────────────────┐
│                    Segment Selector (16 bits)                          │
├────────────────────────────────────────────────────────────────────────┤
│  15                              3    2      1    0                    │
│  ├──────────────────────────────┼────┼──────────┤                     │
│  │         Index (13 bits)      │ TI │   RPL   │                      │
│  │     (GDT/LDT entry number)   │    │ (Ring)  │                      │
│  └──────────────────────────────┴────┴─────────┘                      │
│                                                                        │
│  TI = 0: Use GDT (Global Descriptor Table)                            │
│  TI = 1: Use LDT (Local Descriptor Table)                             │
│  RPL = Requested Privilege Level (0-3)                                │
└────────────────────────────────────────────────────────────────────────┘
```

### The GDT in Long Mode

Even in 64-bit mode, you need a GDT with at least:
- Null descriptor (index 0)
- 64-bit code segment (L=1, D=0)
- Data segment
- TSS descriptor (for interrupt handling)

```asm
; Minimal GDT for 64-bit mode
gdt64:
    dq 0                        ; Null descriptor
.code: equ $ - gdt64
    dq 0x00209A0000000000       ; 64-bit code: Execute/Read
.data: equ $ - gdt64
    dq 0x0000920000000000       ; Data: Read/Write
.end:

gdt64_ptr:
    dw gdt64.end - gdt64 - 1    ; Limit
    dq gdt64                     ; Base
```

> **AMD Manual Reference**: Volume 2, Chapter 4 "Segmented Virtual Memory"

### FS and GS: Thread-Local Storage

FS and GS bases are set via MSRs, not the GDT:

```
MSR Address    Name              Purpose
0xC0000100     FS_BASE           FS segment base (user mode TLS)
0xC0000101     GS_BASE           GS segment base (kernel mode)
0xC0000102     KERNEL_GS_BASE    Swapped with GS_BASE on SWAPGS
```

Linux convention:
- **FS**: User-space thread-local storage (glibc's `__thread` variables)
- **GS**: Kernel per-CPU data (after SWAPGS on syscall entry)

---

## 3. Paging: The Real Memory Virtualizer

### Page Translation Overview

AMD64 uses a 4-level page table hierarchy (5-level with LA57 extension):

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    64-bit Virtual Address                               │
├─────────────────────────────────────────────────────────────────────────┤
│ 63    48 │ 47    39 │ 38    30 │ 29    21 │ 20    12 │ 11          0   │
├──────────┼──────────┼──────────┼──────────┼──────────┼─────────────────┤
│  Sign    │  PML4    │   PDPT   │    PD    │    PT    │     Offset      │
│Extension │  Index   │  Index   │  Index   │  Index   │   (4KB page)    │
│ (16 bits)│ (9 bits) │ (9 bits) │ (9 bits) │ (9 bits) │   (12 bits)     │
└──────────┴──────────┴──────────┴──────────┴──────────┴─────────────────┘
     │          │          │          │          │            │
     │          │          │          │          │            └──► Byte in page
     │          │          │          │          └──────────────► 512 entries
     │          │          │          └─────────────────────────► 512 entries
     │          │          └────────────────────────────────────► 512 entries
     │          └───────────────────────────────────────────────► 512 entries
     └──────────────────────────────────────────────────────────► Must match bit 47
```

### Translation Walkthrough

```
CR3 (Page Map Level 4 Base)
  │
  ▼
┌─────────────┐    PML4E                    ┌─────────────┐
│   PML4[0]   │───────────────────────────►│   PDPT[0]   │
│   PML4[1]   │                            │   PDPT[1]   │
│    ...      │                            │    ...      │
│  PML4[511]  │                            │  PDPT[511]  │
└─────────────┘                            └─────────────┘
                                                  │
                   PDPTE                          ▼
              ┌─────────────┐              ┌─────────────┐
              │    PD[0]    │◄─────────────│    PT[0]    │
              │    PD[1]    │              │    PT[1]    │
              │    ...      │              │    ...      │
              │   PD[511]   │              │   PT[511]   │
              └─────────────┘              └─────────────┘
                                                  │
                                                  ▼
                                           Physical Page
                                          (4KB aligned)
```

### Page Table Entry Format (4KB Pages)

```
┌────────────────────────────────────────────────────────────────────────┐
│  63  │ 62   52 │ 51        M │ M-1      12 │ 11  9 │ 8  │ 7  │ 6  │   │
├──────┼─────────┼─────────────┼─────────────┼───────┼────┼────┼────┼───┤
│  XD  │ Available│   Reserved │Physical Addr│ Avail │ G  │PAT │ D  │...│
│(NX)  │  (11b)  │   (must be 0)│  Bits M:12 │ (3b)  │    │    │    │   │
└──────┴─────────┴─────────────┴─────────────┴───────┴────┴────┴────┴───┘

│ 5  │ 4  │ 3   │ 2   │ 1   │ 0   │
├────┼────┼─────┼─────┼─────┼─────┤
│ A  │PCD │ PWT │ U/S │ R/W │  P  │
└────┴────┴─────┴─────┴─────┴─────┘

Bit  Name        Description
───────────────────────────────────────────────────────────
 0   P           Present (1 = mapped, 0 = not present → #PF)
 1   R/W         Read/Write (1 = writable, 0 = read-only)
 2   U/S         User/Supervisor (1 = user accessible, 0 = kernel only)
 3   PWT         Page Write-Through
 4   PCD         Page Cache Disable
 5   A           Accessed (set by CPU on read)
 6   D           Dirty (set by CPU on write)
 7   PAT/PS      Page Attribute Table / Page Size (for huge pages)
 8   G           Global (don't flush from TLB on CR3 write)
63   XD/NX       Execute Disable (1 = no execute, requires EFER.NXE=1)
```

> **AMD Manual Reference**: Volume 2, Section 5.3 "Long-Mode Page Translation"

### Huge Pages (2MB and 1GB)

For performance-critical applications, larger pages reduce TLB pressure:

**2MB Pages**: Set PS bit in Page Directory Entry
```
Virtual Address for 2MB page:
│ 47:39 │ 38:30 │ 29:21 │ 20:0        │
│ PML4  │ PDPT  │  PD   │ Page Offset │
                    ▲
                    └── PS=1 stops translation here
```

**1GB Pages**: Set PS bit in PDPT Entry
```
Virtual Address for 1GB page:
│ 47:39 │ 38:30 │ 29:0           │
│ PML4  │ PDPT  │ Page Offset    │
             ▲
             └── PS=1 stops here
```

---

## 4. The TLB: Translation Lookaside Buffer

### Why TLBs Are Critical

Without TLB: Every memory access requires 4 additional memory accesses (page table walk).
With TLB: Virtual-to-physical translation in 1 cycle (on hit).

```
┌─────────────────────────────────────────────────────────────────────────┐
│                         TLB Operation                                   │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│   Virtual Address ──►┌─────────┐                                       │
│                      │   TLB   │──► HIT ──► Physical Address (fast!)   │
│                      │ Lookup  │                                        │
│                      └────┬────┘                                        │
│                           │                                             │
│                          MISS                                           │
│                           │                                             │
│                           ▼                                             │
│                    Page Table Walk                                      │
│                    (4 memory reads)                                     │
│                           │                                             │
│                           ▼                                             │
│                    Fill TLB Entry                                       │
│                           │                                             │
│                           ▼                                             │
│                    Physical Address                                     │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### TLB Structure (Typical AMD)

```
┌─────────────────────────────────────────────────────────────────────────┐
│                      AMD Zen TLB Hierarchy (Typical)                    │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  L1 ITLB (Instructions):     64 entries, 4KB pages, fully associative  │
│  L1 DTLB (Data):             64 entries, 4KB pages, fully associative  │
│                                                                         │
│  L2 TLB (Unified):           512-2048 entries, 4KB/2MB pages           │
│                                                                         │
│  Separate entries for 1GB pages (smaller count)                        │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### TLB Invalidation

TLB entries must be invalidated when page tables change:

```asm
; Flush entire TLB (write to CR3)
mov rax, cr3
mov cr3, rax

; Flush single page (INVLPG)
invlpg [address]

; Flush by PCID (Process Context ID) - if supported
; INVPCID instruction with various types
```

**PCID (Process Context Identifier)**: 12-bit identifier in CR3 that allows TLB entries from different processes to coexist. Avoids full TLB flush on context switch.

> **AMD Manual Reference**: Volume 2, Section 5.5 "Translation-Lookaside Buffer (TLB)"

---

## 5. Page Faults: When Translation Fails

### Page Fault Error Code

When a page fault (#PF, vector 14) occurs, CPU pushes an error code:

```
┌────────────────────────────────────────────────────────────────────────┐
│                    Page Fault Error Code                               │
├────────────────────────────────────────────────────────────────────────┤
│  31                              5     4     3     2     1     0      │
│  ├──────────────────────────────┼─────┼─────┼─────┼─────┼─────┤       │
│  │         Reserved             │ I/D │RSVD │ U/S │ W/R │  P  │       │
│  └──────────────────────────────┴─────┴─────┴─────┴─────┴─────┘       │
│                                                                        │
│  P   (bit 0): 0 = Non-present page, 1 = Protection violation          │
│  W/R (bit 1): 0 = Read access, 1 = Write access                       │
│  U/S (bit 2): 0 = Supervisor mode, 1 = User mode                      │
│  RSVD(bit 3): 1 = Reserved bit set in page structure                  │
│  I/D (bit 4): 1 = Instruction fetch (requires NX support)             │
│                                                                        │
│  CR2 contains the faulting virtual address                            │
└────────────────────────────────────────────────────────────────────────┘
```

### Common Page Fault Scenarios

| Scenario | Error Code | Handler Action |
|----------|------------|----------------|
| Demand paging | P=0 | Allocate frame, map page, resume |
| Copy-on-Write | P=1, W=1 | Copy page, make writable, resume |
| Stack growth | P=0, addr near stack | Extend stack mapping |
| NULL pointer | P=0, addr≈0 | Kill process (SIGSEGV) |
| NX violation | P=1, I/D=1 | Kill process (security) |

---

## 6. CR3: The Page Table Root

### CR3 Format (Long Mode)

```
┌────────────────────────────────────────────────────────────────────────┐
│                         CR3 Register                                   │
├────────────────────────────────────────────────────────────────────────┤
│  63                              12  11        5  4    3   2   0      │
│  ├──────────────────────────────────┼───────────┼────┼───┼─────┤      │
│  │      PML4 Physical Base          │  Reserved │PCD │PWT│PCID │      │
│  │      (bits 51:12)                │           │    │   │     │      │
│  └──────────────────────────────────┴───────────┴────┴───┴─────┘      │
│                                                                        │
│  PML4 Base: Physical address of PML4 table (4KB aligned)              │
│  PCD/PWT: Caching attributes for page table accesses                  │
│  PCID: Process Context ID (if CR4.PCIDE=1)                            │
└────────────────────────────────────────────────────────────────────────┘
```

### Context Switch and CR3

On process switch, the OS must:
1. Save current process state
2. Load new CR3 (switches address space)
3. TLB entries for old PCID may remain (if PCID enabled)
4. Load new process state
5. Return to new process

```c
// Simplified context switch (conceptual)
void switch_to(struct task *next) {
    // Save current registers to current->context
    
    // Switch page tables
    write_cr3(next->mm->pgd_phys | next->mm->pcid);
    
    // Load next registers from next->context
}
```

---

## 7. Memory Protection: NX, SMEP, SMAP

### NX (No-Execute) Bit

Prevents code execution from data pages—critical for security.

Enable: `EFER.NXE = 1`
Use: Set bit 63 in page table entries for data pages

```
Stack, heap, data sections → XD=1 (no execute)
Code sections → XD=0 (executable)
```

### SMEP (Supervisor Mode Execution Prevention)

Prevents kernel from executing user-space code (blocks ret2usr attacks).

Enable: `CR4.SMEP = 1`
Effect: #PF if CPL=0 tries to fetch from U/S=1 page

### SMAP (Supervisor Mode Access Prevention)

Prevents kernel from accessing user-space data (except when explicitly allowed).

Enable: `CR4.SMAP = 1`
Override: RFLAGS.AC=1 allows supervisor access to user pages
Instructions: `STAC` (set AC), `CLAC` (clear AC)

```asm
; Kernel accessing user buffer
stac                    ; Enable user-page access
mov rax, [user_ptr]     ; Access user memory
clac                    ; Disable user-page access
```

> **AMD Manual Reference**: Volume 2, Section 5.6 "Page-Protection Checks"

---

## 8. Practical: Examining Page Tables

### Reading CR3 and Walking Tables (Linux /proc)

```bash
# Get CR3 for a process
sudo cat /proc/[pid]/pagemap

# Better: Use crash or custom kernel module
```

### Manual Page Table Walk (Conceptual)

```c
// Pseudocode for 4-level page walk
uint64_t translate(uint64_t cr3, uint64_t vaddr) {
    uint64_t pml4_base = cr3 & ~0xFFF;
    uint64_t pml4_idx = (vaddr >> 39) & 0x1FF;
    uint64_t pdpt_idx = (vaddr >> 30) & 0x1FF;
    uint64_t pd_idx   = (vaddr >> 21) & 0x1FF;
    uint64_t pt_idx   = (vaddr >> 12) & 0x1FF;
    uint64_t offset   = vaddr & 0xFFF;
    
    uint64_t *pml4 = (uint64_t *)pml4_base;
    if (!(pml4[pml4_idx] & 1)) return -1;  // Not present
    
    uint64_t *pdpt = (uint64_t *)(pml4[pml4_idx] & ~0xFFF);
    if (!(pdpt[pdpt_idx] & 1)) return -1;
    if (pdpt[pdpt_idx] & 0x80) {  // 1GB page
        return (pdpt[pdpt_idx] & ~0x3FFFFFFF) | (vaddr & 0x3FFFFFFF);
    }
    
    uint64_t *pd = (uint64_t *)(pdpt[pdpt_idx] & ~0xFFF);
    if (!(pd[pd_idx] & 1)) return -1;
    if (pd[pd_idx] & 0x80) {  // 2MB page
        return (pd[pd_idx] & ~0x1FFFFF) | (vaddr & 0x1FFFFF);
    }
    
    uint64_t *pt = (uint64_t *)(pd[pd_idx] & ~0xFFF);
    if (!(pt[pt_idx] & 1)) return -1;
    
    return (pt[pt_idx] & ~0xFFF) | offset;
}
```

---

# Key Readings

## Required

### AMD64 Architecture Programmer's Manual Volume 2

| Section | Topic | Time |
|---------|-------|------|
| Chapter 4 (all) | Segmented Virtual Memory | 2 hours |
| Chapter 5, Sections 5.1-5.4 | Page Translation | 3 hours |
| Chapter 5, Section 5.5 | TLB | 1 hour |
| Chapter 5, Section 5.6 | Page Protection | 1.5 hours |
| Chapter 3, Section 3.1.1-3.1.5 | CR0-CR4 registers | 1 hour |

## Supplementary

- **Intel SDM Volume 3, Chapter 4**: Paging (different perspective, same architecture)
- **Linux kernel source**: `arch/x86/mm/` directory
- **OSDev Wiki**: "Paging" and "Setting Up Paging"

---

# Recommended Videos

### 1. OpenSecurityTraining2: Architecture 2001 - x86-64 OS Internals
**Instructor**: Xeno Kovah

| Section | Topic | Priority |
|---------|-------|----------|
| "Physical vs Virtual Memory" | Concepts | **Critical** |
| "Page Tables" | 4-level paging | **Critical** |
| "Segmentation" | GDT, segments | High |
| "TLB" | Translation caching | High |

### 2. LiveOverflow: "How Do Page Tables Work?"
**Platform**: YouTube  
**Length**: ~15 minutes  
Good visual explanation of page table walks.

### 3. "What Every Programmer Should Know About Memory" - Ulrich Drepper
**Format**: Paper (PDF), often discussed in video form  
Sections 2-4 cover virtual memory and TLB in depth.

---

# Exercises & Mini-Projects

## Exercise 1: Page Table Mathematics

Calculate by hand:

1. Virtual address `0x00007FFF12345678`:
   - PML4 index = ?
   - PDPT index = ?
   - PD index = ?
   - PT index = ?
   - Page offset = ?

2. How many 4KB pages can be addressed with:
   - One PML4 entry?
   - One PDPT entry?
   - One PD entry?

3. How much physical memory do page tables consume for a process with 1GB of mapped virtual memory (assume 4KB pages, no huge pages)?

## Exercise 2: /proc/[pid]/maps Analysis

```bash
cat /proc/self/maps
```

For each region, identify:
- Address range
- Permissions (r/w/x)
- What it maps (code, heap, stack, libraries)
- Which would have XD bit set?

## Exercise 3: Page Fault Handler Pseudocode

Write pseudocode for a page fault handler that implements:
1. Demand paging (lazy allocation)
2. Copy-on-write for fork()
3. Stack growth detection

## Exercise 4: Build Page Tables (Mini-Project)

Write a program (can be in C or assembly) that:
1. Allocates aligned memory for page tables
2. Creates a minimal identity mapping for first 2MB
3. Creates a higher-half mapping (e.g., 0xFFFF800000000000 → physical 0)
4. Outputs the table entries in hex

This is conceptual—you won't load it into CR3 (would crash user space).

## Exercise 5: TLB Shootdown Simulation

Research and document:
1. What is a TLB shootdown?
2. When is it necessary?
3. How does Linux implement it (IPI)?
4. What is the performance impact?

---

# Quiz

**Q1**: What are the four levels of page tables in AMD64 long mode? What does each level's index come from in the virtual address?

**Q2**: A page table entry has these bits set: P=1, R/W=0, U/S=1, XD=1. Describe what access is allowed to this page from user mode.

**Q3**: Why are FS and GS segment bases still functional in 64-bit mode when other segment bases are ignored?

**Q4**: Calculate the physical address if:
- CR3 = 0x00000000001F3000
- Virtual address = 0x00007FFF00001234
- All present bits are set
- PML4[255] = 0x00000000002F4003
- PDPT[510] = 0x00000000003F5003
- PD[0] = 0x00000000004F6003
- PT[1] = 0x00000000ABCDE003

**Q5**: What is the difference between INVLPG and reloading CR3 for TLB invalidation?

**Q6**: A page fault occurs with error code 0x17. What can you determine about the fault?

**Q7**: Explain why SMEP prevents ret2usr attacks but SMAP is also needed.

**Q8**: How does PCID improve context switch performance?

---

# Priority Guide

## CRITICAL
- 4-level page table structure
- Page table entry format and bits (P, R/W, U/S, XD)
- CR3 purpose and format
- TLB concept and invalidation

## IMPORTANT
- Huge pages (2MB, 1GB)
- Page fault error code interpretation
- NX/XD bit security implications
- SMEP and SMAP

## OPTIONAL (This Week)
- Full GDT structure in long mode
- PCID details
- 5-level paging (LA57)
- Memory type attributes (PAT, MTRR)

---

# Summary

This week you learned how AMD64 creates the illusion of isolated, contiguous address spaces through paging. The key insight: **every memory access involves either a TLB hit or a 4-level page table walk**. Understanding this hierarchy is essential for performance optimization, security research, and OS development.

**Next Week**: Instruction Set Deep Dive—encoding, decoding, and performance characteristics of the AMD64 instruction set.
