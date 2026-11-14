# Fundamental Scaling Metrics: Speedup, Efficiency & Scalability Realities

In Modules 01 and 02, we established the physical and architectural foundations of parallel computing—from single-core instruction-level parallelism and memory hierarchies to multi-core ccNUMA sockets and distributed clusters.

Now, we turn to the central question every parallel software engineer, scientist, and university examiner asks:  
**"We added 8 processors to our program. Did it actually get 8 times faster? If not, why?"**

To answer this quantitatively, we must move beyond raw execution time and master the fundamental metrics of parallel computing: **Speedup ($S$)**, **Parallel Efficiency ($E$)**, and **Scalability**. In this subtopic, you will learn the exact mathematical definitions of these metrics, explore the rare phenomenon of superlinear speedup, examine the real-world reasons why speedup degrades, and understand the critical academic warning from your lecture notes: *why speedup and efficiency are NOT performance metrics*.

---

## 1. Execution Time in Parallel Computing

### The Notation
Before formulating metrics, let us define our operational variables:
- Let $N$ (or $p$) denote the **number of processing elements** (processor cores, execution threads, or compute nodes) allocated to execute a problem.
- Let $T(1)$ (or $T_1$) denote the **sequential execution time**—the total elapsed wall-clock time required to solve the problem using exactly **one worker** ($N = 1$).
- Let $T(N)$ (or $T_p$) denote the **parallel execution time**—the total elapsed wall-clock time required to solve the exact same problem using **$N$ workers**.

$$\text{Clock on Wall starts} \longrightarrow \text{Compute + Wait + Sync} \longrightarrow \text{Clock on Wall stops} \implies T(N)$$

---

## 2. Parallel Speedup ($S$)

### Basic Intuition: The Moving Company
Suppose you are moving to a new apartment. 
- If **1 person** moves all the boxes alone, the move takes **10 hours** ($T(1) = 10\text{ hours}$).
- If you hire **4 workers**, and they complete the move in **2.5 hours** ($T(4) = 2.5\text{ hours}$):
  $$\text{Speedup} = \frac{10\text{ hours}}{2.5\text{ hours}} = 4.0\times$$
  The move was completed 4 times faster!

---

### The Formal Definition
$$\boxed{S(N) = \frac{T(1)}{T(N)} \quad \text{or} \quad S(p) = \frac{T_1}{T_p}}$$

Where:
- $S(N)$ is a dimensionless ratio representing **how many times faster** the workload executes on $N$ workers compared to 1 worker.
- $T(1)$ is the execution time on 1 worker.
- $T(N)$ is the execution time on $N$ workers.

---

### The Three Categories of Speedup

Depending on the relationship between $S(N)$ and the processor count $N$, speedup falls into three distinct regimes:

```
                       THE THREE SPEEDUP REGIMES
                       
   Speedup S(N)
        ▲
        │                                         / Superlinear Speedup S(N) > N
        │                                       /   (Cache expansion effect)
        │                                     /
        │                                   /  <--- Ideal Linear Speedup S(N) = N
        │                                 /
        │                               /
        │                             / - - - Sublinear Speedup S(N) < N
        │                           /         (Real-world parallel systems)
        │                         /
        │                       /
        │                     /
        │                   /
        │                 /
        │               /
        └─────────────┴────────────────────────────────────► Number of Processors (N)
        1             2             4             8
```

#### 1. Ideal Linear Speedup ($S(N) = N$)
- If doubling the number of processors cuts execution time exactly in half, speedup scales linearly:
  $$S(2) = 2, \quad S(4) = 4, \quad S(8) = 8, \quad \dots \quad S(N) = N$$
- This represents the theoretical upper bound under classical scaling assumptions where all work is perfectly divisible and parallel overhead is zero.

#### 2. Sublinear Speedup ($S(N) < N$)
- In virtually all practical parallel programs:
  $$\boxed{S(N) < N}$$
- For example, running on 8 cores might yield a speedup of $5.6\times$ rather than $8.0\times$. 
- Why? As we will analyze in Section 4, real systems suffer from sequential code fractions, communication overhead, lock contention, and memory bus saturation.

