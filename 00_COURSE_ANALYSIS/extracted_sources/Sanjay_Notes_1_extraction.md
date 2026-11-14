# Source Extraction: Sanjay_Notes_1.pdf

**Source File:** `input_notes/Sanjay_Notes_1.pdf`  
**Total Pages:** 12  
**Topics:** Performance metrics, processor architecture (pipelining, superscalarity, SIMD, SMT), memory hierarchy, latency/bandwidth models, Flynn's taxonomy, UMA vs ccNUMA, distributed memory architecture, parallel speedup & efficiency.

---

## Page 01

### Topic: Performance Metric & Stored-Program Computer Architecture
- **Performance Metric:**
  $$P = \frac{\text{Work}}{\text{Time}}$$
  - **Work:** "FLOPs (+, -, *, /)", "lattice site update iterations", "solving the problem"
  - **Time:** "wall-clock time"
- **Stored-program computer (Von Neumann Architecture):**
  - Diagram:
    - `Memory` $\longleftrightarrow$ `Arithmetic/Logic Unit` & `Control Unit`
    - Connected to `Input/Output`
  - Memory access: "high latency and low bandwidth"
- **Instruction Execution:**
  - "This is the primary resource of the processor. All efforts in hardware design are targeted towards increasing the instruction throughput."
  - "Instructions are the concept of 'work' as seen by processor designers."
  - "Not all instructions count as 'work' as seen by application developers."

---

## Page 02

### Topic: Data Transfer & Modern Superscalar Core Microarchitecture
- **Data Transfer:**
  - "Data transfers are a consequence of instruction execution and therefore a secondary resource."
  - "Maximum bandwidth is determined by the request rate of executed instructions and technical limitations."
- **Measures to Improve Performance:**
  - Instruction execution is pipelined.
  - Instructions are executed out of program order (Out-of-Order Execution).
  - Instructions can be inherently parallel (Instruction-Level Parallelism / ILP).
  - Caches store often-used data for quick reference.
- **Detailed Core Architecture Diagram:**
  - `L1 I-cache` feeds 4 `Decoders`
  - Decoders feed `Reorder Buffer / Register Renaming Scheduler`
  - Scheduler dispatches to 6 Execution Ports:
    - `Port 0`: ALU MUL DIV
    - `Port 1`: ALU ADD
    - `Port 2`: LOAD
    - `Port 3`: LOAD ADRS
    - `Port 4`: STORE ADRS
    - `Port 5`: ALU JMP
  - Interfaces with: `Register File`, `L1 D-cache`, and `Memory Control`.

---

## Page 03

### Topic: In-Core Features & Pipelining of Functional Units
- **Important In-Core Features:**
  1. **Pipelining:** Instruction execution in multiple steps.
  2. **Superscalarity:** Multiple instructions per cycle.
  3. **Single Instruction Multiple Data (SIMD):** Multiple operations per instruction.
  4. **Simultaneous Multi-Threading (SMT):** Multiple instruction sequences in parallel.
- **Pipelining of Functional Units:**
  - **Idea:**
    - Split complex instruction into several simple/fast steps.
    - Each step takes the same amount of time, e.g. a single cycle.
    - Execute different steps on different instructions at the same time (in parallel).
  - **Benefits:**
    - Core can work on several independent instructions simultaneously.
    - One instruction finished each cycle after the pipeline is full.
  - **Drawbacks:**
    - Pipeline must be filled: large number of independent instructions required.
    - Requires complex instruction scheduling by hardware (out-of-order execution) or compiler (software pipelining).

---

## Page 04

### Topic: 5-Stage Floating Point Multiplication Pipeline & Instruction Pipeline
- **Floating-Point Multiplication Pipeline Example:**
  - Operation: $A(i) = B(i) \times C(i)$ for $i = 1, 2, \dots, N$.
  - Example numbers: $3 \times 10^{12} \times 5 \times 10^9$
  - 5 Stages:
    1. Separate mantissa / exponent ($3 \times 10^{12}$ and $5 \times 10^9$)
    2. Multiply mantissas ($3 \times 5 = 15$)
    3. Add exponents ($12 + 9 = 21$)
    4. Normalize results ($15 \times 10^{21} \rightarrow 1.5 \times 10^{22}$)
    5. Insert sign
  - Latency: First result is available after 5 cycles (= latency of pipeline).
  - Wind-up / wind-down phases: Empty pipeline stages. Wind-up = 4 cycles.
- **The Instruction Pipeline:**
  - Stages: `Fetch instruction from L1` $\rightarrow$ `Decode instruction` $\rightarrow$ `Execute instruction`.
  - "Beside functional units, instruction execution itself is also pipelined with at least 3 steps."
  - "Branches can also stall the pipeline (speculative execution, branch prediction)."
  - "Each unit is a pipeline itself; each pipeline is chained."

