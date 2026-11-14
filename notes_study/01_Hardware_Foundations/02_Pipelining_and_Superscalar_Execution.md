# In-Core Instruction-Level Parallelism: Pipelining & Superscalar Execution

In the previous subtopic, we established that single-core clock frequencies cannot be increased indefinitely due to the Power Wall. However, long before multi-core processors became ubiquitous, processor architects devised ingenious techniques to execute multiple operations concurrently *inside a single CPU core*. 

This form of parallelism is called **Instruction-Level Parallelism (ILP)**. In this subtopic, you will master the two primary hardware engines of ILP: **Pipelining** of functional units and **Superscalar Out-of-Order Execution**.

---

## 1. The Principle of Pipelining

### Basic Intuition: The Industrial Assembly Line
Imagine an automobile factory. To assemble a car from scratch, workers must complete four major steps:
1. Install Chassis (Takes 1 hour)
2. Install Engine (Takes 1 hour)
3. Attach Wheels (Takes 1 hour)
4. Paint the Body (Takes 1 hour)

**Approach A: Non-Pipelined (Sequential Execution)**
- One team of workers builds Car 1 from step 1 to step 4 ($4\text{ hours}$).
- Only after Car 1 is painted does the team start Car 2.
- Total time for 10 cars: $10 \times 4 = 40\text{ hours}$.
- Completion rate: 1 car every 4 hours.

**Approach B: Pipelined Execution**
- Divide the factory into 4 specialized, sequential stations.
- Hour 1: Station 1 installs Chassis on Car 1.
- Hour 2: Car 1 moves to Station 2 (Engine). Simultaneously, Station 1 starts Car 2 (Chassis).
- Hour 3: Station 3 works on Car 1, Station 2 works on Car 2, Station 1 starts Car 3.
- Hour 4: Car 1 finishes completely!
- From Hour 5 onward: **One completed car rolls off the line every single hour!**
- Total time for 10 cars: $4\text{ hours (to fill line)} + 9\text{ hours} = 13\text{ hours}$ (compared to $40\text{ hours}$).

```
                     PIPELINING EXECUTION TIMELINE
                     
 Clock Cycle:    1      2      3      4      5      6      7      8
 Station 4:   [      ][      ][      ][ C1  ][ C2  ][ C3  ][ C4  ][ C5  ] ---> 1 Car/Cycle
 Station 3:   [      ][      ][ C1  ][ C2  ][ C3  ][ C4  ][ C5  ][      ]
 Station 2:   [      ][ C1  ][ C2  ][ C3  ][ C4  ][ C5  ][      ][      ]
 Station 1:   [ C1  ][ C2  ][ C3  ][ C4  ][ C5  ][      ][      ][      ]
              ▲                      ▲
              │                      │
         Start Car 1             Car 1 Exits
         (Wind-up Begins)        (Steady State Reached)
```

---

### How Pipelining Works in Microprocessors
Inside a CPU, instructions such as floating-point multiplications or memory loads take multiple clock cycles if executed as a single indivisible step. Pipelining applies the assembly line principle to hardware functional units:

1. **Decomposition:** A complex mathematical operation is split into $k$ simple, independent sub-steps (stages).
2. **Cycle Balancing:** Each stage is engineered to take exactly the same amount of time—typically **one clock cycle**.
3. **Pipelined Registers:** High-speed intermediate flip-flops/latches are placed between stages to hold partial results from one cycle to the next.
4. **Concurrent Execution:** Different stages work on different instructions concurrently on every clock pulse.

---

## 2. Walkthrough: The 5-Stage Floating-Point Multiplier Pipeline

Your course notes highlight a classic, high-yield examination example: a **5-stage floating-point multiplication pipeline** executing an array multiplication:

$$A(i) = B(i) \times C(i) \quad \text{for } i = 1, 2, \dots, N$$

Suppose we multiply two floating-point numbers represented in scientific notation:
$$B(i) = 3.0 \times 10^{12}, \quad C(i) = 5.0 \times 10^9$$

