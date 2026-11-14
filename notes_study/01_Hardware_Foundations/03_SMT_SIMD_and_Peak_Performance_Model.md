# Thread & Data Parallelism: SMT, SIMD Vectorization & The Peak Performance Model

In the previous subtopic, we explored how single cores extract parallelism using pipelining and superscalar execution. However, even with an out-of-order scheduler and six execution ports, CPU execution units frequently sit idle because real-world programs have dependencies, branch mispredictions, and long memory delays.

To push silicon utilization even higher, computer architects introduced two complementary in-core mechanisms:
1. **Thread-Level Parallelism within a single core:** **Simultaneous Multi-Threading (SMT)**.
2. **Data-Level Parallelism within a single instruction:** **Single Instruction, Multiple Data (SIMD) Vectorization**.

This subtopic explores how these mechanisms work, examines x86 vector extensions (SSE, AVX, AVX-512), and introduces the single most important mathematical formula in hardware performance modeling: **The Master Core Peak Performance Formula**.

---

## 1. Simultaneous Multi-Threading (SMT / Hyper-Threading)

### Basic Intuition: The Idle Waiting Room
Imagine a skilled doctor in an examination room. The doctor can diagnose a patient in 2 minutes, but each patient frequently needs to wait 10 minutes for blood test results. 

- **Single-Threaded Model:** The doctor brings in Patient 1, orders a test, and sits idle for 10 minutes doing nothing while waiting for the lab.
- **Multi-Threaded (SMT) Model:** The doctor has *two examination chairs* in the same room. While Patient 1 is waiting for test results, the doctor immediately turns to Patient 2 in the second chair and performs a diagnosis. The doctor's time is utilized continuously.

In this analogy:
- The **doctor** is the processor's shared execution engine (ALUs, FPUs).
- The **chairs** are the separate architectural hardware register states.
- The **lab test wait** is a high-latency cache miss to main memory.

---

### The Problem SMT Solves: Pipeline Bubbles
As observed in your course notes:
> *"Pipelines are often underutilized due to dependencies, waiting time, etc. 'Pipeline bubbles' mean wasted resources."*

In a superscalar processor capable of issuing 4 instructions per cycle, an ideal cycle looks like this:
```
Ideal Cycle:    [ Instruction 1 ][ Instruction 2 ][ Instruction 3 ][ Instruction 4 ] (100% Full)
```
However, in typical single-threaded code, data dependencies and memory latency prevent the scheduler from finding 4 independent instructions every cycle:
```
Real Cycle:     [ Instruction 1 ][    BUBBLE    ][    BUBBLE    ][    BUBBLE    ] (25% Full!)
```
These empty execution slots are called **pipeline bubbles**. The transistors are physically present and drawing power, but performing zero useful work.

```
                      PIPELINE SLOTS UNDER SMT
                      
  Single-Threaded Execution               Simultaneous Multi-Threading (SMT)
  (Heavy Wasted Bubbles)                  (Two Threads Fill Empty Slots)
  
  Cycle 1: [ T1 ][ T1 ][ -- ][ -- ]       Cycle 1: [ T1 ][ T1 ][ T2 ][ T2 ]
  Cycle 2: [ T1 ][ -- ][ -- ][ -- ]       Cycle 2: [ T1 ][ T2 ][ T2 ][ T2 ]
  Cycle 3: [ -- ][ -- ][ -- ][ -- ]       Cycle 3: [ T2 ][ T2 ][ T2 ][ -- ]
  Cycle 4: [ T1 ][ T1 ][ T1 ][ -- ]       Cycle 4: [ T1 ][ T1 ][ T1 ][ T2 ]
           ▲                                       ▲
           │                                       │
     White spaces are                        Threads interleave dynamically;
     wasted bubbles!                         Bubbles are drastically reduced!
```

---

### How SMT Actually Works in Hardware

SMT makes a **single physical core** appear to the operating system as **two (or more) logical cores**.

#### What is Duplicated (Hardware Cost):
To maintain two independent threads simultaneously, the processor duplicates only the lightweight **architectural state**:
- Register File copies (separate architectural registers for Thread 1 and Thread 2)
- Program Counters (PC)
- Instruction Pointer registers
- Return address stacks

