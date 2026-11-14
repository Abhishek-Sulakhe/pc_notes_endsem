# Shared-Memory Multiprocessors: UMA vs ccNUMA Architectures

In Subtopic 02.1, we introduced the MIMD classification and established that shared-memory parallel systems allow multiple processors or cores to access a common memory space.

However, *how* that shared memory is physically wired to the processor cores dramatically alters system performance, scalability, and programming considerations. 

In this subtopic, you will master the two major physical implementations of shared memory: **Uniform Memory Access (UMA)** and **Cache-Coherent Non-Uniform Memory Access (ccNUMA)**. You will also examine the multi-core chip layout from your lecture notes and understand the two greatest hardware hazards of shared memory: **The Cache Coherence Problem** and **False Sharing**.

---

## 1. What is Shared Memory?

### Precise Definition from Course Notes
> *"A shared-memory parallel system is a system in which multiple processors or cores access a single, common memory address space."*

In a shared-memory computer:
- There is only **one physical address range** (e.g., address `0x0000` to `0xFFFF`).
- Any processor core can read from or write to any memory address in that range using standard load and store machine instructions (`mov`).
- Communication between parallel threads is **implicit**: if Thread 0 writes a value to shared variable `x`, Thread 1 can immediately read that updated value from `x`.

```
                        SHARED-MEMORY CONCEPT
                        
                     ┌────────────────────────┐
                     │      SHARED MEMORY     │
                     │  (Common Address Space)│
                     │         x = 42         │
                     └───────────┬────────────┘
                                 │
                     ┌───────────┼───────────┐
                     │           │           │
                     ▼           ▼           ▼
                   Core 0      Core 1      Core 2
                   writes      reads       reads
                   x = 42      x (sees 42) x (sees 42)
```

---

## 2. Uniform Memory Access (UMA)

### Basic Intuition: The Round Dining Table
Imagine four people sitting around a small round dining table with a bowl of salad placed precisely in the center. Every person is equidistant from the bowl. Reaching out to take salad takes exactly the same physical effort, time, and reach for everyone at the table.

This is **Uniform Memory Access (UMA)**:

```
                    UNIFORM MEMORY ACCESS (UMA)
                    
       ┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐
       │  Core 0  │    │  Core 1  │    │  Core 2  │    │  Core 3  │
       └────┬─────┘    └────┬─────┘    └────┬─────┘    └────┬─────┘
            │               │               │               │
       ─────┴───────────────┴───────────────┴───────────────┴─────
                            SHARED SYSTEM BUS
       ─────────────────────────────┬─────────────────────────────
                                    │
                        ┌───────────┴───────────┐
                        │      MAIN MEMORY      │
                        │        (DRAM)         │
                        └───────────────────────┘
```

---

### Key Properties of UMA
1. **Equal Access Latency & Bandwidth:**
   - Every processor core has physical access to memory with identical latency and identical bandwidth, regardless of which memory bank or core is involved.
2. **Symmetric Multiprocessing (SMP):**
   - In traditional hardware literature, UMA systems are also known as **Symmetric Multiprocessors (SMP)** because all processors have symmetric relationships to memory and I/O devices.
3. **Physical Implementation:**
   - Found inside a single multicore CPU chip where all cores connect to an on-die integrated memory controller.

---

### The Fundamental Limitation of UMA: Bus Saturation
Why don't we build 1,000-core UMA supercomputers?

Look at the shared system bus in the diagram above. Every single memory access from Core 0, Core 1, Core 2, and Core 3 must traverse the **exact same physical bus wires**:

$$\boxed{\text{Bus Bandwidth Available per Core} = \frac{\text{Total Bus Bandwidth}}{N_{\text{cores}}}}$$

As you add more cores:
- Cores spend most of their time waiting for bus arbitration (bus contention).
- The memory bus quickly saturates at roughly **16 to 32 cores**.
- Beyond this limit, adding more cores produces zero additional performance because memory contention throttles the entire system.

---

## 3. Cache-Coherent Non-Uniform Memory Access (ccNUMA)

### Basic Intuition: The Banquet Hall
To feed 100 people without everyone crowding around one small table, you set up multiple tables across a banquet hall. Each table has its own local salad bowl. 
- Taking salad from your own table takes **2 seconds** (Local Access).
- If your table runs out of dressing, you can walk across the hall to another table and borrow theirs, but walking takes **20 seconds** (Remote Access).

Everyone can still access any food in the entire hall (a single shared dining room), but **access time depends on physical distance**.

This is **Non-Uniform Memory Access (NUMA)**:

```
           CACHE-COHERENT NON-UNIFORM MEMORY ACCESS (ccNUMA)
           
          SOCKET 0 (Domain 0)                   SOCKET 1 (Domain 1)
   ┌───────────────────────────────┐     ┌───────────────────────────────┐
   │ ┌──────────┐     ┌──────────┐ │     │ ┌──────────┐     ┌──────────┐ │
   │ │  Core 0  │     │  Core 1  │ │     │ │  Core 2  │     │  Core 3  │ │
   │ └────┬─────┘     └────┬─────┘ │     │ └────┬─────┘     └────┬─────┘ │
   │      │                │       │     │      │                │       │
   │ ─────┴────────────────┴────── │     │ ─────┴────────────────┴────── │
   │   On-Chip Interconnect Bus    │     │   On-Chip Interconnect Bus    │
   │              │                │     │              │                │
   │              ▼                │     │              ▼                │
   │      [ LOCAL MEMORY 0 ]       │     │      [ LOCAL MEMORY 1 ]       │
   └──────────────┬────────────────┘     └──────────────┬────────────────┘
                  │                                     │
                  └──────────────► INTERCONNECT ◄───────┘
                                   POINT-TO-POINT
                            (UPI / Infinity Fabric)
                                  High Latency!
```

---

### Key Properties of ccNUMA
1. **Physical Distribution, Logical Sharing:**
   - Physical memory is partitioned into distinct local domains, each wired directly to a dedicated CPU socket.
   - However, the hardware presents all these disjoint memory banks as a **single, unified global address space**. A thread running on Core 0 can access memory physically plugged into Socket 1 simply by providing its address!
2. **Non-Uniform Latency & Bandwidth:**
   - **Local Memory Access:** Accessing the memory directly attached to the core's own socket is extremely fast ($\approx 60\text{ to } 70\text{ ns}$).
   - **Remote Memory Access:** Accessing memory attached to another socket requires traversing the inter-socket point-to-point link (e.g., Intel UPI or AMD Infinity Fabric), adding substantial routing latency ($\approx 120\text{ to } 150\text{ ns}$) and consuming inter-socket link bandwidth.
3. **The NUMA Factor:**
   $$\boxed{\text{NUMA Factor} = \frac{\text{Remote Memory Latency}}{\text{Local Memory Latency}} \approx 1.5\text{ to } 2.5\times}$$

---

### Why Use ccNUMA? The Architectural Trade-Off
Your lecture notes summarize the architectural trade-off with great insight:
> - **Advantage of ccNUMA:** *"Easier to build multiple memory domains with smaller local bandwidth than one massive UMA domain with ultra-high monolithic bandwidth."*
> - **Disadvantage of ccNUMA:** *"Adds NUMA topology complexity."*

By distributing memory controllers across sockets, total system memory bandwidth scales up linearly with the number of sockets! A 4-socket server has 4 times the memory bandwidth of a 1-socket server, provided threads predominantly access their local memory banks.

---

## 4. Multi-Core Chip Microarchitecture from Course Notes

Your notes present a specific architectural diagram of a modern shared-memory multi-core CPU chip. Below is the complete structural transcription:

```
            DETAILED MULTI-CORE SOCKET MICROARCHITECTURE
            
 ┌─────────────────────────────────────────────────────────────────────────────┐
 │ CPU CHIP (1 Socket = 8 Physical Cores / 16 SMT Threads)                     │
 │                                                                             │
 │ ┌─────────┬─────────┬─────────┬─────────┬─────────┬─────────┬─────────┬───────┐ │
 │ │ T1 │ T2 │ T1 │ T2 │ T1 │ T2 │ T1 │ T2 │ T1 │ T2 │ T1 │ T2 │ T1 │ T2 │ T1│T2│ │
 │ ├─────────┼─────────┼─────────┼─────────┼─────────┼─────────┼─────────┼───────┤ │
 │ │ Core P0 │ Core P1 │ Core P2 │ Core P3 │ Core P4 │ Core P5 │ Core P6 │Core P7│ │
 │ ├─────────┼─────────┼─────────┼─────────┼─────────┼─────────┼─────────┼───────┤ │
 │ │ L1D/L1I │ L1D/L1I │ L1D/L1I │ L1D/L1I │ L1D/L1I │ L1D/L1I │ L1D/L1I │L1D/L1I│ │
 │ ├─────────┼─────────┼─────────┼─────────┼─────────┼─────────┼─────────┼───────┤ │
 │ │   L2    │   L2    │   L2    │   L2    │   L2    │   L2    │   L2    │  L2   │ │
 │ └────┬────┴────┬────┴────┬────┴────┬────┴────┬────┴────┬────┴────┬────┴───┬───┘ │
 │      │         │         │         │         │         │         │        │     │
 │ ─────┴─────────┴─────────┴─────────┴─────────┴─────────┴─────────┴────────┴───  │
 │                       ON-DIE HIGH-SPEED RING / MESH BUS                     │
 │ ──────────────────────────────────────┬────────────────────────────────────  │
 │                                       │                                     │
 │ ┌─────────────────────────────────────┴───────────────────────────────────┐ │
 │ │                     SHARED ON-CHIP L3 CACHE                             │ │
 │ └─────────────────────────────────────┬───────────────────────────────────┘ │
 │                                       │                                     │
 │ ┌─────────────────────────────────────┴───────────────────────────────────┐ │
 │ │                   INTEGRATED MEMORY CONTROLLER                          │ │
 │ └─────────────────────────────────────┬───────────────────────────────────┘ │
 └───────────────────────────────────────┼─────────────────────────────────────┘
                                         │ Memory Channels
                                         ▼
                                   [ LOCAL DRAM ]
                                   
  NOTE FROM LECTURE NOTES:
  1 Chip = 8 Cores (16 Threads via SMT)
  2 Chips = 1 Dual-Socket Node = 16 Cores (32 Threads via SMT)
```

