# Master Reference Knowledge Base: Complete Content Extraction

**Scope:** Unified academic content extracted across all 5 primary sources (90 pages total)  
**Primary Sources:**
- `Sanjay_Notes_1.pdf` (12 pages)
- `3d773f8d-2e9e-426e-8ef8-0f93e447813a.pdf` (26 pages)
- `Pthreada_Aagam_Notes.pdf` (20 pages)
- `IMG_1333.pdf` (27 pages)
- `Document 12.pdf` (5 pages)

---

# MODULE 1: HARDWARE FOUNDATIONS & PROCESSOR MICROARCHITECTURE

## 1. Computational Performance & The Von Neumann Computer
- **Performance Definition:**
  $$P = \frac{\text{Work}}{\text{Time}}$$
  - **Work:** Scientific/computational units: FLOPs (Floating Point Operations: addition, subtraction, multiplication, division), lattice site updates, matrix elements calculated.
  - **Time:** Real physical elapsed time ("wall-clock time").
- **Stored-Program / Von Neumann Computer:**
  - Components: CPU (Arithmetic/Logic Unit + Control Unit) connected to Memory and I/O.
  - **The Memory Wall Problem:** Memory access exhibits high latency and low bandwidth compared to ALU speed. CPU executes instructions rapidly but is starved waiting for data.
- **Instruction Execution vs Useful Work:**
  - Instructions are the concept of "work" as seen by processor designers.
  - Not all executed instructions represent useful "work" as seen by application developers (e.g., loop overhead, index calculations, address arithmetic, register shuffling).

---

## 2. In-Core Microarchitectural Performance Pillars
Modern CPUs achieve high performance through four fundamental microarchitectural techniques:

### A. Pipelining
- **Working Principle:** Split a complex instruction into multiple simple, balanced stages (each taking 1 clock cycle). Multiple instructions overlap across different stages simultaneously.
- **5-Stage Floating-Point Multiplier Walkthrough:**
  $$A(i) = B(i) \times C(i) \quad \text{for } i = 1, \dots, N$$
  Example values: $3 \times 10^{12} \times 5 \times 10^9$
  - Stage 1: Separate mantissa and exponent ($3, 10^{12}$ and $5, 10^9$).
  - Stage 2: Multiply mantissas ($3 \times 5 = 15$).
  - Stage 3: Add exponents ($12 + 9 = 21$).
  - Stage 4: Normalize result ($15 \times 10^{21} \rightarrow 1.5 \times 10^{22}$).
  - Stage 5: Insert sign bit.
- **Latency vs Throughput:**
  - Pipeline latency: 5 clock cycles (time for 1 instruction to pass through).
  - Wind-up phase: 4 cycles to fill empty stages.
  - Steady-state throughput: 1 instruction completed per cycle once pipeline is full.
- **Instruction Pipeline:** Fetch from L1 $\rightarrow$ Decode $\rightarrow$ Execute. Can stall due to branch mispredictions (speculative execution and branch predictors mitigate this).

### B. Superscalar Execution & Out-of-Order (OoO) Processing
- Multiple execution units enable Instruction-Level Parallelism (ILP).
- Instructions are dynamically analyzed and parallelized on the fly.
- Hardware structure:
  - L1 Instruction Cache $\rightarrow$ 4 Decoders $\rightarrow$ Reorder Buffer / Register Renaming Scheduler.
  - 6 Parallel Execution Ports:
    - Port 0: ALU, MUL, DIV
    - Port 1: ALU, ADD
    - Port 2: LOAD
    - Port 3: LOAD ADDRESS
    - Port 4: STORE ADDRESS
    - Port 5: ALU, JUMP
- An $m$-way superscalar processor can issue $m$ concurrent instructions per cycle (typically 3 to 6 instructions, including up to 2 FP instructions per cycle).

### C. Simultaneous Multi-Threading (SMT / Hyper-Threading)
- Solves the problem of pipeline underutilization ("pipeline bubbles") caused by memory latency, branch mispredictions, or instruction dependencies.
- Hardware provides multiple architectural thread contexts (separate register sets, program counters) sharing the same execution units, execution ports, and caches.
- Does not add more computational execution units, but increases hardware utilization.

