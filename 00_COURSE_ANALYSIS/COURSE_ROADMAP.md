# Master Course Roadmap: Parallel Computing University Study System

**System Architecture:** Hierarchical Master Curriculum  
**Output Target Directory:** `notes_study/`  
**Total Main Topics (Modules):** 6  
**Total Subtopics (Study Documents):** 28  
**Pedagogical Order:** Strictly sequential, prerequisite-driven progression from silicon physics to distributed supercomputing.

---

## High-Level Course Architecture

```
notes_study/
├── 01_Hardware_Foundations/
│   ├── 01_Performance_and_Von_Neumann_Architecture.md
│   ├── 02_Pipelining_and_Superscalar_Execution.md
│   ├── 03_SMT_SIMD_and_Peak_Performance_Model.md
│   └── 04_Memory_Hierarchy_and_Data_Transfer_Models.md
├── 02_Parallel_Architectures/
│   ├── 01_Flynns_Taxonomy_and_Modern_Hybrids.md
│   ├── 02_Shared_Memory_UMA_vs_ccNUMA.md
│   └── 03_Distributed_Memory_Clusters_and_Interconnects.md
├── 03_Performance_and_Scaling_Laws/
│   ├── 01_Speedup_Efficiency_and_Scalability_Metrics.md
│   ├── 02_Amdahls_Law_and_Overhead_Failure_Models.md
│   ├── 03_Gustafson_Barsis_Law_and_Weak_Scaling.md
│   ├── 04_Hardware_Resource_Bottlenecks_and_DAXPY_Roofline.md
│   └── 05_Benchmarking_Methodologies_and_The_Speedup_Fallacy.md
├── 04_Pthreads_Programming/
│   ├── 01_POSIX_Standards_and_Pthreads_Lifecycle.md
│   ├── 02_Thread_Attributes_and_Scheduling_Policies.md
│   ├── 03_Mutex_Synchronization_and_Critical_Sections.md
│   ├── 04_Condition_Variables_and_Producer_Consumer.md
│   ├── 05_POSIX_Semaphores_and_Read_Write_Locks.md
│   └── 06_POSIX_Barriers_Spinlocks_and_The_Contention_Cliff.md
├── 05_OpenMP_Programming/
│   ├── 01_OpenMP_Architecture_and_Fork_Join_Model.md
│   ├── 02_Thread_Management_and_Runtime_APIs.md
│   ├── 03_Data_Scoping_Private_Shared_and_Masking.md
│   ├── 04_Loop_Worksharing_and_Scheduling_Policies.md
│   ├── 05_Synchronization_Critical_Atomic_and_Barriers.md
│   ├── 06_Reductions_and_Ordered_Execution.md
│   ├── 07_Task_Parallelism_and_Task_Loops.md
│   └── 08_OpenMP_SIMD_Vectorization_and_Latent_Bugs.md
└── 06_MPI_Distributed_Programming/
    ├── 01_MPI_Architecture_and_Process_Lifecycle.md
    ├── 02_Point_to_Point_Communication_Send_Recv.md
    ├── 03_Execution_Management_and_CLI_Controls.md
    ├── 04_Collective_Communication_Overview.md
    └── 05_Dynamic_Client_Server_Connections_and_Ports.md
```

---

## Detailed Topic-by-Topic Breakdown & Source Mapping

### TOPIC 01: Hardware Foundations & Processor Microarchitecture
*Foundational silicon architecture explaining why parallel computing is required and how a single core extracts performance.*

#### Subtopic 01.1: Computational Performance, Von Neumann Architecture & The Memory Wall
- **Concepts Covered:** Definition of performance ($P = \text{Work}/\text{Time}$); FLOPs vs wall-clock time; Stored-program computer architecture (ALU, Control Unit, Memory, I/O); Memory latency/bandwidth bottlenecks; Instruction throughput as the hardware definition of work vs application useful work.
- **Source Mapping:** `Sanjay_Notes_1.pdf` (Pages 1–2).
- **Prerequisites:** Basic computer organization.
- **University Exam Value:** High (Standard opening descriptive question on Von Neumann bottleneck and performance metrics).

