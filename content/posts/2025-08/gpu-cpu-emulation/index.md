---
title: "Why can't a GPU act as a CPU?"
date: 2025-08-02T19:11:23
description: "A thought experiment that has absolutely no real-world value."
tags: ["HPC", "GPU Architecture", "CPU Architecture"]
draft: False
---

I recently had this question pop up in my head for no real reason- given a standard Nvidia GPU and x86 CPU, can we theoretically have the GPU emulate the CPU?

At first glance, this question is quite pointless. GPU's are obviously not designed to mimic a CPU, infact a standard desktop computer comes with a CPU and GPU because they go so well together, the CPU handling latency sensitive tasks while the GPU handles throughput heavy tasks. This difference in core design philosophy makes it fairly certain that the two wouldn't be interchangable, but alright. Let's assume we don't care about speed, what hardware architectural elements could make it impossible, that can't directly be solved by software? 

Let's look at some of the possible issues. 

### Instruction Set
This is the most obvious difference, but also a solvable one. A CPU's instruction set (like x86) and a GPU's instruction set (like NVIDIA's PTX) are completely different. A GPU can't natively understand x86 code.

To get around this, you would need a sophisticated interpreter, which reads one x86 instruction, translates it into GPU code, executes it, and then moves to the next. This works, but it's incredibly slow.

A much better, though more complex, approach is a Just-In-Time (JIT) compiler. "Just-in-time" means it translates code dynamically as the program is running. Instead of translating one instruction at a time, a JIT compiler would translate whole chunks of x86 code into optimized GPU code right before they're needed. If it encounters a loop, it translates the entire loop once and then runs the highly efficient GPU version multiple times. This is far faster than re-interpreting the loop every single time. Or, you could just compile Linux for PTX :)

So, while it's a huge software challenge, the instruction set difference isn't a fundamental hardware blocker.

-----

### Interconnects
This one is more subtle than just "speed." CPU cores are linked by high-speed, coherent interconnects, like AMD's Infinity Fabric or Intels UPI. The key word here is coherent. This is a hardware feature that ensures every core has a consistent, unified view of memory. If Core 1 changes a value, the hardware automatically makes sure Core 2 sees that updated data, no matter if its in L1 or L2 cache. Operating systems absolutely depend on this hardware guarantee to function correctly.

A GPU's internal interconnects are designed for a different purpose: maximum bandwidth to shovel data between its core SM's and VRAM/Global Memory. Within an SM, they aren't coherent (L1 Cache). This L1 cache is actually side-by-side with the shared memory block. In some Nvidia GPU architectures it is customizable, whether to increase the memory allocation for shared memory vs L1 Cache. The L2 cache is coherent on the entire GPU, allowing different SM's to utilize it before repeating global memory access. L1 cache may thus have duplication. 

Trying to emulate memory coherency in software across thousands of GPU cores would be a nightmare and would never provide the guarantees an OS needs. However, I don't think this is an unsolvable issue. If you just disable the L1 cache (similar to the DeepSeek PTX optimization), you could provide the OS with a synchronized L2 cache :)

-----

### Bootstrapping
Okay, here's our first real hardware wall. How does a computer even start? The motherboard's firmware (BIOS/UEFI) is hard-coded to look for a CPU in a specific socket, initialize it, and then hand over control to it to begin the boot process. It has absolutely no mechanism to "boot from the GPU." The system is physically and logically wired to depend on a CPU to kick things off.

Well, one could theoretically design a custom motherboard with firmware to boot a GPU, but that doesn't exist today. So, in any practical sense, this is our first dead end. 

-----

### Protection Rings
A CPU has hardware-enforced security levels called privilege rings. The OS kernel runs in the all-powerful Ring 0, while your apps run in the limited Ring 3 (kernel mode vs user mode). If an app tries to do something illegal (like access kernel memory), the CPU hardware itself stops it and throws a protection fault. This is implemented by hardware components that, on almost each action the CPU takes, check the last two significant bits of the Code Segment (CS) register, which designate the Current Privilege Level (CPL) as 0 or 3.

My first thought was, "Can't we simply add a software layer check for privileged instructions?" The problem is that the entire point of hardware protection is that it's non-bypassable. If the security check is just another piece of code in our emulator, a malicious or buggy program could be written to simply sidestep that check and take over the system. 

So, if software is out, what about the GPU's hardware? A key example of protection rings in use is the CPU's Memory Management Unit (MMU). It reads the CPL from the CS register and compares it with a User/Supervisor (U/S) bit on each memory page to ensure privileged regions aren't accessed by user programs. While a GPU also has an MMU for virtual addressing, it's designed with a different philosophy. Focused on throughput, it does not have the integrated logic to consider or implement protection rings. The link between a CPU core's privilege state (the CPL) and the MMU's security check is a deeply integrated, hard-wired circuit, not a programmable feature you can enable.

Unlike the interconnect issue, which we could cleverly work around by disabling L1, there's no equivalent trick here. This absence of a hardware-enforced privilege model presents an architectural barrier that is a surprising blocker from making a GPU emulate a CPU.

-----

### Hardware Interrupt Controller
What about handling input from your keyboard or network? Well, Akeel, aren't you skipping the most obvious issue? How do you even connect your keyboard to your GPU, wouldn't it need a USB host controller? Alas, [GPUDirect RDMA](https://docs.nvidia.com/cuda/gpudirect-rdma/) exists! 

Now, we know that a CPU uses a hardware interrupt controller to actually receive input. The moment you press a key, the keyboard sends a signal that immediately forces the CPU to pause its work and handle that input. Without this, our GPU would have to poll for input—constantly asking, "Is there a key press now? How about now?" This isn't just inefficient; for much of the hardware in a PC, it's functionally invalid. A high-speed device like an NVMe SSD expects its interrupt signal to be serviced within a few milliseconds. If the processor doesn't respond in time because it's in the middle of a polling cycle, the SSD's own controller will time out and register a fatal I/O error. This would lead to constant device failures and data corruption. Polling simply can't meet the real-time demands of the underlying hardware. 

I may be wrong, but this is the first time the speed inefficiency of this CPU mimicry has caused an actual hardware fault. In both the interconnect and intruction set cases, the software should not crash, just run several hundred times slower than usual. Although, maybe this would cause the same hardware faults.

-----

To conclude, there's not really much purpose to this thought experiment, there's no good reason for a GPU to emulate a CPU, and there would need to be a staggering amount of robust software to get some of the things I've said are "theoretically possible" in actual production. However, I did learn quite alot about CPU and GPU architecture. If anyone reading this is an actual expert in computer architecture and operating systems, I'd love for some feedback (reach out to me on LinkedIn!) to know how accurate I am! This isn't really the kind of answer I can find in a textbook, haha.