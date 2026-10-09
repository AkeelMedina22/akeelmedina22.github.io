---
title: "Revealing NVIDIA Closed-Source Driver Command Streams for CPU–GPU Runtime Behavior Insight"
date: 2026-09-25T00:00:00
draft: false
summary: "Reverse engineering the NVIDIA driver command streams."
categories: ["GPU Runtime", "Profiling"]
tags: ["CUDA", "GPU driver", "command submission", "DMA", "CUDA Graphs", "reverse engineering", "PCIe"]
paper:
  title: "Revealing NVIDIA Closed-Source Driver Command Streams for CPU–GPU Runtime Behavior Insight"
  authors: "Yan, Karlin, and Grant"
  venue: "arXiv"
  year: 2026
  url: "https://arxiv.org/abs/2604.26889"
---

[M²NDP](/papers/m2ndp/) describes the conventional accelerator launch path as a ring buffer plus a doorbell, with a kernel-mode transition at the start, and shows that the launch can cost about as much as the kernel itself. This paper from Queen's University looks at that path on a real NVIDIA GPU, and it turns out the kernel-mode transition isn't there. The userspace driver maps the GPU's PCIe BAR into its own address space and writes the doorbell directly. That makes the path fast, but also hard to observe.

## What problem is the paper trying to solve?

A lot of what a CUDA call does doesn't live in the CUDA runtime. It lives in NVIDIA's userspace driver underneath it, which is still proprietary. The kernel driver was open-sourced in 2022, but it only handles operations needing OS privilege, such as memory mapping, so the translation from an API call into the hardware commands the GPU consumes stays opaque.

One issue with this is that you cannot understand performance, Nsight Systems reports a single duration at the runtime level, aggregating the entire process into one number, which matters most in disaggregated systems where software-side cost dominates small messages. It is also hard to observe this path directly, as mentioned above. There is no syscall to hook, and prior work that polled MMIO registers cannot keep up with a full command stream, since its sampling rate cannot guarantee catching every submission and the sampled state may be torn.

## What are the key ideas of the paper?

The idea is that the userspace driver can bypass the kernel for submission, but it cannot bypass the kernel to get the mapping in the first place. Every mapping request goes through `nv_mmap`, where userspace supplies the target physical address. This is a mandatory kernel-side action in the entire process, and it allows the authors to achieve telemetry.

Inside `nv_mmap`, if the requested range covers the doorbell register, they install a CPU hardware watchpoint on the corresponding userspace virtual address, like a watchpoint set in GDB. When the driver later writes the doorbell, the watchpoint traps into the kernel and pauses it, creating a static observation window at the exact point of a submission.

What makes this better than polling, is that the channel ID has already been written by the time the handler runs, and execution stays in kernel space until the observation finishes, so no new commands can appear during the window. Each submission is captured whole. One issue is that reading the doorbell back always returned zero, so they allocate a RAM page as a shadow doorbell, let the write land there, observe, then forward the value to the real register.

## What are the key mechanisms/implementation?

Submitting work to the GPU has three parts. The pushbuffer is the actual list of commands, sitting in host memory. The GPFIFO is a queue of pointers, where each entry says "the next batch of commands is here, and it's this long". The doorbell is how the driver tells the GPU to go check the queue. So the driver writes commands into the pushbuffer, adds an entry to the GPFIFO pointing at them, and rings the doorbell with its channel ID. The GPU then reads the new GPFIFO entry, follows it to the commands, and runs them.

The watchpoint only gives the authors the channel ID, so they work backwards from there. The open-source kernel driver keeps track of where each channel's queue lives and where the driver's write pointer currently is. From that they find the newest GPFIFO entry, follow it to the pushbuffer, and decode the commands using the headers NVIDIA publishes with the open-source driver.

Once they know where everything lives, they can write their own commands into the pushbuffer and ring the doorbell themselves, skipping CUDA entirely. This works because the GPU and the process share the same addresses (UVM), so they can match up which of the process's memory allocations is the pushbuffer, which is the GPFIFO, and so on. For timing they use the same thing CUDA events use under the hood: a command that writes a timestamp once everything before it has finished.

## What are the key results?

Host-to-device `cudaMemcpy` actually has 2 pathways. Below 24 KiB the driver uses inline DMA, embedding the source data directly in the pushbuffer payload and having the compute engine write it out. At 24 KiB and above it switches to direct DMA, carrying both addresses and using a dedicated copy engine. CUDA doesn't expose these to the user, so it cannot be compared with the API. The authors issue both directly. They coalesce the warmup iterations, a progress tracker, the measured iterations and a second tracker into one pushbuffer segment submitted once, so no driver intervention occurs during the measurement.

The engines behave very differently. Inline DMA starts at roughly 24 ns against roughly 500 ns for the copy engine, but saturates at about 17.5 GiB/s by 8 KiB, while the copy engine scales to about 22 GiB/s around 1 MiB. The comparison against Nsight is interesting. For an 8-byte copy, Nsight reports a "CUDA HW" duration of 468 ns where the raw engine time is 24 ns, so roughly 95% of what Nsight reports is not engine time. The gap closes as transfers grow, down to 0.36% at 32 MiB. This is pretty reminiscent of my MSc thesis research.

The second case study compares CUDA Graphs under versions 11.8 and 13.0. For a 2000-kernel chain, launch time is 209 µs against 5.9 µs, the command footprint is 45,476 B against 2,216 B, and doorbell writes grow with graph length on 11.8 but stay at exactly one on 13.0. The authors also find the pushbuffer lives in host RAM while the GPFIFO lives in GPU memory, so 11.8's repeated cycles alternate local and remote writes across PCIe each time, while 13.0 assembles one pushbuffer and notifies once. Under 11.8 the command size grows in discrete steps and the launch time steps at the same breakpoints.

## Strengths

All performance numbers also come from the unmodified kernel module, with only the logging done on the edited one. It is nice that this is mentioned.

The CUDA Graphs result is pretty useful information, in my opinion. Often, HPC environments have fixed CUDA versions. If relying on default, older versions of CUDA, this is a huge consideration to make. NVIDIA did announce the faster graph launches, but this paper shows why, and it is worth knowing that for launch-heavy workloads, upgrading to CUDA 13.0 can improve performance.

## Thoughts

For CUDA Graphs, I think most of the speedup is just 13.0 doing less work. It's about 35x faster but writes about 20x fewer bytes, so the RAM/PCIe switching only really has to explain the leftover 2x. The authors do word this carefully ("suggests", "could"), but it's easy to read it as the main cause. The driver version also changes between the two stacks, not just the CUDA version.

Everything is on one A40 over PCIe Gen4. The authors say the submission process is the same across GPU generations, which is probably true, but things like the 24 KiB switch point and where the pushbuffer and GPFIFO live could easily be different on another platform. I'd be really curious to see this on a GH200, where the CPU and GPU sit on a coherent NVLink-C2C link, and "local" versus "remote" writes might not look anything like this.

Overall I think this is a really useful tool, and it changes how I read M²NDP. The kernel-mode transition M²NDP blames on ring buffers isn't there on NVIDIA's path, so what M²func actually saves over a GPU-style launch is the slower CXL.io protocol and the extra round trips, not a syscall.