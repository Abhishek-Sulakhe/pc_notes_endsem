# Amdahl's Law, Strong Scaling & Communication Overhead Failure Models

In Subtopic 03.1, we introduced parallel speedup ($S = T_1 / T_p$) and efficiency ($E = S/p$). If you buy an expensive parallel computer with 1,000 processor cores, you might naturally expect your scientific simulations to execute 1,000 times faster.

Yet, when you run your code, you might find that it only runs **8 times faster**, and adding another 1,000 cores produces virtually no further improvement!

Why does adding more processors eventually stop helping? What places a fundamental, inescapable mathematical speed limit on parallel execution?

The answer was formulated in 1967 by computer pioneer Gene Amdahl. In this subtopic, you will derive **Amdahl's Law** from first principles, explore the physics of **Strong Scaling**, understand the asymptotic speedup limit, and examine a deadly failure mode highlighted in your lecture notes: **The Hazardous Linear Overhead Model**, where unscaled communication causes speedup to collapse completely to zero.

---

## 1. The Fundamental Motivation: The Bottleneck Principle

### Basic Intuition: The Highway Bottleneck
Imagine a 100-kilometer cross-country highway:
- **Section A (80 km):** A wide, 8-lane expressway. Cars can travel side by side at high speed.
- **Section B (20 km):** A narrow, 1-lane mountain bridge. Cars must pass strictly one at a time in single file.

```
                     THE HIGHWAY BOTTLENECK ANALOGY
                     
  ◄────────────────── 80 km ──────────────────►◄──────── 20 km ────────►
  ═════════════════════════════════════════════  ─────────────────────
  ═════════════════════════════════════════════  [1-LANE BRIDGE BOTTLENECK]
  ═════════════════════════════════════════════  ─────────────────────
  ═════════════════════════════════════════════
          PARALLELIZABLE HIGHWAY                  STRICTLY SEQUENTIAL
          Can widen to 100 lanes!                Cannot be widened!
```

Suppose you spend billions of dollars widening Section A from 8 lanes to **1,000 lanes**. Traffic on Section A moves almost instantaneously.

*What happens to the total travel time?*

Cars still get completely backed up at the 1-lane mountain bridge! Because Section B must remain single-file, no matter how infinitely wide you make Section A, the total trip time can **never** drop below the time it takes to cross Section B.

### The Parallel Computing Reality
Every computer program consists of two types of operations:
1. **Parallelizable Work:** Loops and calculations that operate on independent data (e.g., updating 10 million array elements). This work can be divided across $p$ processor cores.
2. **Inherently Sequential Work:** Operations that have strict serial dependencies and can only be executed by one processor at a time:
   - Initializing hardware and reading input files from disk.
   - Setting up data structures and allocating memory.
   - Master coordination, thread creation, and dispatching.
   - Sequential I/O, writing final output results, and printing to the screen.

Even if you have infinitely many processors, the sequential fraction must execute sequentially on one core. **The sequential portion is the mountain bridge of your software.**

---

## 2. Strong Scaling: Definition & Assumptions

In parallel performance analysis, there are two fundamentally different ways to scale a problem. The first is **Strong Scaling**:

$$\boxed{\text{Strong Scaling: The total problem size remains strictly FIXED while the number of processors increases.}}$$

```
                          STRONG SCALING MODEL
                          
   1 Processor:    [                     FIXED WORKLOAD W                     ]
                   ◄─────────────────────────── T1 ───────────────────────────►
                   
   2 Processors:   Core 1: [        W / 2        ]
                   Core 2: [        W / 2        ]
                   ◄───────────── T2 ────────────►
                   
   4 Processors:   Core 1: [ W/4 ]
                   Core 2: [ W/4 ]
                   Core 3: [ W/4 ]
                   Core 4: [ W/4 ]
                   ◄────── T4 ────►
```

### The Strong Scaling Question:
*"How much faster can we solve the exact same fixed-size scientific problem by buying more processor cores?"*

---

## 3. Step-by-Step Derivation of Amdahl's Law

Let us derive Amdahl's Law from first principles using the exact mathematical formulation from your course notes.

