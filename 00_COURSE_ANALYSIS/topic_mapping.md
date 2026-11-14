# Reverse Source-to-Topic Mapping

**Scope:** Complete cross-index connecting every source document and page range to the corresponding module and subtopic in the study system.

---

## 1. Source-by-Source Incorporation Matrix

### SRC-01: `3d773f8d-2e9e-426e-8ef8-0f93e447813a.pdf` (26 Pages)

| Page Range | Topic in Source Note | Incorporated Study File Path |
| :--- | :--- | :--- |
| **Pages 1–2** | Scalability assumptions, execution time $T(N)$, Speedup $S(N)$, Efficiency $\mathcal{E}(N)$ | `notes_study/03_Performance_and_Scaling_Laws/01_Speedup_Efficiency_and_Scalability_Metrics.md` |
| **Pages 2–3** | Amdahl's Law derivation, asymptotic limits, linear overhead failure model $C(N) = kN$ | `notes_study/03_Performance_and_Scaling_Laws/02_Amdahls_Law_and_Overhead_Failure_Models.md` |
| **Pages 4–5** | Weak scaling, Gustafson's Law derivation, socket boundary scaling drop-offs | `notes_study/03_Performance_and_Scaling_Laws/03_Gustafson_Barsis_Law_and_Weak_Scaling.md` |
| **Pages 6–7** | Hardware bottlenecks ($T_{\min} = \max(T_i)$), load imbalance, DAXPY bandwidth numerical | `notes_study/03_Performance_and_Scaling_Laws/04_Hardware_Resource_Bottlenecks_and_DAXPY_Roofline.md` |
| **Pages 8–9** | Benchmarking Venn diagram (Application, Micro, Mini-apps), speedup vs performance fallacy | `notes_study/03_Performance_and_Scaling_Laws/05_Benchmarking_Methodologies_and_The_Speedup_Fallacy.md` |
| **Pages 9–11** | OpenMP architecture, layers, Fork-Join execution model, syntax sentinels, version macro | `notes_study/05_OpenMP_Programming/01_OpenMP_Architecture_and_Fork_Join_Model.md` |
| **Page 12** | Thread query/control APIs (`omp_get_thread_num`, `omp_get_num_threads`), execution order | `notes_study/05_OpenMP_Programming/02_Thread_Management_and_Runtime_APIs.md` |
| **Pages 12–16**| Data scoping rules, thread stacks, `OMP_STACKSIZE`, shared races, variable masking, `firstprivate` | `notes_study/05_OpenMP_Programming/03_Data_Scoping_Private_Shared_and_Masking.md` |
| **Pages 16–18**| Loop work-sharing (`#pragma omp for`), 4 scheduling policies (static, dynamic, guided, auto) | `notes_study/05_OpenMP_Programming/04_Loop_Worksharing_and_Scheduling_Policies.md` |
| **Pages 19–21**| Synchronization: Critical section, atomic memory update, barriers, single (`nowait`), ordered | `notes_study/05_OpenMP_Programming/05_Synchronization_Critical_Atomic_and_Barriers.md` |
| **Page 22** | Reduction clause (`reduction(+: sum)`), associative/commutative rules, identity init | `notes_study/05_OpenMP_Programming/06_Reductions_and_Ordered_Execution.md` |
| **Pages 23–24**| Task parallelism (`#pragma omp task`), single producer pattern, `taskloop`, grainsize | `notes_study/05_OpenMP_Programming/07_Task_Parallelism_and_Task_Loops.md` |
| **Pages 25–26**| SIMD vectorization (`#pragma omp simd`), compiler flags, latent concurrency bug hazard | `notes_study/05_OpenMP_Programming/08_OpenMP_SIMD_Vectorization_and_Latent_Bugs.md` |

---

### SRC-02: `Document 12.pdf` (5 Pages)

| Page Range | Topic in Source Note | Incorporated Study File Path |
| :--- | :--- | :--- |
| **Pages 1–3** | MPI Dynamic Connection architecture, `ompi-server`, port files, sender/receiver workflow | `notes_study/06_MPI_Distributed_Programming/05_Dynamic_Client_Server_Connections_and_Ports.md` |
| **Pages 4–5** | Complete C source code for `sender.c` and `receiver.c`, port handshake, disconnect | `notes_study/06_MPI_Distributed_Programming/05_Dynamic_Client_Server_Connections_and_Ports.md` |

---

### SRC-03: `IMG_1333.pdf` (27 Pages)