#### 3. Superlinear Speedup ($S(N) > N$)
- Occasionally, an engineer benchmarks a code on 4 processors and observes a speedup of **$4.8\times$** ($S(4) > 4$).
- *How is it mathematically possible to get more than $N\times$ speedup from $N$ cores?*
- **The Hardware Explanation (Cache Expansion):**
  - When a problem with a large data set (e.g. $40\text{ MB}$) runs on **1 core**, the data exceeds the core's private cache ($8\text{ MB}$). The single core suffers massive, slow cache misses to DRAM throughout execution.
  - When the problem is distributed across **8 cores**, each core receives only $\frac{40\text{ MB}}{8} = 5\text{ MB}$ of data!
  - Suddenly, each core's sub-dataset **fits entirely inside its fast on-chip cache**!
  - DRAM accesses drop to zero. The parallel execution runs entirely at ultra-fast SRAM cache speed, resulting in an artificial speedup higher than the processor count!

---

## 3. Parallel Efficiency ($E$)

Speedup alone does not tell the complete engineering story.

Suppose Algorithm A running on 64 cores achieves a speedup of $S = 32\times$.  
Is that good?

To evaluate whether those 64 cores are being used economically, we compare the speedup achieved against the ideal speedup we paid for. This metric is **Parallel Efficiency ($E$)**.

### The Formal Definition
$$\boxed{E(N) = \frac{S(N)}{N} = \frac{T(1)}{N \cdot T(N)} \quad \text{or} \quad E(p) = \frac{S(p)}{p}}$$

Where:
- $E(N)$ is a dimensionless fraction, typically expressed as a **percentage ($0\%\text{ to } 100\%$)**.
- It measures the **fraction of time that processors spend doing useful computational work** compared to ideal linear parallelization.

```
                    PARALLEL EFFICIENCY INTERPRETATION
                    
    E(N) = 1.0  (100%)  ──► Ideal Linear Scaling (Zero wasted overhead)
    E(N) = 0.8  (80%)   ──► Excellent real-world scaling
    E(N) = 0.5  (50%)   ──► Marginal scaling (Half the compute power is wasted!)
    E(N) < 0.2  (20%)   ──► Poor scaling (Processors spend 80% of time idle/stalled)
```

### Numerical Demonstration:
If $T(1) = 100\text{ s}$ and $T(4) = 25\text{ s}$:
$$S(4) = \frac{100}{25} = 4.0$$
$$E(4) = \frac{4.0}{4} = 1.0 = 100\% \quad (\text{Ideal!})$$

If instead $T(4) = 31.25\text{ s}$:
$$S(4) = \frac{100}{31.25} = 3.2$$
$$E(4) = \frac{3.2}{4} = 0.80 = 80\%$$
- **Physical Interpretation:** On average, each of the 4 cores spent $80\%$ of its time computing useful application instructions and $20\%$ of its time waiting at synchronization barriers or stalling on memory.

---

## 4. The Critical Distinction: Speedup vs Performance

Your lecture notes feature an explicit, highlighted warning that is high-yield for university examinations:
> *"Warning: These metrics are not performance metrics."*

### Why Speedup and Efficiency Are NOT Performance Metrics
Students frequently make the fatal mistake of equating high speedup with high computational performance. They are fundamentally different concepts:

$$\boxed{\text{Performance } P = \frac{\text{Work}}{\text{Time}} \quad \text{vs} \quad \text{Speedup } S = \frac{T(1)}{T(N)}}$$

```
                THE SPEEDUP VS PERFORMANCE FALLACY
                
                   ALGORITHM 1                     ALGORITHM 2
               (Unoptimized Code)              (Highly Optimized Code)
               
  1 Core Time:    T(1) = 1000 seconds             T(1) = 100 seconds
  8 Core Time:    T(8) = 125 seconds              T(8) = 20 seconds
  ────────────────────────────────────────────────────────────────────────
  Speedup S(8):   1000 / 125 = 8.0x !             100 / 20 = 5.0x
  Efficiency:     8.0 / 8 = 100% !                5.0 / 8 = 62.5%
  ────────────────────────────────────────────────────────────────────────
  WINNER OF SPEEDUP:    ALGORITHM 1  (8.0x Speedup, 100% Efficiency)
  WINNER OF REAL WORLD: ALGORITHM 2  (Takes 20s vs 125s — 6x faster!)
```

