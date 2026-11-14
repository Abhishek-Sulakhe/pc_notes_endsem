# Gustafson-Barsis' Law, Weak Scaling & Scaling Realities

In Subtopic 03.2, we derived Amdahl's Law and saw its grim prediction: if even $5\%$ of a program is sequential ($s = 0.05$), the maximum possible speedup on an infinite number of processors is strictly capped at $20\times$.

For more than twenty years, computer scientists believed that building massive supercomputers with tens of thousands of processors was pointless because Amdahl's ceiling would render most of the processors useless.

Then, in 1988, researchers John Gustafson and Edwin Barsis at Sandia National Laboratories ran three real-world scientific applications on a 1,024-processor nCUBE supercomputer—**and achieved speedups exceeding $1,000\times$!**

How could this happen? Did Gustafson violate the laws of mathematics?

No. Gustafson realized that **Amdahl made a flawed real-world assumption about how scientists use computers**. In this subtopic, you will master the physics of **Weak Scaling**, derive **Gustafson-Barsis' Law** step by step, understand why weak scaling enables massive supercomputers to achieve near-linear speedups, and examine the model-fitting realities highlighted in your lecture notes.

---

## 1. Strong Scaling vs Weak Scaling

To understand Gustafson's breakthrough, we must contrast the two fundamental ways to evaluate a parallel computer:

```
                      STRONG SCALING vs WEAK SCALING
                      
        STRONG SCALING (Amdahl)                    WEAK SCALING (Gustafson)
        Fixed Total Problem Size                  Scaled Total Problem Size
        
  1 CPU:  [      Problem 1x      ]          1 CPU:  [ Problem 1x ]
  
  2 CPUs: [ Prob 1/2 ][ Prob 1/2 ]          2 CPUs: [ Prob 1x ][ Problem 1x ]
  
  4 CPUs: [P1/4][P1/4][P1/4][P1/4]          4 CPUs: [Prob 1x][Prob 1x][Prob 1x][Prob 1x]
  
  Goal: Solve the SAME problem faster.      Goal: Solve a 4x LARGER problem in
                                                  the SAME amount of time!
```

---

### Strong Scaling (Amdahl's Assumption)
- **Premise:** The problem size is held strictly constant. We add more processors to solve that fixed problem in less time.
- **The Scientist's Perspective:** *"I have a 100-kilometer weather simulation grid. I want the results in 10 minutes instead of 80 minutes."*
- **The Inherent Limit:** As processors increase, the workload per processor shrinks towards zero, but the sequential setup overhead remains constant. The sequential fraction eventually dominates, hitting Amdahl's wall.

---

### Weak Scaling (Gustafson's Real-World Insight)
- **Premise:** The workload assigned to **each processor core remains constant**. As we add more processors, we proportionally increase the total problem size.
- **The Scientist's Perspective:** *"I don't want to solve yesterday's coarse 100-km weather model in 1 second. I have an 8-hour overnight compute window. With 1,000 processors, I want to simulate a hyper-detailed 1-km grid with cloud dynamics that was completely impossible on a single core!"*
- **The Weak Scaling Question:**
  $$\boxed{\text{"Can we solve a proportionally larger problem in approximately the same amount of time?"}}$$

---

## 2. Derivation of Gustafson-Barsis' Law

Let us derive Gustafson-Barsis' Law from first principles using the exact mathematical framework from your lecture notes.

### Step 1: Normalize Parallel Execution Time on $N$ Processors
In Gustafson's model, we observe the application running on the **actual parallel machine** with $N$ processors. 

Let the total parallel execution time on $N$ processors be normalized to $1$ unit of time:
$$\boxed{T(N) = 1}$$

### Step 2: Partition Time on the Parallel System
Let $s$ denote the fraction of time spent in serial code on the parallel system ($0 \le s \le 1$).

Since the program spends time only in serial and parallel execution:
- Time spent in serial execution on $N$ processors: $s$
- Time spent in parallel execution on $N$ processors: $1 - s$