This requires only a tiny fraction (typically $\approx 5\%$) of additional silicon area on the chip.

#### What is Shared:
Both threads share the heavy computational and storage machinery:
- The Execution Units (Integer ALUs, Floating-Point Multipliers, Dividers)
- The Out-of-Order Schedulers and Reorder Buffers
- The L1 Instruction and Data Caches
- The L2 and L3 Caches
- The Memory Controller and Bus Interfaces

> [!IMPORTANT]
> **Key Academic Observation from Notes:**
> *"SMT can improve the utilization of pipelines. It does NOT improve any other resource on the chip."*
> SMT does not give you more ALUs or more memory bandwidth! If both threads compete for the exact same execution port or flood the same cache, they may slow each other down. SMT improves throughput, not single-thread latency.

---

## 2. Single Instruction, Multiple Data (SIMD) Vectorization

### Basic Intuition: The Scalar vs Vector Painter
Suppose you need to paint 4 fence pickets with white paint.

- **Scalar Processing:** You take a small 1-inch paintbrush. You dip it in paint, paint picket 0, dip, paint picket 1, dip, paint picket 2, dip, paint picket 3. That takes **4 separate dipping and painting actions**.
- **SIMD (Vector) Processing:** You take a wide 4-inch roller brush. In **one single motion**, you paint all 4 pickets simultaneously!

In microprocessors:
- **Scalar Operation:** An instruction performs arithmetic on one pair of numbers at a time.
- **SIMD Operation:** An instruction performs the same mathematical operation simultaneously on an entire array of numbers packed inside a single wide "vector register".

```
                         SCALAR vs SIMD EXECUTION
                         
         SCALAR ADDITION (4 Instructions)         SIMD VECTOR ADDITION (1 Instruction)
         
          A[0]   +   B[0]   =   C[0]             ┌───────┬───────┬───────┬───────┐
          A[1]   +   B[1]   =   C[1]          A: │ A[3]  │ A[2]  │ A[1]  │ A[0]  │
          A[2]   +   B[2]   =   C[2]             └───────┴───────┴───────┴───────┘
          A[3]   +   B[3]   =   C[3]                         +   (SIMD VADD)
                                                 ┌───────┬───────┬───────┬───────┐
                                              B: │ B[3]  │ B[2]  │ B[1]  │ B[0]  │
                                                 └───────┴───────┴───────┴───────┘
                                                             =
                                                 ┌───────┬───────┬───────┬───────┐
                                              C: │ C[3]  │ C[2]  │ C[1]  │ C[0]  │
                                                 └───────┴───────┴───────┴───────┘
                                                 All 4 additions happen in ONE cycle!
```

---

### x86 SIMD Instruction Sets & Register Widths

Over the past three decades, Intel and AMD expanded the x86 architecture by introducing progressively wider vector registers:

```
                      EVOLUTION OF x86 VECTOR REGISTERS
                      
   128-bit (SSE)       [  Double 1  ][  Double 0  ]                      = 2 DP FLOPs
   
   256-bit (AVX/AVX2)  [  DP 3  ][  DP 2  ][  DP 1  ][  DP 0  ]          = 4 DP FLOPs
   
   512-bit (AVX-512)   [ DP 7 ][ DP 6 ][ DP 5 ][ DP 4 ][ DP 3 ][ DP 2 ][ DP 1 ][ DP 0 ] = 8 DP FLOPs
```

Let us quantify the capacity of these registers for **Double-Precision (DP, 64-bit)** floating-point operands:

1. **SSE (Streaming SIMD Extensions):**
   - Register width: **128 bits**.
   - Capacity: $\frac{128\text{ bits}}{64\text{ bits}} = \mathbf{2\text{ double-precision operands}}$ (or 4 single-precision 32-bit floats).
2. **AVX / AVX2 (Advanced Vector Extensions):**
   - Register width: **256 bits**.
   - Capacity: $\frac{256\text{ bits}}{64\text{ bits}} = \mathbf{4\text{ double-precision operands}}$ (or 8 single-precision 32-bit floats).
3. **AVX-512:**
   - Register width: **512 bits**.
   - Capacity: $\frac{512\text{ bits}}{64\text{ bits}} = \mathbf{8\text{ double-precision operands}}$ (or 16 single-precision 32-bit floats).