---

## Page 05

### Topic: Superscalar Execution, SMT & SIMD Processing
- **Superscalar Execution:**
  - Multiple execution units enable Instruction-Level Parallelism (ILP).
  - Instruction stream is "parallelized on the fly".
  - Issuing $m$ concurrent instructions per cycle: $m$-way superscalar.
  - Modern processors are 3- to 6-way superscalar and can perform 2 floating-point instructions per cycle.
- **Simultaneous Multi-Threading (SMT):**
  - Pipelines are often underutilized due to dependencies, cache misses, waiting time.
  - "Pipeline bubbles mean wasted resources."
  - SMT improves pipeline utilization by filling empty issue slots from multiple independent threads.
  - "It does not improve any other resource on the chip (shares ALUs, caches, execution ports)."
  - Requires extra hardware (architectural registers, thread states).
- **SIMD Processing:**
  - Single Instruction Multiple Data (SIMD) allows the execution of the same operation on "wide" vector registers from a single instruction.

---

## Page 06

### Topic: x86 SIMD Instruction Sets, Vectorization & Core Peak Performance Formula
- **x86 SIMD Instruction Sets & Register Widths:**
  - **SSE:** Register width = 128 bit $\rightarrow$ 2 double-precision (64-bit) floating-point operands.
  - **AVX / AVX2:** Register width = 256 bit $\rightarrow$ 4 double-precision floating-point operands.
  - **AVX-512:** Register width = 512 bit $\rightarrow$ 8 double-precision floating-point operands.
  - Concurrency: Operations are mostly concurrent on modern standard CPUs.
- **SIMD by Compiler (Vectorization via Loop Unrolling):**
  - Original scalar loop:
    ```c
    for (int i=0; i<n; i++)
        C[i] = A[i] + B[i];
    ```
  - Unrolled loop (by factor of 4 for AVX):
    ```c
    for (int i=0; i<n; i+=4) {
        C[i]   = A[i]   + B[i];
        C[i+1] = A[i+1] + B[i+1];
        C[i+2] = A[i+2] + B[i+2];
        C[i+3] = A[i+3] + B[i+3];
    }
    ```
- **Peak Performance of a Core (Crucial Formula):**
  $$P_{\text{core}} = n_{\text{superscalar}}^{\text{FP}} \times n_{\text{FMA}} \times n_{\text{SIMD}} \times f$$
  - $n_{\text{superscalar}}^{\text{FP}}$: Superscalarity (number of FP execution ports/units)
  - $n_{\text{FMA}}$: Fused Multiply-Add factor (2 FLOPs per FMA operation: 1 multiply + 1 add)
  - $n_{\text{SIMD}}$: SIMD vectorization factor (e.g., 4 for AVX2 DP, 8 for AVX-512 DP)
  - $f$: Processor clock speed / frequency.

---

## Page 07

### Topic: Memory Hierarchy, Latency vs Bandwidth
- **Memory Hierarchy Concepts:**
  - "Data transfers are the #1 limiting factor in computing. Main memory is too slow to keep up with the CPU's hunger for data."
  - "You can either build a small and fast memory or a large and slow memory."
  - Caches hold frequently accessed data for fast reference.
  - Multiple cache levels: L1 (fastest, smallest), L2, L3 (shared, larger, slower).
  - Data transfers occur in bursts of single cache lines (typically 64 bytes).
  - "The purpose of many optimizations is to avoid slow data paths."
- **Quantitative Latency & Bandwidth Hierarchy Table:**
  | Memory Level | Typical Latency (s) | Typical Bandwidth (bytes/s) |
  | :--- | :--- | :--- |
  | **Core Registers** | $< 1$ cycle | Extremely high |
  | **L1 Cache** | $10^{-9}$ s (~1 ns / ~4 cycles) | $10^{12}$ B/s (~1 TB/s) |
  | **L2 Cache** | Few ns | High |
  | **L3 Cache** | $10^{-8}$ s (~10 ns) | Intermediate |
  | **Main Memory (DRAM)** | $10^{-7}$ s (~100 ns) | $10^{11}$ B/s (~100 GB/s) |
  | **Disk Storage** | $10^{-4}$ s (~0.1–1 ms for SSD) | $10^9$ B/s (~1 GB/s) |

---

## Page 08

### Topic: Characterization of Data Paths & Flynn's Taxonomy
- **Basic Data Transfer Time Model:**
  $$T = \lambda + \frac{N}{b}$$
  - $T$: Total transfer time for a message of $N$ bytes
  - $\lambda$: Latency (startup/setup time) in seconds [s]
  - $b$: Asymptotic bandwidth of the data path in [bytes/s]
  - $N$: Message size in bytes