### D. Single Instruction Multiple Data (SIMD) Vectorization
- Executes the same operation simultaneously on wide vector registers containing multiple data elements.
- **x86 SIMD Register Widths & Double-Precision (DP, 64-bit) Capacities:**
  - **SSE:** 128-bit wide $\rightarrow$ 2 double-precision floating-point numbers.
  - **AVX / AVX2:** 256-bit wide $\rightarrow$ 4 double-precision floating-point numbers.
  - **AVX-512:** 512-bit wide $\rightarrow$ 8 double-precision floating-point numbers.
- **Compiler Vectorization via Loop Unrolling:**
  Scalar loop:
  ```c
  for (int i=0; i<n; i++) C[i] = A[i] + B[i];
  ```
  Vectorized loop (unrolled by factor of 4 for AVX):
  ```c
  for (int i=0; i<n; i+=4) {
      C[i]   = A[i]   + B[i];
      C[i+1] = A[i+1] + B[i+1];
      C[i+2] = A[i+2] + B[i+2];
      C[i+3] = A[i+3] + B[i+3];
  }
  ```

---

## 3. Master Peak Core Performance Formula
The theoretical peak floating-point performance of a single processor core is governed by:
$$\boxed{P_{\text{core}} = n_{\text{superscalar}}^{\text{FP}} \times n_{\text{FMA}} \times n_{\text{SIMD}} \times f}$$
- $n_{\text{superscalar}}^{\text{FP}}$: Number of FP execution units/ports issuing instructions per cycle.
- $n_{\text{FMA}}$: Fused Multiply-Add factor ($= 2$, because an FMA instruction executes 1 multiply + 1 addition = 2 FLOPs).
- $n_{\text{SIMD}}$: Vectorization factor (number of operands packed into vector registers; e.g. 4 for AVX2 DP, 8 for AVX-512 DP).
- $f$: Clock frequency (in Hz / GHz).

---

## 4. Memory Hierarchy & Data Path Transfer Model
- Quantitative Latency and Bandwidth Hierarchy:
  - **L1 Cache:** $\approx 10^{-9}\text{ s}$ (1 ns), Bandwidth $\approx 10^{12}\text{ B/s}$ (1 TB/s).
  - **L2 Cache:** A few nanoseconds.
  - **L3 Cache (Shared):** $\approx 10^{-8}\text{ s}$ (10 ns).
  - **Main Memory (DRAM):** $\approx 10^{-7}\text{ s}$ (100 ns), Bandwidth $\approx 10^{11}\text{ B/s}$ (100 GB/s).
  - **Disk Storage (SSD/NVMe):** $\approx 10^{-4}\text{ s}$ (0.1–1 ms), Bandwidth $\approx 10^{9}\text{ B/s}$ (1 GB/s).
- Cache lines are transferred in fixed bursts (typically 64 bytes).
- **Analytical Data Path Transfer Model:**
  $$\boxed{T = \lambda + \frac{N}{b}}$$
  - $T$: Total transfer time for a message of $N$ bytes.
  - $\lambda$: Latency (connection startup/setup time) [seconds].
  - $b$: Asymptotic bandwidth of the data channel [bytes/second].
  - $N$: Message size [bytes].
- **Effective Bandwidth ($b_{\text{eff}}$):**
  $$\boxed{b_{\text{eff}} = \frac{N}{T} = \frac{N}{\lambda + \frac{N}{b}}}$$
  - For small messages ($N \ll \lambda \cdot b$), transfer time is dominated by latency $\lambda$.
  - For large messages ($N \gg \lambda \cdot b$), effective bandwidth approaches peak bandwidth $b$.

---

# MODULE 2: PARALLEL ARCHITECTURES & MEMORY SYSTEMS

## 1. Flynn's Classical Taxonomy (1972)
Classifies computers based on instruction stream and data stream:
| | Single Instruction (SI) | Multiple Instruction (MI) |
| :--- | :--- | :--- |
| **Single Data (SD)** | **SISD** (Traditional Von Neumann uniprocessor) | **MISD** (Systolic arrays, specialized fault-tolerant systems) |
| **Multiple Data (MD)** | **SIMD** (Vector processors, GPUs, x86 SSE/AVX extensions) | **MIMD** (Multiprocessors, multicore systems, clusters, supercomputers) |