### Analysis of the Fallacy:
1. **Algorithm 1** was poorly written sequential code (e.g. unoptimized Python or C compiled with `-O0`). Because its 1-core baseline was atrocious ($1000\text{ s}$), running it on 8 cores produced an "impressive" $8\times$ speedup!
2. **Algorithm 2** was heavily optimized with AVX-512 vectorization and cache blocking. Its sequential baseline was already blazingly fast ($100\text{ s}$). When parallelized, memory bandwidth bottlenecks limited its speedup to $5\times$.
3. **The Trap:** If you only look at speedup, Algorithm 1 looks superior ($8\times$ vs $5\times$). But in reality, Algorithm 2 finished the user's scientific job in **20 seconds**, while Algorithm 1 took **125 seconds**!
4. **Conclusion:** Speedup measures **relative scalability** against a baseline. It does **not** measure absolute computational productivity ($P = \text{Work}/\text{Time}$).

---

## 5. Scalability Realities & Hardware Boundaries

### What is Scalability?
$$\boxed{\text{Scalability is the ability of a parallel computing system to deliver a proportional increase in performance as additional processing resources are added.}}$$

A parallel program is described as **scalable** if its speedup continues to increase monotonically as processors are added, without prematurely plateauing.

---

### Hardware Boundary Drop-Offs: The Socket Cliff
In theoretical textbooks, speedup curves are drawn as smooth, monotonic logarithmic arcs. In real multi-socket multi-core servers, however, speedup curves exhibit sharp **inflection points and cliffs** at hardware boundaries.

Your lecture notes highlight this empirical reality:

```
            REAL-WORLD SPEEDUP CURVE ACROSS SOCKET BOUNDARIES
            
   Speedup S(N)
        ▲
        │                                         / Ideal Linear
        │                                       /
        │                                     /
        │                          Socket 1 /
        │                         Boundary /
        │                        ┌───────/
        │                       /│      /  <--- Inflection Point:
        │                      / │     /        Memory bandwidth saturates!
        │                     /  │    /
        │                    /   │   /
        │                   /    │  /
        │                  /     │ /
        │                 /      │/
        │                /       │
        │               /        │
        └──────────────┴─────────┴─────────────────────────► Core Count (N)
        1              8         16 (Dual Socket Boundary)
                       ▲         ▲
                       │         └── Inter-Socket Link Latency (UPI) + NUMA Penalties
                       └──────────── Intra-Socket L3 Contention
```

### Why Real Systems Suffer Boundary Drop-Offs:
1. **Intra-Socket Contention (Cores 1 to 8):**
   - Cores share the on-chip L3 cache and memory controller. As core count increases, memory channels saturate, causing the curve to bend below the ideal line.
2. **Inter-Socket Boundary Crossing (Cores 9 to 16):**
   - When the workload spills from Socket 0 onto Socket 1, execution becomes **ccNUMA**.
   - Threads on Socket 1 accessing data allocated on Socket 0 must traverse the high-latency inter-socket interconnect (UPI / Infinity Fabric).
   - Remote memory latency ($120\text{ ns}$) and cross-socket cache coherence invalidations hit the application, causing a sharp drop in scaling efficiency!

---

## Important Formulas

### 1. Parallel Speedup
$$\boxed{S(N) = \frac{T(1)}{T(N)} \quad \text{or} \quad S(p) = \frac{T_1}{T_p}}$$

### 2. Parallel Efficiency
$$\boxed{E(N) = \frac{S(N)}{N} = \frac{T(1)}{N \cdot T(N)} \quad \text{or} \quad E(p) = \frac{S(p)}{p}}$$

### 3. Execution Time with Linear Speedup
If speedup is ideal linear ($S(N) = N$):
$$\boxed{T(N) = \frac{T(1)}{N}}$$

---

## Exam-Ready Definitions

### Q1. Define Parallel Speedup.
> **Answer:**  
> Parallel speedup ($S(N) = T(1)/T(N)$) is a dimensionless metric that quantifies the relative performance improvement gained by executing a workload on $N$ parallel processors compared to executing the same workload on a single processor.

### Q2. Define Parallel Efficiency.
> **Answer:**  
> Parallel efficiency ($E(N) = S(N)/N$) is the ratio of observed speedup to the number of processors used, measuring the fraction of time processors spend performing useful computational work compared to ideal linear scaling.

### Q3. What is Superlinear Speedup and what causes it?
> **Answer:**  
> Superlinear speedup is an anomalous condition where a parallel program achieves a speedup greater than the number of processors used ($S(N) > N$). It is primarily caused by hardware cache expansion: distributing a large problem across multiple processors divides the working dataset so that it fits entirely within the fast aggregate on-chip CPU caches, eliminating slow DRAM accesses that penalized single-core execution.