| Page Range | Topic in Source Note | Incorporated Study File Path |
| :--- | :--- | :--- |
| **Pages 1–4** | POSIX standards, OS compliance, Pthreads vs OpenMP, `pthread_t`, `pthread_create`, `join` | `notes_study/04_Pthreads_Programming/01_POSIX_Standards_and_Pthreads_Lifecycle.md` |
| **Page 5** | `pthread_attr_t`, stack size, scheduling policies (`SCHED_OTHER`, `FIFO`, `RR`), detach state | `notes_study/04_Pthreads_Programming/02_Thread_Attributes_and_Scheduling_Policies.md` |
| **Pages 6–9** | Mutex lifecycle (`pthread_mutex_t`), critical section protection, counter increment code | `notes_study/04_Pthreads_Programming/03_Mutex_Synchronization_and_Critical_Sections.md` |
| **Pages 10–12**| Condition variables (`pthread_cond_t`), Producer-Consumer Bounded Buffer complete code | `notes_study/04_Pthreads_Programming/04_Condition_Variables_and_Producer_Consumer.md` |
| **Pages 13–16**| POSIX Semaphores (`<semaphore.h>`), Read-Write Locks (`pthread_rwlock_t`), Blackboard analogy | `notes_study/04_Pthreads_Programming/05_POSIX_Semaphores_and_Read_Write_Locks.md` |
| **Pages 17–20**| POSIX Barriers (`pthread_barrier_t`), Spinlocks vs Mutex, Performance cliff curve | `notes_study/04_Pthreads_Programming/06_POSIX_Barriers_Spinlocks_and_The_Contention_Cliff.md` |
| **Pages 16–17**| MPI distributed memory fundamentals, implementations, `MPI_Init`, `rank`, `size`, `Finalize` | `notes_study/06_MPI_Distributed_Programming/01_MPI_Architecture_and_Process_Lifecycle.md` |
| **Pages 17–19**| Point-to-point communication: `MPI_Send` and `MPI_Recv`, message envelopes, data types | `notes_study/06_MPI_Distributed_Programming/02_Point_to_Point_Communication_Send_Recv.md` |
| **Pages 20–21**| Execution management: `mpicc`, `mpirun`, `--oversubscribe`, hostfiles, `getpid()` | `notes_study/06_MPI_Distributed_Programming/03_Execution_Management_and_CLI_Controls.md` |
| **Page 21** | Collective communication overview: Broadcast, Reduction, Scatter, Gather, Barrier | `notes_study/06_MPI_Distributed_Programming/04_Collective_Communication_Overview.md` |
| **Pages 22–27**| Dynamic client-server connection, port management, `ompi-server`, receiver loop throttling | `notes_study/06_MPI_Distributed_Programming/05_Dynamic_Client_Server_Connections_and_Ports.md` |

---

### SRC-04: `Pthreada_Aagam_Notes.pdf` (20 Pages)

| Page Range | Topic in Source Note | Incorporated Study File Path |
| :--- | :--- | :--- |
| **Pages 1–4** | POSIX standards, OS compliance, Pthreads vs OpenMP, `pthread_t`, `pthread_create`, `join` | `notes_study/04_Pthreads_Programming/01_POSIX_Standards_and_Pthreads_Lifecycle.md` |
| **Page 5** | `pthread_attr_t`, stack size, scheduling policies (`SCHED_OTHER`, `FIFO`, `RR`), detach state | `notes_study/04_Pthreads_Programming/02_Thread_Attributes_and_Scheduling_Policies.md` |
| **Pages 6–9** | Mutex lifecycle (`pthread_mutex_t`), critical section protection, counter increment code | `notes_study/04_Pthreads_Programming/03_Mutex_Synchronization_and_Critical_Sections.md` |
| **Pages 10–12**| Condition variables (`pthread_cond_t`), Producer-Consumer Bounded Buffer complete code | `notes_study/04_Pthreads_Programming/04_Condition_Variables_and_Producer_Consumer.md` |
| **Pages 13–16**| POSIX Semaphores (`<semaphore.h>`), Read-Write Locks (`pthread_rwlock_t`), Blackboard analogy | `notes_study/04_Pthreads_Programming/05_POSIX_Semaphores_and_Read_Write_Locks.md` |
| **Pages 17–20**| POSIX Barriers (`pthread_barrier_t`), Spinlocks vs Mutex, Performance cliff curve | `notes_study/04_Pthreads_Programming/06_POSIX_Barriers_Spinlocks_and_The_Contention_Cliff.md` |

---

### SRC-05: `Sanjay_Notes_1.pdf` (12 Pages)

| Page Range | Topic in Source Note | Incorporated Study File Path |
| :--- | :--- | :--- |
| **Pages 1–2** | Performance definition ($P = \text{Work}/\text{Time}$), Von Neumann computer, memory wall | `notes_study/01_Hardware_Foundations/01_Performance_and_Von_Neumann_Architecture.md` |
| **Pages 2–5** | Pipelining principles, 5-stage FP multiplier pipeline, OoO execution, execution ports | `notes_study/01_Hardware_Foundations/02_Pipelining_and_Superscalar_Execution.md` |
| **Pages 3, 5–6**| SMT / Hyper-Threading, SIMD vectorization (SSE, AVX, AVX-512), Peak Core Performance | `notes_study/01_Hardware_Foundations/03_SMT_SIMD_and_Peak_Performance_Model.md` |
| **Pages 7–8** | Memory hierarchy latency/bandwidth table, cache lines, analytical data transfer model | `notes_study/01_Hardware_Foundations/04_Memory_Hierarchy_and_Data_Transfer_Models.md` |
| **Pages 8–9, 11**| Flynn's classical taxonomy 2x2 matrix (SISD, MISD, SIMD, MIMD), modern hybrid clusters | `notes_study/02_Parallel_Architectures/01_Flynns_Taxonomy_and_Modern_Hybrids.md` |
| **Pages 9–10** | Shared-memory architecture, UMA vs ccNUMA, multi-core chip socket organization | `notes_study/02_Parallel_Architectures/02_Shared_Memory_UMA_vs_ccNUMA.md` |
| **Page 11** | Distributed-memory architecture, historical clusters vs modern ccNUMA clusters | `notes_study/02_Parallel_Architectures/03_Distributed_Memory_Clusters_and_Interconnects.md` |
| **Page 12** | Speedup and Efficiency definitions, distinction between scalability and performance | `notes_study/03_Performance_and_Scaling_Laws/01_Speedup_Efficiency_and_Scalability_Metrics.md` |
| **Page 10** | OpenMP directives overview, syntax sentinel, parallel for reduction snippet | `notes_study/05_OpenMP_Programming/01_OpenMP_Architecture_and_Fork_Join_Model.md` & `notes_study/05_OpenMP_Programming/06_Reductions_and_Ordered_Execution.md` |