- **Effective Bandwidth ($b_{\text{eff}}$):**
  $$b_{\text{eff}} = \frac{N}{T} = \frac{N}{\lambda + \frac{N}{b}}$$
- **Definitions:**
  - **Parallel computing:** A number of compute elements solve a problem in a cooperative way.
  - **Parallel computers:** A number of compute elements connected in such a way as to do parallel computing for a large set of applications.
  - **Flynn's Classification:** Michael J. Flynn (1972) (DOI: 10.1109/TC.1972.5009071).

---

## Page 09

### Topic: Flynn's 2x2 Matrix & Modern Multi-Core Chip Architecture
- **Flynn's 2x2 Taxonomy Table:**
  | | Single Instruction (SI) | Multiple Instruction (MI) |
  | :--- | :--- | :--- |
  | **Single Data (SD)** | **SISD** (Traditional Von Neumann stored-program computer) | **MISD** (Systolic arrays, fault-tolerant redundant computing) |
  | **Multiple Data (MD)** | **SIMD** (Vector instructions: SSE, AVX, GPUs) | **MIMD** (Shared Memory & Distributed Memory) |
- **MIMD Sub-classification:**
  - **Shared Memory:** Single cache-coherent address space.
  - **Distributed Memory:** No cache-coherent address space (clusters, supercomputers).
  - Modern supercomputers are hybrid shared/distributed memory architectures.
- **Multi-Core Processor Architecture Diagram:**
  - CPU Chip contains 8 Cores (Processors $P$).
  - Each Core has SMT threads ($T_1, T_2$), private L1D & L1I cache, private L2 cache.
  - All 8 Cores share an on-chip L3 Cache.
  - L3 connects via Memory Interface to off-chip Memory.
  - 2 chips = 1 dual-socket node (16 cores total).

---

## Page 10

### Topic: UMA vs ccNUMA Architectures & OpenMP Directives
- **UMA (Uniform Memory Access):**
  - All processors access all memory with equal latency and bandwidth.
  - Single address space for all cores.
  - Typically found within a single multicore chip.
- **ccNUMA (Cache-Coherent Non-Uniform Memory Access):**
  - Multiple sockets/domains where memory is physically distributed but logically shared in a single cache-coherent address space.
  - Latency and bandwidth vary depending on mutual distance between the core and the physical memory bank (local vs. remote memory).
  - **Advantage:** Easier to build scalable systems with multiple domains than one massive UMA bus.
  - **Disadvantage:** Introduces NUMA topology complexity and potential remote memory latency penalties.
- **Shared Memory Programming with OpenMP:**
  - Directives interpreted by compiler.
  - Syntax breakdown:
    - Sentinel: `#pragma omp`
    - Parallelization directive: `parallel`
    - Work-sharing directive: `for`
    - Clause: `reduction(+: s)`
  - Code Example:
    ```c
    #pragma omp parallel for reduction(+: s)
    for (int i=0; i<n; i++) {
        a[i] = b[i] + c[i];
        s = s + a[i];
    }
    ```

---

## Page 11

### Topic: Distributed Memory Systems ("Back in the Day" vs Today)
- "Multithreading is easier in OpenMP than in general purpose languages."
- **"Pure" Distributed Memory System (Historical):**
  - Individual processors $P$ with cache $C$, local memory $M$, and network interface $NI$.
  - Interconnected via communication network.
  - Network protocols: TCP/IP and UDP/IP.
  - Parallel program == one process per node.
  - Data exchange via explicit **message passing** over the network.
- **Distributed Memory Systems Today (Modern HPC Clusters):**
  - Cluster of networked shared-memory nodes.
  - Each node features ccNUMA architecture with multi-core CPUs.
  - Hierarchy of latency ($\lambda$) and bandwidth ($b$):
    - Intra-socket (core-to-core on same chip)
    - Inter-socket (chip-to-chip on same board)
    - Inter-node (node-to-node across interconnect fabric: InfiniBand, Ethernet)
  - Network structure topology influences communication performance.

---

## Page 12

### Topic: Parallel Computing Limits — Speedup, Efficiency & Scaling Metrics
- **Performance Metrics to Quantify Parallel Computing:**
  - $T(N)$: Execution time of some fixed workload with $N$ workers.
  - **Parallel Speedup:**
    $$S(N) = \frac{T(1)}{T(N)}$$
    (Answers: "How much faster than with a single worker?")
  - **Parallel Efficiency:**
    $$E(N) = \frac{S(N)}{N} = \frac{T(1)}{N \cdot T(N)}$$
    (Answers: "How efficiently do those $N$ workers do their work?")
- **Critical Conceptual Distinction:**
  - "Warning: These metrics are not performance metrics."
  - Performance is absolute work done per unit time ($P = \frac{\text{Work}}{\text{Time}}$). Speedup and efficiency are relative scalability metrics comparing parallel execution to serial execution.