### Q4. What is Scalability in parallel systems?
> **Answer:**  
> Scalability is the capacity of a parallel program and hardware architecture to effectively utilize increasing numbers of computational processing elements, delivering proportional gains in performance or handling proportionally larger problem sizes without severe efficiency degradation.

---

## Important Comparisons

### Speedup vs Parallel Efficiency

| Attribute | Parallel Speedup ($S$) | Parallel Efficiency ($E$) |
| :--- | :--- | :--- |
| **Formula** | $S(N) = \frac{T(1)}{T(N)}$ | $E(N) = \frac{S(N)}{N} = \frac{T(1)}{N \cdot T(N)}$ |
| **Units** | Dimensionless multiplier ($\times$). | Dimensionless fraction or percentage ($\%$). |
| **Ideal Value** | $S(N) = N$ (e.g. $4\times$ on 4 cores). | $E(N) = 1.0$ ($100\%$). |
| **What it answers** | "How many times faster did the program become?" | "How effectively are the processors being utilized?" |
| **Typical Trend** | Increases monotonically, then plateaus. | Decreases monotonically as $N$ increases. |

---

## Solved Numerical Questions

### Question 1: Speedup and Efficiency Calculations
A sequential weather modeling program takes $T(1) = 360\text{ seconds}$ to simulate a 24-hour forecast on 1 processor core. When executed on an 8-core workstation, the runtime drops to $T(8) = 54\text{ seconds}$. When executed on a 16-core dual-socket server, the runtime drops to $T(16) = 36\text{ seconds}$.

1. Calculate the speedup $S(8)$ and parallel efficiency $E(8)$ on 8 cores.
2. Calculate the speedup $S(16)$ and parallel efficiency $E(16)$ on 16 cores.
3. Analyze the efficiency trend as the core count doubles from 8 to 16.

#### Solution:
**Step 1: 8-Core Performance:**
$$S(8) = \frac{T(1)}{T(8)} = \frac{360\text{ s}}{54\text{ s}} \approx 6.67$$
$$E(8) = \frac{S(8)}{8} = \frac{6.667}{8} \approx 0.833 = 83.3\%$$

$$\boxed{S(8) = 6.67\times, \quad E(8) = 83.3\%}$$

**Step 2: 16-Core Performance:**
$$S(16) = \frac{T(1)}{T(16)} = \frac{360\text{ s}}{36\text{ s}} = 10.0$$
$$E(16) = \frac{S(16)}{16} = \frac{10.0}{16} = 0.625 = 62.5\%$$

$$\boxed{S(16) = 10.0\times, \quad E(16) = 62.5\%}$$

**Step 3: Engineering Analysis:**
When scaling from 8 cores to 16 cores (doubling hardware resources):
- Speedup increased from $6.67\times$ to $10.0\times$ (a $50\%$ improvement in speed).
- Efficiency dropped significantly from $83.3\%$ down to $62.5\%$.
- On 16 cores, almost $37.5\%$ of aggregate compute capability is lost to parallel overhead, memory bandwidth saturation, and inter-socket NUMA penalties.

---

### Question 2: Superlinear Speedup Identification
A molecular dynamics code executes with the following timings:
- 1 Core: $T(1) = 500\text{ seconds}$
- 2 Cores: $T(2) = 240\text{ seconds}$
- 4 Cores: $T(4) = 100\text{ seconds}$

1. Calculate the speedup and efficiency for 2 cores and 4 cores.
2. Identify which core count exhibits superlinear speedup and explain the physical reason.

#### Solution:
**For 2 Cores:**
$$S(2) = \frac{500}{240} \approx 2.083$$
$$E(2) = \frac{2.083}{2} \approx 1.042 = 104.2\%$$

**For 4 Cores:**
$$S(4) = \frac{500}{100} = 5.0$$
$$E(4) = \frac{5.0}{4} = 1.25 = 125.0\%$$

$$\boxed{S(4) = 5.0\times \quad (\text{Superlinear Speedup, since } S(4) > 4)}$$

**Physical Reason:**
Both 2 cores and 4 cores exhibit superlinear speedup ($E > 100\%$). The 4-core run demonstrates massive superlinear behavior ($5.0\times$ speedup on 4 cores). This occurs because the total memory footprint of the molecular dataset was too large for a single core's L3 cache, causing heavy DRAM page faults and cache misses. When split across 4 cores, each core's sub-problem fit completely into its on-die L2/L3 cache, shifting memory access times from $100\text{ ns}$ (DRAM) to $10\text{ ns}$ (Cache).