```
            5-STAGE FLOATING-POINT MULTIPLICATION PIPELINE
            
   Inputs: B(i), C(i)
        │
        ▼
   ┌─────────────────────────────────────────────────────────┐
   │ STAGE 1: SEPARATE MANTISSA & EXPONENT                   │
   │ Extract mantissas (3.0, 5.0) and exponents (12, 9)      │
   └────────────────────────────┬────────────────────────────┘
                                │
                                ▼
   ┌─────────────────────────────────────────────────────────┐
   │ STAGE 2: MULTIPLY MANTISSAS                             │
   │ Fixed-point integer multiplication: 3.0 x 5.0 = 15.0    │
   └────────────────────────────┬────────────────────────────┘
                                │
                                ▼
   ┌─────────────────────────────────────────────────────────┐
   │ STAGE 3: ADD EXPONENTS                                  │
   │ Integer addition of exponents: 12 + 9 = 21              │
   └────────────────────────────┬────────────────────────────┘
                                │
                                ▼
   ┌─────────────────────────────────────────────────────────┐
   │ STAGE 4: NORMALIZE RESULT                               │
   │ Shift mantissa to standard form: 15.0 x 10^21           │
   │ Normalized: 1.5 x 10^22 (adjust exponent to 22)         │
   └────────────────────────────┬────────────────────────────┘
                                │
                                ▼
   ┌─────────────────────────────────────────────────────────┐
   │ STAGE 5: INSERT SIGN & PACK RESULT                      │
   │ Determine sign bit (positive) and assemble IEEE format  │
   └────────────────────────────┬────────────────────────────┘
                                │
                                ▼
   Final Output: A(i) = 1.5 x 10^22
```

---

### Key Operational Metrics of a Pipeline

1. **Pipeline Latency ($L$):**
   - The time required for a single instruction to traverse all stages from input to final output.
   - For a $k$-stage pipeline operating at clock cycle $\tau$:
     $$\boxed{\text{Latency} = k \times \tau}$$
   - In our 5-stage multiplier, **the first result $A(1)$ appears only after 5 clock cycles**.

2. **Wind-Up (Fill) Phase:**
   - The initial transient period during which the pipeline is filling up with instructions.
   - During wind-up, the pipeline is not yet operating at full capacity because downstream stages are waiting for the initial instructions to propagate.
   - For a $k$-stage pipeline, the wind-up phase takes:
     $$\boxed{\text{Wind-up cycles} = k - 1}$$
   - In our 5-stage multiplier, the wind-up takes $5 - 1 = 4\text{ clock cycles}$.

3. **Steady-State Phase (Full Pipeline):**
   - Once all 5 stages are filled (from cycle 5 onward), **one completed result $A(i)$ emerges every single clock cycle**.
   - Throughput reaches its maximum theoretical ceiling:
     $$\boxed{\text{Throughput} = \frac{1}{\text{cycle time}} = f \text{ operations/second}}$$

4. **Wind-Down (Drain) Phase:**
   - The final transient period after the last instruction has entered Stage 1. As trailing instructions complete, stages become progressively idle until the pipeline is completely empty.

5. **Total Execution Time for $N$ Operations in a $k$-Stage Pipeline:**
   $$\boxed{T(N) = [k + (N - 1)] \times \tau}$$
   Where:
   - $k$: Latency of the first instruction (in cycles).
   - $N - 1$: Remaining instructions, each finishing 1 cycle after the previous one.
   - $\tau$: Clock cycle duration.

---

## 3. The Instruction Pipeline & Pipeline Hazards

Beyond arithmetic units, the CPU's core instruction execution itself is pipelined. At minimum, instruction execution is structured as a **3-stage pipeline**:

$$\boxed{\text{Fetch Instruction (from L1 I-cache)} \longrightarrow \text{Decode Instruction} \longrightarrow \text{Execute Instruction}}$$

In modern processors (such as Intel Skylake or AMD Zen), the instruction pipeline is deepened to **14 to 19 stages** to allow clock frequencies up to $4\text{ to } 5\text{ GHz}$.

```
                BASIC 3-STAGE INSTRUCTION PIPELINE
                
   ┌────────────────┐      ┌────────────────┐      ┌────────────────┐
   │     FETCH      │ ───► │     DECODE     │ ───► │    EXECUTE     │
   │  Fetch 32/64b  │      │  Decode opcode │      │  ALU operation │
   │  from L1 Cache │      │  read register │      │  or mem access │
   └────────────────┘      └────────────────┘      └────────────────┘
```

### Why Pipelining Does Not Always Achieve Ideal Throughput: Pipeline Hazards

In real programs, instructions cannot always flow seamlessly one after another. Interruptions called **hazards** cause **pipeline stalls** (or "pipeline bubbles"—idle stages where no useful work occurs).