### Architectural Breakdown:
1. **Core-Level Private Resources:**
   - Each physical core ($P_0$ through $P_7$) has two hardware SMT threads ($T_1, T_2$).
   - Each core has private, ultra-fast **L1 Data (L1D)** and **L1 Instruction (L1I)** caches ($32\text{ KB}$ each, $\approx 1\text{ ns}$).
   - Each core has a private **L2 Cache** ($512\text{ KB}\text{ to } 1\text{ MB}$, $\approx 3\text{ to } 4\text{ ns}$).
2. **Chip-Level Shared Resources:**
   - All 8 cores connect via an on-die high-speed Ring or Mesh interconnect to a massive **Shared L3 Cache** ($16\text{ to } 32\text{ MB}$, $\approx 10\text{ ns}$).
   - The L3 cache interfaces with the Integrated Memory Controller.
3. **Dual-Socket Scaling:**
   - Combining 2 such chips creates a **Dual-Socket Node** with 16 physical cores, 32 threads, and a ccNUMA memory interconnect.

---

## 5. The Cache Coherence Problem

Because each core has its own private L1 and L2 caches, a profound correctness hazard arises when multiple cores cache the same shared memory location.

### The Problem Walkthrough
Consider two cores and a shared variable `x = 10` stored in main memory:

```
               THE CACHE COHERENCE INCONSISTENCY HAZARD
               
  Initial State: Variable x = 10 in Main Memory.
                 Both Core 1 and Core 2 read x.
                 
        Core 1 Cache                          Core 2 Cache
       ┌──────────────┐                      ┌──────────────┐
       │   x = 10     │                      │   x = 10     │
       └──────────────┘                      └──────────────┘
              ▲                                     ▲
              └──────────────────┬──────────────────┘
                                 │
                        ┌────────┴────────┐
                        │ Main Memory: 10 │
                        └─────────────────┘
                        
  Step 1: Core 1 executes: x = 20;
          Core 1 updates its local L1 cache to 20.
          
        Core 1 Cache                          Core 2 Cache
       ┌──────────────┐                      ┌──────────────┐
       │   x = 20     │                      │   x = 10     │  <--- STALE DATA!
       └──────────────┘                      └──────────────┘
              │                                     │
              ▼                                     ▼
        Core 1 sees 20                        Core 2 still sees 10!
        
  CONSEQUENCE: System state is inconsistent. If Core 2 executes y = x + 1,
               it computes 11 instead of 21. Program crashes or produces silent bugs!
```

---

### What is Cache Coherence?
$$\boxed{\text{Cache Coherence is the hardware mechanism that guarantees that all cached copies of a shared memory location remain strictly consistent.}}$$

### How Hardware Solves Cache Coherence
Modern hardware maintains coherence automatically using one of two protocols:

1. **Snooping Protocols (Bus-Based - UMA):**
   - Every core's cache controller continuously "snoops" (listens to) the shared memory bus.
   - When Core 1 writes to `x`, it broadcasts an **Invalidation Signal** across the bus.
   - Core 2's cache controller detects the signal and immediately marks its own cached copy of `x` as **Invalid**.
   - The next time Core 2 attempts to read `x`, it detects the invalid state, suffers a cache miss, and fetches the fresh value ($20$) from Core 1.
   - Standard protocol: **MESI** (Modified, Exclusive, Shared, Invalid).