### Step 1: Normalize Execution Time
Let the total execution time of the program on a single processor ($p = 1$) be normalized to $1$ unit of time:
$$T_1 = 1$$

### Step 2: Partition the Workload
Let $s$ denote the **sequential fraction** of the program—the fraction of total runtime that must execute sequentially ($0 \le s \le 1$).

Since the program consists only of sequential and parallelizable work, the **parallelizable fraction** is:
$$\text{Parallel Fraction} = 1 - s$$

For example, if $20\%$ of the program is sequential, then $s = 0.20$ and $1 - s = 0.80$.

### Step 3: Compute Execution Time on $p$ Processors ($T_p$)
When executing on $p$ parallel processors:
- The sequential portion cannot be divided and still takes time $s \cdot T_1 = s$.
- The parallelizable portion is divided equally among the $p$ processors, taking ideally $\frac{1 - s}{p} \cdot T_1 = \frac{1 - s}{p}$.

The total parallel execution time $T_p$ is the sum of these two phases:
$$\boxed{T_p = s + \frac{1 - s}{p}}$$

### Step 4: Calculate Speedup $S(p)$
By definition, speedup is the ratio of serial time to parallel time:
$$S(p) = \frac{T_1}{T_p} = \frac{1}{s + \frac{1 - s}{p}}$$

This yields the world-famous equation:

$$\boxed{S(p) = \frac{1}{s + \frac{1 - s}{p}}}$$

*(In notation using $N$ workers):*
$$\boxed{S(N) = \frac{1}{s + \frac{1 - s}{N}}}$$

This is **Amdahl's Law**.

---

## 4. The Asymptotic Speedup Limit ($p \to \infty$)

What happens if you have an unlimited budget and purchase a supercomputer with infinitely many processors ($p \to \infty$)?

Let us evaluate the mathematical limit of Amdahl's Law:

$$\lim_{p \to \infty} S(p) = \lim_{p \to \infty} \left( \frac{1}{s + \frac{1 - s}{p}} \right)$$

As $p \to \infty$, the parallel term shrinks to zero:
$$\frac{1 - s}{p} \longrightarrow 0$$

Therefore:
$$\boxed{S_{\max} = \lim_{p \to \infty} S(p) = \frac{1}{s}}$$

```
                       AMDAHL'S SPEEDUP CEILING
                       
   Speedup S(p)
        ▲
        │                                         / Ideal Linear S(p) = p
        │                                       /
        │                                     /
   1/s  ┼ - - - - - - - - - - - - - - - - - ┌─────────────────────────────
        │                                 /   <--- ASYMPTOTIC CEILING S_max = 1/s
        │                               /
        │                             /
        │                           /
        │                         /
        │                       /  Curve flattens as sequential fraction
        │                     /   dominates total runtime!
        │                   /
        │                 /
        └───────────────┴──────────────────────────────────► Processors (p)
        1               10               100               1000
```

---

### The Harsh Reality of Amdahl's Ceiling
Look at how severely the sequential fraction caps the maximum achievable speedup, even with an infinite supercomputer:

| Sequential Fraction ($s$) | Parallel Fraction ($1 - s$) | Maximum Possible Speedup ($S_{\max} = 1/s$) |
| :---: | :---: | :---: |
| **50% ($s = 0.50$)** | 50% | **$2\times$** |
| **25% ($s = 0.25$)** | 75% | **$4\times$** |
| **10% ($s = 0.10$)** | 90% | **$10\times$** |
| **5% ($s = 0.05$)** | 95% | **$20\times$** |
| **1% ($s = 0.01$)** | 99% | **$100\times$** |
| **0.1% ($s = 0.001$)** | 99.9% | **$1,000\times$** |

> [!WARNING]
> **The Critical Examination Insight:**
> If just **$5\%$** of your program is sequential ($s = 0.05$), you can NEVER achieve more than a **$20\times$ speedup**, even if you run on 1,000,000 processor cores! The remaining $95\%$ of your program runs in near-zero time, but the $5\%$ sequential tail dominates the entire runtime.

---

## 5. Parallel Efficiency Under Amdahl's Law