### Step 3: Compute Hypothetical Sequential Time on 1 Processor ($T(1)$)
Now, ask the reverse question: **How long would it take a single processor to execute this exact same expanded workload?**
1. The serial portion would still take time $s$ (executed on 1 processor).
2. The parallel portion—which took time $(1 - s)$ when split concurrently across $N$ processors—would have to be executed **sequentially, one piece after another**, by the single core!
   $$\text{Sequential time for parallel work} = N \times (1 - s)$$

Therefore, the total time required by 1 processor to complete this massive workload is:
$$\boxed{T(1) = s + N(1 - s)}$$

### Step 4: Calculate Scaled Speedup ($S(N)$)
By definition, speedup is the ratio of single-processor time to $N$-processor time:

$$S(N) = \frac{T(1)}{T(N)} = \frac{s + N(1 - s)}{1}$$

Expanding the equation:
$$S(N) = s + N - N \cdot s = N - s(N - 1)$$

$$\boxed{S(N) = s + (1 - s)N \quad \text{or} \quad S(N) = N - s(N - 1)}$$

*(In notation using $p$ processors):*
$$\boxed{S(p) = s + (1 - s)p = p - s(p - 1)}$$

This is **Gustafson-Barsis' Law** (often called simply **Gustafson's Law**).

---

## 3. The Power of Gustafson's Law: Linear Scaling Unlocked

Look closely at the algebraic structure of Gustafson's Law compared to Amdahl's Law:

$$\text{Amdahl's Law (Strong Scaling): } S(p) = \frac{1}{s + \frac{1 - s}{p}} \quad \longrightarrow \quad S_{\max} = \frac{1}{s}$$
$$\text{Gustafson's Law (Weak Scaling): } S(p) = p - s(p - 1) \quad \longrightarrow \quad S(p) \propto p$$

```
                   AMDAHL'S LAW vs GUSTAFSON'S LAW
                   
   Speedup S(p)
        ▲
        │                                         / Gustafson's Weak Scaling
        │                                       /   (Near-Linear Speedup!)
        │                                     /     S(p) = p - s(p - 1)
        │                                   /
        │                                 /
        │                               /
   1/s  ┼ - - - - - - - - - - - - - - ┌────────────────── Amdahl's Strong Scaling
        │                            /                     S_max = 1/s (Asymptote)
        │                          /
        │                        /
        │                      /
        │                    /
        └──────────────────┴──────────────────────────────► Processors (p)
        1                  100                           1000
```

---

### Why Weak Scaling Overcomes Amdahl's Ceiling
In Amdahl's Law, the problem size is fixed, so as $p \to \infty$, the parallel runtime collapses towards zero while the serial runtime remains constant. The serial fraction grows as a percentage of total time until it completely dominates.

In Gustafson's Law, as $p \to \infty$, the problem size scales up with $p$. The parallel work expands proportionally with the hardware capability, keeping the parallel phase dominant.

#### Concrete Numerical Example:
Suppose a program has a sequential fraction of **$5\%$** ($s = 0.05$) when running on **$p = 1,024$ cores**:

1. **Under Amdahl's Law (Fixed Workload):**
   $$S(1,024) = \frac{1}{0.05 + \frac{0.95}{1,024}} = \frac{1}{0.05 + 0.000927} = \frac{1}{0.050927} \approx \mathbf{19.6\times}$$
   *(Amdahl predicts an abysmal $19.6\times$ speedup on 1,024 cores—a parallel efficiency of less than $2\%$!)*

2. **Under Gustafson's Law (Scaled Workload):**
   $$S(1,024) = 1,024 - 0.05(1,024 - 1)$$
   $$S(1,024) = 1,024 - 0.05(1,023) = 1,024 - 51.15 = \mathbf{972.85\times}$$
   *(Gustafson predicts a spectacular $\mathbf{972.85\times}$ speedup on 1,024 cores—a parallel efficiency of $\mathbf{95\%}$!)*

This mathematical result validated the construction of massively parallel supercomputers containing millions of cores.

---

## 4. Model Parameter Estimation: The Realities of Curve Fitting

Your lecture notes address an important practical question in parallel performance modeling:  
**How do engineers determine the parameters (such as $s$) for these scaling models?**

There are two primary approaches:
1. **Manual / Analytical Code Inspection:** Profiling the source code line-by-line using timers to measure the exact runtime of sequential sections vs parallel loops.
2. **Empirical Curve Fitting:** Running the program on $1, 2, 4, 8$ cores, recording the execution times, and using non-linear regression to fit Amdahl's equation to the measured data points.

```
                 CURVE FITTING REALITIES FROM LECTURE NOTES
                 
   Speedup
        ▲
        │                                         / Fitted Model Curve
        │                                       /
        │                                     /
        │                          *        /
        │                         / \     /
        │                        *   \  /
        │                       /     *   <--- ACTUAL BENCHMARK DATA
        │                      *       \        (Drops at socket boundary!)
        │                     /         *
        │                    *
        │                   /
        │                  *
        └─────────────────┴──────────────────────────────► Cores (N)
                          Critical Socket Boundary
```

### The Warning from Course Notes:
> *"Best fit is not a good fit at all."*

Why does standard mathematical curve fitting often fail disastrously in parallel computing?

1. **Parameter Shifts Across Thresholds:** A regression model assumes that the sequential fraction $s$ is a static mathematical constant across all core counts. In physical reality, once execution crosses a **critical number of cores** (e.g., crossing from 1 socket to 2 sockets), hardware parameters shift abruptly:
   - Remote NUMA memory latency hits.
   - Cache coherence invalidation traffic spikes non-linearly.
   - Memory bus bandwidth saturates.
2. **Blind Extrapolation Danger:** A curve fit that achieves an $R^2 = 0.99$ on 1 to 8 cores will catastrophically fail to predict performance on 16 or 32 cores because it cannot foresee the physical saturation of memory channels.

---

## Important Comparisons

### Amdahl's Law vs Gustafson's Law

| Dimension | Amdahl's Law | Gustafson-Barsis' Law |
| :--- | :--- | :--- |
| **Scaling Model** | **Strong Scaling** | **Weak Scaling** |
| **Problem Size** | Strictly Fixed. | Scales proportionally with $p$. |
| **Work per Processor** | Shrinks proportionally ($W/p$). | Stays approximately constant. |
| **Perspective** | How fast can we solve a fixed task? | How large a problem can we solve in fixed time? |
| **Speedup Formula** | $S(p) = \frac{1}{s + \frac{1 - s}{p}}$ | $S(p) = s + (1 - s)p = p - s(p - 1)$ |
| **Speedup Limit ($p \to \infty$)**| Strictly bounded by $S_{\max} = 1/s$. | Unbounded (scales linearly with $p$). |
| **Efficiency Trend** | Decays to $0\%$ as $p \to \infty$. | Remains high and stable near $(1 - s)$. |
| **Primary Domain** | Real-time systems, desktop computing. | Large-scale HPC, scientific supercomputing. |

---

## Exam-Ready Definitions

### Q1. State Gustafson-Barsis' Law.
> **Answer:**  
> Gustafson-Barsis' Law states that for a scaled workload where the parallel work increases proportionally with the number of processors $p$, keeping per-processor execution time constant, the achievable speedup is given by:
> $$S(p) = s + (1 - s)p = p - s(p - 1)$$
> where $s$ is the sequential fraction of execution time on the parallel system. Under this weak scaling model, speedup scales linearly with processor count, overcoming Amdahl's asymptotic limit.

### Q2. What is Weak Scaling?
> **Answer:**  
> Weak scaling is a parallel scaling model in which the total workload or problem size increases proportionally with the number of processing elements, keeping the workload assigned to each processor constant, with the objective of solving proportionally larger problems in approximately the same total wall-clock time.

### Q3. Explain why Amdahl's Law and Gustafson's Law do not contradict each other.
> **Answer:**  
> Amdahl's Law and Gustafson's Law do not contradict each other because they evaluate fundamentally different physical scaling scenarios. Amdahl's Law evaluates **Strong Scaling** (fixed problem size, shrinking work per core), where sequential overhead inevitably caps speedup. Gustafson's Law evaluates **Weak Scaling** (variable problem size, constant work per core), where the parallel workload expands to keep processors fully utilized. Both laws are mathematically correct within their respective operating assumptions.

---

## Solved Numerical Questions

### Question 1: Weak Scaling Speedup Calculation
An astrophysics simulation modeling star formation is benchmarked on an HPC cluster with $p = 512$ processor cores under a weak scaling regime. On the 512-core cluster, profiling reveals that $s = 0.02$ ($2\%$ of total parallel execution time is spent in sequential I/O and coordination).

1. Calculate the scaled speedup achieved by the simulation using Gustafson's Law.
2. Calculate the parallel efficiency of this weak scaling execution.
3. Compare this result with what Amdahl's Law would predict for a fixed-size problem with the same $s = 0.02$ on 512 cores.

#### Solution:
**Step 1: Calculate Scaled Speedup using Gustafson's Law:**
$$S(p) = p - s(p - 1)$$
Given $p = 512$ and $s = 0.02$:
$$S(512) = 512 - 0.02(512 - 1)$$
$$S(512) = 512 - 0.02(511) = 512 - 10.22 = 501.78$$

$$\boxed{S(512) \approx 501.78\times}$$

**Step 2: Calculate Parallel Efficiency:**
$$E(p) = \frac{S(p)}{p} = \frac{501.78}{512} \approx 0.980 = 98.0\%$$

$$\boxed{E(512) = 98.0\%}$$

**Step 3: Contrast with Amdahl's Law:**
Under Amdahl's strong scaling model:
$$S_{\text{Amdahl}}(512) = \frac{1}{0.02 + \frac{0.98}{512}} = \frac{1}{0.02 + 0.001914} = \frac{1}{0.021914} \approx 45.63\times$$
$$E_{\text{Amdahl}}(512) = \frac{45.63}{512} \approx 0.089 = 8.9\%$$

$$\boxed{\text{Gustafson Weak Speedup: } 501.78\times \ (98.0\% \text{ Eff}) \quad \text{vs} \quad \text{Amdahl Strong Speedup: } 45.63\times \ (8.9\% \text{ Eff})}$$
> *Key Takeaway:* Weak scaling maintains $98\%$ efficiency on 512 cores, whereas strong scaling drops efficiency to below $9\%$.

---

### Question 2: Gustafson Speedup on Different Core Counts
A parallel finite element simulation has a sequential fraction of $s = 0.04$ ($4\%$). Calculate the scaled speedup using Gustafson's Law on:
1. $p = 64$ cores
2. $p = 256$ cores
3. $p = 1,024$ cores

#### Solution:
Using $S(p) = p - s(p - 1)$:

**1. For $p = 64$:**
$$S(64) = 64 - 0.04(63) = 64 - 2.52 = 61.48\times$$
$$E(64) = \frac{61.48}{64} \approx 96.1\%$$

**2. For $p = 256$:**
$$S(256) = 256 - 0.04(255) = 256 - 10.20 = 245.80\times$$
$$E(256) = \frac{245.80}{256} \approx 96.0\%$$

**3. For $p = 1,024$:**
$$S(1,024) = 1,024 - 0.04(1,023) = 1,024 - 40.92 = 983.08\times$$
$$E(1,024) = \frac{983.08}{1,024} \approx 96.0\%$$

$$\boxed{S(64) = 61.48\times, \quad S(256) = 245.80\times, \quad S(1,024) = 983.08\times}$$
> Notice that across all core counts, parallel efficiency remains virtually constant at $96\%$!

---

## Conceptual Examination Questions

### Q1. Why is weak scaling considered a more realistic model for evaluating high-performance supercomputing systems than strong scaling?
> **Model Answer:**  
> When research institutions invest millions of dollars into supercomputers with hundreds of thousands of cores, their goal is rarely to run the exact same small, simple problem in milliseconds. Real-world computational science is limited by resolution and accuracy:
> 1. In climate modeling, running a coarse 50-km resolution model on 10,000 cores provides no scientific value. Instead, scientists use the 10,000 cores to increase resolution to 1 km, simulating clouds, hurricanes, and ocean eddies that could never fit into the memory of a smaller system.
> 2. Because available memory capacity grows linearly with node count, the problem size naturally expands to fill the available memory.
> 3. Therefore, weak scaling accurately reflects real-world scientific practice: scaling the computational workload alongside machine size to solve larger, higher-fidelity problems within a constant, acceptable wall-clock turnaround time.

### Q2. Explain why the empirical observation "best fit is not a good fit at all" applies to scaling models in multi-socket server nodes.
> **Model Answer:**  
> Standard mathematical regression minimizes error across a set of measured data points by fitting a continuous, smooth function (like Amdahl's hyperbolic curve). However, computer hardware is not smooth; it is partitioned by physical architectural boundaries.  
> When scaling within a single socket (e.g. Cores 1 to 8), data transfers remain on high-speed on-die caches, producing a smooth scaling curve. But the moment the core count crosses to Core 9 (Socket 2), execution transitions abruptly from UMA to ccNUMA: memory latency doubles, and inter-socket link contention begins.  
> A mathematical "best fit" curve calculated on Cores 1–8 will completely miss this physical socket boundary and provide dangerously inaccurate performance predictions for larger core counts.

---

## Quick Revision Sheet

- **Weak Scaling Definition:** Workload scales proportionally with processor count; work per processor remains constant.
- **Gustafson-Barsis' Law Formula:**
  $$\boxed{S(p) = s + (1 - s)p = p - s(p - 1)}$$
- **Speedup Trajectory:** Scales **linearly** with $p$; does NOT have an asymptotic speedup ceiling!
- **Parallel Efficiency under Gustafson's Law:**
  $$E(p) \approx 1 - s \quad (\text{remains high and stable as } p \to \infty)$$
- **Amdahl vs Gustafson Summary:**
  - Amdahl: Fixed problem, strong scaling, speedup bounded by $1/s$.
  - Gustafson: Scaled problem, weak scaling, speedup unbounded ($p - s(p - 1)$).
- **Curve-Fitting Pitfall:** Mathematical best-fit models fail across multi-core and multi-socket NUMA boundaries.

---

## What You Should Be Able to Do After This Topic

1. [ ] Define weak scaling and distinguish it from strong scaling with clear diagrams.
2. [ ] Derive Gustafson-Barsis' Law step by step from first principles.
3. [ ] Explain why Gustafson's Law allows near-linear speedup on massively parallel supercomputers.
4. [ ] Compare Amdahl's Law and Gustafson's Law in a comprehensive examination table.
5. [ ] Explain why Amdahl's Law and Gustafson's Law do not contradict each other.
6. [ ] Explain the observation "best fit is not a good fit at all" regarding multi-socket scaling.
7. [ ] Solve numerical exam problems calculating scaled speedup and efficiency for weak scaling.

---

## Source References
- `3d773f8d.pdf`: Pages 4–5 (Weak scaling concept, Gustafson's Law formulation $S(N) = s + (1-s)N$, Scalability realities, Model parameter estimation: manual analysis vs curve fitting, "Best fit is not good fit at all", Critical core count and socket boundaries).
- `reference/teaching_style_sample.md`: Section 28 (Strong scaling vs Weak scaling, Definitions, Question answered by each, Strong scaling example).