```
Cycle:         1      2      3      4      5      6      7
Fetch:      [ I1 ] [ I2 ] [ I3 ] [ -- ] [ -- ] [ I4 ] [ I5 ]
Decode:            [ I1 ] [ I2 ] [ -- ] [ -- ] [ I3 ] [ I4 ]
Execute:                  [ I1 ] [ BUBBLE ]    [ I2 ] [ I3 ]
                                 ▲
                                 └── Stall / Bubble (Hardware Idles!)
```

There are three fundamental types of hazards:

1. **Structural Hazards (Hardware Resource Contention):**
   - Two instructions require the same physical hardware unit simultaneously (e.g., two instructions both attempting to access a single memory port in the same cycle).
2. **Data Hazards (Data Dependencies):**
   - An instruction depends on the result of an earlier instruction that has not yet finished executing.
   ```c
   a = b + c; // Instruction 1 (Takes 3 cycles in ALU)
   d = a * e; // Instruction 2: CANNOT execute until 'a' is written back!
   ```
3. **Control Hazards (Branch Hazards):**
   - Caused by conditional branch instructions (`if-else`, loops).
   - When the CPU fetches instruction $I_{k+1}$, it does not yet know whether the branch condition evaluated by instruction $I_k$ in the Execute stage will be taken or not taken!
   - **Mitigation:** Modern CPUs employ sophisticated **Branch Predictors** and **Speculative Execution**. The CPU guesses which path the branch will take and speculatively executes ahead. If the guess was wrong, the pipeline must be completely flushed, discarding all speculative results at a severe cost of $15\text{ to } 20\text{ wasted cycles}$.

---

## 4. Superscalar Execution & Out-of-Order (OoO) Processing

While pipelining allows multiple instructions to be in different stages of completion, a purely pipelined scalar processor still retires at most **1 instruction per cycle**. 

To break this barrier and achieve **multiple instructions retired per cycle**, modern processors use **Superscalar Architecture**.

### What is Superscalar Execution?
A superscalar processor contains **multiple parallel execution pipelines** fed by a shared instruction dispatch unit. 

If a processor has 4 decoders and multiple ALUs, it can issue $m$ instructions simultaneously in the exact same clock cycle. Such a processor is called an **$m$-way superscalar processor**:

$$\boxed{\text{Ideal Superscalar Throughput} = m \text{ instructions per cycle (IPC)}}$$

Modern server and desktop processors (Intel Core, AMD Ryzen, Apple M-series) are typically **4-way to 6-way superscalar**, capable of issuing up to 2 floating-point arithmetic instructions per cycle.

---

### Modern Superscalar Out-of-Order Microarchitecture

Your notes present the detailed microarchitectural schematic of a modern superscalar execution core. Below is the complete structural architecture:

```
            MODERN SUPERSCALAR OUT-OF-ORDER MICROARCHITECTURE
            
                        ┌────────────────────────┐
                        │  L1 Instruction Cache  │
                        └───────────┬────────────┘
                                    │ Instruction Stream
                                    ▼
                        ┌────────────────────────┐
                        │  4 Parallel Decoders   │
                        │ (Convert to Micro-ops) │
                        └───────────┬────────────┘
                                    │
                                    ▼
            ┌──────────────────────────────────────────────────┐
            │       REORDER BUFFER (ROB) & REGISTER RENAMING   │
            │   • Removes false dependencies (WAR, WAW)        │
            │   • Buffers instructions out-of-order            │
            │   • Retires instructions in strict program order │
            └───────────────────────┬──────────────────────────┘
                                    │
                                    ▼
            ┌──────────────────────────────────────────────────┐
            │             OUT-OF-ORDER SCHEDULER               │
            │ Dispatches ready micro-ops to available ports    │
            └───────┬─────┬──────┬──────┬──────┬───────┬───────┘
                    │     │      │      │      │       │
            ┌───────┘     │      │      │      │       └───────┐
            ▼             ▼      ▼      ▼      ▼               ▼
         Port 0        Port 1  Port 2 Port 3 Port 4         Port 5
       ┌────────┐    ┌───────┐ ┌────┐ ┌────┐ ┌─────┐      ┌────────┐
       │ALU, MUL│    │  ALU  │ │LOAD│ │LOAD│ │STORE│      │  ALU   │
       │  DIV   │    │  ADD  │ │    │ │ADRS│ │ADRS │      │  JMP   │
       └────┬───┘    └───┬───┘ └─┬──┘ └─┬──┘ └──┬──┘      └───┬────┘
            │            │       │      │       │             │
            └────────────┴───────┴──────┴───────┴─────────────┘
                                 │
                                 ▼
                     ┌───────────────────────┐
                     │ Physical Register File│
                     │   & L1 Data Cache     │
                     └───────────────────────┘
```