What happens to parallel efficiency $\mathcal{E}(p) = S(p)/p$ as the processor count grows?

Substitute Amdahl's Law into the efficiency equation:

$$\mathcal{E}(p) = \frac{S(p)}{p} = \frac{\frac{1}{s + \frac{1 - s}{p}}}{p}$$

Multiply the denominator:
$$\mathcal{E}(p) = \frac{1}{p \cdot \left( s + \frac{1 - s}{p} \right)} = \frac{1}{p \cdot s + (1 - s)}$$

Factoring out $s$:
$$\boxed{\mathcal{E}(p) = \frac{1}{s(p - 1) + 1}}$$

### The Efficiency Decay Theorem:
Let us take the limit as $p \to \infty$:
$$\lim_{p \to \infty} \mathcal{E}(p) = \lim_{p \to \infty} \left( \frac{1}{s(p - 1) + 1} \right) = 0$$

$$\boxed{\lim_{p \to \infty} \mathcal{E}(p) = 0}$$

Under strong scaling with any non-zero sequential fraction ($s > 0$), **parallel efficiency asymptotically decays to zero**. As you add more processors, each additional processor spends a larger and larger fraction of its time sitting completely idle waiting for the sequential code to finish.

---

## 6. The Real World: Communication Overhead & Speedup Collapse

Amdahl's classical derivation assumes an **idealized parallel machine** where:
- Parallel work can be partitioned with zero scheduling overhead.
- Threads communicate with zero latency.
- Synchronization barriers take zero time.

In real systems, adding more processors introduces **communication and synchronization overhead**. Your lecture notes formalize this real-world behavior by adding an **overhead function $C(N)$**:

$$\boxed{T(N) = s + \frac{1 - s}{N} + C(N)}$$

Where $C(N)$ represents the time lost to lock contention, barrier synchronization, cache invalidations, and network message transfers.

---

### The Hazardous Linear Overhead Model: $C(N) = k \cdot N$

Your lecture notes analyze a dangerous, real-world case: **What happens if communication overhead grows linearly with the number of processors?**

In algorithms requiring all-to-all communication (such as naive N-body simulations, global reductions without tree topologies, or heavy lock contention), the communication cost per processor grows proportionally to the team size:
$$C(N) = k \cdot N \quad (k > 0)$$

Let us substitute this linear overhead into our parallel runtime equation:
$$T(N) = s + \frac{1 - s}{N} + k \cdot N$$

The resulting speedup is:

$$\boxed{S(N) = \frac{1}{s + \frac{1 - s}{N} + k \cdot N}}$$

```
            SPEEDUP COLLAPSE UNDER LINEAR OVERHEAD C(N) = k*N
            
   Speedup S(N)
        ▲
        │                           / Amdahl's Theoretical Curve (k = 0)
        │                         /
        │                       /
        │             Peak    ┌───────
        │           Speedup  /│       ──────
        │              *    / │             ──────
        │             / \  /  │                   ──────
        │            /   \/   │                         ──────
        │           /    /\   │                               ──────
        │          /    /  \  │                                     ───
        │         /    /    \ │  <--- SPEEDUP COLLAPSES TO ZERO!
        │        /    /      \│       (Communication overwhelms computation)
        │       /    /        \
        └──────┴────┴─────────┴────────────────────────────► Processors (N)
        1           N_opt
```

---

### Mathematical Proof of Speedup Collapse
Let us evaluate what happens as $N$ becomes large:

$$\lim_{N \to \infty} S(N) = \lim_{N \to \infty} \left( \frac{1}{s + \frac{1 - s}{N} + k \cdot N} \right)$$

As $N \to \infty$:
- $\frac{1 - s}{N} \to 0$
- $k \cdot N \to \infty$

The denominator explodes to infinity:
$$s + 0 + \infty = \infty$$

Therefore:
$$\boxed{\lim_{N \to \infty} S(N) = 0}$$

