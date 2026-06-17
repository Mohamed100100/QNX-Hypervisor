
# QNX Hypervisor — Architecture Deep Dive

## Overview

This section covers the fundamental architecture of the QNX Hypervisor: how it runs multiple operating systems on a single SoC, the difference between hypervisors and emulators, the role of the `qvm` process, memory virtualization with two-stage address translation, and how to configure guests via `.qvmconf` files.

---

## 1. What is the QNX Hypervisor?

### Definition
The QNX Hypervisor is **software that runs multiple operating systems on the same SoC (System on Chip)** — the same chip, the same board.

**Example:** QNX, Linux, and Android all running simultaneously on one board.

**Why use it?** Consolidation. Instead of multiple boards in a vehicle (e.g., one for infotainment, one for cluster, one for ADAS), you run them all on a single SoC, reducing hardware cost.

---

## 2. Hypervisor vs. Emulator

| | **Hypervisor** | **Emulator** |
|---|---|---|
| **Instruction Execution** | Most instructions run **directly on CPU** (bare metal speed) | Every instruction is **emulated** in software |
| **Performance** | Fast — near-native speed | Slow — 10x to 1000x slower |
| **Hardware Requirement** | Guest OS architecture **must match** host CPU (ARM guest → ARM CPU) | Can run ARM OS on x86 hardware |
| **Example** | QNX Hypervisor, VMware ESXi, KVM | QEMU (in emulation mode) |

**Key Insight:** With a hypervisor, only **privileged instructions** (e.g., accessing protected memory, certain system calls) trigger a "guest exit" — the CPU traps to the hypervisor, which handles it and returns. The overhead is negligible.

---

## 3. QNX Hypervisor Architecture

### Core Principle: "Just Normal QNX"

The QNX Hypervisor is **not a separate, exotic piece of software**. It is built on the proven **QNX Neutrino RTOS** microkernel architecture:

- **procnto** — The QNX process manager + microkernel (memory management, thread scheduling, IPC, interrupts)
- **Host processes** — Normal QNX drivers (`io-sock`, `devb-*`, `devc-ser*`, etc.)
- **qvm processes** — One `qvm` (QNX Virtual Machine) process **per guest**

> **"The hypervisor line is just normal QNX, plain old QNX."**

Each `qvm` is a **normal QNX process**. The host (normal QNX + all `qvm` processes) manages the guests.

---

## 4. Supported Guests & Hardware

### Guest Operating Systems
| Guest | Notes |
|-------|-------|
| **QNX 8** | Native RTOS guest |
| **Linux** | Ubuntu 20.04, Yocto-built kernels |
| **Android** | From hypervisor's view, Android **is** Linux (same kernel). From customer view, different software stack |
| **QNX OS for Safety** | For safety-certified applications |

### Hardware Platforms
- **x86-64** (Intel/AMD with VT-x)
- **ARMv8** using **AArch64** (with virtualization extensions)

> **64-bit OSes only.** No 32-bit support.

---

## 5. Starting Guests: The `qvm` Process

### Live Demonstration Pattern (Raspberry Pi Example)

```bash
# Terminal 1: Serial connection to host QNX
# (Boot the board, see normal QNX running)

# Terminal 2: SSH into host, start QNX guest
ssh root@<<pi-ip>
cd /data/guests/qnx
qvm qnx-rpi.qvmconf          # Boots QNX guest in this terminal

# Terminal 3: SSH into host, start Linux guest
ssh root@<<pi-ip>
cd /data/guests/linux
qvm linux-rpi.qvmconf          # Boots Linux guest in this terminal
```

**Result:** On the host, `pidin` shows:
- `procnto` (host microkernel)
- `io-sock`, `devb-*` (host drivers)
- `qvm` (for QNX guest)
- `qvm` (for Linux guest)

Each guest runs in its own `qvm` process, isolated from each other.

---

## 6. Guest Configuration: The `.qvmconf` File

