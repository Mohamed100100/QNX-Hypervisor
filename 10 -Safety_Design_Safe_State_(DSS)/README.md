

# QNX Hypervisor — Design Safe State (DSS)

## Overview

**Design Safe State (DSS)** defines how a system responds when an unexpected error occurs — either software or hardware — that the system was not designed to handle. In the QNX Hypervisor context, there are two levels of DSS: **local** (per-guest) and **global** (system-wide). This module covers both, their implementation, and the critical caveats around DMA and pass-through devices.

---

## 1. What is Design Safe State?

> **Definition:** A state that the system moves to when an internal error occurs that is **not designed to be handled** — neither software nor hardware failure that was anticipated.

| Aspect | Description |
|--------|-------------|
| **Trigger** | Unexpected, unrecoverable error |
| **Response** | Move to a known-safe configuration |
| **Goal** | Prevent harm, limit damage, enable recovery |
| **Key Principle** | **No attempt at recovery** — just get to safety |

---

## 2. Two Levels of DSS in Hypervisor

```
┌─────────────────────────────────────────────────────────────────────┐
│                    DESIGN SAFE STATE HIERARCHY                       │
│                                                                     │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │  GLOBAL DSS                                                  │    │
│  │  "Something in host failed — entire system unsafe"          │    │
│  │                                                              │    │
│  │  Action: Terminate ALL guests → Reboot host                │    │
│  │                                                              │    │
│  │  ┌─────────────────┐    ┌─────────────────┐    ┌─────────┐ │    │
│  │  │  Guest A         │    │  Guest B         │    │  Host   │ │    │
│  │  │  (terminated)    │    │  (terminated)    │    │ (reboot)│ │    │
│  │  └─────────────────┘    └─────────────────┘    └─────────┘ │    │
│  │                                                              │    │
│  └─────────────────────────────────────────────────────────────┘    │
│                              ▲                                      │
│                              │                                      │
│                    Host failure detected                              │
│                              │                                      │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │  LOCAL DSS                                                   │    │
│  │  "Something in one guest failed — others may be OK"         │    │
│  │                                                              │    │
│  │  Action: Terminate that qvm (guest stops)                  │    │
│  │                                                              │    │
│  │  ┌─────────────────┐    ┌─────────────────┐    ┌─────────┐ │    │
│  │  │  Guest A         │    │  Guest B         │    │  Host   │ │    │
│  │  │  (terminated)    │    │  (running)       │    │ (running)│ │    │
│  │  │  qvm killed      │    │  qvm running     │    │         │ │    │
│  │  └─────────────────┘    └─────────────────┘    └─────────┘ │    │
│  │                                                              │    │
│  └─────────────────────────────────────────────────────────────┘    │
│                              ▲                                      │
│                              │                                      │
│                    Guest A failure detected                           │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 3. Local DSS: Guest Failure

### Concept

When a single guest encounters an unrecoverable error, the local DSS is to **terminate that guest's `qvm` process**.

### Why This Works

| Property | Explanation |
|----------|-------------|
| `qvm` is just a process | Normal QNX process, can be killed with `kill()` |
| vCPU threads belong to `qvm` | Killing `qvm` stops all vCPU threads |
| Guest code stops executing | No vCPU threads = no guest execution |
| Theoretical isolation | Should not affect host or other guests |

### Execution Flow

```
Guest A detects unrecoverable error
    │
    ▼
