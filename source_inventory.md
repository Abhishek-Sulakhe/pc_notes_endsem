# Source Inventory: Parallel Computing Course Materials

**Total Sources Analyzed:** 5 PDF Documents  
**Total Pages Inspected:** 90 Scanned & Handwritten Pages  
**Extraction Methodology:** 100% Visual and Multimodal Transcription with Page-Level Verification  

---

## 1. Document Inventory Table

| Source Identifier | Original Filename | Format | Page Count | Primary Subject Areas | Extracted Text Reference |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **SRC-01** | `3d773f8d-2e9e-426e-8ef8-0f93e447813a.pdf` | Scanned handwritten notes | 26 pages | Parallel Scaling, Amdahl's & Gustafson's Laws, Hardware Bottlenecks (Roofline/DAXPY), Benchmarking, OpenMP Core Directives, Scoping, Scheduling, Synchronization, Tasks, SIMD | [`3d773f8d_extraction.md`](file:///d:/Academics/3rd%20year/PC/study/00_COURSE_ANALYSIS/extracted_sources/3d773f8d_extraction.md) |
| **SRC-02** | `Document 12.pdf` | Scanned handwritten notes | 5 pages | MPI-2 Dynamic Process Architecture, Port-based Inter-communicators, `ompi-server`, Client-Server Handshake | [`Document_12_extraction.md`](file:///d:/Academics/3rd%20year/PC/study/00_COURSE_ANALYSIS/extracted_sources/Document_12_extraction.md) |
| **SRC-03** | `IMG_1333.pdf` | Scanned handwritten notes | 27 pages | POSIX Threads (Pthreads) lifecycle & attributes, Complete Synchronization Suite, Mutex vs Spinlock Cliff, MPI Architecture, Point-to-Point, Collective Ops, Execution Flags, MPI-2 Dynamic Ports & `ompi-server` | [`IMG_1333_extraction.md`](file:///d:/Academics/3rd%20year/PC/study/00_COURSE_ANALYSIS/extracted_sources/IMG_1333_extraction.md) |
| **SRC-04** | `Pthreada_Aagam_Notes.pdf` | Scanned handwritten notes | 20 pages | POSIX standards, Pthreads API, `pthread_attr_t`, Scheduling policies, Mutexes, Condition Variables, Producer-Consumer Bounded Buffer, Semaphores, RW Locks, Barriers, Spinlocks | [`Pthreada_Aagam_Notes_extraction.md`](file:///d:/Academics/3rd%20year/PC/study/00_COURSE_ANALYSIS/extracted_sources/Pthreada_Aagam_Notes_extraction.md) |
| **SRC-05** | `Sanjay_Notes_1.pdf` | Scanned handwritten notes | 12 pages | Fundamental Performance Metrics, Modern CPU Microarchitecture, Pipelining (5-stage FP), Superscalarity, SMT, SIMD/AVX, Peak Core Performance Formula, Memory Hierarchy, Latency/Bandwidth Model, Flynn's Taxonomy, UMA vs ccNUMA, Speedup vs Efficiency | [`Sanjay_Notes_1_extraction.md`](file:///d:/Academics/3rd%20year/PC/study/00_COURSE_ANALYSIS/extracted_sources/Sanjay_Notes_1_extraction.md) |

---

## 2. In-Depth Source Profiles

### Source 1: `3d773f8d-2e9e-426e-8ef8-0f93e447813a.pdf` (26 Pages)
- **Author/Context:** Comprehensive lecture and lab notes covering performance theory and deep OpenMP programming.
- **Key Academic Topics Covered:**
  - Mathematical formalization of Speedup $S(N)$ and Parallel Efficiency $\mathcal{E}(N)$.
  - Amdahl's Law derivation, asymptotic limits, and the linear overhead failure model ($C(N) = kN$).
  - Weak scaling and Gustafson's Law ($S(N) = s + (1-s)N$).
  - Load imbalance and hardware resource bottleneck modeling ($T_{min} = \max(T_1, \dots, T_n)$).
  - Concrete DAXPY numerical problem calculating memory bandwidth saturation versus floating-point peak.
  - Benchmarking taxonomy (Application vs Microbenchmarking vs Mini-apps / Proxy apps) and the speedup vs absolute performance fallacy (Commodity cluster vs NEC SX vector supercomputer).
  - OpenMP architecture: programmer view, runtime view, OS threads, hardware cores.
  - Fork-Join model, pragma sentinels, fallback comments, and `_OPENMP` version macro.
  - Data scoping rules, thread-local stacks, `OMP_STACKSIZE`, shared variable races, variable masking, `firstprivate`.
  - Loop scheduling policies (`static`, `dynamic`, `guided`, `auto`), chunk size effects, idle thread bugs.
  - OpenMP synchronization: `#pragma omp critical` vs `#pragma omp atomic`, `#pragma omp barrier`, `#pragma omp single` with `nowait`, `#pragma omp ordered`, `reduction` clause.
  - Advanced OpenMP: Task parallelism (`#pragma omp task`, `#pragma omp taskloop`, grainsize, num_tasks), SIMD vectorization (`#pragma omp simd`, compiler flags, latent concurrency bugs).
- **Unique Value / Standout Elements:**
  - Highly quantitative mathematical derivations.
  - Fully worked-out DAXPY numerical problem.
  - Latent concurrency bugs in vectorization.
  - Taskloop granularity analysis.
- **Extraction Challenges:**
  - Handwritten mathematical subscripts and equations required precise KaTeX formatting.

---

### Source 2: `Document 12.pdf` (5 Pages)
- **Author/Context:** Lab-focused implementation notes on advanced MPI-2 client-server communication.
- **Key Academic Topics Covered:**
  - Dynamic inter-communicator establishment between independently started MPI executables outside `MPI_COMM_WORLD`.
  - Server process workflow: `MPI_Open_port`, publishing port via file (`port.txt`), `MPI_Comm_accept`.
  - Client process workflow: reading port from file, `MPI_Comm_connect`.
  - Message exchange and clean teardown (`MPI_Comm_disconnect`, `MPI_Close_port`, `MPI_Finalize`).
  - Open MPI server daemon (`ompi-server`) command-line flags and PMIx server URI binding.
- **Unique Value / Standout Elements:**
  - Complete, compilable C code implementations for both `sender.c` and `receiver.c`.
  - Practical shell command invocations with Open MPI.
- **Overlaps:** Overlaps directly with Pages 22–27 of `IMG_1333.pdf`.

---

### Source 3: `IMG_1333.pdf` (27 Pages)
- **Author/Context:** Unified student notebook covering both shared-memory (Pthreads) and distributed-memory (MPI).
- **Key Academic Topics Covered:**
  - **Pages 01–15, 18:** POSIX standards, Pthreads API (`pthread_t`, `pthread_create`, `pthread_join`, `pthread_attr_t`, scheduling policies `SCHED_OTHER`, `SCHED_FIFO`, `SCHED_RR`), complete synchronization library (Mutex, Condition Variables, Bounded Buffer Producer-Consumer, Semaphores via `<semaphore.h>`, Read-Write Locks, Barriers, Spinlocks).
  - **Page 19:** Quantitative Spinlock vs Mutex performance curve (Spinlock advantage at $<10^2$ cycles, steep cliff under contention).
  - **Pages 16–21:** MPI distributed architecture, core API (`MPI_Init`, `MPI_Comm_rank`, `MPI_Comm_size`, `MPI_Finalize`), point-to-point communication (`MPI_Send`, `MPI_Recv`), process management flags (`--oversubscribe`, hostfiles).
  - **Pages 22–27:** MPI-2 dynamic client-server communication, `ompi-server` setup, and asynchronous receiver throttling.
- **Unique Value / Standout Elements:**
  - The cleanest bridge connecting shared-memory multi-threading (Pthreads) directly to distributed-memory message passing (MPI).
  - Detailed CLI command-line explanations for Open MPI.
- **Overlaps:**
  - Pages 01–20 share almost identical content and diagrams with `Pthreada_Aagam_Notes.pdf`.
  - Pages 22–27 share identical client-server MPI code with `Document 12.pdf`.

---

### Source 4: `Pthreada_Aagam_Notes.pdf` (20 Pages)
- **Author/Context:** Dedicated, high-quality handwritten reference solely focused on POSIX Threads (Pthreads).
- **Key Academic Topics Covered:**
  - POSIX standard definitions, OS certifications, and compliance differences.
  - Complete thread lifecycle management in C.
  - Detailed breakdown of `pthread_attr_t` attributes and real-time scheduling.
  - Comprehensive synchronization implementations: Mutexes, Condition Variables, Semaphores, RW Locks, Barriers, Spinlocks.
  - Real-world analogies (e.g., Blackboard reader-writer analogy).
  - Complete Producer-Consumer bounded buffer code.
  - Mutex vs Spinlock engineering trade-offs and contention curve.
- **Unique Value / Standout Elements:**
  - Clear, methodical code structures with explicit parameter-by-parameter explanations.
  - Specific examination warnings (e.g., *"He will ask all this!"* regarding thread attributes).
- **Overlaps:** Corresponds directly to pages 01–20 of `IMG_1333.pdf`.

---

### Source 5: `Sanjay_Notes_1.pdf` (12 Pages)
- **Author/Context:** Foundational hardware and systems architecture lecture notes.
- **Key Academic Topics Covered:**
  - Definition of computational performance ($P = \text{Work} / \text{Time}$).
  - Von Neumann stored-program computer model and the memory-wall problem.
  - Modern CPU microarchitecture: instruction pipeline, out-of-order execution, reorder buffer, register renaming, execution port scheduling.
  - 4 Pillars of in-core performance: Pipelining, Superscalarity, SIMD, SMT.
  - 5-stage floating-point multiplication pipeline walkthrough.
  - x86 SIMD vector registers (SSE 128-bit, AVX 256-bit, AVX-512 512-bit) and compiler vectorization.
  - Master Core Peak Performance Formula: $P_{\text{core}} = n_{\text{superscalar}}^{\text{FP}} \cdot n_{\text{FMA}} \cdot n_{\text{SIMD}} \cdot f$.
  - Memory hierarchy: quantitative latency (1 ns to 1 ms) and bandwidth (1 TB/s to 1 GB/s) table.
  - Data transfer model: $T = \lambda + N/b$, Effective Bandwidth $b_{\text{eff}} = N / (\lambda + N/b)$.
  - Flynn's 2x2 Taxonomy (SISD, MISD, SIMD, MIMD).
  - Shared memory architectures: UMA vs ccNUMA.
  - Evolution of distributed memory: historical pure clusters vs modern hierarchical clusters (cores $\rightarrow$ ccNUMA sockets $\rightarrow$ networked nodes).
  - Speedup and Efficiency definitions.
- **Unique Value / Standout Elements:**
  - Essential physical and microarchitectural foundation required to understand why OpenMP, Pthreads, and MPI behave the way they do.
  - Exact quantitative latency/bandwidth numbers and core peak performance formula.
- **Extraction Challenges:**
  - Detailed microarchitecture diagrams with multiple execution ports and buses transcribed into ASCII schemas.

---

## 3. Page-Level Extraction Checklist

| Document | Page Range | Extraction Status | Dedicated Extracted File |
| :--- | :--- | :--- | :--- |
| `3d773f8d-2e9e-426e-8ef8-0f93e447813a.pdf` | Pages 1–26 (26/26) | **100% COMPLETE** | `00_COURSE_ANALYSIS/extracted_sources/3d773f8d_extraction.md` |
| `Document 12.pdf` | Pages 1–5 (5/5) | **100% COMPLETE** | `00_COURSE_ANALYSIS/extracted_sources/Document_12_extraction.md` |
| `IMG_1333.pdf` | Pages 1–27 (27/27) | **100% COMPLETE** | `00_COURSE_ANALYSIS/extracted_sources/IMG_1333_extraction.md` |
| `Pthreada_Aagam_Notes.pdf` | Pages 1–20 (20/20) | **100% COMPLETE** | `00_COURSE_ANALYSIS/extracted_sources/Pthreada_Aagam_Notes_extraction.md` |
| `Sanjay_Notes_1.pdf` | Pages 1–12 (12/12) | **100% COMPLETE** | `00_COURSE_ANALYSIS/extracted_sources/Sanjay_Notes_1_extraction.md` |
| **TOTAL** | **90 / 90 Pages** | **100% INSPECTED & TRANSCRIBED** | **All 5 Sources Preserved** |
