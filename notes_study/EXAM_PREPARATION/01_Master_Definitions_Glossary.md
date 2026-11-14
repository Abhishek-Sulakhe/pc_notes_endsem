# Master Definitions & Technical Terminology Glossary

This reference glossary provides formal, precise, university-examination-ready definitions and technical explanations for all core terminology across the entire Parallel Computing curriculum.

---

## Module 01: Hardware Foundations & Microarchitecture

### 1. Von Neumann Architecture & Performance
- **Von Neumann Architecture:** A computer architecture model where CPU processing units and memory are separate entities interconnected by a shared system bus, and where programs (instructions) and data share the identical physical memory space (Stored-Program Concept).
- **Von Neumann Bottleneck (Memory Wall):** The severe performance disparity and throughput limit caused by the shared physical bus between the high-frequency CPU execution unit and the comparatively high-latency main memory.
- **Cycles Per Instruction (CPI):** The average number of clock cycles required by a microprocessor to execute an instruction: $\text{CPI} = \frac{\text{Clock Cycles}}{\text{Instruction Count}}$.
- **Instructions Per Cycle (IPC):** The inverse of CPI ($\text{IPC} = \frac{1}{\text{CPI}}$); measures instruction retirement throughput per clock period.
- **Clock Frequency ($f$):** The operational frequency of the processor crystal oscillator, expressed in Hertz ($\text{GHz} = 10^9\text{ cycles/second}$); cycle time $\tau = \frac{1}{f}$.
- **Stored-Program Concept:** The architectural principle where machine code instructions are encoded as binary numbers and stored in the same addressable memory space as computational data.

### 2. Pipelining & Superscalar Execution
- **Instruction Pipelining:** An execution technique that overlaps multiple stages of instruction processing (Fetch, Decode, Execute, Memory Access, Write-Back) across different hardware stages simultaneously.
- **Pipeline Depth ($k$):** The number of discrete stages in an execution pipeline.
- **Structural Hazard:** A hardware conflict occurring when two or more concurrent pipeline stages attempt to access the same physical hardware resource simultaneously.
- **Data Hazard:** A condition where an instruction depends on the result of a previous instruction that is still in flight through the pipeline.
  - **Read-After-Write (RAW):** True dependency; instruction $J$ tries to read before instruction $I$ writes.
  - **Write-After-Read (WAR):** Anti-dependency; instruction $J$ tries to write before instruction $I$ reads.
  - **Write-After-Write (WAW):** Output dependency; instruction $J$ tries to overwrite an operand before instruction $I$ writes.
- **Control Hazard (Branch Hazard):** A pipeline stall occurring when a branch or jump instruction alters the Program Counter ($PC$), invalidating pre-fetched instructions.
- **Branch Prediction:** Hardware heuristics (static or dynamic branch history tables) that anticipate whether a conditional jump will be taken before the condition is evaluated.
- **Superscalar Processor:** A CPU microarchitecture containing multiple redundant execution pipelines that can fetch, decode, dispatch, and execute multiple instructions per single clock cycle ($\text{IPC} > 1$).
- **Instruction-Level Parallelism (ILP):** The intrinsic potential in a sequential machine instruction stream to execute operations concurrently without altering program semantics.

### 3. SMT, SIMD & Arithmetic Intensity
- **Simultaneous Multithreading (SMT / Hyper-Threading):** A hardware technique where a single physical processor core maintains multiple architectural execution contexts (registers, program counters) while sharing execution units and caches to hide pipeline latency.
- **SIMD (Single Instruction, Multiple Data):** A data-parallel architecture where a single machine instruction operates simultaneously on multiple data elements packed into wide vector registers (e.g., AVX-512, SSE).
- **FMA (Fused Multiply-Add):** A floating-point instruction that calculates $a \cdot b + c$ in a single clock cycle with a single rounding step, doubling floating-point throughput.
- **Peak Performance ($R_{peak}$):** The theoretical upper bound of floating-point computations a processor can execute per second: $R_{peak} = \text{Sockets} \times \text{Cores} \times f \times \text{SIMD Width} \times \text{FMA Factor}$.
- **Arithmetic Intensity ($I$):** The ratio of total floating-point arithmetic operations performed to the total bytes of data transferred to/from main memory ($\text{FLOP/byte}$).