---

## 2. MIMD Shared Memory: UMA vs ccNUMA
- **Shared Memory Definition:** Multiple processors or cores share a single, unified address space. Communication occurs implicitly through read/write operations to shared memory locations.
- **UMA (Uniform Memory Access):**
  - Equal access time, latency, and bandwidth from any core to any physical memory location.
  - Typical implementation: Single multicore CPU chip with on-die memory controller.
- **ccNUMA (Cache-Coherent Non-Uniform Memory Access):**
  - Multiple sockets/domains where physical memory is distributed among processor nodes, but logically presented as a single address space.
  - Memory local to a core exhibits lower latency and higher bandwidth than remote memory attached to another socket.
  - Hardware maintains cache coherence automatically across sockets.
  - **Advantage:** Scalability (easier to build multiple modest domains than one monolithic UMA bus).
  - **Disadvantage:** Memory placement topology impacts application runtime.

---

## 3. Distributed Memory Architectures
- **Distributed Memory Definition:** Independent compute nodes, each with private memory and network interface ($NI$). No global address space.
- **Historical Pure Systems:** Independent single-core nodes communicating strictly over TCP/IP or UDP/IP via explicit message passing.
- **Modern HPC Hybrid Clusters:**
  - Hierarchical structure: Nodes connected via high-speed network fabrics (InfiniBand, Omni-Path).
  - Each node is a multi-socket ccNUMA system containing multi-core processors.
  - Hierarchy of communication costs:
    $$\lambda_{\text{intra-socket}} < \lambda_{\text{inter-socket}} < \lambda_{\text{inter-node}}$$
    $$b_{\text{intra-socket}} > b_{\text{inter-socket}} > b_{\text{inter-node}}$$

---

# MODULE 3: PERFORMANCE ANALYSIS, SCALING LAWS & BOTTLENECK MODELING

## 1. Speedup & Efficiency Formalisms
- Let $T_1$ (or $T(1)$) be the execution time on 1 worker, and $T_p$ (or $T(N)$) be the execution time on $p$ (or $N$) workers.
- **Parallel Speedup:**
  $$\boxed{S(p) = \frac{T_1}{T_p}}$$
- **Ideal Linear Speedup:** $S(p) = p$.
- **Real-World Sub-Linear Speedup:** $S(p) < p$ due to sequential portions, synchronization barriers, communication delays, memory bandwidth contention, and thread overhead.
- **Parallel Efficiency:**
  $$\boxed{E(p) = \frac{S(p)}{p} = \frac{T_1}{p \cdot T_p}}$$
  - Measures the fraction of time processors spend doing useful computational work compared to ideal parallelization.

---

## 2. Amdahl's Law (Strong Scaling)
- **Premise:** Fixed problem size executed on an increasing number of processors ($p$).
- **Derivation:**
  - Let total serial time $T_1 = 1$.
  - Let $s$ be the fraction of the program that is strictly sequential ($0 \le s \le 1$).
  - The parallelizable fraction is $(1 - s)$.
  - Time on $p$ processors:
    $$T_p = s + \frac{1 - s}{p}$$
  - Speedup:
    $$\boxed{S(p) = \frac{1}{s + \frac{1 - s}{p}}}$$
- **Asymptotic Speedup Limit ($p \to \infty$):**
  $$\boxed{S_{\max} = \lim_{p \to \infty} S(p) = \frac{1}{s}}$$
- **Parallel Efficiency under Amdahl's Law:**
  $$\boxed{\mathcal{E}(p) = \frac{S(p)}{p} = \frac{1}{s(p - 1) + 1}}$$
  - As $p \to \infty$, efficiency decays asymptotically to zero: $\lim_{p \to \infty} \mathcal{E}(p) = 0$.

---

## 3. Communication Overhead & The Linear Overhead Failure Model
- When communication and synchronization overhead $C(p)$ is included:
  $$T(p) = s + \frac{1 - s}{p} + C(p)$$
