# AMD64 Systems Programming
### A Comprehensive 12-Week University Course

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)
[![AMD64](https://img.shields.io/badge/Architecture-AMD64-red.svg)](https://www.amd.com)
[![Level](https://img.shields.io/badge/Level-Advanced-blue.svg)](#prerequisites)

---

## Overview

**AMD64 Systems Programming** is a rigorous, university-level curriculum designed to provide deep understanding of x86-64 processor architecture and low-level systems programming. Using the **AMD64 Architecture Programmer's Manual** as the primary text, this course takes students from CPU fundamentals through advanced topics like virtualization and hardware security.

> **Target Audience**: Computer science students, systems programmers, security researchers, OS developers, and anyone seeking mastery of x86-64 internals.

---

## Table of Contents

- [Quick Start](#quick-start)
- [Course Structure](#course-structure)
- [Prerequisites](#prerequisites)
- [Curriculum Overview](#curriculum-overview)
- [Weekly Breakdown](#weekly-breakdown)
- [Learning Outcomes](#learning-outcomes)
- [Required Materials](#required-materials)
- [Development Environment](#development-environment)
- [How to Use This Course](#how-to-use-this-course)
- [Contributing](#contributing)
- [License](#license)
- [Acknowledgments](#acknowledgments)

---

## Quick Start

```bash
# Clone the repository
git clone https://github.com/yourusername/amd64-systems-programming.git
cd amd64-systems-programming

# Start with Week 1
cat week01_cpu_fundamentals.md

# Set up development environment (Ubuntu/Debian)
sudo apt update
sudo apt install nasm gcc gdb qemu-system-x86 build-essential
```

**Estimated Time**: 180-240 hours (15-20 hours/week × 12 weeks)

---

## Course Structure

```
amd64-systems-programming/
│
├── README.md                          # This file
├── week01_cpu_fundamentals.md         # Week 1: Registers, modes, first program
├── week02_memory_management.md        # Week 2: Paging, segmentation, TLB
├── week03_instruction_set.md          # Week 3: Encoding, decoding, performance
├── week04_interrupts_exceptions.md    # Week 4: IDT, handlers, TSS
├── week05_protection_privilege.md     # Week 5: Rings, CPL, SMEP/SMAP
├── week06_system_calls.md             # Week 6: SYSCALL/SYSRET deep dive
├── week07_caching_memory_types.md     # Week 7: MTRR, PAT, cache coherency
├── week08_simd_programming.md         # Week 8: SSE, AVX, AVX-512
├── week09_virtualization.md           # Week 9: AMD-V (SVM)
├── week10_security_features.md        # Week 10: SME, SEV, hardware security
├── week11_performance_analysis.md     # Week 11: PMU, profiling, optimization
└── week12_capstone_project.md         # Week 12: Comprehensive final project
```

Each weekly module contains:

| Section | Description |
|---------|-------------|
| **Lecture Notes** | Detailed explanations with diagrams and code examples |
| **Key Readings** | Specific AMD manual sections with time estimates |
| **Recommended Videos** | Curated video resources from experts |
| **Exercises** | Hands-on practice problems |
| **Mini-Projects** | Larger implementation challenges |
| **Quiz** | Self-assessment questions |
| **Priority Guide** | Critical vs. optional topics |

---

## Prerequisites

### Required Knowledge

| Topic | Proficiency Level | Notes |
|-------|-------------------|-------|
| C Programming | Intermediate | Pointers, structs, memory management |
| Basic Assembly | Beginner | Familiarity with any assembly language |
| Operating Systems | Fundamentals | Processes, memory, files |
| Computer Architecture | Introductory | CPU basics, memory hierarchy |
| Linux Command Line | Comfortable | Navigation, compilation, debugging |

### Recommended Background

- Data structures and algorithms
- Digital logic fundamentals
- Previous exposure to x86 (32-bit) helpful but not required

---

## Curriculum Overview

### Phase 1: Foundations (Weeks 1-4)

Build core understanding of AMD64 architecture.

```
Week 1: CPU Fundamentals
├── Historical context (AMD64 vs IA-64)
├── Operating modes (Long, Legacy, Real)
├── Register architecture (16 GPRs, zero-extension)
├── Memory addressing modes
├── System V ABI calling convention
└── First assembly program

Week 2: Memory Management
├── Virtual memory concepts
├── 4-level page table hierarchy
├── Page table entry format
├── TLB operation
├── Segmentation in 64-bit mode
└── FS/GS for thread-local storage

Week 3: Instruction Set Deep Dive
├── Variable-length encoding
├── REX, VEX, EVEX prefixes
├── ModR/M and SIB bytes
├── Manual instruction decoding
├── Performance characteristics
└── CPUID feature detection

Week 4: Interrupts and Exceptions
├── Exception taxonomy
├── IDT structure (16-byte entries)
├── Stack frame on interrupt entry
├── TSS and IST
├── SYSCALL/SYSRET introduction
└── APIC basics
```

### Phase 2: System Programming (Weeks 5-8)

Dive into OS-level concepts and SIMD.

```
Week 5: Protection and Privilege
├── Ring model (0-3)
├── CPL, DPL, RPL
├── Segment-based protection
├── Page-level protection (U/S, R/W, NX)
├── SMEP, SMAP, Protection Keys
└── Control-flow Enforcement (CET)

Week 6: System Calls
├── SYSCALL architecture
├── MSR configuration
├── Linux syscall ABI
├── Kernel entry/exit sequence
├── SYSRET vulnerability
└── vDSO mechanism

Week 7: Caching and Memory Types
├── Cache hierarchy (L1/L2/L3)
├── Memory types (WB, UC, WC, WT, WP)
├── MTRR programming
├── PAT configuration
├── MESI protocol
└── Cache control instructions

Week 8: SIMD Programming
├── Register evolution (MMX→AVX-512)
├── Feature detection
├── SSE packed operations
├── AVX 3-operand form
├── FMA instructions
└── SSE/AVX transition states
```

### Phase 3: Advanced Topics (Weeks 9-12)

Explore virtualization, security, and performance.

```
Week 9: Virtualization (AMD-V)
├── SVM architecture
├── VMCB structure
├── VMRUN/VMEXIT flow
├── Intercept configuration
├── Nested Page Tables (NPT)
└── Basic hypervisor structure

Week 10: Security Features
├── SME (Secure Memory Encryption)
├── SEV (Secure Encrypted Virtualization)
├── SEV-ES (Encrypted State)
├── SEV-SNP (Integrity Protection)
├── C-bit in page tables
└── Attestation

Week 11: Performance Analysis
├── Performance Monitoring Unit
├── Core performance events
├── IBS (Instruction-Based Sampling)
├── Using perf and AMD μProf
├── RDTSC/RDTSCP
└── Optimization strategies

Week 12: Capstone Project
├── Project options (Kernel, Hypervisor, etc.)
├── Requirements and milestones
├── Deliverables
└── Presentation guidelines
```

---

## Weekly Breakdown

| Week | Topic | Hours | Key Concepts |
|------|-------|-------|--------------|
| 1 | CPU Fundamentals | 15-20 | Registers, modes, ABI, first program |
| 2 | Memory Management | 15-20 | Paging, TLB, segmentation |
| 3 | Instruction Set | 15-20 | Encoding, ModR/M, SIB, performance |
| 4 | Interrupts/Exceptions | 15-20 | IDT, handlers, TSS, IST |
| 5 | Protection/Privilege | 15-20 | Rings, CPL, SMEP/SMAP |
| 6 | System Calls | 15-20 | SYSCALL/SYSRET, Linux ABI |
| 7 | Caching | 15-20 | MTRR, PAT, MESI |
| 8 | SIMD | 15-20 | SSE, AVX, optimization |
| 9 | Virtualization | 15-20 | AMD-V, VMCB, NPT |
| 10 | Security | 15-20 | SME, SEV, SEV-SNP |
| 11 | Performance | 15-20 | PMU, IBS, profiling |
| 12 | Capstone | 20-30 | Comprehensive project |

**Total**: ~180-240 hours

---

## Learning Outcomes

Upon successful completion, students will be able to:

### Architecture Knowledge
- [ ] Explain AMD64 operating modes and privilege levels
- [ ] Describe the complete instruction encoding format
- [ ] Understand 4-level page table translation
- [ ] Explain interrupt and exception handling mechanisms

### Practical Skills
- [ ] Write and debug x86-64 assembly programs
- [ ] Implement interrupt handlers and system calls
- [ ] Program SIMD operations for performance
- [ ] Use hardware performance counters for profiling

### Systems Understanding
- [ ] Implement memory management at the page level
- [ ] Understand kernel/user privilege separation
- [ ] Configure hardware virtualization (AMD-V)
- [ ] Apply CPU security features

### Professional Capabilities
- [ ] Read and apply AMD architecture documentation
- [ ] Analyze system performance bottlenecks
- [ ] Debug low-level system issues
- [ ] Design systems software with security in mind

---

## Required Materials

### Primary Text

**AMD64 Architecture Programmer's Manual**
- Volume 1: Application Programming
- Volume 2: System Programming
- Volume 3: General-Purpose and System Instructions
- Volume 4: 128-Bit and 256-Bit Media Instructions
- Volume 5: 64-Bit Media and x87 Floating-Point Instructions

📥 **Download**: [AMD Developer Resources](https://developer.amd.com/resources/developer-guides-manuals/)

### Supplementary Resources

| Resource | Use |
|----------|-----|
| System V AMD64 ABI | Calling convention reference |
| Intel SDM | Alternative perspective |
| OSDev Wiki | Practical implementation guides |
| Agner Fog's Manuals | Optimization reference |

---

## Development Environment

### Recommended Setup

```bash
# Ubuntu/Debian
sudo apt update
sudo apt install \
    nasm \              # Assembler
    gcc \               # C compiler
    gdb \               # Debugger
    qemu-system-x86 \   # Emulator
    build-essential \   # Build tools
    linux-tools-common  # perf
    
# Optional: Cross-compiler for bare-metal
sudo apt install gcc-multilib

# Verify installation
nasm --version
gcc --version
qemu-system-x86_64 --version
```

### IDE/Editor Recommendations

| Editor | Notes |
|--------|-------|
| VS Code | With x86/x86_64 assembly extensions |
| CLion | Excellent for mixed C/ASM projects |
| Vim/Neovim | With proper syntax highlighting |
| Ghidra | For disassembly analysis |

---

## How to Use This Course

### Self-Study Path

1. **Read** the lecture notes for the week
2. **Study** the referenced AMD manual sections
3. **Watch** recommended videos
4. **Complete** exercises in order
5. **Build** the mini-project
6. **Take** the quiz for self-assessment
7. **Review** any weak areas before proceeding

### Classroom Use

Instructors may:
- Use lecture notes as presentation material
- Assign exercises as homework
- Use quizzes for assessment
- Modify capstone project requirements

### Time Management

```
Suggested Weekly Schedule (15-20 hours):

Day 1-2:  Lecture notes + AMD manual reading (6-8 hours)
Day 3-4:  Videos + initial exercises (4-5 hours)
Day 5-6:  Advanced exercises + mini-project (4-5 hours)
Day 7:    Quiz + review + catch-up (2-3 hours)
```

---

## API Reference

### Key Data Structures

Throughout the course, you'll work with these critical structures:

```c
// Page Table Entry (64-bit)
typedef union {
    uint64_t raw;
    struct {
        uint64_t present       : 1;
        uint64_t writable      : 1;
        uint64_t user          : 1;
        uint64_t write_through : 1;
        uint64_t cache_disable : 1;
        uint64_t accessed      : 1;
        uint64_t dirty         : 1;
        uint64_t page_size     : 1;
        uint64_t global        : 1;
        uint64_t available     : 3;
        uint64_t page_frame    : 40;
        uint64_t reserved      : 11;
        uint64_t no_execute    : 1;
    };
} pte_t;

// IDT Entry (16 bytes in long mode)
typedef struct __attribute__((packed)) {
    uint16_t offset_low;
    uint16_t selector;
    uint8_t  ist;
    uint8_t  type_attr;
    uint16_t offset_mid;
    uint32_t offset_high;
    uint32_t reserved;
} idt_entry_t;

// VMCB Control Area (partial)
typedef struct {
    uint32_t intercept_cr;
    uint32_t intercept_dr;
    uint32_t intercept_exc;
    uint32_t intercept_ctrl1;
    uint32_t intercept_ctrl2;
    // ... additional fields
    uint64_t exitcode;
    uint64_t exitinfo1;
    uint64_t exitinfo2;
} vmcb_control_t;
```

---

## Frequently Asked Questions

<details>
<summary><strong>Do I need AMD hardware?</strong></summary>

No. Most concepts apply to both AMD and Intel x86-64 processors. QEMU can emulate AMD-specific features. However, for virtualization (AMD-V) and security features (SEV), AMD hardware is beneficial.
</details>

<details>
<summary><strong>Is this course suitable for beginners?</strong></summary>

This is an advanced course. You should be comfortable with C programming and have basic familiarity with assembly concepts. Complete a basic systems programming or architecture course first.
</details>

<details>
<summary><strong>How long does the course take?</strong></summary>

Plan for 180-240 hours total (15-20 hours per week for 12 weeks). Adjust based on your background—those with prior x86 experience may move faster.
</details>

<details>
<summary><strong>Can I skip weeks?</strong></summary>

Weeks build on each other. Skipping is not recommended. If you have strong background in a topic, spend less time but still review the material.
</details>

---

## Contributing

Contributions are welcome! Please see [CONTRIBUTING.md](CONTRIBUTING.md) for guidelines.

### Ways to Contribute

- 🐛 Report errors or unclear explanations
- 📝 Suggest additional exercises
- 🔧 Submit fixes or improvements
- 📚 Add supplementary resources
- 🌐 Translations

---

## License

This curriculum is released under the **MIT License**. See [LICENSE](LICENSE) for details.

```
MIT License

Copyright (c) 2025

Permission is hereby granted, free of charge, to any person obtaining a copy
of this software and associated documentation files (the "Software"), to deal
in the Software without restriction, including without limitation the rights
to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
copies of the Software...
```

---

## Acknowledgments

### Resources and Inspiration

- **AMD** for comprehensive public documentation
- **OpenSecurityTraining2** for excellent x86 education
- **OSDev Community** for practical implementation knowledge
- **Agner Fog** for optimization research

### Special Thanks

- The systems programming education community
- Contributors and reviewers
- Students who provided feedback

---

<div align="center">

**[⬆ Back to Top](#amd64-systems-programming)**

Made with dedication for systems programming education.

</div>