---

### How the Compiler Vectorizes Code: Loop Unrolling

Your lecture notes illustrate the exact mechanism compilers use to exploit SIMD registers: **Loop Unrolling by the Vector Factor ($n_{\text{SIMD}}$)**.

Consider an elemental vector addition loop:
```c
// Original Scalar Loop: Executes 1 addition per iteration
for (int i = 0; i < n; i++) {
    C[i] = A[i] + B[i];
}
```

To vectorize this loop for an AVX target (where register width allows 4 double-precision operands simultaneously), the optimizing compiler unrolls the loop by a factor of 4:

```c
// Unrolled Loop (SIMD Processing with n_SIMD = 4):
for (int i = 0; i < n; i += 4) {
    C[i]     = A[i]     + B[i];
    C[i + 1] = A[i + 1] + B[i + 1];
    C[i + 2] = A[i + 2] + B[i + 2];
    C[i + 3] = A[i + 3] + B[i + 3];
}
```

The compiler replaces these four scalar additions with a single AVX machine instruction:
```assembly
vmovupd (%rsi,%rax), %ymm0      ; Load 4 doubles from A into 256-bit ymm0
vmovupd (%rdx,%rax), %ymm1      ; Load 4 doubles from B into 256-bit ymm1
vaddpd  %ymm1, %ymm0, %ymm2     ; Add all 4 doubles in a single clock cycle!
vmovupd %ymm2, (%rdi,%rax)      ; Store 4 results into C
```

---

## 3. The Master Peak Core Performance Formula

One of the most essential formulas in high-performance computing enables us to calculate the **theoretical peak floating-point performance ($P_{\text{core}}$)** of a single processor core:

$$\boxed{P_{\text{core}} = n_{\text{superscalar}}^{\text{FP}} \times n_{\text{FMA}} \times n_{\text{SIMD}} \times f}$$

Where:
- $\mathbf{P_{\text{core}}}$: Theoretical peak performance of the core (measured in $\text{FLOP/s}$ or $\text{GFLOP/s}$).
- $\mathbf{n_{\text{superscalar}}^{\text{FP}}}$: The floating-point superscalarity factor—the number of floating-point execution ports/units capable of issuing a vector instruction in the same cycle (typically $1\text{ or } 2$).
- $\mathbf{n_{\text{FMA}}}$: The **Fused Multiply-Add (FMA)** factor. Modern execution units can compute $D = A \times B + C$ in a single hardware cycle. Because an FMA performs **one multiplication AND one addition**, it counts as **2 FLOPs per cycle**:
  $$n_{\text{FMA}} = 2$$
  *(If the processor does not support FMA, $n_{\text{FMA}} = 1$)*.
- $\mathbf{n_{\text{SIMD}}}$: The vectorization factor—the number of data elements processed simultaneously in one vector register:
  - For SSE (Double Precision): $n_{\text{SIMD}} = 2$
  - For AVX / AVX2 (Double Precision): $n_{\text{SIMD}} = 4$
  - For AVX-512 (Double Precision): $n_{\text{SIMD}} = 8$
- $\mathbf{f}$: The processor clock frequency (in $\text{Hz}$ or $\text{GHz}$).

---

### Physical Meaning of the Formula

```
           THE FOUR MULTIPLICATIVE TIERS OF CORE PERFORMANCE
           
   Base Clock Frequency (f)              -----> Ticks per second
             │
             ▼
   x Superscalar FP Units (n_super)      -----> Vector instructions issued per tick
             │
             ▼
   x SIMD Vector Width (n_SIMD)          -----> Numbers processed per vector instruction
             │
             ▼
   x Fused Multiply-Add (n_FMA = 2)      -----> Operations executed per number (Mul + Add)
             │
             ▼
   ========================================================================
   P_core: Total Floating-Point Operations Completed Per Second (FLOP/s)!
   ========================================================================
```

Notice how these four factors multiply together. If you double the vector width from 256 bits to 512 bits, performance doubles. If you add FMA capability, performance doubles again. This compounding multiplication is how modern silicon cores achieve astronomical compute rates.

---

## Important Formulas