- **The Hazardous Linear Overhead Model ($C(p) = k \cdot p$):**
  $$S(p) = \frac{1}{s + \frac{1 - s}{p} + k \cdot p}$$
  - As $p$ increases, the term $k \cdot p$ dominates the denominator.
  - As $p \to \infty$, speedup drops catastrophically to zero:
    $$\lim_{p \to \infty} S(p) = 0$$
- **Engineering Mitigations:** Non-blocking asynchronous network communications, overlapping communication with compute, coarsening task granularity.

---

## 4. Gustafson-Barsis' Law (Weak Scaling)
- **Premise:** Workload is scaled proportionally with the number of processors ($p$). The time spent in the parallel portion grows with $p$, keeping per-processor workload constant.
- **Derivation:**
  - Let parallel execution time on $p$ processors be normalized: $T_p = s' + (1 - s') = 1$, where $s'$ is the fraction of time spent in serial code on $p$ processors.
  - On 1 processor, the parallel work would take $p \times (1 - s')$.
  - Total time on 1 processor: $T_1 = s' + p(1 - s')$.
  - Scaled Speedup:
    $$\boxed{S(p) = \frac{T_1}{T_p} = s' + p(1 - s') = p - s'(p - 1)}$$
  - Alternatively expressed in source notes:
    $$\boxed{S(N) = s + (1 - s)N}$$
- **Insight:** Overcomes Amdahl's limit because the parallel workload scales up with system size, enabling near-linear speedups on massively parallel supercomputers.

---

## 5. Hardware Resource Bottlenecks & The Roofline DAXPY Model
- Theoretical execution time with multiple independent resource constraints:
  $$\boxed{T_{\min} = \max(T_1, T_2, \dots, T_n)}$$
- **DAXPY Numerical Problem from Notes:**
  $$Y[i] = a \cdot X[i] + Y[i] \quad \text{for } 10^7 \text{ double-precision elements}$$
  - **Workload:**
    - Floating-Point Operations: 1 multiply + 1 add = 2 FLOPs per iteration $\implies W_{\text{flops}} = 2 \times 10^7\text{ FLOPs} = 0.02\text{ GFLOPs}$.
    - Memory Traffic: Read $X[i]$ (8 B), Read $Y[i]$ (8 B), Write $Y[i]$ (8 B) = 24 Bytes per iteration $\implies W_{\text{data}} = 24 \times 10^7\text{ B} = 240\text{ MB}$.
  - **Hardware Parameters:**
    - Peak Compute: $R_{\text{flops}}^{\max} = 192\text{ GFLOP/s}$.
    - Peak Memory Bandwidth: $R_{\text{BW}}^{\max} = 40\text{ GB/s}$.
  - **Time Requirements:**
    - Compute time: $T_{\text{flops}} = \frac{0.02}{192} \approx 104\ \mu\text{s}$.
    - Memory transfer time: $T_{\text{BW}} = \frac{0.240\text{ GB}}{40\text{ GB/s}} = 6\text{ ms} = 6000\ \mu\text{s}$.
  - **Bottleneck Resolution:**
    $$T_{\min} = \max(0.104\text{ ms}, 6.000\text{ ms}) = 6\text{ ms}$$
  - **Attainable Performance:**
    $$P = \frac{0.02\text{ GFLOPs}}{0.006\text{ s}} = 3.33\text{ GFLOP/s}$$
  - **Core Takeaway:** The kernel achieves only $\frac{3.33}{192} \approx 1.73\%$ of theoretical compute peak because performance is 100% memory-bandwidth bound. Adding more compute units produces zero speedup!

---

## 6. Benchmarking Methodologies
- Three categories of performance benchmarks:
  1. **Application Benchmarks:** Full, production-scale scientific codes. High real-world fidelity, but difficult to port and profile.
  2. **Micro-benchmarks:** Synthetic, isolated codes targeting a single hardware subsystem (e.g. STREAM benchmark for memory bandwidth, Linpack for dense matrix multiplication).
  3. **Mini-Apps / Proxy Apps:** Compact extracts of representative computational kernels from large applications (e.g., Mantevo suite).