#### Subtopic 01.2: In-Core Instruction-Level Parallelism: Pipelining & Superscalar Execution
- **Concepts Covered:** Pipelining principles; 5-stage floating-point multiplication pipeline walkthrough ($A(i) = B(i) \times C(i)$); Latency, wind-up/wind-down phases, throughput; Instruction pipelines; Branch hazards and branch prediction; Superscalar execution; Reorder buffer, register renaming, execution ports (Ports 0–5).
- **Source Mapping:** `Sanjay_Notes_1.pdf` (Pages 2–5).
- **Prerequisites:** Subtopic 01.1.
- **University Exam Value:** Very High (5-stage FP multiplier pipeline numericals and superscalar port diagrams are frequent exam questions).

#### Subtopic 01.3: Thread & Data Parallelism: SMT, SIMD Vectorization & The Peak Performance Model
- **Concepts Covered:** Simultaneous Multi-Threading (SMT / Hyper-Threading); Pipeline bubbles and resource utilization; SIMD processing concept; x86 SIMD register widths (SSE 128-bit, AVX 256-bit, AVX-512 512-bit); Loop unrolling by compiler; Master Peak Core Performance Formula:
  $$P_{\text{core}} = n_{\text{superscalar}}^{\text{FP}} \times n_{\text{FMA}} \times n_{\text{SIMD}} \times f$$
- **Source Mapping:** `Sanjay_Notes_1.pdf` (Pages 3, 5–6).
- **Prerequisites:** Subtopic 01.2.
- **University Exam Value:** Critical (Core numerical formula for calculating theoretical peak FLOP/s).

#### Subtopic 01.4: Memory Hierarchy, Cache Organization & Analytical Data Path Transfer Models
- **Concepts Covered:** Multi-level cache organization (L1, L2, L3); Quantitative latency/bandwidth hierarchy table ($10^{-9}$ s to $10^{-4}$ s; $10^{12}$ B/s to $10^9$ B/s); Cache line transfer bursts (64 bytes); Analytical message transfer model ($T = \lambda + N/b$); Effective bandwidth ($b_{\text{eff}} = N/T$); Asymptotic limits for small vs large messages.
- **Source Mapping:** `Sanjay_Notes_1.pdf` (Pages 7–8).
- **Prerequisites:** Subtopic 01.1.
- **University Exam Value:** High (Analytical transfer time calculations and memory hierarchy comparison tables).

---

### TOPIC 02: Parallel Computer Architectures & Memory Systems
*Categorization and physical topologies of multiprocessor and cluster systems.*

#### Subtopic 02.1: Flynn's Classical Taxonomy & Modern Hybrid Systems
- **Concepts Covered:** Flynn's 2x2 matrix: SISD, MISD, SIMD, MIMD; Historical origins (Flynn 1972); Examples and operational characteristics of each class; Modern supercomputers as hierarchical hybrids (MIMD clusters of ccNUMA nodes containing multicore SIMD processors).
- **Source Mapping:** `Sanjay_Notes_1.pdf` (Pages 8–9, 11).
- **Prerequisites:** Topic 01.
- **University Exam Value:** Very High (Classic 5-mark university exam classification question).

#### Subtopic 02.2: Shared-Memory Multiprocessors: UMA vs ccNUMA Architectures
- **Concepts Covered:** Shared-memory paradigm; Single cache-coherent address space; Uniform Memory Access (UMA) bus-based systems; Cache-Coherent Non-Uniform Memory Access (ccNUMA); Local vs remote memory latency/bandwidth; Multi-core chip architecture (private L1/L2, shared L3, dual-socket nodes); Cache coherence and false sharing problems.
- **Source Mapping:** `Sanjay_Notes_1.pdf` (Pages 9–10).
- **Prerequisites:** Subtopic 02.1.
- **University Exam Value:** Very High (UMA vs NUMA comparison table is a recurring exam favorite).