### Physical Meaning of the Collapse:
1. **The Optimal Operating Point ($N_{\text{opt}}$):** At low core counts, adding processors helps because $\frac{1-s}{N}$ drops faster than $kN$ grows. Speedup rises to a maximum peak.
2. **The Collapse Regime ($N > N_{\text{opt}}$):** Past the optimal point, the communication overhead $k \cdot N$ grows larger than the entire useful computation! Processors spend 99% of their time passing messages and waiting for locks.
3. **The Disaster:** Adding more processors actually makes the program **slower**, eventually driving speedup towards zero!

---

## Important Formulas

### 1. Amdahl's Law (Strong Scaling Speedup)
$$\boxed{S(p) = \frac{1}{s + \frac{1 - s}{p}}}$$
- $s$: Sequential fraction of program ($0 \le s \le 1$).
- $p$ (or $N$): Number of processors.

### 2. Maximum Asymptotic Speedup
$$\boxed{S_{\max} = \frac{1}{s}}$$

### 3. Parallel Efficiency Under Amdahl's Law
$$\boxed{\mathcal{E}(p) = \frac{1}{s(p - 1) + 1}}$$

### 4. Speedup with Linear Communication Overhead ($C(N) = kN$)
$$\boxed{S(N) = \frac{1}{s + \frac{1 - s}{N} + k \cdot N}}$$

---

## Exam-Ready Definitions

### Q1. State Amdahl's Law.
> **Answer:**  
> Amdahl's Law states that the maximum speedup achievable by parallelizing a program on $p$ processors is strictly limited by the time needed for the sequential fraction ($s$) of the program:
> $$S(p) = \frac{1}{s + \frac{1 - s}{p}}$$
> As the number of processors approaches infinity, the maximum possible speedup is asymptotically bounded by $S_{\max} = 1/s$.

### Q2. What is Strong Scaling?
> **Answer:**  
> Strong scaling is a parallel scaling model in which the total workload or problem size is held strictly constant while the number of processing elements is increased, with the objective of minimizing the total time required to solve that fixed problem.

### Q3. Explain why parallel efficiency approaches zero as processor count approaches infinity under Amdahl's Law.
> **Answer:**  
> Under Amdahl's Law, as $p \to \infty$, the speedup reaches a constant upper bound ($S_{\max} = 1/s$). Because parallel efficiency is defined as $\mathcal{E}(p) = S(p)/p$, dividing a finite constant ($1/s$) by an infinitely increasing processor count ($p$) drives efficiency asymptotically to zero:
> $$\lim_{p \to \infty} \frac{1/s}{p} = 0$$

### Q4. What is the Linear Overhead Model and what is its consequence on parallel speedup?
> **Answer:**  
> The Linear Overhead Model is a real-world performance model where communication, synchronization, or contention overhead grows linearly with processor count ($C(N) = kN$). Its mathematical consequence is that for large $N$, overhead dominates computation, causing execution time to increase and driving parallel speedup catastrophically to zero ($\lim_{N \to \infty} S(N) = 0$).

---

## Solved Numerical Questions

### Question 1: Amdahl's Law Speedup & Efficiency
A seismic data analysis program has a sequential fraction of $s = 0.12$ ($12\%$ sequential execution).
1. Calculate the theoretical maximum possible speedup ($S_{\max}$) even if infinitely many processors are available.
2. Calculate the speedup and parallel efficiency achieved when running on $p = 4$ processors.
3. Calculate the speedup and parallel efficiency achieved when running on $p = 16$ processors.
4. Calculate the speedup achieved when running on $p = 64$ processors.

#### Solution:
**Step 1: Calculate $S_{\max}$:**
$$S_{\max} = \frac{1}{s} = \frac{1}{0.12} = \frac{100}{12} \approx 8.33$$
$$\boxed{S_{\max} = 8.33\times}$$
No matter how many millions of cores are used, speedup can never exceed $8.33\times$!

**Step 2: Performance on $p = 4$ processors:**
Parallel fraction: $1 - s = 1 - 0.12 = 0.88$.
$$S(4) = \frac{1}{0.12 + \frac{0.88}{4}} = \frac{1}{0.12 + 0.22} = \frac{1}{0.34} \approx 2.941$$
$$E(4) = \frac{S(4)}{4} = \frac{2.941}{4} \approx 0.735 = 73.5\%$$
$$\boxed{S(4) \approx 2.94\times, \quad E(4) = 73.5\%}$$