- **The Performance vs Speedup Fallacy:**
  - A slow, inefficient sequential code ported to a cluster can show "superb" 100x speedup because the baseline was atrocious.
  - A highly optimized vector supercomputer (like NEC SX) may show only 2x speedup on the same problem, but its absolute execution time is 10x faster than the cluster!
  - Always evaluate absolute performance ($P = \text{Work}/\text{Time}$) alongside relative speedup.

---

# MODULE 4: SHARED-MEMORY PROGRAMMING WITH PTHREADS

## 1. POSIX Standards & System Conformance
- POSIX: Portable Operating System Interface (IEEE standard).
- Guarantees portable multithreading across UNIX-like environments.
- Systems: Certified (macOS); Compliant via GNU C Library (Linux distributions); BSD operating systems; Android (Bionic C library).
- Pthreads provides a low-level, explicit thread management model.

---

## 2. Pthreads Lifecycle & Core APIs
- Header: `#include <pthread.h>`
- Compilation: `gcc -pthread program.c -o program`
- **Thread Handle:** `pthread_t` (opaque data type).
- **Thread Creation:**
  ```c
  int pthread_create(pthread_t *thread, const pthread_attr_t *attr,
                     void *(*start_routine)(void *), void *arg);
  ```
- **Thread Termination & Joining:**
  ```c
  void pthread_exit(void *retval);
  int pthread_join(pthread_t thread, void **retval);
  ```
- **Thread Attributes (`pthread_attr_t`):**
  - Customized prior to creation.
  - Stack size configuration (`pthread_attr_setstacksize`).
  - Detach state: Joinable (`PTHREAD_CREATE_JOINABLE`) vs Detached (`PTHREAD_CREATE_DETACHED`).
  - Scheduling policy:
    - `SCHED_OTHER`: Standard time-sharing round-robin.
    - `SCHED_FIFO`: Real-time FIFO priority.
    - `SCHED_RR`: Real-time round-robin priority.

---

## 3. Pthreads Synchronization Suite
1. **Mutex (`pthread_mutex_t`):**
   - Mutual exclusion lock protecting critical sections.
   - Functions: `pthread_mutex_init`, `pthread_mutex_lock`, `pthread_mutex_trylock`, `pthread_mutex_unlock`, `pthread_mutex_destroy`.
2. **Condition Variables (`pthread_cond_t`):**
   - Coordinates thread state changes; always paired with a mutex.
   - Functions: `pthread_cond_wait(&cond, &mutex)`, `pthread_cond_signal(&cond)`, `pthread_cond_broadcast(&cond)`.
   - Always checked in a `while` loop to guard against spurious wakeups.
3. **Producer-Consumer Bounded Buffer:**
   - Uses 1 mutex and 2 condition variables (`not_full`, `not_empty`).
   - Producer waits when `count == SIZE`; Consumer waits when `count == 0`.
4. **POSIX Semaphores (`<semaphore.h>`):**
   - Counting semaphore: `sem_t`.
   - Real-time extension; `sem_init(&sem, 0, initial_value)`.
   - `sem_wait(&sem)` (decrement/block), `sem_post(&sem)` (increment/signal).
5. **Read-Write Locks (`pthread_rwlock_t`):**
   - Multiple concurrent readers (`pthread_rwlock_rdlock`) or one exclusive writer (`pthread_rwlock_wrlock`).
   - Blackboard analogy: everyone can read simultaneously; writer blocks all reads and writes.
6. **Barriers (`pthread_barrier_t`):**
   - Rendezvous synchronization point initialized with a thread count:
     `pthread_barrier_init(&barrier, NULL, NUM_THREADS)`.
   - Threads block at `pthread_barrier_wait(&barrier)` until all threads arrive.
7. **Spinlocks (`pthread_spinlock_t`):**
   - Busy-waits in a CPU loop without putting thread to sleep.
   - `pthread_spin_init(&lock, PTHREAD_PROCESS_PRIVATE)`.
   - `pthread_spin_lock(&lock)`, `pthread_spin_unlock(&lock)`.

---