---

### Step-by-Step Microarchitectural Flow

1. **Instruction Fetch & Multi-Decode:**
   - The CPU fetches a 16-byte chunk of machine code from the L1 Instruction Cache.
   - Four parallel decoders parse the instructions simultaneously, converting complex x86 instructions into simpler internal RISC-like operations called **micro-operations ($\mu\text{ops}$)**.

2. **Register Renaming (Eliminating False Dependencies):**
   - Programs frequently reuse a limited number of architectural registers (e.g. `%rax`, `%rbx`), creating artificial dependencies:
     - **WAR (Write-After-Read / Anti-dependency):** Instruction 2 writes to a register that Instruction 1 is still reading.
     - **WAW (Write-After-Write / Output dependency):** Instruction 2 writes to the same register as Instruction 1.
   - The hardware eliminates these false dependencies by dynamically mapping the few architectural registers to a much larger pool of **physical internal registers** (e.g. 180+ physical registers).

3. **Reorder Buffer (ROB) & Out-of-Order Scheduler:**
   - Instructions are placed into the **Scheduler** (reservation stations).
   - If Instruction 2 is stalled waiting for memory, but Instruction 3 and Instruction 4 have all their operands ready, **the scheduler issues Instruction 3 and 4 immediately, bypassing Instruction 2!**
   - The instruction stream is thus *parallelized on the fly*.
   - Crucially, the **Reorder Buffer (ROB)** ensures that even though instructions *execute* out of order, they *commit and retire* their results to architectural state in strict **original program order**, preserving software correctness and precise exception handling.

4. **Execution Ports (Dispatch Units):**
   - As shown in your lecture notes, modern cores feature dedicated execution ports:
     - **Port 0:** Complex integer ALU, Floating-point Multiplication (`MUL`), Division (`DIV`).
     - **Port 1:** Standard ALU, Floating-point Addition (`ADD`).
     - **Port 2 & Port 3:** Memory Load pipelines (loading data from L1 D-Cache).
     - **Port 4:** Store Data pipeline.
     - **Port 5:** Standard ALU, Branch evaluation (`JMP`).
   - Because Port 0 handles FP Multiply while Port 1 handles FP Add, the core can dispatch **one multiply and one add simultaneously in the exact same cycle**!

---

## Important Formulas

### 1. Total Time for $N$ Operations in a $k$-Stage Pipeline
$$\boxed{T_{\text{pipe}}(N) = [k + (N - 1)] \times \tau}$$
Where:
- $k$: Number of pipeline stages.
- $N$: Number of independent operations.
- $\tau$: Clock cycle duration ($\tau = 1/f$).

### 2. Speedup of Pipelined vs Non-Pipelined Execution
For a non-pipelined system where each operation takes $k \times \tau$:
$$T_{\text{seq}}(N) = N \times k \times \tau$$
The speedup achieved by pipelining is:
$$\boxed{S_{\text{pipe}}(N) = \frac{T_{\text{seq}}(N)}{T_{\text{pipe}}(N)} = \frac{N \cdot k}{k + N - 1}}$$

**Asymptotic Limit ($N \to \infty$):**
$$\boxed{\lim_{N \to \infty} S_{\text{pipe}}(N) = k}$$
> *Key Insight:* The maximum theoretical speedup achievable by pipelining is bounded by the number of pipeline stages $k$.

---

## Exam-Ready Definitions

### Q1. Define pipelining in processor architecture.
> **Answer:**  
> Pipelining is an implementation technique whereby a complex computational process is divided into a sequence of distinct, balanced stages separated by storage registers, allowing multiple instructions to be processed simultaneously across different stages in an overlapping manner.

### Q2. What are pipeline latency and throughput?
> **Answer:**  
> - **Pipeline Latency:** The elapsed time required for a single instruction to traverse the entire pipeline from input stage to final completion ($L = k \cdot \tau$).  
> - **Pipeline Throughput:** The rate at which completed instructions emerge from the pipeline in steady state, ideally one instruction per clock cycle ($1/\tau$).