**Step 3: Performance on $p = 16$ processors:**
$$S(16) = \frac{1}{0.12 + \frac{0.88}{16}} = \frac{1}{0.12 + 0.055} = \frac{1}{0.175} \approx 5.714$$
$$E(16) = \frac{S(16)}{16} = \frac{5.714}{16} \approx 0.357 = 35.7\%$$
$$\boxed{S(16) \approx 5.71\times, \quad E(16) = 35.7\%}$$

**Step 4: Performance on $p = 64$ processors:**
$$S(64) = \frac{1}{0.12 + \frac{0.88}{64}} = \frac{1}{0.12 + 0.01375} = \frac{1}{0.13375} \approx 7.48$$
$$E(64) = \frac{7.48}{64} \approx 0.117 = 11.7\%$$
$$\boxed{S(64) \approx 7.48\times, \quad E(64) = 11.7\%}$$

> *Observation:* Quadrupling processors from 16 to 64 increased speedup only slightly (from $5.71\times$ to $7.48\times$), while efficiency plummeted from $35.7\%$ to $11.7\%$.

---

### Question 2: Finding Required Sequential Fraction
A research laboratory purchases an 8-core CPU. The director mandates that any parallel software deployed must achieve at least a **$6.0\times$ speedup** on these 8 cores.
Calculate the **maximum permissible sequential fraction ($s$)** the program can have to meet this performance target.

#### Solution:
**Step 1: Set up the equation using Amdahl's Law:**
$$S(p) = \frac{1}{s + \frac{1 - s}{p}}$$
Given $p = 8$ and target $S(8) \ge 6.0$:
$$\frac{1}{s + \frac{1 - s}{8}} \ge 6.0$$

**Step 2: Invert both sides:**
$$s + \frac{1 - s}{8} \le \frac{1}{6.0}$$

**Step 3: Multiply through by 24 (the LCM of 8 and 6):**
$$24s + 3(1 - s) \le 4$$
$$24s + 3 - 3s \le 4$$
$$21s + 3 \le 4$$
$$21s \le 1$$
$$s \le \frac{1}{21} \approx 0.0476$$

$$\boxed{s \le 4.76\%}$$
To achieve a $6\times$ speedup on 8 cores, the sequential portion of the software must not exceed **$4.76\%$**!

---

### Question 3: Speedup with Linear Communication Overhead
A parallel algorithm has a sequential fraction of $s = 0.05$ ($5\%$) and a linear communication overhead factor of $k = 0.005$.
1. Calculate the speedup achieved on $N = 10$ processors.
2. Calculate the speedup achieved on $N = 50$ processors.
3. Calculate the speedup achieved on $N = 200$ processors.
4. Identify whether scaling past $N = 50$ improves or degrades performance.

#### Solution:
The speedup formula with linear overhead is:
$$S(N) = \frac{1}{s + \frac{1 - s}{N} + k \cdot N}$$
Here $s = 0.05$, $1 - s = 0.95$, and $k = 0.005$.

**Step 1: For $N = 10$:**
$$S(10) = \frac{1}{0.05 + \frac{0.95}{10} + (0.005 \times 10)} = \frac{1}{0.05 + 0.095 + 0.05} = \frac{1}{0.195} \approx 5.13\times$$

**Step 2: For $N = 50$:**
$$S(50) = \frac{1}{0.05 + \frac{0.95}{50} + (0.005 \times 50)} = \frac{1}{0.05 + 0.019 + 0.25} = \frac{1}{0.319} \approx 3.13\times$$

**Step 3: For $N = 200$:**
$$S(200) = \frac{1}{0.05 + \frac{0.95}{200} + (0.005 \times 200)} = \frac{1}{0.05 + 0.00475 + 1.0} = \frac{1}{1.05475} \approx 0.95\times$$

$$\boxed{S(10) \approx 5.13\times, \quad S(50) \approx 3.13\times, \quad S(200) \approx 0.95\times}$$

