---
title: "Low-overhead General-purpose Near-Data Processing in CXL Memory Expanders"
date: 2026-09-03T00:00:00
summary: "M²NDP puts a general-purpose RISC-V vector engine inside the CXL memory controller, builds the kernel launch path out of ordinary CXL.mem loads and stores."
categories: ["NDP", "CXL"]
tags: ["near-data processing", "CXL", "RISC-V", "inference", "simulation"]
paper:
  title: "Low-overhead General-purpose Near-Data Processing in CXL Memory Expanders"
  authors: "Ham et al."
  venue: "MICRO"
  year: 2024
  url: "https://arxiv.org/pdf/2404.19381"
  code: "https://github.com/PSAL-POSTECH/M2NDP-public/tree/main"
---

## What problem is the paper trying to solve?

CXL is a protocol building on top of PCIe, allowing for memory disaggregation. The CXL.mem protocol allows for low-latency, but the overall link bandwidth is limited to the PCIe bandwidth, bottlenecking memory-bound applications. A common approach is Near-Data Processing (NDP), implemented as hardware logic in the CXL memory controller. Problem 1 is that this cannot be implemented in a general case. You have to put fixed-function blocks on the memory expander die, which is not viable. Problem 2 lies with the conventional NDP kernel launch path: CXL.io (PCIe) ring buffer and doorbell. This is how GPUs work, with userspace writing a command, the driver takes a kernel-mode transition and pushes a descriptor into a ring in kernel memory, the host writes the ring tail pointer to a device MMIO register, the device sees the doorbell, DMAs the descriptor, then the command it points to, DMAs a completion back, then you poll or take an interrupt. Just kernel launch is roughly the length of the NDP kernel execution. Another pathway is direct to MMIO registers, skipping the ring. However, this is serial, only one kernel can be executed at a time, and it cannot be safely shared between processes. This paper proposes a solution to both issues.

## What are the key ideas of the paper?

The authors propose M²NDP, memory-mapped NDP which allows for general-purpose NDP in CXL memory, explicitly solving problem 1. M²func is a component of this which explicitly solves problem 2. M²µthread maximizes the NDP kernel's memory bandwidth utilization with lightweight, hardware created threads optimized for NDP execution.

M²func gives the fast but simple CXL.mem the expressiveness of the slower CXL.io, without using CXL.io at all. Without changing the CXL standard or the host silicon, the authors reserve physical memory space of the CXL memory referred to as the M²func region. With host communication, such as NDP kernel launch, solely taking place in that region, a packet filter is added to determine the mode of communication. Different functions can be called with different offsets. CXL.io is no longer needed for NDP.

M²µthread starts from an observation about what memory-bound kernel execution actually needs. When dealing with hundreds of concurrent memory requests, you would typically spawn many threads, or apply out-of-order execution. Out-of-order execution is too expensive for a memory expander, and many threads are expensive only because a conventional thread carries the full ISA-defined register set whether it uses it or not. However, memory-bound NDP kernels barely require compute resources. So, M²µthread works to optimize the execution side. If the compiler declares how many registers a kernel needs, the hardware can provision exactly that many, and the same register file can then support far more concurrent threads. They also bind a thread directly to the address it is responsible for, which removes the index arithmetic that can dominate short memory-bound kernels.

## What are the key mechanisms/implementation?

A packet filter at the device's input port checks every incoming address against a small table of per-process regions. The table is installed once over CXL.io at initialization, after which CXL.io is no longer needed. A matching write is interpreted as a function call rather than a store, and the NDP controller, a microcontroller much like the ones already in GPUs, handles it. The only issue is that a CXL.mem write response cannot return data, so return values are retrieved by a subsequent read to the same address, with a fence in between. Kernel registration, launch, etc. all work this way.

The NDP unit itself is a RISC-V core with the vector extension, split into sub-cores each holding a pool of thread slots, switching between them on every stall to hide DRAM latency. A hardware thread generator spawns one µthread per chunk of the address range (sized to DRAM access granularity). Virtual memory is supported with an on-chip TLB, expanded for large data applications with a low-overhead DRAM-TLB.

## What are the key results?

Almost all results are simulated on an in-house cycle-level simulator built on Ramulator. The exception is the CPU-NDP baseline for OLAP, which was measured on a real dual-socket EPYC system chosen to match the modeled bandwidth. Energy is derived from analytical power models as there is no physical implementation of this yet.

Likely the key result presented is that a general-purpose memory expander chip performs within a few percent of the domain-specific implementations on their intended workloads. However, this paper also displays a framework to bypass slow CXL.io access for frequent kernel launches. The clearest evidence for this is the key-value store, where the same NDP engine behind a conventional CXL.io launch path ends up worse than doing nothing at all, because the launch costs more than the kernel it is launching. Only with M²func does NDP become a win. The authors then rerun the comparison with CXL.io and CXL.mem forced to the same latency, and M²func still wins, which separates the benefit of fewer round trips from the benefit of riding a faster protocol. This is a novel implementation, and it works without modifying the CXL standard or the host processor.

Sensitivity analysis results confirm the chip is bandwidth-bound by design, since raising its clock buys almost nothing, and performance scales close to linearly when work is partitioned across several devices.

## Strengths

M²func requires no change to the CXL standard, no new host instructions, no host hardware modification and no kernel-mode transition. It holds potential for appearing in a real product.

The evaluation is thorough in its measured cases. Comparing against domain-specific accelerators, equalizing protocol latency to isolate the mechanism, and reporting the distance from an ideal-bandwidth bound are all measurements that risked unflattering results, but the authors took the initiative to show them upfront, proving the efficacy. The simulator is also public, and can be investigated.

The paper is also written expertly, building each concept onto the last. The paper is self-contained, and a reader without prior understanding of CXL, NDP, etc. can learn from it.

## Weaknesses

Every kernel is hand-written assembly, evaluated against production baselines like vLLM and Polars that contain years of tuning. The authors acknowledge this and leave it to future work, but it is a genuinely large amount of future work.

The programming model requires the user to decide which kernels to offload and when the host is a better choice. This is not exactly a flaw, but also provides future research direction.

The overhead of the packet filter was not mentioned in the paper. I thought this was odd, because it sits at the device input port, and examines every single CXL.mem packet, including ordinary read and writes that have nothing to do with NDP. Surely this costs some latency?

The overhead on the host-side is absent, such as core occupancy, uncacheable store latency (typical stores are buffered in the cache), and socket topology (depending on where the CXL device is slotted in, potential UPI/IF contention), etc. On the device-side, the multi-tenant case isn't tested.

M²NDP is proven to work exceedingly well in isolation, and I'd be curious how it works under some more production use-cases, to see if it can actually be implemented by a memory company. After all, this work was done in conjunction with SK hynix.