### Q3. What is the wind-up phase of a pipeline?
> **Answer:**  
> The wind-up (or fill) phase is the initial operational period of $(k - 1)$ clock cycles during which the pipeline stages are progressively loaded with instructions before the very first result emerges at the output.

### Q4. What is superscalar execution?
> **Answer:**  
> Superscalar execution is a microarchitectural capability enabling a single processor core to fetch, decode, and issue multiple independent instructions simultaneously within the same clock cycle using multiple parallel execution units.

### Q5. What is the Reorder Buffer (ROB) and why is it necessary?
> **Answer:**  
> The Reorder Buffer (ROB) is a hardware structure in out-of-order processors that tracks instructions issued and executed out of program order, ensuring they write their results to architectural registers and memory in strict original program order to maintain program correctness and precise interrupts.

---

## Important Comparisons

### Pipelined Execution vs Superscalar Execution

| Architectural Feature | Pipelined Processor | Superscalar Processor |
| :--- | :--- | :--- |
| **Execution Model** | Single instruction stream overlapped across stages. | Multiple instruction streams issued simultaneously. |
| **Instructions Issued / Cycle** | Maximum $1\text{ instruction/cycle}$. | $m\text{ instructions/cycle}$ ($m > 1$, typically 3–6). |
| **Functional Units** | One functional unit divided into stages. | Multiple independent, replicated functional units (multiple ALUs, FPUs). |
| **Theoretical Maximum IPC** | $\text{IPC} \le 1.0$. | $\text{IPC} \le m$ (e.g., $\text{IPC} = 4.0$). |
| **Hardware Complexity** | Modest (pipeline registers, basic hazard detection). | Extremely high (decoders, reorder buffers, register renaming, multi-port scheduling). |

---

## Solved Numerical Questions

### Question 1: 5-Stage Floating-Point Multiplier Pipeline
A 5-stage floating-point multiplier pipeline operates at a clock frequency of $f = 2.5\text{ GHz}$.
1. Calculate the clock cycle duration $\tau$.
2. Calculate the latency required to compute the very first multiplication result.
3. Determine the total execution time required to perform $N = 10,000$ independent array multiplications:
   $$A[i] = B[i] \times C[i]$$
4. Calculate the speedup obtained compared to a non-pipelined multiplier where each multiplication takes 5 clock cycles sequentially.

#### Solution:
**Step 1: Calculate clock cycle duration $\tau$:**
$$\tau = \frac{1}{f} = \frac{1}{2.5 \times 10^9\text{ Hz}} = 0.4 \times 10^{-9}\text{ s} = 0.4\text{ ns}$$

**Step 2: Calculate pipeline latency for 1st result:**
$$\text{Latency} = k \times \tau = 5 \times 0.4\text{ ns} = 2.0\text{ ns}$$
The first result emerges after $5\text{ clock cycles}$ ($2.0\text{ ns}$).

**Step 3: Calculate total time for $N = 10,000$ operations:**
$$T_{\text{pipe}} = [k + (N - 1)] \times \tau$$
$$T_{\text{pipe}} = [5 + (10,000 - 1)] \times 0.4\text{ ns}$$
$$T_{\text{pipe}} = [5 + 9,999] \times 0.4\text{ ns} = 10,004 \times 0.4\text{ ns} = 4,001.6\text{ ns} \approx 4.002\ \mu\text{s}$$

**Step 4: Calculate speedup compared to non-pipelined execution:**
Non-pipelined time:
$$T_{\text{seq}} = N \times k \times \tau = 10,000 \times 5 \times 0.4\text{ ns} = 20,000\text{ ns}$$
Speedup:
$$S = \frac{T_{\text{seq}}}{T_{\text{pipe}}} = \frac{20,000\text{ ns}}{4,001.6\text{ ns}} \approx 4.998$$

$$\boxed{S \approx 4.998 \approx 5.0\times}$$
> Notice that for large $N$, the speedup closely approaches the theoretical maximum of $k = 5$.

---

### Question 2: Superscalar IPC vs Clock Frequency
Processor Core A is a single-issue pipelined processor running at $f_A = 3.6\text{ GHz}$ with an average $\text{IPC} = 0.85$.  
Processor Core B is a 4-way superscalar out-of-order processor running at a lower frequency $f_B = 2.4\text{ GHz}$ with an average $\text{IPC} = 2.2$.  
Determine which processor delivers higher instruction throughput (in Million Instructions Per Second, MIPS).

#### Solution:
Throughput in MIPS is given by:
$$\text{Throughput} = \text{IPC} \times f$$

