# Source Extraction: 3d773f8d-2e9e-426e-8ef8-0f93e447813a.pdf

**Source File:** `input_notes/3d773f8d-2e9e-426e-8ef8-0f93e447813a.pdf`  
**Total Pages:** 26  
**Primary Topics:** Parallel Computing Limits & Scalability Models (Speedup, Efficiency, Amdahl's Law Strong Scaling, Linear Communication Overhead Models, Weak Scaling, Gustafson's Law, Scalability Curve Fitting & Node Boundaries, Load Imbalance), Hardware Resource Bottlenecks & Roofline / Execution Time Modeling (Memory Bandwidth vs Compute Limits, DAXPY Light Speed Calculation), Benchmarking Methodologies (Application vs Micro-benchmarking, Mini-apps), OpenMP Architecture & Foundations (Fork-Join Model, Compiler Directives & Environment, Shared vs Private Scoping, Stack vs Heap Architecture, Masking, `firstprivate`), OpenMP Worksharing & Loop Scheduling (`schedule(static)`, `schedule(dynamic)`, `schedule(guided)`, `schedule(auto)`), Synchronization & Mutual Exclusion (Critical Sections, Atomic Operations, Barriers, Single Construct, `nowait`, Ordered Directive, Reduction Clauses), OpenMP Task Parallelism (`#pragma omp task`, `#pragma omp taskloop`, Task Pool Mechanics, `num_tasks`, `grainsize`), OpenMP SIMD Vectorization (`#pragma omp simd`, Vector Register Architectures AVX/AVX-512/SSE, GCC Vectorization Flags, Latent Vectorization Bugs, Combined `#pragma omp parallel for simd`, SIMD Reductions).

---

## Page 01

### Topic: Parallel Computing and Its Limits & Basic Scalability Metrics
- **Main Heading:** Parallel Computing and Its Limits
- **Subheading:** Metrics to quantify the efficiency of parallel computing.

#### Fundamental Definitions & Mathematical Formulations
- **Execution Time $T(N)$:**
  - $T(N)$: Execution time of some fixed workload with $N$ workers (processors/threads/cores).
  - $T(1)$: Execution time of the workload executed by a single worker.

- **Parallel Speedup $S(N)$:**
  - *Question:* How much faster than with a single worker?
  - *Formula:*
    $$S(N) = \frac{T(1)}{T(N)}$$

- **Parallel Efficiency $\mathcal{E}(N)$:**
  - *Question:* How efficiently do those $N$ workers do their work?
  - *Formula:*
    $$\mathcal{E}(N) = \frac{S(N)}{N} = \frac{T(1)}{N \cdot T(N)}$$

- **Analysis Methodology:**
  - Theoretical and empirical analysis $\to$ practically analyzed.
  - **Warning:** *These metrics are not performance metrics!* (Speedup and efficiency measure relative scalability, not absolute throughput or execution speed).

---

### Topic: Assumptions for Basic Scalability Models
1. **Scalable hardware:**
   - $N$ times the iron (hardware resources) can work $N$ times faster.
2. **Binary parallelizability of work:**
   - Work is either fully parallelizable or not parallelizable at all (purely serial).

---

### Topic: A Simple Speedup Model for Fixed Workload (Strong Scaling Basis)

#### Workload Decomposition Diagram
```
Sequential Execution: Total Workload Normalized to 1
+---------------------------------------------------+---------------+
|                P = 1 - s                          |       s       |
|       (Perfectly parallelizable part)             | (Serial part) |
+---------------------------------------------------+---------------+
|<----------------------- Total Runtime T(1) = 1 ------------------>|

Parallel Execution with N Workers:
+-----------------------+---------------+
|         P / N         |       s       |
| (Parallel execution)  | (Serial part) |
+-----------------------+---------------+
|<--------- T(N) ------>|
```

- **Variables:**
  - $s$: Runtime of the purely serial part (cannot be parallelized).
  - $P$: Runtime of the perfectly parallelizable part ($P = 1 - s$).
  - $N$: Number of parallel workers.

- **Parallel Execution Time Formulation:**
  $$T(N) = s + \frac{P}{N}$$

---

## Page 02

### Topic: Amdahl's Law — Strong Scaling

