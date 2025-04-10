---
title: "Utilizing PMU Event Counters on Apple M3 and M4"
date: 2025-03-15T12:53:00+08:00
draft: false
tags: ["Apple", "Architecture"]
categories: ["技术"]
---

The [Asahi Linux Docs](https://asahilinux.org/docs/hw/cpu/system-registers/) include detailed information for PMU registers on Apple M1 and M2 processors. However, on M3 and M4, there are notable differences.

## Register Definition

The PMU register definitions are as follows, according to the [Asahi Linux Docs](https://asahilinux.org/docs/hw/cpu/system-registers/) and my experiments on M3 and M4 chips.

### SYS_APL_PMCR0_EL1 

s3_1_c15_c0_0

* **[7:0] Counter enable for PMC #7-0**
* [10:8] Interrupt mode (0=off 1=PMI 2=AIC 3=HALT 4=FIQ)
* [11] PMI interrupt is active (write 0 to clear)
* [19:12] Enable PMI for PMC #7-0
* [20] Disable counting on a PMI
* [22] Block PMIs until after eret
* [23] Count global (not just core) L2C events
* **[30] User-mode access to registers enable**
* **[33:32] Counter enable for PMC #9-8**
* [45:44] Enable PMI for PMC #9-8

### SYS_APL_PMCR1_EL1

s3_1_c15_c1_0

Controls which ELx modes count events

* [7:0] EL0 A32 enable PMC #0-7 (not implemented on modern chips)
* **[15:8] EL0 A64 enable PMC #0-7**
* **[23:16] EL1 A64 enable PMC #0-7**
* [31:24] EL3 A64 enable PMC #0-7 (not implemented except on old chips with EL3)
* [33:32] EL0 A32 enable PMC #9-8 (not implemented on modern chips)
* **[41:40] EL0 A64 enable PMC #9-8**
* **[49:48] EL1 A64 enable PMC #9-8**
* [57:56] EL3 A64 enable PMC #9-8 (not implemented on modern chips)

### SYS_APL_PMESR0_EL1

s3_1_c15_c5_0

Event selection register for PMC #2-5

#### On M1/M2:

* [7:0] event for PMC #2
* [15:8] event for PMC #3
* [23:16] event for PMC #4
* [31:24] event for PMC #5

#### On M3/M4:

* [15:0] event for PMC #2
* [31:16] event for PMC #3
* [47:32] event for PMC #4
* [63:48] event for PMC #5

### SYS_APL_PMESR1_EL1

s3_1_c15_c6_0

Event selection register for PMC #6-9

#### On M1/M2:

* [7:0] event for PMC #6
* [15:8] event for PMC #7
* [23:16] event for PMC #8
* [31:24] event for PMC #9

#### On M3/M4:

* [15:0] event for PMC #6
* [31:16] event for PMC #7
* [47:32] event for PMC #8
* [63:48] event for PMC #9

### SYS_APL_PMC0-9_EL1

Performance counter.

On M1: 48 bits, bit 47 triggers PMI. 
On M2/M3/M4: 64 bits, bit 63 triggers PMI.

* PMC #0: fixed cpu cycle count if enabled
* PMC #1: fixed instruction count if enabled

According to my experiments, the main difference of PMUs on M3 and M4 is that their ESR regs are 64-bit, and each event takes 16 bits.

## Usage

First, according to the register definitions, we need to set correct bit in `SYS_APL_PMCR0_EL1` and `SYS_APL_PMCR1_EL1` to enable PMC2-9 and allow EL0 and EL1 access to them, on a machine with a [patched kernel](https://github.com/jprx/PacmanPatcher), we can write to the two regs in user mode. 

```C
asm volatile("mrs %0, s3_1_c15_c0_0" : "=r"(SYS_APL_PMCR0));
asm volatile("mrs %0, s3_1_c15_c1_0" : "=r"(SYS_APL_PMCR1));
SYS_APL_PMCR0 |= 3ULL << 32;        // enable PMC8-9
SYS_APL_PMCR0 |= 255ULL << 0;       // enable PMC0-7
SYS_APL_PMCR1 |= 3ULL << 40;        // enable PMC8-9 EL0 access
SYS_APL_PMCR1 |= 3ULL << 48;        // enable PMC8-9 EL1 access
SYS_APL_PMCR1 |= 255ULL << 8;       // enable PMC0-7 EL0 access
SYS_APL_PMCR1 |= 255ULL << 16;      // enable PMC0-7 EL1 access    
asm volatile("msr s3_1_c15_c0_0, %0" ::"r"(SYS_APL_PMCR0));
asm volatile("msr s3_1_c15_c1_0, %0" ::"r"(SYS_APL_PMCR1));
```

It’s worth noting that `SYS_APL_PMCR0_EL1` may be repeatedly overwritten by a kernel process, and any modifications to that register typically won't persist longer than ~100 µs. Thus, we can't enable the PMU event counter for longer than that time unless we patch the kernel again with a modified patcher, which is theoretically practical.

Then we need to set the event ID to the PMESR regs (SYS_APL_PMESR0_EL1, SYS_APL_PMESR1_EL1). Notice that the bit widths are different on M1/M2 and M3/M4, the former is 8bit for each event, the latter is 16bit.

```C
asm volatile("mrs %0, s3_1_c15_c5_0" : "=r"(SYS_APL_PMESR0));
asm volatile("mrs %0, s3_1_c15_c6_0" : "=r"(SYS_APL_PMESR1));
SYS_APL_PMESR0 = 1443ULL << 0;      // L1D_CACHE_MISS_LD on PMC2
SYS_APL_PMESR1 = 1441ULL << 0;      // L1D_TLB_MISS on PMC6
asm volatile("msr s3_1_c15_c5_0, %0" ::"r"(SYS_APL_PMESR0));
asm volatile("msr s3_1_c15_c6_0, %0" ::"r"(SYS_APL_PMESR1));
```

After setting the PMCR and PMESR regs, we can just read PMC2-9 (depending on the bits you set in PMESR) to get the counting of the event of interest.

```C
asm volatile("mrs x9, S3_2_c15_c2_0\n\t" ::: "x9");     // read PMC2
asm volatile("mrs x10, s3_2_c15_c6_0\n\t" ::: "x10");   // read PMC6
```

Finally, after you've got all the counter data, read SYS_APL_PMCR0_EL1 back and check the bits you set haven't been overwritten by the kernel yet. This ensures the PMU event counter data you've read is still valid.

```C
asm volatile("mrs %0, s3_1_c15_c0_0" : "=r"(SYS_APL_PMCR0));    // confirm PMCR0 changed
```