---

## Module 02: Parallel Computer Architectures & Memory Systems

### 1. Flynn's Taxonomy
- **SISD (Single Instruction, Single Data):** Traditional sequential uniprocessor architecture executing one instruction stream on one data stream.
- **SIMD (Single Instruction, Multiple Data):** Architecture where a single control unit broadcasts one instruction to multiple processing elements operating on distinct data elements.
- **MISD (Multiple Instruction, Single Data):** Architecture where multiple distinct instruction streams operate on a single shared data stream (primarily used in fault-tolerant voting systems like spacecraft flight controllers).
- **MIMD (Multiple Instruction, Multiple Data):** Systems where multiple autonomous processing nodes execute different instruction streams on completely independent data streams.
- **SPMD (Single Program, Multiple Data):** A high-level software programming model where identical binary code runs across all processing nodes, but each node branches conditionally based on its unique rank or ID.

### 2. Memory Organization & Interconnects
- **UMA (Uniform Memory Access):** Shared-memory symmetric multiprocessing (SMP) where all physical memory is equally accessible to all processors through a shared bus, resulting in identical latency for all cores.
- **ccNUMA (Cache-Coherent Non-Uniform Memory Access):** Distributed shared-memory architecture where physical memory is partitioned among local processor nodes, resulting in lower latency for local memory and higher latency for remote memory across the interconnect, with hardware-enforced cache coherency.
- **Distributed-Memory Cluster:** An architecture where each compute node possesses private physical memory; nodes share no address space and communicate exclusively via network message passing.
- **Network Diameter:** The maximum shortest path (measured in hops or links) between any two nodes in an interconnection network.
- **Bisection Bandwidth:** The minimum data transfer bandwidth across the smallest cut that divides the network into two equal halves.

---

## Module 03: Parallel Performance Analysis & Scaling Laws

### 1. Performance Metrics & Scaling
- **Speedup ($S_P$):** The ratio of sequential execution time on one processor ($T_1$) to parallel execution time on $P$ processors ($T_P$): $S_P = \frac{T_1}{T_P}$.
- **Linear Speedup:** Ideal parallel performance where $S_P = P$.
- **Superlinear Speedup:** An anomaly where $S_P > P$, typically caused by aggregate cache capacity effects where the working set fits entirely in fast cache when partitioned across multiple cores.
- **Parallel Efficiency ($E_P$):** The fraction of total processing power effectively utilized: $E_P = \frac{S_P}{P} = \frac{T_1}{P \cdot T_P}$.
- **Strong Scaling:** Scaling behavior where the total problem size is kept **fixed** while increasing the number of processors $P$.
- **Weak Scaling:** Scaling behavior where the problem size per processor is kept **fixed**, causing the total problem size to increase proportionally with $P$.

### 2. Theoretical Laws & Performance Modeling
- **Amdahl's Law:** A mathematical model stating that the maximum speedup achievable under strong scaling is strictly bounded by the serial fraction $s$ of the program: $S_P = \frac{1}{s + \frac{1-s}{P}} \le \frac{1}{s}$.
- **Serial Fraction ($s$):** The portion of an algorithm that cannot be parallelized and must execute sequentially ($s = 1 - p$).
- **Overhead Failure Model:** An extended performance model incorporating parallel coordination overhead $T_o(P)$, proving that beyond an optimal core count $P^*$, speedup degrades due to synchronization contention.
- **Gustafson-Barsis's Law:** A mathematical model for weak scaling stating that if the problem size expands with $P$, the scaled speedup grows linearly: $S_{scaled} = P - s'(P - 1)$.
- **Roofline Model:** A visual 2D performance model that plots computational performance ($G\text{FLOP/s}$) as a function of arithmetic intensity ($I$), bounded by memory bandwidth and peak processor compute performance.
- **Machine Balance ($B_m$):** The inflection point on the Roofline chart ($\text{FLOP/byte}$) representing the minimum arithmetic intensity required to transition from a memory-bound state to a compute-bound state: $B_m = \frac{\text{Peak Compute Performance}}{\text{Memory Bandwidth}}$.
- **The Speedup Fallacy:** The flawed experimental practice of calculating speedup by comparing an unoptimized parallel program against an unoptimized sequential baseline, producing artificially inflated speedup numbers.