## 4. Mutex vs Spinlock: Engineering Analysis & The Contention Cliff
| Feature | Mutex (Blocking Lock) | Spinlock (Busy-Wait Lock) |
| :--- | :--- | :--- |
| **Wait Mechanism** | OS deschedules thread; puts it to sleep | Spins in tight loop testing lock state |
| **Context Switch Overhead**| Incurs 2 context switches (~2–5 $\mu$s) | Zero context switch overhead |
| **CPU Utilization** | Zero CPU cycles burned while waiting | 100% of core burned spinning |
| **Ideal Workload** | Long critical sections ($> \text{context switch}$) | Extremely short critical sections ($< 10^2$ cycles) |
- **Contention Curve Dynamics:**
  - At low critical section lengths ($10^0$ to $10^2$ CPU cycles), spinlocks achieve higher throughput (~13.5M ops/s vs ~8M ops/s) by avoiding OS context switches.
  - At $\approx 10^2$ cycles under multi-core contention, the spinlock curve plunges into a catastrophic cliff (dropping to ~2.5M ops/s) due to cache-line bouncing and wasted CPU cycles.
  - Engineering rule: Use spinlocks only for verified microsecond-scale non-blocking operations on dedicated cores. Otherwise, use mutexes.

---

# MODULE 5: SHARED-MEMORY PROGRAMMING WITH OPENMP

## 1. Architecture & Execution Model
- **Definition:** Standardized API for shared-memory parallel programming across C, C++, and Fortran using compiler directives (`#pragma omp`), runtime library routines, and environment variables.
- **Layers:**
  $$\text{Application Code} \xrightarrow{\text{Pragmas}} \text{OpenMP Runtime Library} \xrightarrow{\text{OS Threads}} \text{Hardware CPU Cores}$$
- **Fork-Join Model:**
  - Serial execution on Master Thread (Thread 0).
  - `#pragma omp parallel`: Master thread forks a team of worker threads.
  - Enclosed structured block executed concurrently by all team members.
  - Implicit barrier at end of parallel region: threads join and synchronize before master thread resumes serial execution.
- **Compilation:**
  - GCC/Clang: `-fopenmp`
  - Version macro: `_OPENMP` (e.g., date code integer `201511L`).
  - Fallback: Non-supporting compilers ignore `#pragma` directives cleanly.

---

## 2. Thread Management & Control APIs
- Header: `#include <omp.h>`
- `omp_get_thread_num()`: Returns unique thread ID ($0 \le \text{ID} < p$).
- `omp_get_num_threads()`: Returns current number of threads in team.
- `omp_set_num_threads(N)`: Requests $N$ threads for subsequent parallel regions.
- Clause: `#pragma omp parallel num_threads(4)`: Explicitly sets team size.
- Environment variable: `export OMP_NUM_THREADS=8`.

---

## 3. Data Scoping Architecture & Scoping Rules
- **Process Memory Layout:**
  - Global/Heap: Shared by all threads.
  - Stacks: Each thread has its own private execution stack.
  - Shell stack size control: `export OMP_STACKSIZE=64M`.
- **Default Scoping Rules:**
  - Variables declared outside the parallel construct are **shared** by default.
  - Variables declared inside the parallel block are **private** by default (allocated on thread stack).
  - Loop index variables in parallel loops are **private** by default.
  - Static variables inside parallel blocks remain **shared**.
- **Scoping Clauses:**
  - `shared(var)`: One common instance in shared memory.
  - `private(var)`: Each thread allocates an uninitialized local copy on its stack.
    - *Critical Warning:* Private variables have **undefined initial values** inside the construct, and their value becomes **undefined outside** upon join!
  - `firstprivate(var)`: Private copy initialized with the variable's value from before the construct.
  - `lastprivate(var)`: Copies the value from the sequentially last loop iteration back to the outer variable.
  - `default(none)`: Forces programmer to explicitly scope every variable, preventing race conditions.

---