┌─────────────────┐
│  Error Handler  │  "This is bad — I can't recover"
│  (in guest or   │
│   monitoring    │
│   process)      │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│  kill(qvm_pid)  │  or: qvm receives fatal signal
│  or: qvm crashes│
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│  qvm process    │  "I'm dying..."
│  termination    │
│                 │
│  • All vCPU     │
│    threads stop │  Guest A no longer executes
│  • Guest memory │
│    freed by     │
│    procnto      │
│  • Virtual      │
│    devices      │
│    unloaded     │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│  Guest A: DEAD  │
│  Guest B:       │  "I'm fine, still running"
│  Host:          │  "I'm fine, still running"
│  "Theoretical    │
│   isolation"    │
└─────────────────┘
```

### The DMA Caveat (Critical)

> **"Terminating a guest doesn't harm anything else... EXCEPT if DMA was in progress."**

```
┌─────────────────────────────────────────────────────────────────────┐
│  THE DMA PROBLEM: Guest Crash During Pass-Through                   │
│                                                                     │
│  t=0   Guest A driver starts DMA transfer                         │
│        ┌─────────┐                                                  │
│        │ Guest A │  "Start DMA from 0x40000000 to device"        │
│        │ Driver  │                                                  │
│        └────┬────┘                                                  │
│             │                                                       │
│  t=1   DMA engine running autonomously (hardware)                   │
│        ┌─────────┐                                                  │
│        │ DMA     │  "Transferring... 30% complete"                 │
│        │ Engine  │  "Reading from guest physical memory"            │
│        │ (HW)    │                                                  │
│        └────┬────┘                                                  │
│             │                                                       │
│  t=2   Guest A crashes! (null pointer, watchdog timeout, etc.)      │
│        ┌─────────┐                                                  │
│        │ Guest A │  "Segmentation fault — dying!"                  │
│        │ CRASH   │                                                  │
│        └────┬────┘                                                  │
│             │                                                       │
│  t=3   qvm killed (local DSS)                                     │
│        ┌─────────┐                                                  │
│        │ qvm     │  "Process terminated"                             │
│        │ (dead)  │                                                  │
│        └────┬────┘                                                  │
│             │                                                       │
│  t=4   DMA engine STILL RUNNING! (hardware autonomous)              │
│        ┌─────────┐                                                  │
│        │ DMA     │  "Still going... 60% complete"                  │
│        │ Engine  │  "But memory at 0x40000000 is now UNMAPPED!"    │
│        │ (HW)    │  "Reading garbage or causing bus fault"         │
│        └────┬────┘                                                  │
│             │                                                       │
│  t=5   DMA completes (or hangs) with CORRUPTED data               │
│        ┌─────────┐                                                  │
│        │ Device  │  "I got bad data"                                 │
│        │ State   │  "Registers in undefined state"                   │
│        │ (BAD)   │                                                  │
│        └────┬────┘                                                  │
│             │                                                       │
│  t=6   System decides to restart Guest A                            │
│        ┌─────────┐                                                  │
│        │ New qvm │  "Starting fresh..."                              │
│        │ (Guest A)│ "Let me initialize the device..."                │
│        └────┬────┘                                                  │
│             │                                                       │
│  t=7   Driver tries to use device — FAILS                           │
│        ┌─────────┐                                                  │
│        │ Driver  │  "Device not responding!"                         │
│        │ (ERROR) │  "DMA registers show 0xDEADBEEF — weird!"       │
│        │         │  "Cannot recover — DMA left in bad state"         │
│        └─────────┘                                                  │
│                                                                     │
│  RESULT: Even after "safe" termination, hardware state is corrupted  │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### Mitigation: Driver Reset on Initialization

```c
// CRITICAL: Always reset hardware before assuming it's clean

int driver_init() {
    // Step 1: Stop any ongoing DMA
    write_reg(DMA_CONTROL_REG, DMA_STOP_BIT);
    
    // Step 2: Wait for DMA to quiesce (hardware-specific timeout)
    int timeout = 1000;  // 1000 * 10us = 10ms max
    while (read_reg(DMA_STATUS_REG) & DMA_ACTIVE_BIT) {
        delay_us(10);
        if (--timeout == 0) {
            log_error("DMA failed to stop — hardware fault?");
            return -EIO;
        }
    }
    
    // Step 3: Reset DMA controller (hardware-specific sequence)
    write_reg(DMA_RESET_REG, 1);
    delay_us(100);  // Hardware reset pulse width
    write_reg(DMA_RESET_REG, 0);
    
    // Step 4: Verify clean state
    uint32_t status = read_reg(DMA_STATUS_REG);
    if (status != DMA_IDLE_STATE) {
        log_error("DMA reset failed: status=0x%08x", status);
        return -EIO;
    }
    
    // Step 5: Clear any pending interrupts
    write_reg(DMA_IRQ_CLEAR_REG, 0xFFFFFFFF);
    
    // Step 6: Now safe to configure
    write_reg(DMA_CONFIG_REG, DEFAULT_CONFIG);
    
    log_info("Device reset complete — safe to use");
    return 0;
}
```