### 1. Single-Core Peak Performance Formula
$$\boxed{P_{\text{core}} = n_{\text{superscalar}}^{\text{FP}} \times n_{\text{FMA}} \times n_{\text{SIMD}} \times f}$$

### 2. Multi-Core Processor Chip Peak Performance
For a processor chip containing $N_{\text{cores}}$ identical physical cores:
$$\boxed{P_{\text{chip}} = N_{\text{cores}} \times P_{\text{core}} = N_{\text{cores}} \times n_{\text{superscalar}}^{\text{FP}} \times n_{\text{FMA}} \times n_{\text{SIMD}} \times f}$$

### 3. SIMD Vector Width Factor
$$\boxed{n_{\text{SIMD}} = \frac{\text{Register Width in Bits}}{\text{Operand Size in Bits}}}$$
- For Double Precision (64-bit float):
  $$n_{\text{SIMD}}^{\text{DP}} = \frac{W_{\text{reg}}}{64}$$
- For Single Precision (32-bit float):
  $$n_{\text{SIMD}}^{\text{SP}} = \frac{W_{\text{reg}}}{32} = 2 \times n_{\text{SIMD}}^{\text{DP}}$$

---

## Exam-Ready Definitions

### Q1. What is Simultaneous Multi-Threading (SMT)?
> **Answer:**  
> Simultaneous Multi-Threading (SMT) is an in-core hardware multithreading technique in which multiple independent execution threads issue instructions simultaneously to the shared execution units of a superscalar processor in the same clock cycle, thereby maximizing pipeline utilization and reducing idle bubbles.

### Q2. What is SIMD vectorization?
> **Answer:**  
> Single Instruction Multiple Data (SIMD) is a computer architecture classification and execution model where a single machine instruction performs the identical arithmetic or logical operation concurrently on multiple data elements stored in wide vector registers.

### Q3. What is FMA (Fused Multiply-Add) and what is its contribution to peak performance?
> **Answer:**  
> Fused Multiply-Add (FMA) is a hardware instruction that computes the operation $A \times B + C$ in a single execution step with a single rounding stage. Its contribution factor is $n_{\text{FMA}} = 2$ because it completes two distinct floating-point operations (one multiply and one add) in a single cycle.

### Q4. What is the difference between SMT and a multi-core processor?
> **Answer:**  
> A multi-core processor replicates the entire physical core (including dedicated execution units, ALUs, and L1 caches), providing truly independent execution hardware. SMT replicates only the lightweight architectural registers and program counters within a single core, while sharing the underlying execution units, caches, and memory buses.

---

## Important Comparisons

### SMT (Hyper-Threading) vs Multi-Core Processing

| Dimension | SMT (Logical Cores) | Multi-Core (Physical Cores) |
| :--- | :--- | :--- |
| **Physical Hardware** | Replicates registers and state ($\approx 5\%$ silicon area). | Replicates complete silicon cores (ALUs, FPUs, L1/L2 caches). |
| **Execution Units** | Shared dynamically between threads. | Completely independent and dedicated per core. |
| **Performance Scaling** | Modest ($\approx 15\%\text{ to } 30\%$ throughput gain). | Linear ($\approx 100\%$ per added core under ideal conditions). |
| **Resource Contention** | Threads can thrash shared caches and contend for ports. | No internal port contention; separate L1 caches. |
| **Power Consumption** | Extremely low additional power. | Scales proportionally with the number of physical cores. |

---

### x86 SIMD Instruction Sets

| Feature | SSE | AVX / AVX2 | AVX-512 |
| :--- | :--- | :--- | :--- |
| **Register Name** | `%xmm0`–`%xmm15` | `%ymm0`–`%ymm15` | `%zmm0`–`%zmm31` |
| **Register Width** | 128 bits | 256 bits | 512 bits |
| **DP Operands ($n_{\text{SIMD}}$)** | **2** doubles | **4** doubles | **8** doubles |
| **SP Operands ($n_{\text{SIMD}}$)** | **4** floats | **8** floats | **16** floats |
| **FMA Support** | No (separate mul/add) | Yes (AVX2 introduces FMA3) | Yes (full 512-bit FMA) |

---

## Solved Numerical Questions