---

## Module 04: Pthreads Programming (Shared Memory)

### 1. POSIX Threads Architecture & Management
- **Thread:** A lightweight unit of CPU execution within a process that shares the process virtual address space, heap, file descriptors, and global variables, but maintains private registers, program counter, and stack.
- **POSIX Threads (Pthreads):** The IEEE POSIX 1003.1c standard API for thread creation, control, and synchronization in shared-memory C environments.
- **Thread Control Block (TCB):** Operating system kernel data structure storing thread identification, execution state, scheduling parameters, and CPU register snapshots.
- **Joinable Thread:** A thread whose memory resources (stack and termination status) remain allocated after completion until another thread calls `pthread_join()`.
- **Detached Thread:** A thread whose storage is automatically reclaimed by the OS immediately upon termination (`pthread_detach()`); cannot be joined.

### 2. Synchronization Primitives
- **Critical Section:** A segment of code that accesses a shared mutable resource and must not be executed concurrently by multiple threads.
- **Race Condition:** A programming flaw where the final program output depends unpredictably on the precise execution timing or interleaving of concurrent threads.
- **Mutex (`pthread_mutex_t`):** A mutual exclusion lock providing exclusive access to a critical section; only the thread that acquired the lock can release it.
- **Condition Variable (`pthread_cond_t`):** A synchronization primitive that allows threads to suspend execution and yield the CPU inside a mutex until a specific state condition becomes true.
- **Spurious Wakeup:** An event where a waiting thread unblocks from a condition variable without any explicit signal; mandates that condition checks must always be wrapped in a `while` loop.
- **Counting Semaphore (`sem_t`):** A synchronization object maintaining an atomic integer counter initialized to $N$, regulating concurrent access to a finite pool of resources (`sem_wait` decrements; `sem_post` increments).
- **Binary Semaphore:** A semaphore initialized to 1; differs from a mutex because any thread can signal (`sem_post`) to unlock it, lacking strict ownership semantics.
- **Read-Write Lock (`pthread_rwlock_t`):** A synchronization lock permitting concurrent access by multiple reader threads simultaneously, but enforcing strict exclusivity for writer threads.
- **Barrier Synchronization (`pthread_barrier_t`):** A rendezvous construct where all $N$ participating threads block until all $N$ threads have arrived at the barrier.
- **Spinlock (`pthread_spinlock_t`):** A low-level lock where waiting threads execute an active busy-waiting loop ("spinning") on the CPU instead of context-switching to sleep.
- **The Contention Cliff:** The catastrophic collapse of spinlock throughput occurring when the number of competing threads exceeds physical CPU cores, causing excessive cache-line bouncing and wasted CPU cycles.

---

## Module 05: OpenMP Programming (Shared Memory Directives)

### 1. Execution Model & Data Scoping
- **OpenMP:** Open Multi-Processing; an industry-standard API specification for compiler-directed shared-memory parallelism in C, C++, and Fortran.
- **Fork-Join Model:** The execution paradigm where a program starts as a single master thread (serial execution), forks a team of parallel threads upon encountering `#pragma omp parallel`, and joins back to the master at an implicit barrier upon exiting the region.
- **Structured Block:** A block of executable code with exactly one point of entry at the top and one point of exit at the bottom (cannot branch in or out using `goto` or `return`).
- **Shared Variable:** A variable residing in process memory that is directly read and written by all threads in the parallel team.
- **Private Variable:** A variable replicated such that each thread in the team allocates an independent, unshared copy on its private stack.
- **`firstprivate` Clause:** An OpenMP data-sharing clause that creates a private instance for each thread and initializes it with the master variable's value prior to entering the region.
- **`lastprivate` Clause:** A clause specifying that the private value from the sequentially last loop iteration or last section is copied back to the original master variable upon exiting the construct.
- **`default(none)`:** A safety clause that disables implicit variable scoping, requiring the developer to explicitly categorize every referenced variable as `shared`, `private`, or `reduction`.