> **Rule:** Any driver using pass-through or DMA **must** implement hardware reset on initialization. Never assume hardware is in a clean state after a crash.

---

## 4. Global DSS: Host Failure

### Concept

When something in the **host** fails unexpectedly — an error that cannot be cleanly recovered — the global DSS is to **terminate all guests and reboot the entire system**.

### Why Global?

| Reason | Explanation |
|--------|-------------|
| Host is the foundation | If host fails, guests cannot be trusted |
| Shared resources | Host manages memory, interrupts, scheduling for all guests |
| Unknown state | Host failure may have corrupted guest state too |
| Safety first | Better to reboot than risk running in undefined state |

### Execution Flow

```
Host detects unrecoverable error
    │
    ▼
┌─────────────────┐
│  Error in Host  │  "procnto error, memory corruption, etc."
│  (unrecoverable)│
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│  Global DSS     │  "System is unsafe — shut everything down"
│  Handler        │
└────────┬────────┘
         │
         ├──► Terminate Guest A (kill qvmA)
         │
         ├──► Terminate Guest B (kill qvmB)
         │
         ├──► Terminate Guest C (kill qvmC)
         │
         └──► ... all guests
         │
         ▼
┌─────────────────┐
│  Reboot Host    │  "Full system restart"
│  (QNX reboot)   │
└─────────────────┘
```

### Implementation Example

```c
// Global DSS handler (runs in host)

void global_dss_handler(error_code_t error) {
    log_critical("GLOBAL DSS: Unrecoverable host error %d", error);
    
    // Step 1: Notify all guests (if possible)
    // They may want to log state before dying
    for (int i = 0; i < num_guests; i++) {
        signal_guest_shutdown(guests[i], SHUTDOWN_IMMEDIATE);
    }
    
    // Step 2: Terminate all qvm processes
    for (int i = 0; i < num_guests; i++) {
        kill(guests[i]->qvm_pid, SIGKILL);
        log_info("Terminated guest %s (pid %d)", 
                 guests[i]->name, guests[i]->qvm_pid);
    }
    
    // Step 3: Allow brief time for cleanup (optional)
    delay_ms(100);
    
    // Step 4: Reboot the host
    log_critical("Rebooting system...");
    reboot(RB_AUTOBOOT);
    
    // Should never reach here
    while (1);
}
```

---

## 5. Recovery vs. No Recovery

### The DSS Philosophy: No Recovery

| Approach | DSS Philosophy | Rationale |
|----------|---------------|-----------|
| **No recovery** | ✅ Standard DSS | Unknown error → unknown state → don't try to fix |
| **Attempt recovery** | ⚠️ Optional | If you understand the failure, you can try |

> **"No attempts are made for recovery with those two suggestions [local DSS, global DSS], but you can, if you want."**

### When Recovery Might Be Appropriate

```c
// Optional: Monitored restart with recovery attempt

void monitored_guest_restart(guest_t *guest) {
    // Step 1: Log the failure
    log_error("Guest %s crashed — attempting restart", guest->name);
    
    // Step 2: Reset any pass-through hardware
    // (Driver must implement reset on init)
    
    // Step 3: Restart the guest
    pid_t new_qvm = start_qvm(guest->config_file);
    
    if (new_qvm < 0) {
        // Restart failed — escalate to global DSS
        log_critical("Guest restart failed — escalating to global DSS");
        global_dss_handler(ERROR_RESTART_FAILED);
        return;
    }
    
    // Step 4: Monitor the new guest
    guest->qvm_pid = new_qvm;
    guest->restart_count++;
    
    if (guest->restart_count > MAX_RESTARTS) {
        // Too many restarts — something is fundamentally wrong
        log_critical("Guest %s restarted %d times — global DSS", 
                     guest->name, guest->restart_count);
        global_dss_handler(ERROR_TOO_MANY_RESTARTS);
    }
}
```

### Recovery Decision Matrix