### Question 1: Single-Core Peak Performance Calculation
An Intel Xeon processor core has the following specifications:
- Clock Frequency: $f = 3.0\text{ GHz}$
- Floating-Point Execution Ports: 2 FMA-capable execution units ($n_{\text{superscalar}}^{\text{FP}} = 2$)
- Instruction Set: AVX-512 (512-bit registers)

1. Calculate $n_{\text{SIMD}}$ for Double-Precision (64-bit) floating-point calculations.
2. State the value of $n_{\text{FMA}}$.
3. Calculate the theoretical peak double-precision performance of this core ($P_{\text{core}}$) in $\text{GFLOP/s}$.
4. Calculate the peak performance if the program uses Single-Precision (32-bit) arithmetic.

#### Solution:
**Step 1: Calculate $n_{\text{SIMD}}$ for double precision:**
$$n_{\text{SIMD}}^{\text{DP}} = \frac{512\text{ bits}}{64\text{ bits}} = 8$$

**Step 2: Determine $n_{\text{FMA}}$:**
Since the units support FMA (1 multiply + 1 add):
$$n_{\text{FMA}} = 2$$

**Step 3: Calculate Double-Precision $P_{\text{core}}$:**
$$P_{\text{core}}^{\text{DP}} = n_{\text{superscalar}}^{\text{FP}} \times n_{\text{FMA}} \times n_{\text{SIMD}}^{\text{DP}} \times f$$
$$P_{\text{core}}^{\text{DP}} = 2 \times 2 \times 8 \times 3.0 \times 10^9\text{ Hz}$$
$$P_{\text{core}}^{\text{DP}} = 32 \times 3.0 \times 10^9 = 96.0 \times 10^9\text{ FLOP/s} = 96.0\text{ GFLOP/s}$$

$$\boxed{P_{\text{core}}^{\text{DP}} = 96.0\text{ GFLOP/s}}$$

**Step 4: Calculate Single-Precision $P_{\text{core}}$:**
For single precision (32-bit), the register holds twice as many operands:
$$n_{\text{SIMD}}^{\text{SP}} = \frac{512\text{ bits}}{32\text{ bits}} = 16$$
$$P_{\text{core}}^{\text{SP}} = 2 \times 2 \times 16 \times 3.0 \times 10^9\text{ Hz} = 192.0\text{ GFLOP/s}$$

$$\boxed{P_{\text{core}}^{\text{SP}} = 192.0\text{ GFLOP/s}}$$

---

### Question 2: Multi-Core Server Node Peak Performance
A high-performance computing server consists of **2 CPU sockets**. Each CPU socket houses a processor with:
- $N_{\text{cores}} = 16\text{ physical cores}$ per socket
- Clock Frequency: $f = 2.5\text{ GHz}$
- Floating-Point Superscalarity: $n_{\text{superscalar}}^{\text{FP}} = 2$ units per core
- Instruction Set: AVX2 ($256\text{-bit}$ registers) with FMA support ($n_{\text{FMA}} = 2$)

Calculate the theoretical peak double-precision floating-point performance of the entire 2-socket server node in $\text{TFLOP/s}$.

#### Solution:
**Step 1: Calculate $n_{\text{SIMD}}$ for AVX2 Double Precision:**
$$n_{\text{SIMD}} = \frac{256\text{ bits}}{64\text{ bits}} = 4$$

**Step 2: Calculate peak performance for a single core ($P_{\text{core}}$):**
$$P_{\text{core}} = n_{\text{superscalar}}^{\text{FP}} \times n_{\text{FMA}} \times n_{\text{SIMD}} \times f$$
$$P_{\text{core}} = 2 \times 2 \times 4 \times 2.5\text{ GHz} = 16 \times 2.5\text{ GFLOP/s} = 40.0\text{ GFLOP/s}$$

**Step 3: Calculate total physical cores across the server node:**
$$\text{Total Cores} = 2\text{ sockets} \times 16\text{ cores/socket} = 32\text{ cores}$$

**Step 4: Calculate total server node performance ($P_{\text{node}}$):**
$$P_{\text{node}} = 32 \times 40.0\text{ GFLOP/s} = 1,280\text{ GFLOP/s}$$

**Step 5: Convert to $\text{TFLOP/s}$ ($1\text{ TFLOP/s} = 1,000\text{ GFLOP/s}$):**
$$P_{\text{node}} = \frac{1,280}{1,000}\text{ TFLOP/s} = 1.28\text{ TFLOP/s}$$