QNX does not use a GUI wizard to create VMs. You **describe the VM in a text configuration file** with the `.qvmconf` extension.

### Configuration File Structure

```qvmconf
# ============================================
# QNX Guest Configuration Example
# ============================================

system name=qnx-guest              # Name of the VM

# Memory allocation
ram addr=0x40000000,size=0x8000000   # 128 MB at Guest Physical Address 0x40000000

# Virtual CPUs
cpu cluster=0,cores=2              # 2 vCPUs for this guest

# Boot image (normal QNX boot image, created with mkifs)
load addr=0x40000000,file=/data/guests/qnx/qnx-boot.img

# Virtual Devices
vdev pl011,addr=0x9000000          # Virtual ARM PL011 serial port
vdev gicd,addr=0x8000000           # Virtual GIC interrupt controller
vdev generic_timer                 # Virtual ARM timer

# Pass-through: Map host device memory directly into guest
pass addr=0xFE000000,host=0x3F000000,size=0x100000   # Video memory bypass
```

### Linux Guest Configuration

```qvmconf
# ============================================
# Linux Guest Configuration Example
# ============================================

system name=linux-guest

# Memory (Linux needs more than QNX)
ram addr=0x80000000,size=0x19000000   # 400 MB at GPA 0x80000000

cpu cluster=0,cores=2

# Boot images (built with Yocto)
load addr=0x80000000,file=/data/guests/linux/Image
load addr=0x84000000,file=/data/guests/linux/initrd.img

# Linux kernel command line
cmdline "console=ttyAMA0 root=/dev/ram0 rw"

# Virtual Devices
vdev pl011,addr=0x9000000          # Serial console
vdev virtio_net                    # Para-virtualized network
vdev virtio_blk,file=/data/guests/linux/rootfs.ext4   # Virtual disk
```

---

## 7. Memory Architecture: Two-Stage Address Translation

This is the **most critical concept** in hypervisor architecture. The QNX Hypervisor uses the MMU (Memory Management Unit) with **two layers of page tables** to isolate guests while allowing near-native performance.

### The Three Address Spaces

| Layer | Name | Set Up By | Description |
|-------|------|-----------|-------------|
| **VA** | Virtual Address | Guest OS | Process address space within the guest |
| **GPA** | Guest Physical Address | `qvm` (hypervisor) | What the guest OS thinks is "physical" memory |
| **HPA** | Host Physical Address | Hardware / BIOS | Actual DRAM addresses on the board |

### Two-Stage Translation

```
┌─────────────────────────────────────┐
│  Guest Process Virtual Address (VA)  │
│  (e.g., 0x00010000)                 │
└──────────────┬──────────────────────┘
               │ Stage 1: Guest OS Page Tables
               ▼
┌─────────────────────────────────────┐
│  Guest Physical Address (GPA)        │
│  (e.g., 0x40000000)                 │
│  ← Guest thinks this is "physical"  │
└──────────────┬──────────────────────┘
               │ Stage 2: qvm / Hypervisor Page Tables
               │ (ARM: Stage 2 tables, x86: Extended Page Tables)
               ▼
┌─────────────────────────────────────┐
│  Host Physical Address (HPA)         │
│  (e.g., 0x10000000)                  │
│  ← Actual DRAM on the board          │
└─────────────────────────────────────┘
```

### How It Works