#### Subtopic 02.3: Distributed-Memory Systems: Clusters, Interconnects & Message Passing Networks
- **Concepts Covered:** Pure distributed memory model (processor + local memory + network interface); Communication network fabrics; TCP/IP and UDP/IP vs high-speed interconnects (InfiniBand); Modern hierarchical cluster topology; Latency and bandwidth hierarchy across sockets, nodes, and network switches.
- **Source Mapping:** `Sanjay_Notes_1.pdf` (Page 11).
- **Prerequisites:** Subtopic 02.2.
- **University Exam Value:** Medium-High (Foundational conceptual bridge to MPI programming).

---

### TOPIC 03: Parallel Performance Analysis, Scaling Laws & Bottlenecks
*Mathematical foundations of parallel efficiency, scaling limits, and hardware bottlenecks.*

#### Subtopic 03.1: Fundamental Scaling Metrics: Speedup, Efficiency & Scalability Realities
- **Concepts Covered:** Execution time $T(N)$; Parallel Speedup $S(N) = T(1)/T(N)$; Ideal linear speedup; Superlinear speedup (cache effects); Parallel Efficiency $E(N) = S(N)/N$; Physical vs relative performance distinction; Scalability definition; Multi-socket boundary drop-offs.
- **Source Mapping:** `Sanjay_Notes_1.pdf` (Page 12); `3d773f8d.pdf` (Pages 1–2, 5).
- **Prerequisites:** Topic 01 & Topic 02.
- **University Exam Value:** Critical (Mandatory numerical calculations of speedup and efficiency).