| Scenario | DSS Only | Recovery Attempt |
|----------|----------|------------------|
| Unknown error | ✅ Yes | ❌ No |
| Known error with known fix | ⚠️ Maybe | ✅ Yes |
| Guest crash, no pass-through | ✅ Yes | ✅ Yes (safe to restart) |
| Guest crash, with DMA/pass-through | ✅ Yes | ⚠️ Only with driver reset |
| Multiple guest crashes | ✅ Yes | ❌ No (systemic problem) |
| Host error | ✅ Yes | ❌ No (must reboot) |

---

## 6. DSS Configuration Examples

### Local DSS: Guest Monitoring

```qvmconf
# Guest configuration with health monitoring
system name=safety-critical-guest
ram addr=0x40000000,size=0x4000000
cpu cluster=0,cores=2

# Pass-through device (requires driver reset!)
pass addr=0xFE000000,host=0x3F000000,size=0x100000
pass interrupt=42

# Boot image
load addr=0x40000000,file=/data/guests/safety/guest-boot.img
```

**Host monitoring process:**
```c
// watchdog.c — runs in host, monitors guest health

int main() {
    while (1) {
        sleep_ms(100);  // Check every 100ms
        
        for (int i = 0; i < num_guests; i++) {
            guest_t *g = &guests[i];
            
            // Check if guest is responsive
            if (!guest_is_responsive(g)) {
                log_error("Guest %s unresponsive — local DSS", g->name);
                
                // Local DSS: terminate this guest
                kill(g->qvm_pid, SIGKILL);
                
                // Optional: restart with recovery
                if (g->restart_count < MAX_RESTARTS) {
                    delay_ms(100);  // Allow cleanup
                    start_qvm(g->config_file);
                    g->restart_count++;
                }
            }
        }
    }
}
```

### Global DSS: Host Health Monitor

```c
// host_monitor.c — monitors host health, triggers global DSS

int main() {
    while (1) {
        sleep_ms(50);
        
        // Check host health indicators
        if (check_memory_corruption() || 
            check_kernel_panic_pending() ||
            check_unrecoverable_hardware_error()) {
            
            // GLOBAL DSS: Shut down everything
            global_dss_handler(ERROR_HOST_FATAL);
            // Does not return
        }
    }
}
```

---

## 7. DSS Summary Table

| Aspect | Local DSS | Global DSS |
|--------|-----------|------------|
| **Trigger** | Single guest failure | Host failure |
| **Scope** | One guest | All guests + host |
| **Action** | Terminate `qvm` for that guest | Terminate all guests, reboot host |
| **Isolation** | Theoretical (caveat: DMA) | N/A — system-wide |
| **Recovery** | Optional restart possible | Full reboot required |
| **DMA concern** | Yes — reset hardware on restart | Yes — reboot clears everything |
| **Use case** | Non-safety guest crash | Host microkernel error |

---

## 8. Key Takeaways

| Concept | Key Point |
|---------|-----------|
| **DSS definition** | State system moves to on unexpected, unhandled error |
| **Local DSS** | Terminate the failing guest's `qvm` process |
| **Global DSS** | Terminate all guests, reboot host |
| **qvm is just a process** | Can be killed; killing it stops guest execution |
| **Theoretical isolation** | Guest crash shouldn't affect others, BUT... |
| **DMA caveat** | Pass-through DMA may leave hardware in bad state |
| **Driver reset requirement** | Any DMA driver must reset hardware on init |
| **No recovery in DSS** | Philosophy is safety first, not recovery |
| **Recovery optional** | You can implement restart if you understand the failure |
| **Multiple restarts** | Escalate to global DSS if guest keeps crashing |

---

## 9. Safety Manual Connection

The QNX Hypervisor for Safety **Safety Manual** provides:

| Content | Relevance to DSS |
|---------|-----------------|
| Restrictions on guest/host configurations | Which setups allow safe termination |
| Recommendations for monitoring | How to detect failures |
| Hazard analysis | What can go wrong with DMA/pass-through |
| Validation guidance | How to verify DSS works correctly |

> **Always consult the Safety Manual when implementing DSS in a certified system.**

---

## 10. Screenshots
---

![1](resources/1.png)

---