1. **qvm reads the `.qvmconf`** `ram` option
2. **qvm asks procnto** (host process manager) to allocate memory
3. **procnto returns host physical memory** (may be fragmented — doesn't matter)
4. **qvm programs the MMU** with Stage 2 page tables mapping GPA → HPA
5. **Guest OS boots** and creates its own Stage 1 page tables (VA → GPA)
6. **At runtime:** The CPU's MMU handles both translations in hardware — **most instructions run at full speed**

---

## 8. Memory Configuration Options

### `ram` — Allocated Guest Memory
```qvmconf
ram addr=0x40000000,size=0x8000000   # 128 MB starting at GPA 0x40000000
```
- Memory is **zero-filled** by the host process manager
- Takes time to zero-fill (e.g., 128 MB), but uses fast CPU instructions
- Guest sees this as **contiguous** physical memory

### `pass` — Pass-Through / Bypass Mapping
```qvmconf
pass addr=0xFE000000,host=0x3F000000,size=0x100000
```
- **No zero-filling** — faster boot
- Creates a **direct mapping** from GPA to actual host device memory
- Used for: **video memory**, device registers, DMA buffers
- Guest accesses hardware **directly**, bypassing hypervisor overhead

### Typed Memory + `pass` Trick
You can pre-allocate **typed memory** in the QNX host and map it via `pass` to get RAM that is **not zero-filled**:
```qvmconf
# In host: set up typed memory region
# In .qvmconf: map it directly
pass addr=0x50000000,host=0xA0000000,size=0x10000000   # 256 MB pre-allocated RAM
```

### Unity Guest (1:1 Mapping)
```qvmconf
# GPA == HPA (one-to-one mapping)
ram addr=0x0,size=0x40000000          # Guest physical = Host physical
```
- **MMU still involved** (page tables programmed), but identity mapping
- Used when **no IOMMU** is available and you need device DMA safety
- **Complex to configure** — consult QNX engineering

---

## 9. Key Terminology: x86 vs. ARM

| Concept | x86 Terminology | ARM Terminology |
|---------|----------------|-----------------|
| Guest's "physical" address | **Guest Physical Address (GPA)** | **Intermediate Physical Address (IPA)** |
| Host's physical address | **Host Physical Address (HPA)** | **Physical Address (PA)** |

> QNX documentation uses **x86 terminology** (GPA/HPA) because it is clearer and more explicit.

---

## 10. CPU Virtualization

```qvmconf
cpu cluster=0,cores=2
```
- You may have **8 physical CPU cores** on the SoC
- This guest will see **only 2 vCPUs**
- vCPUs are implemented as **threads within the qvm process**
- Supports **SMP** (Symmetric Multi-Processing) and **BMP** (Bound Multi-Processing) for pinning

---

## 11. Virtual Devices (`vdev`)

Guests need hardware devices: timers, interrupt controllers, serial ports, network, storage. The hypervisor provides **virtual devices**:

| Device Type | Example | Purpose |
|-------------|---------|---------|
| **Emulated** | `pl011`, `gicd` | ARM PL011 UART, GIC interrupt controller |
| **Para-virtualized** | `virtio_net`, `virtio_blk` | High-performance VirtIO devices (guest knows it's virtualized) |
| **Pass-through** | `pass` (memory) | Direct hardware access |

---
## 12. Course Screenshots
The following screenshots from the video course illustrate the key concepts covered in this architecture module.

![Screenshot 1](resources/screenshot_01.png)

![Screenshot 2](resources/screenshot_02.png)

![Screenshot 3](resources/screenshot_03.png)

![Screenshot 4](resources/screenshot_04.png)

![Screenshot 5](resources/screenshot_05.png)

![Screenshot 6](resources/screenshot_06.png)

![Screenshot 7](resources/screenshot_07.png)

![Screenshot 8](resources/screenshot_08.png)

## 13. Summary of Key Architectural Principles

1. **Type 1 Hypervisor** — Runs directly on hardware (bare metal), not on top of another OS
2. **Most instructions execute directly on CPU** — No emulation overhead for normal code
3. **Guest exits only for privileged operations** — MMU, interrupts, certain instructions
4. **One `qvm` process per guest** — Normal QNX process, managed by procnto
5. **Two-stage MMU translation** — Stage 1 (guest OS) + Stage 2 (hypervisor/qvm)
6. **Memory isolation via MMU** — Guests cannot access outside their GPA space
7. **Pass-through for performance** — Direct device memory mapping bypasses hypervisor