#### Subtopic 03.2: Amdahl's Law, Strong Scaling & Communication Overhead Failures
- **Concepts Covered:** Strong scaling definition (fixed problem size); Mathematical derivation of Amdahl's Law from first principles; Asymptotic speedup limit ($S_{\max} = 1/s$); Efficiency decay under Amdahl's Law ($\mathcal{E}(N) = 1/(s(N-1)+1)$); Communication/synchronization overhead term $C(N)$; The hazardous linear overhead model ($C(N) = kN$) and catastrophic speedup collapse to zero.
- **Source Mapping:** `3d773f8d.pdf` (Pages 2–3); `reference/teaching_style_sample.md` (Sections 15–25).
- **Prerequisites:** Subtopic 03.1.
- **University Exam Value:** Highest Priority (Amdahl's law derivations and numerical problems appear on virtually every mid-sem and end-sem paper).

#### Subtopic 03.3: Gustafson-Barsis' Law, Weak Scaling & Scaling Realities
- **Concepts Covered:** Weak scaling definition (scaled problem size, constant workload per worker); Mathematical derivation of Gustafson-Barsis' Law:
  $$S(N) = s + (1 - s)N$$
  Comparison between Amdahl's Law and Gustafson's Law; Why weak scaling enables massive supercomputer scaling; Parameter estimation challenges ("Best fit is not a good fit at all").
- **Source Mapping:** `3d773f8d.pdf` (Pages 4–5); `reference/teaching_style_sample.md` (Section 28).
- **Prerequisites:** Subtopic 03.2.
- **University Exam Value:** Very High (Direct comparison question: Amdahl vs Gustafson).

#### Subtopic 03.4: Hardware Resource Bottlenecks, The Roofline Model & DAXPY Bandwidth Saturation
- **Concepts Covered:** Multi-resource bottleneck theorem: $T_{\min} = \max(T_1, \dots, T_n)$; Compute-bound vs memory-bandwidth-bound applications; Load imbalance at barriers (laggers vs speeders); Complete DAXPY numerical problem ($10^7$ iterations, $W_{\text{flops}} = 20\text{ MFLOPs}$, $W_{\text{BW}} = 240\text{ MB}$, $R_{\text{peak}} = 192\text{ GFLOP/s}$, $B_{\text{peak}} = 40\text{ GB/s}$); Why DAXPY achieves only 1.73% of peak compute.
- **Source Mapping:** `3d773f8d.pdf` (Pages 6–7).
- **Prerequisites:** Subtopic 01.4 & Subtopic 03.1.
- **University Exam Value:** Very High (Complex numerical problem modeling real-world performance ceilings).

#### Subtopic 03.5: Benchmarking Methodologies & The Speedup Fallacy
- **Concepts Covered:** Benchmarking classification Venn diagram: Application benchmarks vs Microbenchmarks vs Mini-apps / Proxy apps; Proper benchmarking rules; The Performance vs Speedup fallacy (Commodity cluster 100x speedup vs Vector supercomputer 2x speedup on low execution time); Why timing a program once is never sufficient.
- **Source Mapping:** `3d773f8d.pdf` (Pages 8–9).
- **Prerequisites:** Subtopic 03.1.
- **University Exam Value:** High (Conceptual examination questions on benchmarking rigor).

---

### TOPIC 04: Shared-Memory Parallel Programming with Pthreads
*Explicit multithreading, thread management, POSIX standards, and low-level synchronization.*

#### Subtopic 04.1: POSIX Standard, Pthreads Architecture & Thread Lifecycle
- **Concepts Covered:** POSIX definition and IEEE origin; Certified vs compliant systems (macOS vs Linux/glibc, BSD, Android/Bionic); Pthreads vs OpenMP comparison; Thread identifier `pthread_t` opaque type; Function pointers in C (`void* (*)(void*)`); Core lifecycle APIs: `pthread_create`, `pthread_exit`, `pthread_join`; Complete Hello World multithreaded program.
- **Source Mapping:** `Pthreada_Aagam_Notes.pdf` (Pages 1–4); `IMG_1333.pdf` (Pages 1–4).
- **Prerequisites:** Topic 02.
- **University Exam Value:** Very High (Foundational Pthreads programming syntax and thread creation questions).

#### Subtopic 04.2: Thread Attributes (`pthread_attr_t`) & Real-Time Scheduling Policies
- **Concepts Covered:** `pthread_attr_t` data type and lifecycle (`init`/`destroy`); Stack size customization; Detach state (`PTHREAD_CREATE_JOINABLE` vs `PTHREAD_CREATE_DETACHED`); POSIX scheduling policies: `SCHED_OTHER` (time-sharing), `SCHED_FIFO` (real-time FIFO), `SCHED_RR` (real-time round robin); Thread priorities (`sched_param`).
- **Source Mapping:** `Pthreada_Aagam_Notes.pdf` (Page 5); `IMG_1333.pdf` (Page 5).
- **Prerequisites:** Subtopic 04.1.
- **University Exam Value:** Very High (*"He will ask all this!"* - High-frequency exam topic on thread attributes).

#### Subtopic 04.3: Mutex Synchronization & Critical Section Protection
- **Concepts Covered:** Race conditions on shared variables (`counter++`); Mutual exclusion definition; `pthread_mutex_t` data type; Lifecycle functions: `pthread_mutex_init`, `pthread_mutex_lock`, `pthread_mutex_trylock`, `pthread_mutex_unlock`, `pthread_mutex_destroy`; Complete multithreaded counter increment program with mutex protection; Mutex serialization overhead.
- **Source Mapping:** `Pthreada_Aagam_Notes.pdf` (Pages 6–9); `IMG_1333.pdf` (Pages 6–9).
- **Prerequisites:** Subtopic 04.1.
- **University Exam Value:** Very High (Standard code-writing exam question on mutex critical section).

#### Subtopic 04.4: Condition Variables & The Producer-Consumer Bounded Buffer Problem
- **Concepts Covered:** Condition synchronization concept; `pthread_cond_t` data type; APIs: `pthread_cond_init`, `pthread_cond_wait`, `pthread_cond_signal`, `pthread_cond_broadcast`, `pthread_cond_destroy`; Atomic release and reacquisition of mutex in `wait`; Guarding predicates with `while` loops; Complete Producer-Consumer bounded buffer implementation with `not_full` and `not_empty` conditions.
- **Source Mapping:** `Pthreada_Aagam_Notes.pdf` (Pages 10–12); `IMG_1333.pdf` (Pages 10–12).
- **Prerequisites:** Subtopic 04.3.
- **University Exam Value:** Highest Priority (Classic 10-mark operating systems / parallel computing exam question).

#### Subtopic 04.5: POSIX Semaphores & Read-Write Locks
- **Concepts Covered:** Counting semaphores (`sem_t`); Real-time extensions header (`<semaphore.h>`); `sem_init`, `sem_wait`, `sem_post`, `sem_destroy`; Binary semaphore vs Mutex; Read-Write locks (`pthread_rwlock_t`); Shared reader lock vs exclusive writer lock; The Blackboard analogy; Complete RW lock reader/writer program.
- **Source Mapping:** `Pthreada_Aagam_Notes.pdf` (Pages 13–16); `IMG_1333.pdf` (Pages 13–16).
- **Prerequisites:** Subtopic 04.3.
- **University Exam Value:** High (Mutex vs Semaphore and Reader-Writer problem implementations).

#### Subtopic 04.6: POSIX Barriers, Spinlocks & The Contention Cliff
- **Concepts Covered:** Phase synchronization; `pthread_barrier_t` lifecycle and rendezvous count; `pthread_barrier_wait`; Spinlocks (`pthread_spinlock_t`); Busy-waiting vs blocking locks; In-depth comparison table (Mutex vs Spinlock); Quantitative throughput vs critical section length curve; Spinlock advantage region ($< 10^2$ cycles) vs the catastrophic contention cliff ($> 10^2$ cycles).
- **Source Mapping:** `Pthreada_Aagam_Notes.pdf` (Pages 17–20); `IMG_1333.pdf` (Pages 17–20).
- **Prerequisites:** Subtopic 04.3.
- **University Exam Value:** Very High (Spinlock vs Mutex comparison and contention curve analysis).

---

### TOPIC 05: Shared-Memory Parallel Programming with OpenMP
*Compiler-directive-based parallel programming, work sharing, advanced synchronization, tasking, and vectorization.*

#### Subtopic 05.1: OpenMP Architecture, Core Syntax & The Fork-Join Execution Model
- **Concepts Covered:** OpenMP definition and history; Software architecture (Application $\rightarrow$ Pragmas $\rightarrow$ Runtime $\rightarrow$ Threads $\rightarrow$ Cores); The Fork-Join execution model; Master thread, worker team, implicit barriers; Pragma sentinels (`#pragma omp`); Fallback comment behavior; Version macro `_OPENMP`; Basic parallel region construct.
- **Source Mapping:** `3d773f8d.pdf` (Pages 9–11); `reference/teaching_style_sample.md` (Sections 11–14).
- **Prerequisites:** Topic 02.
- **University Exam Value:** Very High (Standard introductory OpenMP questions and fork-join diagrams).

#### Subtopic 05.2: Thread Management & Runtime Environment APIs
- **Concepts Covered:** OpenMP runtime functions (`omp_get_thread_num`, `omp_get_num_threads`, `omp_set_num_threads`); `num_threads` clause; Shell environment variable `OMP_NUM_THREADS`; Nondeterminism of concurrent output ordering; Difference between setting thread count and querying thread count.
- **Source Mapping:** `3d773f8d.pdf` (Page 12); `reference/teaching_style_sample.md` (Sections 4–6).
- **Prerequisites:** Subtopic 05.1.
- **University Exam Value:** High (Code tracing and output prediction questions).

#### Subtopic 05.3: Data Scoping Architecture: Shared, Private, Firstprivate & Variable Masking
- **Concepts Covered:** Memory model: shared process heap/global memory vs private thread stacks; Stack limits and `OMP_STACKSIZE`; Scoping rules; Unintentionally shared variables and data races; Variable masking; Private variable uninitialized state hazard; `firstprivate` clause; `lastprivate` clause; `default(none)` clause best practices.
- **Source Mapping:** `3d773f8d.pdf` (Pages 12–16); `reference/teaching_style_sample.md` (Sections 7–12).
- **Prerequisites:** Subtopic 05.1.
- **University Exam Value:** Highest Priority (Data scoping rules and `firstprivate` bugs are major exam traps).

#### Subtopic 05.4: Loop Work-Sharing (`#pragma omp for`) & Loop Scheduling Policies
- **Concepts Covered:** Work-sharing loops; Difference between `parallel` and `parallel for`; Canonical loop requirements; The 4 scheduling policies:
  - `schedule(static [, chunk])`: Round-robin chunking, low overhead, idle thread bug.
  - `schedule(dynamic [, chunk])`: Work queue model, load balancing irregular work, runtime overhead.
  - `schedule(guided [, min_chunk])`: Exponentially decaying chunks, balancing overhead and load balance.
  - `schedule(auto)`: Runtime delegation.
- **Source Mapping:** `3d773f8d.pdf` (Pages 16–18); `reference/teaching_style_sample.md` (Sections 13–27).
- **Prerequisites:** Subtopic 05.3.
- **University Exam Value:** Critical (Scheduling policy comparison tables and iteration distribution numericals).

#### Subtopic 05.5: OpenMP Synchronization: Critical, Atomic, Barrier, Single & Ordered
- **Concepts Covered:** Mutual exclusion with `#pragma omp critical`; Hardware atomic updates with `#pragma omp atomic` (performance advantage); Explicit barriers (`#pragma omp barrier`); Single execution construct (`#pragma omp single [nowait]`); Master construct (`#pragma omp master`); Sequential execution ordering within parallel loops (`#pragma omp ordered`); 3-way synchronization comparison table.
- **Source Mapping:** `3d773f8d.pdf` (Pages 19–21); `reference/teaching_style_sample.md` (Sections 30–42).
- **Prerequisites:** Subtopic 05.4.
- **University Exam Value:** Highest Priority (Critical vs Atomic and Barrier vs Single comparison questions).

#### Subtopic 05.6: The Reduction Clause: Parallel Accumulation Without Races
- **Concepts Covered:** The shared accumulator race condition problem; The `reduction(operator: list)` clause; Mathematical requirements: associative and commutative operators; Thread-local private accumulators; Identity value initialization ($0$ for $+$, $1$ for $*$, $-\infty$ for $\max$); Tree-based reduction at implicit barrier; Array summation and dot product code examples.
- **Source Mapping:** `3d773f8d.pdf` (Page 22); `Sanjay_Notes_1.pdf` (Page 10).
- **Prerequisites:** Subtopic 05.5.
- **University Exam Value:** Very High (Standard code rewriting question: converting unsafe loop to reduction).

#### Subtopic 05.7: OpenMP Task Parallelism: Tasks, Task Loops & Granularity Controls
- **Concepts Covered:** Limitations of loop work-sharing on irregular/recursive problems; The Tasking model: asynchronous work units, task creation (`#pragma omp task`), task execution, and task pool; Single producer pattern (`#pragma omp single { #pragma omp task ... }`); Recursive tree traversal example; Task loops (`#pragma omp taskloop`); Controlling granularity: `grainsize(G)` vs `num_tasks(K)` on massive loops ($8 \times 10^9$ iterations).
- **Source Mapping:** `3d773f8d.pdf` (Pages 23–24).
- **Prerequisites:** Subtopic 05.5.
- **University Exam Value:** High (Modern OpenMP feature question on tasking vs worksharing loops).

#### Subtopic 05.8: OpenMP SIMD Vectorization & Latent Concurrency Bugs
- **Concepts Covered:** Core-level data parallelism with `#pragma omp simd`; Vector register exploitation without thread creation; GCC compilation flags: `-O3`, `-fopenmp`, `-fopt-info-vec`, `-march=native`; Combined construct `#pragma omp parallel for simd`; SIMD reduction; **The Latent Concurrency Bug Hazard:** Missing `private(tmp)` clause passing at `-O0` but failing catastrophically at `-O3 -march=native`.
- **Source Mapping:** `3d773f8d.pdf` (Pages 25–26); `Sanjay_Notes_1.pdf` (Pages 5–6).
- **Prerequisites:** Subtopic 01.3 & Subtopic 05.4.
- **University Exam Value:** Very High (Tricky code debugging question on latent vectorization bugs).

---

### TOPIC 06: Distributed-Memory Programming with MPI
*Message passing programming, process lifecycle, communication paradigms, execution management, and dynamic client-server connections.*

#### Subtopic 06.1: MPI Architecture, SPMD Model & Process Lifecycle
- **Concepts Covered:** Distributed memory architectural foundations; Major MPI implementations (OpenMPI, MPICH, Intel MPI); Single Program Multiple Data (SPMD) execution; Process address space isolation; Communicators and `MPI_COMM_WORLD`; Process rank and size; Core lifecycle APIs: `MPI_Init`, `MPI_Comm_rank`, `MPI_Comm_size`, `MPI_Finalize`; Basic MPI Hello World program.
- **Source Mapping:** `IMG_1333.pdf` (Pages 16–17); `Sanjay_Notes_1.pdf` (Page 11).
- **Prerequisites:** Topic 02.
- **University Exam Value:** Very High (Basic MPI program structure and communicator definitions).

#### Subtopic 06.2: Point-to-Point Communication: `MPI_Send` & `MPI_Recv`
- **Concepts Covered:** Point-to-point communication semantics; Message envelope components (buffer, count, datatype, destination/source rank, message tag, communicator); `MPI_Send` blocking characteristics; `MPI_Recv` blocking characteristics; Status object (`MPI_Status`); Safe communication patterns; Deadlock prevention in bidirectional ping-pong exchanges.
- **Source Mapping:** `IMG_1333.pdf` (Pages 17–19).
- **Prerequisites:** Subtopic 06.1.
- **University Exam Value:** Critical (Writing point-to-point send/receive programs and envelope explanations).

#### Subtopic 06.3: Process Execution Management & CLI Controls
- **Concepts Covered:** Compilation with wrapper compilers (`mpicc`); Process launcher `mpirun` / `mpiexec`; Process placement and CPU affinity; CLI flags: `-np <N>`, `--oversubscribe` (running more processes than physical cores), `--display-allocation`; Hostfiles and multi-node execution; Inspecting runtime process IDs (`getpid()`, `ps aux`).
- **Source Mapping:** `IMG_1333.pdf` (Pages 20–21).
- **Prerequisites:** Subtopic 06.1.
- **University Exam Value:** High (Practical execution questions on oversubscription and process mapping).

#### Subtopic 06.4: Collective Communication Primitives Overview
- **Concepts Covered:** Collective vs point-to-point communication; Participation requirements; Broadcast (`MPI_Bcast`); Reduction (`MPI_Reduce`, `MPI_Allreduce`); Data distribution: `MPI_Scatter` and `MPI_Gather`; Global synchronization (`MPI_Barrier`); Algorithmic communication complexity.
- **Source Mapping:** `IMG_1333.pdf` (Page 21).
- **Prerequisites:** Subtopic 06.2.
- **University Exam Value:** Very High (Definitions and diagrams of collective communication patterns).

#### Subtopic 06.5: Dynamic Client-Server Connections & Port-Based Architecture
- **Concepts Covered:** MPI-2 dynamic process management; Connecting independent MPI programs outside `MPI_COMM_WORLD`; Port architecture: `MPI_Open_port` and `MPI_Close_port`; Publishing port identifiers (file-based `port.txt` vs name server); Handshake primitives: `MPI_Comm_accept` (server) and `MPI_Comm_connect` (client); Inter-communicator establishment; Complete compilable `sender.c` and `receiver.c` implementations; Centralized name server daemon (`ompi-server --no-daemonize -report-uri`) and PMIx server URI binding (`--mca pmix_server_uri`).
- **Source Mapping:** `Document 12.pdf` (Pages 1–5); `IMG_1333.pdf` (Pages 22–27).
- **Prerequisites:** Subtopic 06.2 & Subtopic 06.3.
- **University Exam Value:** Highest Priority (Advanced laboratory and university examination topic on MPI dynamic process management).