**Step 4: Analysis:**
- At $N = 10$, speedup is $5.13\times$.
- At $N = 50$, speedup drops to $3.13\times$ because linear communication overhead ($0.25$) is larger than computation!
- At $N = 200$, speedup drops below $1.0\times$ ($0.95\times$)—the parallel program is actually **slower than the sequential program**!
- This demonstrates the real-world catastrophe of unscaled communication overhead.

---

## Conceptual Examination Questions

### Q1. Why does Amdahl's Law imply that computer architects should prioritize single-core performance alongside multi-core scaling?
> **Model Answer:**  
> Amdahl's Law proves that the sequential fraction $s$ places an insurmountable ceiling on maximum speedup ($S_{\max} = 1/s$). Even if a chip contains 10,000 cores, the sequential code runs on only a single core. If single-core performance is sluggish, the entire multi-core system will sit idle waiting for that single core to complete the serial phase. Therefore, architects cannot simply add low-power, slow cores indefinitely; they must maintain aggressive single-core performance (via high IPC, out-of-order execution, and frequency boosting) to accelerate the sequential bottleneck.

### Q2. How can an engineering team reduce the sequential fraction $s$ of a legacy program?
> **Model Answer:**  
> An engineering team can reduce $s$ through four primary software strategies:
> 1. **Parallel I/O:** Replace single-thread file reads/writes with parallel I/O libraries (e.g. MPI-IO, HDF5) that allow multiple cores to read/write disk blocks concurrently.
> 2. **Algorithmic Redesign:** Replace inherently sequential algorithms (e.g., Gauss-Seidel iteration) with inherently parallel algorithms (e.g., Jacobi iteration or Red-Black ordering).
> 3. **Tree-Based Reductions:** Replace sequential result accumulation with logarithmic $O(\log p)$ parallel reduction trees.
> 4. **Asynchronous Overlapping:** Overlap remaining serial setup phases with background parallel work using non-blocking queues or tasks.

---

## Quick Revision Sheet

- **Amdahl's Law Formula:**
  $$\boxed{S(p) = \frac{1}{s + \frac{1 - s}{p}}}$$
- **Amdahl's Maximum Speedup Ceiling:**
  $$\boxed{S_{\max} = \frac{1}{s}}$$
- **Strong Scaling Definition:** Problem size is fixed; processor count increases.
- **Amdahl's Efficiency Decay:**
  $$\boxed{\mathcal{E}(p) = \frac{1}{s(p - 1) + 1} \longrightarrow 0 \quad \text{as } p \to \infty}$$
- **Linear Overhead Failure Model:**
  $$\boxed{S(N) = \frac{1}{s + \frac{1 - s}{N} + k \cdot N} \longrightarrow 0 \quad \text{as } N \to \infty}$$
  If overhead scales linearly ($C(N) = kN$), speedup eventually collapses to zero!

---

## What You Should Be Able to Do After This Topic

1. [ ] State the definition and assumptions of Strong Scaling.
2. [ ] Derive Amdahl's Law step by step from first principles.
3. [ ] Calculate the asymptotic maximum speedup $S_{\max} = 1/s$ for any given sequential fraction.
4. [ ] Derive the parallel efficiency formula under Amdahl's Law and prove that efficiency decays to zero as $p \to \infty$.
5. [ ] Explain the highway bottleneck analogy and relate it to software execution.
6. [ ] State the formula for speedup with linear communication overhead ($C(N) = kN$) and prove that speedup collapses to zero as $N \to \infty$.
7. [ ] Solve numerical exam problems calculating speedup, efficiency, and finding the maximum permissible sequential fraction.

---

## Source References
- `3d773f8d.pdf`: Pages 2–3 (Amdahl's Law formulation, Strong scaling assumptions, Asymptotic speedup $1/s$, Parallel efficiency $\mathcal{E}(N) = 1/(s(N-1)+1)$, Communication overhead term $C(N)$, Linear overhead model $C(N) = kN$ and speedup collapse to 0).
- `reference/teaching_style_sample.md`: Sections 15, 20–26 (Why parallelization does not give infinite speedup, Amdahl's Law derivation, Intuitive understanding, Maximum speedup table, Why adding more processors stops helping, Bottleneck concept).
