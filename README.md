
# QNX Hypervisor — Course Overview

## Welcome

This is a detailed, in-depth overview of the **QNX Hypervisor**. this course provides a thorough architectural understanding, practical design guidance, and safety considerations for building virtualized embedded systems.

---

## What You Will Learn

### 1. Hypervisor Fundamentals
- What a **hypervisor** is in general and what makes the **QNX Hypervisor** unique
- How the QNX Hypervisor leverages the proven **QNX Neutrino microkernel** architecture, extending it with virtualization capabilities 

### 2. Architectural Deep Dive
- The **core components** of the QNX Hypervisor and how they work together
- How the hypervisor, host environment, and guest virtual machines (VMs) interact
- **Execution model**: How guests run directly on physical CPUs with hardware-assisted virtualization, and how the hypervisor handles traps (guest exits/entrances) via the **Lahav Line** model 

### 3. Integration with Your Software
- What **software components you supply** (your own processes, drivers, applications)
- How your components fit into the hypervisor host and guest environments
- The **QNX OS API** compatibility — develop using familiar POSIX-compliant tools without additional ramp-up time 

### 4. Virtual Machines & Devices
- How VMs are configured and assembled (memory, physical devices, virtual devices)
- **Virtual device library**: VirtIO para-virtualized devices, emulated devices, and pass-through devices
- **Custom virtual device development** using the QNX Virtualization API 

### 5. Design & Tuning Tips
- Best practices for **CPU partitioning**, memory allocation (`ram` vs. `pass` options), and avoiding oversubscription
- **Performance tuning** guidance for embedded workloads
- Design patterns for **mixed-criticality systems** (safety-critical + non-safety guests on the same SoC) 

### 6. Safety & Security
- **Isolation mechanisms**: Memory protection, Access Control Lists (ACLs), Mandatory Access Control, and SMMU-based DMA containment
- **QNX Hypervisor for Safety**: Pre-certified to IEC 61508 SIL 3 and ISO 26262 ASIL D
- Solutions for **freedom from interference** between guests and host
- Implementing **Local Design Safe States (DSS)** in virtualized environments 

---

## Key Concepts Covered

| Concept | Description |
|---------|-------------|
| **Type 1 / Type 2 Flexibility** | QNX Hypervisor scales from a thin separation kernel to a full-featured host OS  |
| **Guest OS Support** | Unmodified Linux, Android, QNX Neutrino RTOS, QNX OS for Safety |
| **Hardware Support** | ARMv8 (AArch64) and x86-64 SoCs with virtualization extensions |
| **Virtual CPUs (vCPUs)** | Implemented as threads within QNX processes; supports SMP and BMP pinning |
| **Memory Sharing** | Guest-to-guest and guest-to-host shared memory regions |
| **Networking** | Guest-to-guest, guest-to-host, and guest-to-world connectivity |
| **Device Hand-off** | Transferring pass-through devices from host to guest at runtime  |

---

## Course Structure

This course goes beyond basic overviews. It has evolved over years to cover:

1. **Architecture** — Static and dynamic views of the hypervisor system
2. **Building Systems** — Assembling bootable images, configuring VMs, using BSPs
3. **Configuration** — VM config files, FDTs/ACPI, device assignment
4. **Virtual Devices** — Using built-in vdevs and developing custom ones
5. **Monitoring & Debugging** — GDB, hypervisor trace events, timeline analysis
6. **Safety & Certification** — Designing for ISO 26262, IEC 61508, and mixed-criticality

---

## Prerequisites

- Familiarity with **QNX Neutrino RTOS** concepts (microkernel, resource managers, message passing)
- Understanding of **embedded system development** and board support packages (BSPs)
- Basic knowledge of **virtualization concepts** (helpful but not required)
- Access to **QNX Software Development Platform (SDP)** and QNX Momentics IDE

---

*This course is designed for system architects, embedded developers, and safety engineers building next-generation automotive, medical, industrial, and aerospace systems.*