2. **Directory-Based Protocols (Interconnect-Based - ccNUMA):**
   - On large multi-socket systems, broadcasting across a bus is impossible because there is no shared bus.
   - Instead, a centralized or distributed **Directory** tracks which sockets currently hold cached copies of each memory block.
   - When Socket 0 writes to a block, it queries the directory, which sends targeted invalidation messages *only* to the specific sockets caching that line, avoiding network flooding.

---

## 6. The Silent Performance Killer: False Sharing

While cache coherence guarantees mathematical correctness, it introduces a severe performance trap called **False Sharing**.

### What Causes False Sharing?
Recall from Subtopic 01.4 that **caches move data in 64-byte lines**, not individual bytes!

Suppose two independent threads execute on Core 1 and Core 2. The programmer declares two completely unrelated variables:
```c
int a; // Used exclusively by Thread 1
int b; // Used exclusively by Thread 2
```
If the compiler happens to place `a` and `b` next to each other in memory, **they will reside inside the exact same 64-byte cache line**!

```
                  THE FALSE SHARING PHENOMENON
                  
              SAME 64-BYTE CACHE LINE IN MEMORY
 ┌──────────────────────────────┬──────────────────────────────┐
 │          int a               │          int b               │
 │   (Updated by Core 1)        │   (Updated by Core 2)        │
 └──────────────────────────────┴──────────────────────────────┘
 
 Core 1 Cache                                     Core 2 Cache
┌──────────────────────────────┐                 ┌──────────────────────────────┐
│ [ a = 5 ] [ b = stale ]      │                 │ [ a = stale ] [ b = 8 ]      │
└──────────────────────────────┘                 └──────────────────────────────┘
        │                                                │
        ▼                                                ▼
 Core 1 writes to 'a' ──► Invalidation ──► Core 2's line invalidated!
                          Broadcast
                          
 Core 2 writes to 'b' ◄── Invalidation ◄── Core 1's line invalidated!
                          Broadcast
                          
 RESULT: The cache line bounces endlessly across the interconnect!
         ("Cache Line Ping-Pong") Execution drops to memory speed!
```

### The "Cache Line Ping-Pong" Disaster
1. Core 1 updates `a`. The cache coherence hardware invalidates the *entire 64-byte line* in Core 2's cache—even though Core 2 never touched `a`!
2. A microsecond later, Core 2 updates `b`. Core 2 must fetch the line from Core 1, and in doing so, invalidates Core 1's cache line!
3. The cache line bounces endlessly back and forth across the slow inter-core bus like a ping-pong ball.
4. **Performance collapses by $90\%\text{ to } 99\%$**, even though the code had zero algorithmic race conditions!

### How to Fix False Sharing
1. **Data Padding:** Insert unused padding bytes between variables so they reside on different cache lines:
   ```c
   struct ThreadData {
       int a;
       char pad[60]; // Pads struct to 64 bytes!
   };
   ```
2. **Use Thread-Local Variables:** Accumulate results in local stack variables and write to shared memory only once at the end.

---

## Important Comparisons

### UMA vs ccNUMA Architectures

| Characteristic | Uniform Memory Access (UMA) | Cache-Coherent NUMA (ccNUMA) |
| :--- | :--- | :--- |
| **Memory Latency** | Identical for all cores and addresses. | Non-uniform (Local is fast, Remote is slow). |
| **Memory Bandwidth** | Shared single domain; bandwidth per core shrinks with $N$. | Distributed domains; total bandwidth scales with sockets. |
| **Physical Topology** | Bus or crossbar switch. | Point-to-point interconnect network (UPI, Infinity Fabric). |
| **Scalability Limit** | Modest ($\le 16\text{ to } 32\text{ cores}$). | High (hundreds of cores across multiple sockets). |
| **Typical Platform** | Single-socket desktop/laptop PCs. | Multi-socket enterprise and HPC servers. |
| **Cache Coherence** | Snooping on shared bus (MESI). | Directory-based over point-to-point fabric. |
| **Programming Trap** | Bus saturation under memory-heavy loops. | NUMA memory placement penalty & false sharing. |

---

## Exam-Ready Definitions

### Q1. What is Uniform Memory Access (UMA)?
> **Answer:**  
> Uniform Memory Access (UMA) is a shared-memory multiprocessing architecture in which all processors access physical memory through a shared system bus with identical access latency and bandwidth, regardless of which core initiates the request.

### Q2. What is Cache-Coherent Non-Uniform Memory Access (ccNUMA)?
> **Answer:**  
> Cache-Coherent NUMA is a shared-memory architecture in which physical memory is distributed among multiple processor sockets while logically presenting a single global address space, where memory access latency and bandwidth vary depending on whether the accessed memory is local or remote, with hardware maintaining cache consistency across all sockets.