### 2. Worksharing, Scheduling & Synchronization
- **Work-Sharing Construct:** An OpenMP directive (such as `#pragma omp for` or `#pragma omp sections`) that distributes the execution of associated structured code across existing threads rather than creating new threads.
- **Canonical Loop Form:** The standardized format required for loops parallelized with OpenMP (integer index variable, relational condition, invariant bounds, constant stride step).
- **Static Scheduling:** A loop iteration distribution policy where iterations are divided into chunks and assigned to threads at compile/launch time using a round-robin schedule.
- **Dynamic Scheduling:** A run-time scheduling policy where threads request chunks from a shared dynamic work queue upon completing their current workload.
- **Guided Scheduling:** An adaptive scheduling policy where chunk sizes decrease exponentially over time, balancing low scheduling overhead initially with fine-grained load balancing near loop completion.
- **`#pragma omp critical`:** Directive that enforces exclusive execution of a structured block by one thread at a time across the entire team.
- **`#pragma omp atomic`:** Hardware-level synchronization directive that updates a specific memory location atomically, bypassing OS lock queues.
- **Reduction Clause:** A high-level clause (`reduction(op: var)`) that provides private accumulators for each thread initialized to the operator's identity value, combining them into a global result at the end of the region via a race-free tree reduction.
- **OpenMP Task:** An independent unit of deferred execution consisting of code, private data, and internal control variables scheduled by the runtime task pool.
- **SIMD Vectorization (`#pragma omp simd`):** Directive instructing the compiler to generate vector instructions operating on hardware SIMD registers without spawning threads.

---

## Module 06: Distributed-Memory Programming with MPI

### 1. Architecture, Lifecycle & Communication
- **MPI (Message Passing Interface):** A standardized, portable message-passing system specification designed for high-performance distributed-memory parallel computing.
- **Communicator:** An opaque handle representing a communication universe consisting of an ordered group of processes with isolated message passing contexts.
- **`MPI_COMM_WORLD`:** The default predefined communicator containing all processes created during program launch via `mpirun`.
- **Rank:** An integer identifier ranging from $0$ to $\text{Size} - 1$ that uniquely identifies a process within a given communicator.
- **Message Envelope:** The six-tuple required to identify and route an MPI message: Buffer address, Count, Datatype, Source/Destination Rank, Message Tag, and Communicator.
- **Eager Protocol:** An MPI point-to-point transmission mode where small messages are sent immediately and buffered at the receiver without waiting for the receiver to post a matching receive.
- **Rendezvous Protocol:** An MPI transmission mode where large messages require a preliminary handshake between sender and receiver before payload data transfer begins.
- **Oversubscription:** The practice of launching more MPI processes than physical CPU hardware slots or cores available on the system (`--oversubscribe`).
- **Collective Communication:** A coordinated communication primitive that requires the simultaneous participation of all processes within a communicator (e.g., `MPI_Bcast`, `MPI_Reduce`).
- **Root Process:** The designated process that acts as the sole source in one-to-all collectives (`Bcast`, `Scatter`) or the sole destination in all-to-one collectives (`Gather`, `Reduce`).
- **Binomial Spanning Tree:** A logarithmic communication topology utilized by MPI collective operations that scales as $\mathcal{O}(\log_2 P)$ steps instead of linear $\mathcal{O}(P)$ serialization.
- **MPI Port:** An opaque string generated by the runtime (`MPI_Open_port`) that encodes network endpoint addresses to enable dynamic client-server rendezvous.
- **Inter-Communicator:** A communication bridge connecting two disjoint groups of MPI processes, where rank indices reference the peer processes in the remote group.
- **`ompi-server`:** A centralized name and lookup daemon in Open MPI that provides DNS-like name publishing and resolution services for dynamic MPI ports.
