# AMD64 Systems Programming — Week 12
## Capstone Project

**Course**: AMD x86-64 Systems Programming  
**Primary Text**: All previous weeks + AMD64 Architecture Programmer's Manual  
**Week Duration**: ~20-30 hours (extended for project work)  

---

# Overview

This final week is dedicated to a comprehensive capstone project that integrates knowledge from all previous weeks. Choose one of the project options below, or propose your own with instructor approval.

---

# Project Options

## Option A: Minimal Operating System Kernel

### Description
Build a minimal 64-bit kernel that boots via UEFI or Multiboot2, sets up long mode properly, and provides basic functionality.

### Requirements

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    Minimal Kernel Requirements                         │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  Core Components (Required):                                           │
│  □ Boot in 64-bit long mode                                           │
│  □ Set up GDT with code/data/TSS segments                             │
│  □ Set up IDT with exception handlers (#DE, #GP, #PF, #DF)            │
│  □ Initialize 4-level paging with identity + higher-half mapping      │
│  □ Implement basic serial/VGA console output                          │
│  □ Handle timer interrupt (PIT or APIC timer)                         │
│  □ Basic physical memory allocator (bitmap or buddy)                  │
│                                                                         │
│  Extended Features (Choose 2+):                                        │
│  □ Virtual memory manager (page fault handler, demand paging)         │
│  □ Keyboard input via PS/2 or USB (basic)                             │
│  □ Process/thread abstraction with context switching                  │
│  □ System call interface (SYSCALL/SYSRET)                             │
│  □ User-mode execution with privilege separation                      │
│  □ Basic shell or command interpreter                                  │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### Suggested Structure

```
myos/
├── boot/
│   ├── multiboot2.asm      # Multiboot2 header and entry
│   └── boot.asm            # 32→64 bit transition
├── kernel/
│   ├── main.c              # Kernel entry point
│   ├── gdt.c               # GDT setup
│   ├── idt.c               # IDT and exception handlers
│   ├── paging.c            # Page table management
│   ├── pmm.c               # Physical memory manager
│   ├── vmm.c               # Virtual memory manager
│   ├── interrupt.c         # Interrupt handling
│   └── syscall.c           # System call entry
├── drivers/
│   ├── serial.c            # Serial port output
│   ├── vga.c               # VGA text mode
│   └── pit.c               # Programmable Interval Timer
├── lib/
│   ├── string.c            # String functions
│   └── printf.c            # Formatted output
├── include/
│   └── *.h                 # Headers
├── linker.ld               # Linker script
└── Makefile
```

### Evaluation Criteria

| Component | Points | Description |
|-----------|--------|-------------|
| Boots to long mode | 15 | Clean boot sequence |
| GDT/IDT correct | 15 | Proper segment/interrupt setup |
| Paging works | 20 | 4-level tables, higher-half |
| Exception handling | 15 | Graceful fault handling |
| Memory allocator | 15 | Working PMM |
| Extended features | 20 | 2+ additional features |

---

## Option B: Hardware Hypervisor

### Description
Build a Type-2 hypervisor using AMD-V (SVM) that can run a simple guest.

### Requirements

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    Hypervisor Requirements                             │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  Core Components (Required):                                           │
│  □ Detect and enable SVM                                              │
│  □ Allocate and initialize VMCB                                       │
│  □ Set up host state save area                                        │
│  □ Configure intercepts (CPUID, I/O, MSR)                             │
│  □ Implement VMRUN loop                                               │
│  □ Handle basic VM exits (CPUID, HLT, I/O)                           │
│  □ Run real-mode guest (e.g., prints "Hello from guest")             │
│                                                                         │
│  Extended Features (Choose 2+):                                        │
│  □ Nested Page Tables (NPT)                                           │
│  □ MSR bitmap and emulation                                           │
│  □ I/O port emulation (serial port)                                   │
│  □ Interrupt injection                                                 │
│  □ Multiple vCPU support                                              │
│  □ Guest loading from file                                            │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### Suggested Structure

```
myhypervisor/
├── driver/
│   ├── main.c              # Linux kernel module entry
│   ├── svm.c               # SVM enable/disable
│   ├── vmcb.c              # VMCB management
│   ├── vmrun.c             # VM entry/exit loop
│   ├── intercept.c         # Exit handlers
│   └── npt.c               # Nested page tables
├── include/
│   ├── svm.h               # SVM definitions
│   └── vmcb.h              # VMCB structure
├── guest/
│   └── guest.asm           # Simple guest code
├── userspace/
│   └── hvctl.c             # Control utility
├── Kbuild
└── Makefile
```

---

## Option C: Performance Analysis Tool

### Description
Build a comprehensive CPU performance analysis tool using PMU and/or IBS.

### Requirements

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    Performance Tool Requirements                       │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  Core Components (Required):                                           │
│  □ Read/write performance counter MSRs                                │
│  □ Support multiple simultaneous events                               │
│  □ Per-process or system-wide monitoring                              │
│  □ Calculate derived metrics (IPC, miss rates)                        │
│  □ Command-line interface                                             │
│  □ Output in human-readable and CSV formats                           │
│                                                                         │
│  Extended Features (Choose 2+):                                        │
│  □ IBS fetch/op sampling                                              │
│  □ Top-down microarchitecture analysis                                │
│  □ Flame graph generation                                             │
│  □ Real-time dashboard (ncurses or web)                               │
│  □ Comparison mode (before/after)                                     │
│  □ Hardware topology awareness (per-core, per-CCX)                    │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## Option D: SIMD-Optimized Library

### Description
Build a high-performance library using SSE/AVX for a specific domain.

### Suggested Domains

```
1. Image Processing:
   - Convolution filters (blur, sharpen, edge detect)
   - Color space conversion (RGB↔YUV)
   - Image resizing (bilinear/bicubic)

2. Signal Processing:
   - FFT implementation
   - FIR/IIR filters
   - Audio processing (mixing, effects)

3. Linear Algebra:
   - Matrix multiplication (tiled + SIMD)
   - Matrix transposition
   - Dot products, norms

4. Cryptography:
   - AES-NI based encryption
   - SHA using SHA-NI
   - Constant-time operations
```

### Requirements

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    SIMD Library Requirements                           │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  Core Components (Required):                                           │
│  □ Runtime CPU feature detection                                       │
│  □ Scalar fallback implementations                                     │
│  □ SSE4.2 optimized path                                              │
│  □ AVX2 optimized path                                                │
│  □ Proper memory alignment handling                                    │
│  □ Comprehensive benchmarks vs. scalar                                │
│  □ Unit tests for correctness                                         │
│                                                                         │
│  Extended Features (Choose 2+):                                        │
│  □ AVX-512 path (if available)                                        │
│  □ Multi-threaded parallelization                                     │
│  □ NUMA-aware allocation                                              │
│  □ Cache blocking optimization                                         │
│  □ Comparison with reference library (OpenCV, BLAS, etc.)            │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## Option E: Custom Project

Propose your own project that demonstrates mastery of AMD64 systems programming. Must be approved and include:

- Clear objectives and scope
- Connection to at least 4 course topics
- Defined deliverables and evaluation criteria
- Realistic timeline

---

# Project Timeline

```
Week 12 Schedule:
┌─────────────────────────────────────────────────────────────────────────┐
│  Day 1-2:   Project selection and planning                             │
│             - Choose project                                            │
│             - Set up development environment                           │
│             - Create project skeleton                                   │
│             - Define milestones                                         │
│                                                                         │
│  Day 3-5:   Core implementation                                        │
│             - Implement required features                              │
│             - Regular testing                                          │
│             - Document as you go                                       │
│                                                                         │
│  Day 6-7:   Extended features + Polish                                 │
│             - Add extended features                                    │
│             - Performance optimization                                 │
│             - Bug fixes                                                │
│                                                                         │
│  Day 8-9:   Documentation + Presentation                               │
│             - Write README and documentation                           │
│             - Create presentation/demo                                 │
│             - Code cleanup                                             │
│                                                                         │
│  Day 10:    Submission + Demo                                          │
│             - Final submission                                         │
│             - Live demonstration                                       │
│             - Q&A                                                      │
└─────────────────────────────────────────────────────────────────────────┘
```

---

# Deliverables

## Required Submissions

1. **Source Code**
   - Clean, well-organized repository
   - Meaningful commit history
   - Build instructions that work

2. **Documentation**
   - README with project overview
   - Architecture/design document
   - API documentation (if applicable)

3. **Technical Report** (3-5 pages)
   - Problem statement and goals
   - Design decisions and rationale
   - Challenges encountered and solutions
   - Results and performance analysis
   - Future improvements

4. **Presentation/Demo** (10-15 minutes)
   - Live demonstration
   - Technical deep-dive on interesting aspects
   - Q&A

---

# Grading Rubric

| Category | Weight | Criteria |
|----------|--------|----------|
| Functionality | 35% | Core requirements met, features work |
| Code Quality | 20% | Clean, readable, well-structured |
| Technical Depth | 20% | Demonstrates course concepts |
| Documentation | 15% | Clear, complete, professional |
| Presentation | 10% | Clear explanation, good demo |

---

# Resources

## Development Tools

```bash
# Cross-compiler for bare-metal
sudo apt install gcc-multilib nasm

# QEMU for testing kernels/hypervisors
sudo apt install qemu-system-x86

# Debugging
sudo apt install gdb

# Performance tools
sudo apt install linux-tools-common linux-tools-$(uname -r)
```

## Reference Projects

- **OSDev Wiki**: wiki.osdev.org
- **SimpleVisor**: GitHub - ionescu007/SimpleVisor
- **xv6**: MIT's teaching OS
- **Agner Fog's libraries**: SIMD examples

## Getting Help

- Course forum/Discord
- Office hours
- AMD documentation
- OSDev forums

---

# Final Notes

This capstone project is your opportunity to demonstrate everything you've learned. Choose a project that excites you—passion shows in the final product.

**Good luck!**

---

# Course Completion Checklist

After completing this course, you should be able to:

```
□ Explain AMD64 operating modes and transitions
□ Program and debug at the assembly level
□ Understand CPU privilege and protection mechanisms
□ Implement interrupt and exception handlers
□ Work with virtual memory and page tables
□ Use system calls and understand their implementation
□ Optimize code using SIMD instructions
□ Profile and analyze program performance
□ Understand hardware virtualization concepts
□ Apply security features (NX, SMEP, SMAP)
□ Read and apply AMD architecture documentation
```

**Congratulations on completing AMD64 Systems Programming!**
