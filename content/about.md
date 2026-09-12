---
title: "About"
description: "Akeel Ather Medina — MSc High Performance Computing, University of Edinburgh. LLM systems, HPC/ML, interconnects."
---

I recently finished an MSc in High Performance Computing at the University of Edinburgh (EPCC), where my thesis measured the cost of KV-cache offloading in LLM inference. I've written more about it in one of my blog posts! Before Edinburgh I spent two years as a software engineer at Abyss Solutions, working on point-cloud analysis pipelines at scale.

My interests lie around MLSys:

- **LLM inference** — KV-cache management, hardware/software co-design, down to the phy.
- **HPC for ML** — ML at scale. Scale-out algorithms and topologies, 
- **Interconnects and memory hierarchy** — CXL, PNM, optimizing tiering performance.

I'm applying for PhD positions in these areas. The best way to reach me is [email](mailto:akeelmedina22@gmail.com).

## Education

<div class="entry">
  <div class="entry-head">
    <h3 class="entry-title">MSc, High Performance Computing</h3>
    <span class="entry-when">2025 &ndash; 2026</span>
  </div>
  <p class="entry-where">The University of Edinburgh &middot; Predicted Distinction</p>
  <div class="entry-body">
    <p><strong>Thesis:</strong> <em>Profiling the Cost of KV Cache Offloading in LLM Inference.</em>
    A low-latency C++ inference profiler that attributes energy cost to CPU&ndash;GPU interconnect
    stalls during KV-cache offload, benchmarked on H200 (PCIe) and GH200 (NVLink-C2C) systems.</p>
    <p><strong>Coursework:</strong> Performance Programming, Accelerated Systems, Machine Learning
    at Scale, Advanced Message Passing Programming, Numerical Algorithms, Threaded Programming,
    HPC Architectures.</p>
  </div>
</div>

<div class="entry">
  <div class="entry-head">
    <h3 class="entry-title">BSc, Computer Science</h3>
    <span class="entry-when">2019 &ndash; 2023</span>
  </div>
  <p class="entry-where">Habib University &middot; CGPA 3.79/4.00</p>
  <div class="entry-body">
    <p>Dean's List in Fall 2022 and Spring 2021. Coursework in Computer Graphics, Image
    Processing, Deep Learning, and GPU-Accelerated Computing.</p>
  </div>
</div>

## Experience

<div class="entry">
  <div class="entry-head">
    <h3 class="entry-title">TeamEPCC — ISC High Performance Student Cluster Competition</h3>
    <span class="entry-when">Oct 2025 &ndash; Jun 2026</span>
  </div>
  <p class="entry-where">The University of Edinburgh</p>
  <div class="entry-body">
    <ul>
      <li>Represented the University of Edinburgh against 10 selected international teams.</li>
      <li>Optimised llama.cpp batched inference through source modification and performance analysis under a strict 6&nbsp;kW power budget.</li>
      <li>Built a distributed parallel file system across 8 AMD EPYC client nodes with a GPU-node metadata backend and direct PCIe-attached NVMe targets, delivering 32&nbsp;GB/s aggregate multi-client throughput.</li>
    </ul>
  </div>
</div>

<div class="entry">
  <div class="entry-head">
    <h3 class="entry-title">Software Engineer</h3>
    <span class="entry-when">Jun 2023 &ndash; Jul 2025</span>
  </div>
  <p class="entry-where">Abyss Solutions</p>
  <div class="entry-body">
    <ul>
      <li>Architected a 3D scan coverage analysis workflow processing terabytes of LIDAR data, identifying unseen regions against CAD models.</li>
      <li>Cut an ML pipeline's processing time by 75% through point-cloud handling optimisations and AWS Batch array jobs.</li>
      <li>Built a cross-sectional analysis tool for mooring-chain photogrammetry, improving measurement accuracy from roughly 15% error to sub-millimetre precision.</li>
      <li>Migrated a legacy Bash data-reconciliation process to Python/Prefect, turning a two-day manual task into a five-minute automated one.</li>
      <li>Promoted from Junior (L3) to Level 4 Engineer, later serving as technical team lead for three developers.</li>
      <li>Built proof-of-concept tools in company hackathons, including mesh generation from TLS data, octree compression, and LIDAR scan-quality metrics (Abyssathon 8 winner).</li>
    </ul>
  </div>
</div>

<div class="entry">
  <div class="entry-head">
    <h3 class="entry-title">Teaching Assistant, CS 440: Computer Graphics</h3>
    <span class="entry-when">Aug &ndash; Dec 2022</span>
  </div>
  <p class="entry-where">Habib University</p>
  <div class="entry-body">
    <p>Taught weekly WebGL recitations covering GLSL, the graphics pipeline, Phong shading,
    mesh loading, and advanced topics including tone mapping, parallax, and shadows.</p>
  </div>
</div>

## Selected Projects

<div class="entry">
  <div class="entry-head">
    <h3 class="entry-title"><a href="https://github.com/AkeelMedina22/cpp-design-patterns">C++ Design Patterns</a></h3>
    <span class="entry-when">Jun 2025</span>
  </div>
  <div class="entry-body">
    <p>A matrix multiplication library built around the Factory, Registry, and Meyers' Singleton
    patterns, with Eigen, tiled CUDA, and cuBLAS backends selected through a JSON config. New
    backends register themselves without modifying the factory.</p>
  </div>
</div>

<div class="entry">
  <div class="entry-head">
    <h3 class="entry-title">CUDA Fast Voxel Traversal</h3>
    <span class="entry-when">Dec 2024</span>
  </div>
  <div class="entry-body">
    <p>A CUDA-accelerated ray casting system for point-cloud intersection testing, using Morton
    encoding and binary search optimisations.</p>
  </div>
</div>

<div class="entry">
  <div class="entry-head">
    <h3 class="entry-title">GPU-Accelerated Gauss-Seidel Sparse Iterative Solver</h3>
    <span class="entry-when">May 2023</span>
  </div>
  <div class="entry-body">
    <p>Compared cuSPARSE CSRColor against the Brook-Vizing randomised colouring algorithm for
    preconditioning iterative SpMV, reordering elements to launch CUDA kernels in parallel.</p>
  </div>
</div>

## Tools

C++ (including CUDA and Pybind), Python, CUDA, MPI, OpenMP, CMake, Linux, Bash, AWS, Git.