**For Processor Core A:**
$$\text{Throughput}_A = 0.85 \times 3.6 \times 10^9 = 3.06 \times 10^9\text{ instructions/s} = 3,060\text{ MIPS}$$

**For Processor Core B:**
$$\text{Throughput}_B = 2.2 \times 2.4 \times 10^9 = 5.28 \times 10^9\text{ instructions/s} = 5,280\text{ MIPS}$$

$$\boxed{\text{Processor Core B is } \frac{5280}{3060} \approx 1.73\times \text{ faster, despite running at a 33\% lower clock speed!}}$$

---

## Conceptual Examination Questions

### Q1. Why does an $m$-stage pipeline require a large number of independent instructions to achieve its theoretical maximum speedup?
> **Model Answer:**  
> A pipeline requires $(k - 1)$ initial cycles (the wind-up phase) to fill its stages before steady-state output begins. If a workload consists of only a small number of instructions ($N \approx k$), the wind-up and wind-down overhead dominates execution time, yielding a speedup far below $k$. Furthermore, if instructions have dependencies (data hazards), dependent instructions cannot enter the pipeline in consecutive cycles and must stall, inserting empty "bubbles". Only when a large stream of independent instructions ($N \gg k$) is supplied can the pipeline remain continuously saturated at 1 completion per cycle.

### Q2. How does register renaming resolve the Write-After-Write (WAW) and Write-After-Read (WAR) hazards in superscalar out-of-order processors?
> **Model Answer:**  
> WAR and WAW are "name dependencies" (or false dependencies) caused by a scarcity of architectural register names in compiler code, rather than true data flow. For example, if Instruction 2 overwrites register `%rax` that Instruction 1 recently used, sequential order mandates that Instruction 2 must not overwrite `%rax` before Instruction 1 finishes reading it (WAR).  
> In register renaming, the hardware dynamically maps the architectural name `%rax` for Instruction 2 to a completely different, unused physical register (e.g. `PR#47`). Instruction 2 can now execute immediately in parallel with Instruction 1, completely eliminating the stall because they write to physically distinct silicon storage cells.

---

## Quick Revision Sheet

- **Pipelining Working Principle:** Divide operation into $k$ balanced 1-cycle stages; overlap execution across stages.
- **Latency Formula:** $\text{Latency} = k \times \tau$.
- **Wind-Up Duration:** $(k - 1)$ cycles.
- **Total Pipeline Execution Time:** $T(N) = [k + (N - 1)] \cdot \tau$.
- **Maximum Pipelining Speedup:** $S_{\max} = k$.
- **Superscalar Architecture:** Issues $m$ concurrent instructions in the same clock cycle using multiple parallel execution ports.
- **Out-of-Order Execution Flow:**
  $$\text{L1 I-Cache} \longrightarrow \text{Decoders} \longrightarrow \text{Register Renaming} \longrightarrow \text{Scheduler} \longrightarrow \text{Execution Ports (0–5)} \longrightarrow \text{ROB Retirement}$$
- **Reorder Buffer (ROB) Purpose:** Allows out-of-order execution while guaranteeing in-order retirement of architectural state.

---

## What You Should Be Able to Do After This Topic

1. [ ] Explain the assembly line analogy of pipelining and contrast it with sequential execution.
2. [ ] Draw the step-by-step diagram of the 5-stage floating-point multiplication pipeline and explain all 5 stages.
3. [ ] Calculate the latency, wind-up cycles, steady-state throughput, and total time for $N$ operations in a $k$-stage pipeline.
4. [ ] Define structural, data, and control hazards and explain how pipeline bubbles arise.
5. [ ] Explain superscalar execution and draw the block diagram showing decoders, reorder buffer, scheduler, and execution ports (Ports 0 to 5).
6. [ ] Explain why register renaming is required to eliminate false WAR and WAW dependencies.
7. [ ] Calculate processor throughput in MIPS using $\text{Throughput} = \text{IPC} \times f$.

---

## Source References
- `Sanjay_Notes_1.pdf`: Pages 2–5 (Measures to improve performance, Modern core microarchitecture diagram with Ports 0–5, In-core features, Pipelining idea, benefits, drawbacks, 5-stage multiplication pipeline walkthrough, Instruction pipeline, Superscalar execution).
- `reference/teaching_style_sample.md`: Sections 2–3 (CPU execution models, Dependency chains, Why operations cannot be arbitrarily overlapped).