#### Mathematical Formulations
- **Speedup Formulation (Amdahl's Law):**
  $$S(N) = \frac{T(1)}{T(N)} = \frac{1}{s + \frac{1 - s}{N}}$$

- **Parallel Efficiency Formulation:**
  $$\mathcal{E}(N) = \frac{S(N)}{N} = \frac{1}{N\left(s + \frac{1 - s}{N}\right)} = \frac{1}{s(N - 1) + 1}$$

- **Marginal Calculation Example:**
  - Demonstration calculation for small serial fraction:
    $$\frac{1}{0.001 + \frac{0.0999}{\dots}} = \frac{10}{1.009}$$

#### Fundamental Limits in Amdahl's Law
- **Asymptotic Speedup Limit:**
  $$\lim_{N \to \infty} S(N) = \frac{1}{s}$$
  - No matter how many workers/cores are added, speedup can never exceed $\frac{1}{s}$.
- **Asymptotic Efficiency Limit:**
  $$\lim_{N \to \infty} \mathcal{E}(N) = 0$$
- **Physical Interpretation:**
  - *"Asymptotically, nobody is doing anything except the worker that gets the serial work."*
  - All other $N - 1$ workers remain idle waiting for the serial fraction to complete.

---

### Topic: Strong Scaling with Overhead
- Real parallel systems incur communication and synchronization penalties.
- Let $C(N)$ be an overhead term that may include communication and/or synchronization:
  $$T(N) = s + \frac{P}{N} + C(N)$$

- **Factors determining overhead $C(N)$:**
  - Communication pattern (point-to-point, all-to-all, master-worker)
  - Synchronization strategy (barriers, locks, lock-free)
  - Message sizes (latency-dominated vs bandwidth-dominated)
  - Network structures (interconnect topology, switch contention, NUMA hops)

---

## Page 03

### Topic: Communication Overhead Models & Hazardous Linear Overhead

#### Domain Decomposition Overhead Scaling
- In physical simulations with Cartesian domain decomposition and nonblocking networks:
  - Surface-to-volume communication cost scales as:
    $$\lambda + k \cdot N^{-2/3}$$
    *(or boundary surface scaling as problem partitions scale).*
  - But in poorly designed algorithms or centralized architectures, possible communication cost can be linear:
    $$C(N) = k \cdot N$$

---

### Topic: Strong Scaling with a Particularly Bad Overhead Model
- Consider linear communication overhead:
  - $C(N) = k \cdot N$
  - Base serial time: $T(1) = s + P = 1$
  - Parallel runtime:
    $$T(N) = s + \frac{1 - s}{N} + k \cdot N$$
- *"Linear overhead is hazardous."*

#### Speedup with Linear Overhead
$$S(N) = \frac{T(1)}{T(N)} = \frac{1}{s + \frac{1 - s}{N} + k \cdot N}$$

- **Large $N$ Asymptotic Behaviour ($N \gg 1$):**
  $$S_k(N) \xrightarrow{N \gg 1} \frac{1}{N \cdot k}$$
  - As $N$ becomes large, the speedup drops inversely proportional to $N$! Adding more workers causes the system to run slower, eventually driving speedup towards zero.

---

### Topic: Mitigating Communication Bottlenecks
- Communication is not necessarily purely serial and/or non-overlapping:
  1. **Nonblocking networks** can transfer many messages concurrently without stalling compute units.
  2. **Overlap of communication and computation:** Communication may be overlapped with useful work for well-designed asynchronous algorithms.
  3. **Increasing problem size:** Increasing the amount of parallel work can mitigate the negative impact of the serial work.

---

## Page 04

### Topic: Weak Scaling & Gustafson's Law

#### Concept of Weak Scaling
- Instead of fixing the problem size as in strong scaling, **weak scaling** increases the problem size proportionally with the number of processors.
- *"Increase the parallel part of work: the larger $P$, the larger the speedup."*

#### Visual Comparison of Scaling Models
```
Fixed Problem Size vs Scaled Problem Size:
Small Workload:
+------------+------+
|     P      |  s   |
+------------+------+
       | Parallel Execution
       v
+------+------+
| P/N  |  s   |  --> Serial fraction s dominates total runtime!
+------+------+

Scaled Workload (Weak Scaling):
+------------------------------------------+------+
|                  P                       |  s   |
+------------------------------------------+------+
       | Parallel Execution
       v
+------------+------+
|   P / N    |  s   |  --> Parallel work per processor remains high!
+------------+------+
```

- **Empirical Ratio Calculations (Notebook Margins):**
  - $\frac{100}{28} \approx 3.9$ (or $\frac{100}{25} = 4$)
  - $\frac{520}{70} \approx 7.43$
  - $\frac{5080}{520} \approx 9.7$

---

### Topic: Gustafson's Law Derivation
- Let $s$ be the normalized serial execution time, and let the parallel workload scale with $N$ workers ($p \cdot N$):
  $$S(N) = \frac{(s + p \cdot N) / (s + p)}{(s + p) / (s + p)}$$
- Normalizing total scaled parallel execution time to $s + p = 1$:
  $$S(N) = s + p \cdot N$$
- Since $p = 1 - s$:
  $$S(N) = s + (1 - s) \cdot N = N + s(1 - N) = s + N - s N$$
- **Significance:** In weak scaling, speedup scales linearly with $N$ with a slope of $(1 - s)$, overcoming the rigid asymptotic bound of Amdahl's Law.

---

## Page 05

### Topic: Scalability Models, Parameter Estimation & Node Boundaries

#### Model Parameter Estimation Methods
1. **Manual Analysis:**
   - Requires in-depth knowledge of both the underlying hardware architecture and the application program.
2. **Curve Fitting:**
   - Provides less theoretical insight, but is much less cumbersome to compute from empirical benchmark runs.
   - **Critical Warning:** *"Best fit is not good fit at all."* Simple mathematical regression often fails to predict architectural bottlenecks.

#### Speedup vs Number of Cores (Node Boundary Effect)
```
Speedup
  ^
  |                     Fitted Curve (Amdahl / Regression)
  |                         . - - - - * - - - - *
  |                     . '       *
  |                 . '       *
  |             . '
  |          * '
  |        * '     *
  |       *|
  |      * |                         * (Actual data drops off!)
  |     *  |
  +--------+---------------------------------------------> # Cores
           |
     Node Boundary
```

- **Key Architectural Insight:**
  - *"Intrasocket scaling is not covered by the model."*
  - As soon as execution crosses a socket or node boundary (e.g. crossing from intra-socket shared cache to inter-socket QPI/UPI interconnect, or across distinct NUMA nodes), memory bandwidth contention and inter-socket latency cause actual speedup to drop significantly below the theoretical fitted curve.

---

## Page 06

### Topic: Load Imbalance (Amdahl Generalized)
- **Load Imbalance at Sync Points:**
  - More specifically: **execution time imbalance** among threads.
  - The standard theoretical assumption that parallel work divides equally as $P / N$ is **no longer valid in general**.

#### Two Corner Cases of Load Imbalance
1. **A few "laggers" waste lots of resources:**
   - Even if $N - 1$ threads complete their assigned work rapidly, the entire parallel region is forced to wait for the slowest thread.
   - *A single severe lagger reduces overall system performance back to Amdahl's Law!*
2. **A few "speeders" might be harmless:**
   - Threads that finish early simply wait at the barrier; they do not introduce additional delay beyond what the average/slowest threads take.

#### Performance Tuning Advice
- **Avoid sync points** (minimize explicit barriers and locks).
- **Turn laggers into speeders** (balance workload dynamically so no single thread lags).

---

### Topic: Resource Bottlenecks in Parallel Architectures
- Amdahl's law assumes **perfect scalability of resources** (independent memory, infinite bandwidth, unconstrained execution units).
- **Reality:** Computer architecture is plagued by bottlenecks!

#### Concrete Example: Array Update Loop (Stream Triad / DAXPY Variant)
```c
#pragma omp parallel for
for (i = 0; i < 1000000; i++) {
    a[i] = a[i] + s * c[i];
}
```

- Under Amdahl's Law analysis:
  - Serial fraction $s = 0$ (every iteration is completely independent).
  - Overhead $C(N) = 0$.
  - Theoretical prediction: Perfect linear speedup ($S(N) = N$).
- Actual Execution Reality:
  - **No perfect speedup.**
  - Speedup quickly saturates and plateaus due to **memory bandwidth exhaustion**! All cores contend for the same shared memory bus.

---

## Page 07

### Topic: Resource Bottleneck & Execution Time Modeling (Roofline Analysis)

#### General Formulation
- Let resource bottleneck $i$ deliver resource throughput at maximum rate $R_i^{max}$.
- Let $W_i$ be the amount of resource $i$ demanded by the workload.
- **Minimum runtime imposed by bottleneck $i$:**
  $$T_i = \frac{W_i}{R_i^{max}}$$
- When multiple architectural bottlenecks exist, total minimum runtime is constrained by the slowest component:
  $$T_{min} = f(T_1, T_2, \dots, T_n) = \max(T_1, T_2, \dots, T_n)$$
- **Overall achievable performance:**
  $$P_{max} = \frac{W}{T_{min}}$$

---

### Topic: Complete Numerical Example (DAXPY Stream Loop Calculation)
Consider the array update loop: `a[i] = a[i] + s * c[i]` for $10^7$ iterations with double-precision floating-point data:

#### 1. Compute Bottleneck ($T_{flops}$)
- Operations per iteration: 1 addition ($+$) and 1 multiplication ($*$) = 2 FLOPs.
- Total computational work:
  $$W_{flops} = 2 \times 10^7\text{ FLOPs}$$
- Hardware maximum compute capacity:
  $$R_{flops}^{max} = 192\text{ Gflops/s} = 192 \times 10^9\text{ FLOPs/s}$$
- Compute time:
  $$T_{flops} = \frac{2 \times 10^7\text{ FLOPs}}{192 \times 10^9\text{ FLOPs/s}} = 1.04 \times 10^{-4}\text{ s} = 104\ \mu\text{s}$$

#### 2. Memory Bandwidth Bottleneck ($T_{BW}$)
- Memory accesses per iteration:
  - Read `a[i]` (8 bytes)
  - Read `c[i]` (8 bytes)
  - Write `a[i]` (8 bytes)
  - (Scalar `s` remains in register).
  - Total memory traffic = 3 doubles $\times$ 8 bytes = 24 bytes/iteration.
- Total memory traffic for $10^7$ iterations:
  $$W_{BW} = 3 \times 8 \times 10^7\text{ bytes} = 2.4 \times 10^8\text{ bytes}$$
- Hardware peak memory bandwidth:
  $$R_{BW}^{max} = 40\text{ Gbyte/s} = 40 \times 10^9\text{ bytes/s}$$
- Bandwidth time:
  $$T_{BW} = \frac{3 \times 8 \times 10^7\text{ bytes}}{40 \times 10^9\text{ bytes/s}} = \frac{24 \times 10^7}{40 \times 10^9} = 6 \times 10^{-3}\text{ s} = 6\text{ ms}$$

#### 3. Bottleneck Determination & "Light Speed" Performance
- Minimum execution time:
  $$T_{min} = \max(T_{flops}, T_{BW}) = \max(104\ \mu\text{s}, 6\text{ ms}) = 6\text{ ms}$$
- Maximum achievable performance ("Light Speed"):
  $$P_{max} = \frac{W_{flops}}{T_{min}} = \frac{2 \times 10^7\text{ FLOPs}}{6 \times 10^{-3}\text{ s}} = 3.33\text{ Gflops/s}$$
- **Crucial Conclusion:** Even on a system with a peak theoretical compute performance of 192 Gflops/s, this memory-bound algorithm can never exceed **3.33 Gflops/s** (~1.7% of compute capacity) because it is strictly bounded by the 40 GB/s memory bandwidth.

---

## Page 08

### Topic: Benchmarking Methodologies: "Two Kinds (and a Half)"
- **Primary Objective:** To rigorously assess the performance of a computing system.

#### Venn Diagram of Benchmarks
```
+------------------------------------+------------------------------------+
|     Application Benchmarking       |        Micro Benchmarking          |
|                                    |                                    |
|  * Real full-scale applications    |  * Simple isolated loops & routines|
|  * Realistic domain problems       |  * Well-understood behavior        |
|                                    |  * Highly configurable             |
|                  +-----------------+-----------------+                  |
|                  |     Mini-Apps (Proxy Apps)       |                  |
|                  |  * Represents core kernels of    |                  |
|                  |    large production codes        |                  |
|                  +-----------------+-----------------+                  |
+------------------------------------+------------------------------------+
```

---

### Topic: Proper Definition of Benchmark Cases
1. **Define proper benchmark cases:**
   - Must reflect authentic **production workloads**.
   - Must maintain a **tolerable runtime** (neither too short to be corrupted by timer jitter/startup costs, nor unmanageably long).
2. **Document system settings and execution environment thoroughly:**
   - **Software environment:** Compilers, compiler flags/optimization levels, numerical library versions, operating system kernel version.
   - **Hardware environment:** CPU model/microarchitecture, memory capacity & speed, cache hierarchy, network interconnect.
   - **Runtime options:** Environment variables, thread-to-core affinity (`OMP_PLACES`, `OMP_PROC_BIND`), process pinning, scheduling policies.

---

## Page 09

### Topic: Performance vs Speedup Deception
- **Core Principles:**
  - `runtime != performance`
  - `speedup != performance`

#### The Speedup Fallacy: Vector Supercomputer (NEC SX) vs Commodity Cluster
```
Speedup Graph:                           Absolute Performance Graph:
Speedup                                 Performance (e.g. Gflops)
  ^                                       ^
  |        / Commodity Cluster            |  *================ NEC SX (Vector)
  |       /  (Steep speedup curve!)       |   \
  |      /                                |    \
  |     /                                 |     *--------------- Commodity Cluster
  |    /     --== NEC SX (Flatter speedup)|     |                (Low absolute perf!)
  +---+------------------------> Cores    +-----+------------------------> Cores
```

- **Analysis:**
  - The commodity cluster starts with a very slow single-core baseline, making speedup appear artificially high and steep as cores are added.
  - The NEC SX vector processor has an extremely optimized, high-throughput single-core pipeline; its speedup curve appears flatter, but its **absolute performance is vastly superior**.
  - **Takeaway:** Never optimize for speedup alone; absolute performance (time to solution or sustained FLOPS) is the only valid metric.

---

### Topic: Basics of OpenMP (Open Multi-Processing)
- **Definition:** Standardized API for incremental and portable parallel programming on shared-memory computers.
- **Original Design Goal:** Data-level shared memory parallelism.
- **Modern Extensions:** Task parallelism, accelerator/GPU offloading, SIMD vectorization support.
- **Specification:**
  - Standardized set of compiler directives, runtime library functions, and environment variables.
  - Official specification authority: `http://www.openmp.org/`
  - Fully defined bindings for Fortran, C, and C++.
  - OpenMP is implemented directly on top of general-purpose programming languages.
  - Broad support across all major compilers (GCC, Clang, Intel oneAPI, MSVC).
  - Free and open-source development tools available.
- **Textbook References:**
  - *Using OpenMP*, MIT Press, 2007 (Barbara Chapman, Gabriele Jost, Ruud van der Pas).
  - *Using OpenMP — The Next Step*, MIT Press, 2019 (Ruud van der Pas, Eric Stotzer, Christian Terboven).

---

## Page 10

### Topic: OpenMP Software Architecture & Shared Memory Model
- **Question:** How many different possibilities to compile C programs with parallel directives?

#### OpenMP Layered Software Architecture
```
+----------------------------------+----------------------------------+
|        Programmer's View         |           User's View            |
|       Application Source         |       Execution Environment      |
|     [Compiler Directives/Pragmas]|     [Environment Variables]      |
+-----------------+----------------+-----------------+----------------+
                  |                                  |
                  +----------------+-----------------+
                                   |
                                   v
             +---------------------------------------------+
             |            OpenMP Runtime Library           |
             +---------------------+-----------------------+
                                   |
                                   v
             +---------------------------------------------+
             |              Threads in OS                  |
             +---------------------+-----------------------+
                                   |
                                   v
             +---------------------------------------------+
             |             Cores in Hardware               |
             +---------------------------------------------+
```

- **Programmer's View:** Directives / Pragmas embedded directly in application code.
- **User's View:** Environment variables control resource allocation, scheduling strategies, dynamic adjustment, and thread pinning.

---

### Topic: OpenMP Shared Memory & Execution Model
- **Threads:**
  - Spawned and scheduled by an operating system process.
  - **Thread-Private State:** Local register set, program counter / instruction pointer, private stack frame.
  - **Shared State:** Global shared virtual address space.
- **Data Categories:**
  - **Shared Data:** Single memory location accessible to all threads in the team.
  - **Private Data:** Dedicated, isolated memory space accessible exclusively to the thread that owns it.
- **OpenMP Fork-Join Model:**
  - Execution begins with a single initial thread called the **master thread** running sequentially.

---

## Page 11

### Topic: Fork-Join Execution Model & General C/C++ Syntax

#### Fork-Join Timeline Diagram
```
Master Thread
     || (Serial Region)
     ||
    ==== FORK (Master thread creates team of threads)
    || | | |
    || | | |  (Parallel Region: Concurrent execution across team)
    || | | |
    ==== JOIN (Implicit synchronization; worker threads suspend/terminate)
     ||
     || (Serial Region)
     ||
    ==== FORK
    || | | |
    || | | |  (Parallel Region)
    || | | |
    ==== JOIN
     || (Serial Region continues)
```

---

### Topic: General C/C++ Directive Syntax
- **Directive Format:**
  ```c
  #pragma omp <directive-name> [clause ...]
      structured-block
  ```
- **Fallback Property:** If OpenMP is not enabled during compilation, `#pragma omp` directives are completely ignored by standard C/C++ compilers and treated as comments.
- **Header File:** `#include <omp.h>` (required when calling runtime API functions).
- **GCC Compilation Flag:** `gcc -fopenmp program.c`

---

### Code Example: Checking OpenMP Macro Support & Version
```c
#include <stdio.h>

int main() {
#ifdef _OPENMP
    printf("OpenMP is enabled\n");
    printf("%d\n", _OPENMP); // Prints version integer in YYYYMM format (e.g. 201511)
#else
    printf("OpenMP is not enabled\n");
#endif
    return 0;
}
```

---

## Page 12

### Topic: The Parallel Construct & Thread Management

#### Basic Parallel Region
```c
#pragma omp parallel
{
    printf("Hello from %d of %d\n", omp_get_thread_num(), omp_get_num_threads());
}
```
- Spawns a team of threads. Every thread in the team executes the structured block concurrently.
- `omp_get_thread_num()`: Returns thread ID within the team ($0 \dots N - 1$).
- `omp_get_num_threads()`: Returns the total number of threads in the team.

#### Restricting the Number of Threads Programmatically
```c
omp_set_num_threads(no_of_threads);
#pragma omp parallel
{
    int id = omp_get_thread_num();
}
```

---

### Topic: Data Scoping — Shared vs Private
- Variables within a parallel region can be:
  1. **Private to each executing thread:**
     - Each thread allocates an independent copy on its own stack.
  2. **Shared between threads:**
     - Only one physical instance exists, accessible by all threads.
     - *Important Caveat:* "This does not mean that the instance is always visible to all threads!" (Hardware store buffers, register caching, and memory consistency models require explicit flushes/fences for visibility).
- **Explicit Scoping Clauses:**
  ```c
  #pragma omp parallel private(var1, tmp) shared(eps)
  ```
  - Specifies that `var1` and `tmp` are thread-private, while `eps` remains shared.

---

## Page 13

### Topic: Memory Architecture of Private vs Shared Data
- Scoping specification: `var1, tmp` are private; `eps` is shared.
- *Core Question:* How is private data physically different from shared data?

#### Scoping & Shadowing Code Example
```c
void f() {
    int a;
    float x, y;
    #pragma omp parallel
    {
        int i;
        float y; // Masking/shadowing outer shared y: local y becomes thread-private
    }
}
```

#### Memory Layout Diagram (Shared Stack vs Local Thread Stacks)
```
     SHARED STACK (Process / Master)                  LOCAL STACK (Thread-Private)
+---------------------------------------+       +------------------------------------+
|  Stack Pointer -> [ <free space>   ]  |       | Stack Pointer -> [ <free space>  ] |
|                   [ <free space>   ]  |       |                  [ <free space>  ] |
|                   [ y (outer)      ]  |       |                  [ y (private)   ] |
|                   [ x              ]  |       |                  [ i (local)     ] |
|                   [ a              ]  |       +------------------------------------+
+---------------------------------------+             ^
     ^                                                |
     | Memory Addresses Grow Upward                   | Allocated individually per thread
```

- **Stack Allocation Principles:**
  - Local variables defined inside functions or parallel blocks are allocated on the stack.
  - Every thread has its own **private stack area**:
    - There is one global/main process stack plus one local stack allocated for each thread.
  - **Stack size is strictly limited!** Large thread-local allocations risk silent stack overflow.

---

## Page 14

### Topic: OpenMP Default Scoping Rules & Exceptions

#### Default Scoping Rule
- **Default:** All variables in a parallel region that were declared outside the region are **shared**. This includes global variables, file-scope variables, and dynamic heap data.

#### Scoping Exceptions (Automatically Private Variables)
1. **Loop iteration variables:** The loop index variable of a parallel worksharing loop (`#pragma omp for`) is automatically made private to each thread.
2. **Local variables declared within the parallel region:** Automatically allocated on each thread's private stack.
3. **Local variables in enclosed function calls:** Any function called from within a parallel region allocates its local variables on the calling thread's private stack (unless explicitly declared `static`).

#### Static Variables & Stack Size Limits
- **Static behavior:** Static variables retain their value across calls/iterations and are **shared among all threads**.
- **Stack size limit remedies:**
  - If a large array exceeds thread stack limits, either declare it `static` (places it in the shared data segment) or allocate it on the `heap` using dynamic allocation (`malloc`).
  - Configure the thread stack size using the environment variable:
    ```bash
    export OMP_STACKSIZE=64M
    ```
- **Lifespan Caveat:** Thread-local stack variables are destroyed when execution exits their enclosing scope.

#### Concurrency Hazards of Unintentional Sharing
- *What happens if a variable is unintentionally shared?*
  - **Read-only access:** Harmless; all threads read identical data.
  - **Write access (at least one thread writes):** **Extremely hazardous!** Induces a race condition resulting in memory inconsistency, non-deterministic bugs, and data corruption.

---

## Page 15

### Topic: Private Variables, Variable Masking & Lifecycle

#### Code Demonstration of Masking
```c
double s;
s = ...; // Outer shared s initialized
#pragma omp parallel private(s)
{
    s = ...;      // Modifies thread-private s (outer shared s is masked!)
    ... = ... + s;
}
... = ... + s;    // Accesses outer shared s
```

#### Thread Lifecycle & Variable Masking Diagram
```
Time
 |   Shared Scope: Variable 's' has defined value
 |   ==================== FORK ====================
 |       Thread T0      Thread T1      Thread T2      Thread T3
 |     +-----------+  +-----------+  +-----------+  +-----------+
 |     |    s_0    |  |    s_1    |  |    s_2    |  |    s_3    |
 |     +-----------+  +-----------+  +-----------+  +-----------+
 |           ^              ^              ^              ^
 |           +--------------+--------------+--------------+
 |                  INITIAL VALUES ARE UNDEFINED!
 |
 |     [Outer shared variable 's' persists in memory but is INACCESSIBLE]
 |
 |     +-----------+  +-----------+  +-----------+  +-----------+
 |     |    s_0    |  |    s_1    |  |    s_2    |  |    s_3    |
 |     +-----------+  +-----------+  +-----------+  +-----------+
 |   ==================== JOIN ====================
 v   Shared Scope: Shared value of 's' recovers after parallel region!
```

- **Key Scoping Rules:**
  - Under `private(s)`, each thread receives an uninitialized copy: **its initial value is completely undefined**.
  - Modifications inside the parallel construct affect only the thread-local instance.
  - Upon exiting the parallel region, the private copies vanish; the outer shared variable retains whatever value it had before the fork.
- **Top Marginal Note:** `thread safe: printf()` (C standard library `printf` is thread-safe from crashing, though character streams may interleave without synchronization).
- *Open Question:* What if the initial value of the outer variable is required inside the parallel region?

---

## Page 16

### Topic: The `firstprivate` Clause & Loop Parallelism

#### The `firstprivate` Clause
- **Definition:** The `firstprivate` clause provides a private copy of each listed variable for each thread, **and initializes each copy with the value of the outer variable prior to entering the construct**.
```c
#pragma omp parallel firstprivate(x)
{
    printf("[First Private] Thread %d: x = %d\n", omp_get_thread_num(), x);
}
```

---

### Topic: Loop Parallelism in OpenMP (`#pragma omp for`)
- **Core Concept:** Threads in the team are assigned an independent subset of the total loop iterations.
- **Barrier Requirement:** Threads must wait at the end of the worksharing construct due to an **implicit barrier**.

#### Worksharing Syntax
```c
// Two-step nested form:
#pragma omp parallel
{
    #pragma omp for
    for (int i = 1; i < 13; i++) {
        c[i] = a[i] + b[i];
    }
}

// Combined directive form:
#pragma omp parallel for
for (int i = 1; i < 13; i++) {
    c[i] = a[i] + b[i];
}
```

#### Iteration Partitioning Example (12 Iterations across 3 Threads)
```
          #pragma omp parallel for (Iterations 1 to 12)
                     /           |            \
                    /            |             \
       Thread 0            Thread 1             Thread 2
       i = 1               i = 5                i = 9
       i = 2               i = 6                i = 10
       i = 3               i = 7                i = 11
       i = 4               i = 8                i = 12
                    \            |             /
                     \           |            /
                      v          v           v
           ===========================================
                        IMPLICIT BARRIER
           ===========================================
```
- *Note:* If no scheduling clause is specified, the schedule is predetermined (implementation-dependent default, typically static chunking).

---

## Page 17

### Topic: OpenMP Loop Scheduling Types & Static Scheduling

#### Four Core OpenMP Scheduling Types
1. **Static Scheduling** (`schedule(static, [chunk])`)
2. **Dynamic Scheduling** (`schedule(dynamic, [chunk])`)
3. **Guided Scheduling** (`schedule(guided, [chunk])`)
4. **Auto Scheduling** (`schedule(auto)`)

---

### Topic: Static Scheduling Mechanics & Pitfalls
- **Default Scheduling Behavior:**
  - Iterations are divided into chunks of size `chunk_size` and statically assigned to threads in round-robin fashion **before execution starts**.
  - **Zero runtime reassignment:** Assignment is fixed; no dynamic redistribution occurs.

#### Example 1: Chunk Size = 2 (8 Iterations, 4 Threads)
```c
#pragma omp parallel for schedule(static, 2)
for (int i = 0; i < 8; i++) { ... }
```
- **Thread Assignments:**
  - Thread 0: iterations 0, 1
  - Thread 1: iterations 2, 3
  - Thread 2: iterations 4, 5
  - Thread 3: iterations 6, 7

#### Example 2: Chunk Size = 4 (8 Iterations, 4 Threads)
```c
#pragma omp parallel for schedule(static, 4)
for (int i = 0; i < 8; i++) { ... }
```
- **Thread Assignments:**
  - Thread 0: iterations 0, 1, 2, 3
  - Thread 1: iterations 4, 5, 6, 7
- **Critical Flaw:**
  - *"If you have taken 4 threads, 2 will remain idle."*
  - Threads 2 and 3 receive zero iterations and sit idle while Threads 0 and 1 perform all the work!

---

## Page 18

### Topic: Dynamic & Guided Scheduling

#### Dynamic Scheduling
```c
int i;
#pragma omp parallel for schedule(dynamic, 1)
for (i = 0; i < 8; i++) {
    printf("Thread %d starting iteration %d\n", omp_get_thread_num(), i);
    sleep(i % 3); // Simulates uneven/varying workload per iteration
    printf("Thread %d finished iteration %d\n", omp_get_thread_num(), i);
}
return 0;
```
- **Mechanics:**
  - Iterations are grouped into chunks (size 1 in this example).
  - Threads dynamically pick up a new chunk from a centralized work queue as soon as they finish their current chunk.
  - **Primary Benefit:** Effectively solves load imbalance when work per iteration is unpredictable or non-uniform.
  - **Trade-off:** High runtime overhead due to synchronized access to the shared work queue.

---

### Topic: Guided Scheduling (Smart Decreasing Chunks)
- **Concept:** Designed to minimize scheduling overhead while maintaining dynamic load balancing.
- **How It Works:**
  - Starts by dispatching **large chunks** of iterations to minimize queue access contention early in the loop.
  - As remaining iterations decrease, chunk sizes are **gradually and proportionally reduced** to fine-tune load balancing toward the end.
  - The parameter `chunk_size` establishes the minimum lower bound for chunk size.
- **Syntax:**
  ```c
  #pragma omp parallel for schedule(guided, chunk_size)
  ```

---

## Page 19

### Topic: Auto Scheduling, Synchronization & Data Protection

#### Auto Scheduling
- **Syntax:**
  ```c
  #pragma omp parallel for schedule(auto)
  ```
- **Mechanism:** Delegates the scheduling decision entirely to the compiler and runtime system.
- **When to Use:**
  1. You trust compiler optimizations.
  2. You want simple, maintainable code.
  3. You are prototyping, benchmarking, or experimenting across diverse architectures.
- **Marginal Mathematical Scribbles:**
  - Arithmetic series formula: $\sum_{i=1}^n i = \frac{n(n+1)}{2}$
  - $\frac{100 \times 101}{2} = 50 \times 101 = 5050$
  - Scale factor calculation: $(5050)(10000) / 2 \dots$

---

### Topic: Synchronization Primitives
- **Mutex (Mutual Exclusion):** Restricts access to a critical section to strictly one thread at a time.
- **Semaphore:** Counter-based synchronization primitive managing access to a finite pool of shared resources.

---

### Topic: Data Protection Constructs (`critical` vs `atomic`)

#### 1. `#pragma omp critical`
- Restricts an entire structured code block to **one single thread at a time**.
- Guarantees complete mutual exclusion across arbitrary statements.
- **Drawbacks:** High execution overhead; implemented as a software lock/mutex.
- Can easily become a **major framework-level performance bottleneck** that serializes parallel execution.

#### 2. `#pragma omp atomic`
- Provides direct **hardware-level atomic operations**.
- Protects a **single specific memory update** (e.g. `x += expr`, `x++`, `x--`).
- Leverages hardware-supported atomic CPU instructions (e.g., bus/cache locking, `CMPXCHG`, `LOCK ADD`).
- **Much faster performance** than critical sections.
- **Limitation:** Strictly restricted to a single scalar assignment/operation.

---

## Page 20

### Topic: Barrier Synchronization & Multi-Stage Pipelines
- **Classification Note:** *"Critical and atomic both belong to mutex."*

#### `#pragma omp barrier` (Synchronization Mechanism)
- **Function:** Forces all threads in the current parallel team to halt execution and wait until **every thread reaches this exact point**.
- **Purpose:** Ensures that all parallel calculations up to that point are completely finished and memory visibility is established before any thread is allowed to continue.

#### Two-Stage Parallel Pipeline Example
```c
#pragma omp parallel num_threads(4)
{
    int i = omp_get_thread_num();

    // Stage 1: Compute square
    b[i] = a[i] * a[i];
    printf("Thread %d finished Stage 1\n", i);

    #pragma omp barrier // All threads must wait until all b[i] are computed!

    // Stage 2: Add 10 (Safely depends on Stage 1 completion across all threads)
    c[i] = b[i] + 10;
    printf("Thread %d finished Stage 2\n", i);
}
```

---

## Page 21

### Topic: The Single Directive & The `nowait` Clause

#### `#pragma omp single`
- **Definition:** Ensures that the enclosed structured block is executed by **only one thread** in the team (the first thread that arrives).
- **Behavior of Other Threads:** All other threads in the team skip the enclosed block entirely.
- **Implicit Barrier:** By default, there is an **implicit barrier at the end of `#pragma omp single`**, meaning all other threads must wait until the executing thread finishes the block.

#### Code Demonstration
```c
#pragma omp parallel num_threads(10)
{
    int tid = omp_get_thread_num();
    printf("Thread %d before single\n", tid);

    #pragma omp single
    {
        printf("Thread %d is inside single block\n", tid);
    }
    // All threads wait here due to the default implicit barrier!

    printf("Thread %d after single barrier\n", tid);
}
```

#### Removing the Implicit Barrier (`nowait`)
- To eliminate the barrier and allow non-executing threads to continue without stalling:
  ```c
  #pragma omp single nowait
  ```

---

## Page 22

### Topic: The Ordered Directive & The Reduction Clause

#### OpenMP Ordered Directive (`#pragma omp ordered`)
- **Problem:** When loops are parallelized, iterations execute out-of-order concurrently.
- **Function:** Forces the code inside the `#pragma omp ordered` block to execute in the **exact sequential order of the original serial loop**.
- **Prerequisite:** The enclosing loop must declare the `ordered` clause on `#pragma omp for`.

---

### Topic: The Reduction Clause
- **Syntax:**
  ```c
  int sum = 0;
  #pragma omp parallel for reduction(+: sum)
  for (int i = 0; i < 5; i++) {
      sum += i;
  }
  ```
- **Mechanism:**
  1. Creates an independent private copy of the reduction variable for each thread.
  2. Initializes each private copy to the operator's mathematical identity (e.g. 0 for `+`, 1 for `*`).
  3. Each thread updates its local private copy during parallel execution without lock contention.
  4. At the end of the region, OpenMP automatically combines all private copies into the original shared variable using the designated operation.
- **Mathematical Requirements:**
  - The reduction operation must be **Associative**: $(a \oplus b) \oplus c = a \oplus (b \oplus c)$.
  - The reduction operation should **preferably be Commutative**: $a \oplus b = b \oplus a$ (allows the runtime to aggregate partial sums in any order).

---

## Page 23

### Topic: Task Parallelism & Task Loops

#### Marginal Notes
- Finding maximum value with reduction vs without reduction.
- Semaphore implementations in OpenMP.

---

### Topic: Task Parallelism Foundations
- **Concept:** Different units of independent work ("tasks") run in parallel asynchronously.
- **How It Works:**
  - Each thread/process performs a different discrete job.
  - Tasks may or may not operate on the same data.
  - Ideal for irregular algorithms, recursive tree structures, linked lists, and unbounded while-loops.
  - Directive: `#pragma omp task`

---

### Topic: Task Loops (`#pragma omp taskloop`)
- **Concept:** A loop (e.g., iterations 0 to 999) is automatically decomposed into discrete task objects:
  - Iterations 0 - 99 $\to$ Task 1
  - Iterations 100 - 199 $\to$ Task 2, etc.
- **Task Pool Architecture:**
  1. A single thread enters, generates the tasks, and deposits them into a centralized **task pool**.
  2. Idle threads in the team retrieve tasks from the task pool and execute them.
- **Contrast with Standard Loop Worksharing:**
  - In a standard `#pragma omp for`, iterations are directly partitioned and scheduled to specific threads.
  - In `#pragma omp taskloop`, iterations are packaged into first-class tasks managed by the OpenMP task scheduler.

---

## Page 24

### Topic: Tasking Construct Syntax & Granularity Control

#### Standard Task Creation Pattern (Single Producer)
```c
#pragma omp parallel
{
    #pragma omp single
    {
        #pragma omp task
        {
            // Task body s executed asynchronously by any available thread
        }
    }
}
```

#### Taskloop Pattern
```c
#pragma omp parallel
{
    #pragma omp single
    {
        #pragma omp taskloop
        for (int i = 0; i < N; i++) {
            // Loop body s
        }
    }
}
```

#### Performance & Granularity Trade-Offs
- For massive loop bounds (e.g., $N = 8 \times 10^9$), dynamic task loops can outperform static worksharing loops on imbalanced or heterogeneous systems.
- **Granularity Control Clauses:**
  1. **`num_tasks(K)`:**
     ```c
     #pragma omp taskloop num_tasks(10)
     ```
     - Explicitly bounds the number of tasks created to exactly 10, regardless of loop bounds.
  2. **`grainsize(G)`:**
     - Fixes the exact number of iterations bundled into each individual task.

---

## Page 25

### Topic: OpenMP SIMD (Single Instruction, Multiple Data) Vectorization

#### Core Definition & Principles
- **Directive:** `#pragma omp simd`
- **Compiler Instruction:** Tells the compiler:
  > *"This loop is safe to vectorize; use SIMD vector instructions (AVX / SSE) to run multiple iterations at once."*
- **Scope:** Represents **data-level parallelism inside a single CPU core** (utilizing vector execution units).

```c
#pragma omp simd
for (int i = 0; i < N; i++) {
    c[i] = a[i] + b[i];
}
```

#### Hardware Vector Register Standards
- **SSE:** 128-bit vector registers (processes 4 single-precision floats or 2 doubles simultaneously).
- **AVX / AVX2:** 256-bit vector registers (processes 8 single-precision floats or 4 doubles simultaneously).
- **AVX-512:** 512-bit vector registers (processes 16 single-precision floats or 8 doubles simultaneously).
- **SIMD Factor:** $\text{SIMD Factor} = \frac{\text{Vector Register Width}}{\text{Data Element Size}}$ (number of operations completed in an instant).

#### GCC Compiler Optimization Flags
- Compilation command:
  ```bash
  gcc -O3 -fopenmp -fopt-info-vec 17.c
  ```
  - `-fopt-info-vec`: Outputs detailed compiler diagnostics reporting which loops were successfully vectorized.
- **Architecture Target Flags (`-march`):**
  - `-march=x86-64`: Safe baseline 64-bit architecture (guarantees compatibility across all modern x86 chips).
  - `-march=haswell`: Targets a specific microarchitecture family (enables AVX2, FMA).
  - `-march=native`: Directs the compiler to detect and leverage all instruction set extensions supported by the host machine's exact CPU.

---

## Page 26

### Topic: SIMD Scoping, Latent Bugs & Multi-Level Parallelism

#### SIMD Scoping Clauses & Latent Vectorization Bugs
- **Directive:**
  ```c
  #pragma omp simd private(variable)
  ```
- **Hazard:** Missing `private(tmp)` inside a vectorized loop induces a **latent bug**!

#### What is a Latent Bug?
- **Definition:** *"A bug that exists in code, but does not always show up immediately."*
- **Mechanism:** The bug manifests only when specific runtime conditions or compiler optimization levels are met.
- **Conditions where buggy code deceptively appears correct:**
  1. When the compiler fails or chooses not to vectorize the loop (executes sequentially).
  2. When problem size $N$ is small (loop overhead dominates or vector lanes are not filled).
  3. At low optimization levels (`-O0`, `-O1`).
  4. On specific CPU architectures that lack hardware SIMD vector units.

---

### Topic: Combined Multi-Level Parallelism (`parallel for simd`)
- **Directive:**
  ```c
  #pragma omp parallel for simd
  ```
- Combines multi-threaded shared-memory parallelism across multiple CPU cores with SIMD vectorization inside each core:
  - Outer level: Iterations partitioned across OS threads.
  - Inner level: Chunk assigned to each thread is vectorized across SIMD vector registers.
- **Compilation Flag:** `gcc -O3 -fopenmp 2.c`
  - *What is -O3?* Level 3 aggressive compiler optimization (enables automatic loop vectorization, unrolling, function inlining, and code motion).
- **Performance Trade-Off:**
  - *"Parallel + SIMD takes more time than only SIMD due to thread creation/join overhead and synchronization time"* when problem sizes are small or memory bandwidth is saturated.

---

### Topic: SIMD Reduction
- Directs the compiler to perform reduction operations across SIMD vector lanes before horizontally reducing to the scalar variable:
```c
#pragma omp simd reduction(+: sum)
for (int i = 0; i < N; i++) {
    sum += a[i];
}
```

---
*End of Extraction — 3d773f8d-2e9e-426e-8ef8-0f93e447813a.pdf (Pages 01–26)*