---

## Conceptual Examination Questions

### Q1. Why does parallel efficiency almost universally decline as the number of processors increases?
> **Model Answer:**  
> Parallel efficiency declines with increasing processor count due to five interacting physical and algorithmic overheads:
> 1. **Sequential Work:** According to Amdahl's Law, the sequential fraction of the program cannot be parallelized and places an upper ceiling on speedup.
> 2. **Communication & Synchronization:** As core count grows, threads spend more time coordinating, acquiring locks, and waiting at barriers.
> 3. **Load Imbalance:** When work cannot be split into perfectly equal slices, faster cores finish early and sit idle at synchronization points.
> 4. **Hardware Contention:** Multiple cores compete for shared memory channels, memory controllers, and interconnect bandwidth.
> 5. **Thread Management Overhead:** Creating, scheduling, and context-switching hundreds of threads consumes CPU cycles.

### Q2. Explain why a benchmark reporting $99\%$ parallel efficiency does not guarantee that the software is well-optimized.
> **Model Answer:**  
> Parallel efficiency measures only how well work is distributed relative to the single-core baseline ($E = T(1) / (N \cdot T(N))$). If the single-core baseline $T(1)$ is extremely unoptimized (e.g. written with inefficient scalar loops, missing compiler optimization flags, and suffering constant cache misses), it is very easy for $N$ processors to divide that bloated workload evenly and show near-perfect $99\%$ efficiency. However, if the single-core baseline were properly optimized with vectorization and cache blocking, $T(1)$ might drop by $10\times$, after which memory contention would lower the reported parallel efficiency. High efficiency on a slow baseline is a classic benchmarking fallacy.

---

## Quick Revision Sheet

- **Parallel Speedup:**
  $$\boxed{S(N) = \frac{T(1)}{T(N)}}$$
  Answers: *How many times faster did the program become?*
- **Parallel Efficiency:**
  $$\boxed{E(N) = \frac{S(N)}{N} = \frac{T(1)}{N \cdot T(N)}}$$
  Answers: *How effectively are the processors utilized compared to ideal linear scaling?*
- **Speedup Regimes:**
  - **Ideal Linear:** $S(N) = N, \quad E(N) = 100\%$.
  - **Sublinear (Normal):** $S(N) < N, \quad E(N) < 100\%$.
  - **Superlinear (Cache Effect):** $S(N) > N, \quad E(N) > 100\%$.
- **The Core Warning:** Speedup and efficiency are **relative scalability metrics**, NOT absolute performance metrics! Performance is $P = \text{Work}/\text{Time}$.
- **Hardware Drop-Offs:** Speedup curves flatten at memory bus boundaries and drop at socket (ccNUMA) boundaries.

---

## What You Should Be Able to Do After This Topic

1. [ ] Write the exact formulas for Parallel Speedup $S(N)$ and Parallel Efficiency $E(N)$.
2. [ ] Explain the difference between linear, sublinear, and superlinear speedup.
3. [ ] Explain the hardware mechanism (cache expansion) that creates superlinear speedup.
4. [ ] Articulate why speedup and efficiency are not performance metrics, using the slow-baseline vs fast-baseline example.
5. [ ] Draw the realistic speedup curve showing inflection points at multi-core and ccNUMA socket boundaries.
6. [ ] List the five practical reasons why parallel efficiency degrades as processor count increases.
7. [ ] Solve numerical exam problems calculating speedup, efficiency, and identifying superlinear behavior.

---

## Source References
- `Sanjay_Notes_1.pdf`: Page 12 (Metrics to quantify efficiency of parallel computing, $T(N)$ execution time, Parallel speedup formula $S(N) = T(1)/T(N)$, Parallel efficiency formula $E(N) = S(N)/N$, Warning: These metrics are not performance metrics).
- `3d773f8d.pdf`: Pages 1–2, 5 (Scalability assumptions, Speedup and efficiency formulations, Multi-socket drop-offs, Contention at hardware boundaries).
- `reference/teaching_style_sample.md`: Sections 15–19 (Speedup formula, Ideal linear speedup, Parallel efficiency, Speedup vs Efficiency comparison).