$$\boxed{P_{\text{node}} = 1.28\text{ TFLOP/s}}$$

---

## Conceptual Examination Questions

### Q1. Why does running two compute-heavy threads on an SMT core rarely double performance?
> **Model Answer:**  
> SMT does not duplicate the underlying physical execution units (ALUs, FP multipliers, dividers). If both threads are compute-bound (e.g. dense matrix multiplication), they will both compete simultaneously for the exact same execution ports (Port 0 and Port 1). Since the execution units are already fully utilized by the first thread, the second thread cannot execute its arithmetic instructions until the ports become free. As a result, SMT typically yields only a modest $15\%\text{ to } 30\%$ throughput improvement, achieving its greatest benefits when threads have diverse, non-conflicting workloads (e.g., one thread performing integer address calculation while another waits on memory).

### Q2. Explain why compilers must unroll loops to generate efficient SIMD vector instructions.
> **Model Answer:**  
> A standard scalar loop processes one array element per iteration ($i = 0, 1, 2, \dots$). A SIMD vector register, however, requires $n_{\text{SIMD}}$ contiguous data elements (e.g., 4 double-precision numbers for AVX) to be available simultaneously so that a single vector load instruction (`vmovupd`) can populate the register. Compilers unroll the loop by a stride of $n_{\text{SIMD}}$ (`i += 4`) to group four consecutive scalar operations into a single logical block, allowing them to be fused into one vector instruction (`vaddpd`). Without unrolling or strip-mining, the compiler cannot map scalar loop iterations onto wide vector registers.

---

## Quick Revision Sheet

- **Simultaneous Multi-Threading (SMT):** Multiple hardware threads interleave instructions into shared execution pipelines to fill pipeline bubbles.
- **What SMT Replicates:** Only architectural register sets and program counters ($\approx 5\%$ area); shares all execution ports and caches.
- **SIMD Processing:** One single instruction operates on wide vector registers containing multiple data elements concurrently.
- **Vector Widths & Double-Precision Capacity:**
  - $\text{SSE (128-bit)} \longrightarrow 2\text{ doubles}$
  - $\text{AVX/AVX2 (256-bit)} \longrightarrow 4\text{ doubles}$
  - $\text{AVX-512 (512-bit)} \longrightarrow 8\text{ doubles}$
- **Master Peak Performance Formula:**
  $$\boxed{P_{\text{core}} = n_{\text{superscalar}}^{\text{FP}} \times n_{\text{FMA}} \times n_{\text{SIMD}} \times f}$$
- **FMA Factor:** $n_{\text{FMA}} = 2$ (computes 1 multiply + 1 add in a single cycle).

---

## What You Should Be Able to Do After This Topic

1. [ ] Explain the intuition behind SMT using the doctor/examination room analogy.
2. [ ] Identify what hardware structures are duplicated in SMT and what structures are shared.
3. [ ] Define SIMD vectorization and draw the diagram comparing scalar addition with vector addition.
4. [ ] List the register widths and double-precision operand capacities for SSE, AVX, and AVX-512.
5. [ ] Write the C code demonstrating how a compiler unrolls a loop to achieve SIMD vectorization.
6. [ ] State from memory and explain every variable in the Master Peak Performance Formula:
   $$P_{\text{core}} = n_{\text{superscalar}}^{\text{FP}} \times n_{\text{FMA}} \times n_{\text{SIMD}} \times f$$
7. [ ] Solve numerical examination problems calculating the peak GFLOP/s and TFLOP/s of single cores, multi-core chips, and dual-socket nodes.

---

## Source References
- `Sanjay_Notes_1.pdf`: Pages 3, 5–6 (In-core features, SMT concept, pipeline bubbles, SIMD processing, x86 instruction sets SSE/AVX/AVX-512, SIMD by compiler loop unrolling, Peak performance of a core formula).
- `3d773f8d.pdf`: Pages 25–26 (OpenMP SIMD vectorization, AVX register widths, compiler vectorization flags).
- `reference/teaching_style_sample.md`: Sections 2–3 (Concurrency, Independent operations, Vector execution).