## 4. Loop Work-Sharing & Scheduling Policies
- Work-sharing construct: `#pragma omp parallel for` divides loop iterations across threads.
- Requirements: Loop must have a canonical form (known iteration count at loop entry; no premature `break`/`goto`).
- **The 4 Scheduling Policies:**
  1. `schedule(static [, chunk])`:
     - Predetermined, compile-time/launch-time round-robin chunk assignment.
     - Lowest runtime scheduling overhead.
     - Best suited for uniform, predictable workloads.
     - *Trap:* If `chunk_size * num_threads > total_iterations`, trailing threads remain completely idle!
  2. `schedule(dynamic [, chunk])`:
     - Centralized work queue; idle threads request new chunks dynamically.
     - Solves load imbalance for irregular workloads (e.g. `sleep(i % 3)`).
     - Higher runtime synchronization overhead per chunk request.
  3. `schedule(guided [, min_chunk])`:
     - Chunks start large to minimize overhead, then shrink exponentially to allow fine-grained load balancing near the end.
     - Balances low overhead with effective load balancing.
  4. `schedule(auto)`:
     - Delegates scheduling decision completely to compiler/runtime heuristics.

---

## 5. Synchronization & Coordination Directives
1. **Critical Section (`#pragma omp critical [name]`):**
   - Enforces mutual exclusion across all threads for a code block.
   - High software overhead; serializes execution.
2. **Atomic Memory Update (`#pragma omp atomic`):**
   - Hardware-level atomic instruction (e.g. `LOCK XADD` on x86).
   - Restricted to simple scalar updates (`x++`, `x += expr`).
   - Vastly superior performance to `critical`.
3. **Barrier (`#pragma omp barrier`):**
   - Explicit team synchronization checkpoint. No thread proceeds until all threads arrive.
4. **Single Construct (`#pragma omp single [nowait]`):**
   - Code block executed by exactly one thread in the team.
   - Has an implicit barrier at the end unless `#pragma omp single nowait` is specified.
5. **Master Construct (`#pragma omp master`):**
   - Executed strictly by Thread 0; has **no** implicit barrier.
6. **Ordered Construct (`#pragma omp ordered`):**
   - Forces execution within a parallel loop to occur in strictly sequential iteration order.
7. **Reduction Clause (`reduction(operator: variable)`):**
   - Eliminates race conditions on shared accumulation variables.
   - Syntax: `#pragma omp parallel for reduction(+: sum)`
   - Each thread maintains a private accumulator initialized to the operator's identity value ($0$ for $+$, $1$ for $*$).
   - At loop completion, private accumulators are combined into the global variable using the specified associative/commutative operator.

---

## 6. Advanced OpenMP: Tasks & SIMD Vectorization
- **Task Parallelism (`#pragma omp task`):**
  - Designed for irregular algorithms, recursive tree traversal, while-loops, and pointer chasing.
  - Tasks represent independent work units packaged and placed into a runtime task pool.
  - Typically generated by a single producer thread:
    ```c
    #pragma omp parallel
    {
        #pragma omp single
        {
            #pragma omp task
            process_node(root->left);
            #pragma omp task
            process_node(root->right);
        }
    }
    ```
- **Task Loops (`#pragma omp taskloop`):**
  - Automatically slices loop iterations into discrete task units.
  - Granularity clauses: `grainsize(G)` or `num_tasks(K)` prevent task pool flooding on massive loops.
- **SIMD Vectorization Directives (`#pragma omp simd`):**
  - Instructs compiler to vectorize the loop using vector registers (SSE/AVX/AVX-512) without creating multiple threads.
  - GCC vectorization flags: `-O3 -fopenmp -fopt-info-vec -march=native`.
  - Combined construct: `#pragma omp parallel for simd` (multithreading across cores + SIMD vector lanes within each core).
  - **Latent Concurrency Bug Hazard:** A temporary variable inside a SIMD loop missing a `private(tmp)` clause may pass tests with `-O0` or non-vectorizing targets, but produces corrupt values when compiled with `-O3 -march=native`.

---

# MODULE 6: DISTRIBUTED-MEMORY PROGRAMMING WITH MPI

## 1. Architecture & Execution Model
- **Message Passing Model:** Independent processes with private address spaces communicate across an interconnect fabric via explicit message passing.
- **Implementations:** OpenMPI, MPICH, Intel MPI, MS-MPI.
- **SPMD (Single Program, Multiple Data):** Identical executable launched on $N$ processes; behavior differentiates based on process rank.