### Q3. State the Cache Coherence Problem.
> **Answer:**  
> The Cache Coherence Problem is the inconsistency that occurs in shared-memory multiprocessors when multiple processors maintain private cached copies of the same memory location, and an update performed by one processor is not immediately reflected in the cached copies held by other processors, leading to stale reads.

### Q4. What is False Sharing?
> **Answer:**  
> False sharing is a performance degradation hazard in multi-threaded programs that occurs when threads running on different cores modify independent, logically distinct variables that reside within the same physical 64-byte cache line, causing the cache coherence hardware to constantly invalidate and bounce the cache line between cores.

---

## Conceptual Examination Questions

### Q1. Why is ccNUMA preferred over UMA for modern high-core-count enterprise servers?
> **Model Answer:**  
> In a UMA system, all memory transactions must share a single, monolithic system bus or central memory controller. As the core count increases, the bus becomes an insurmountable bottleneck, causing cores to stall waiting for memory access. In contrast, ccNUMA partitions memory into local domains attached directly to each processor socket. Each socket has its own dedicated memory controller and memory channels. This allows aggregate memory bandwidth to scale linearly with the number of sockets, enabling servers to scale to dozens or hundreds of cores without bus saturation.

### Q2. How can a programmer detect and eliminate false sharing in a parallel loop accumulating results into an array?
> **Model Answer:**  
> Consider an array `sum[NUM_THREADS]` where each thread accumulates its partial sum into `sum[thread_id]`. Because `sum[0]`, `sum[1]`, etc., occupy contiguous integers (4 bytes each), all thread accumulators sit on the same 64-byte cache line, triggering massive false sharing.  
> **Detection:** Profiling with hardware performance counters (e.g. `perf c2c` on Linux) will reveal extremely high rates of Last-Level Cache (LLC) invalidations and HITM (Hit Modified) snoop cycles.  
> **Elimination:** Have each thread accumulate into a private local variable allocated on its own execution stack (`int local_sum = 0;`). Only after the loop finishes does each thread write its final value once into `sum[thread_id]`. This completely decouples the threads during loop execution.

---

## Quick Revision Sheet

- **Shared Memory:** Single address space accessible by all cores; communication via shared variables.
- **UMA (SMP):** All cores have equal latency and bandwidth to memory; limited to $\le 32$ cores by bus contention.
- **ccNUMA:** Memory physically distributed per socket, logically shared; local memory is $1.5\text{ to } 2.5\times$ faster than remote memory.
- **Microarchitecture Numbers from Notes:**
  - $1\text{ Chip} = 8\text{ Cores} = 16\text{ SMT Threads}$.
  - $2\text{ Chips} = 1\text{ Dual-Socket Node} = 16\text{ Cores} = 32\text{ SMT Threads}$.
  - Private L1/L2 caches per core; shared on-chip L3 cache; off-chip DRAM.
- **Cache Coherence:** Hardware guarantees all cached copies of variable `x` stay identical (Snooping/MESI vs Directory).
- **False Sharing:** Independent variables sharing the same 64-byte cache line cause continuous invalidations ("ping-ponging").
- **Fixing False Sharing:** Use thread-local variables or pad data structures to 64 bytes.

---

## What You Should Be Able to Do After This Topic

1. [ ] Define shared-memory computing and explain the difference between UMA and ccNUMA.
2. [ ] Draw the block diagrams of both UMA and ccNUMA architectures.
3. [ ] Explain why UMA systems cannot scale to hundreds of cores (bus saturation).
4. [ ] Draw the complete multi-core chip socket diagram from your notes (8 cores, SMT threads, L1/L2 private, shared L3, dual sockets).
5. [ ] Illustrate the Cache Coherence Problem with a step-by-step memory trace of stale reads.
6. [ ] Explain how snooping (bus-based) and directory-based protocols resolve cache coherence.
7. [ ] Define false sharing, draw the cache-line diagram explaining the "ping-pong" effect, and demonstrate how to eliminate it in code.

---

## Source References
- `Sanjay_Notes_1.pdf`: Pages 9–10 (Shared memory definition, UMA uniform access speed, ccNUMA varying latency/bandwidth, Advantages and disadvantages of UMA vs ccNUMA, Multi-core chip diagram with 8 cores, SMT $T_1/T_2$, private L1D/L2, shared L3, 2 chips = 1 node).
- `reference/teaching_style_sample.md`: Sections 4, 7–8 (Shared-memory systems, Cache coherence problem, Inconsistent cached copies, Why cache coherence matters).