---

## 2. Core MPI API & Lifecycle
- Header: `#include <mpi.h>`
- **Initialization & Finalization:**
  ```c
  int MPI_Init(int *argc, char ***argv);
  int MPI_Finalize(void);
  ```
- **Communicator Inspection:**
  - `MPI_COMM_WORLD`: Default predefined communicator containing all processes.
  - `MPI_Comm_rank(MPI_COMM_WORLD, &rank)`: Returns ID of calling process ($0 \le \text{rank} < \text{size}$).
  - `MPI_Comm_size(MPI_COMM_WORLD, &size)`: Returns total number of processes.
- **Execution Management:**
  - Compilation: `mpicc program.c -o program`
  - Launch: `mpirun -np 4 ./program`
  - Flags:
    - `--oversubscribe`: Allows launching more MPI processes than physical CPU cores.
    - `--display-allocation`: Shows core/socket process bindings.
    - `--hostfile <file>`: Specifies cluster node network addresses.

---

## 3. Point-to-Point Communication
- **Standard Blocking Send:**
  ```c
  int MPI_Send(const void *buf, int count, MPI_Datatype datatype,
               int dest, int tag, MPI_Comm comm);
  ```
- **Standard Blocking Receive:**
  ```c
  int MPI_Recv(void *buf, int count, MPI_Datatype datatype,
               int source, int tag, MPI_Comm comm, MPI_Status *status);
  ```
- **Message Envelope:** Buffer address, count, MPI datatype (`MPI_INT`, `MPI_DOUBLE`, etc.), source/dest rank, integer tag, communicator handle, status pointer (`MPI_STATUS_IGNORE`).

---

## 4. Collective Communication Overview
- Involves all processes in a communicator simultaneously.
- **Broadcasting:** `MPI_Bcast` (Rank 0 sends data to all processes).
- **Reduction:** `MPI_Reduce` (Gathers and combines values using an operation like `MPI_SUM`, `MPI_MAX`).
- **Scatter / Gather:** `MPI_Scatter` (splits array into chunks across ranks); `MPI_Gather` (re-assembles chunks).
- **Barrier:** `MPI_Barrier(MPI_COMM_WORLD)` (global synchronization).

---

## 5. MPI-2 Dynamic Process Connections & Client-Server Architecture
- Connects independently started MPI programs without a pre-existing `MPI_COMM_WORLD`.
- **Server Workflow (`sender.c`):**
  1. Initialize MPI: `MPI_Init(&argc, &argv)`.
  2. Open communication port: `MPI_Open_port(MPI_INFO_NULL, port)`.
  3. Publish port name string (write to `port.txt` or publish via name server).
  4. Wait for client: `MPI_Comm_accept(port, MPI_INFO_NULL, 0, MPI_COMM_SELF, &client)`.
  5. Exchange data via inter-communicator: `MPI_Send(&data, 1, MPI_INT, 0, 0, client)`.
  6. Disconnect & cleanup: `MPI_Comm_disconnect(&client)`, `MPI_Close_port(port)`, `MPI_Finalize()`.
- **Client Workflow (`receiver.c`):**
  1. Initialize MPI: `MPI_Init(&argc, &argv)`.
  2. Read server's port from `port.txt`.
  3. Connect to server: `MPI_Comm_connect(port, MPI_INFO_NULL, 0, MPI_COMM_SELF, &server)`.
  4. Receive data: `MPI_Recv(&data, 1, MPI_INT, 0, 0, server, &status)`.
  5. Disconnect & cleanup: `MPI_Comm_disconnect(&server)`, `MPI_Finalize()`.
- **Open MPI Name Server (`ompi-server`):**
  - Centralized lookup daemon (acts like a DNS service for MPI ports).
  - Startup command:
    ```bash
    ompi-server --no-daemonize -report-uri /tmp/ompi-server.uri
    ```
  - Launching processes connected to the server URI:
    ```bash
    mpirun -mca pmix_server_uri "$(cat /tmp/ompi-server.uri)" -np 1 ./sender
    ```
