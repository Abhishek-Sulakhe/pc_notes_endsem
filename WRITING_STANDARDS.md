# Writing Standards & Pedagogical Framework

**Reference Standard:** [`reference/teaching_style_sample.md`](file:///d:/Academics/3rd%20year/PC/study/reference/teaching_style_sample.md)  
**Role:** Expert University Parallel Computing Professor, Curriculum Designer, and Examination Author  
**Target Output Directory:** All study notes must be created inside `notes_study/<Topic_Folder>/<Subtopic_File>.md`

---

## 1. Core Teaching Philosophy: Understanding Over Memorization

Every study note must teach the subject from first principles. Do not assume prior knowledge of advanced mechanisms. Follow this conceptual progression:
1. **What is the concept?** Clear, accessible intuition in plain English.
2. **Why do we need it?** What physical, architectural, or computational problem existed before this concept?
3. **How does the concept solve that problem?** The underlying mechanism explained step by step.
4. **How does it actually work?** Concrete flow, memory states, data paths, or thread interactions.
5. **ASCII Architectural/Flow Diagrams:** Clear visual mental models for every major concept.
6. **Code / Mathematical Formalism:** Precise syntax, derivations, and worked examples.
7. **What are its trade-offs / limitations?** Bottlenecks, overhead, failure cases.
8. **Exam-Ready Definitions & Comparison Tables:** High-yield summaries for university exams.
9. **Conceptual & Numerical Exam Questions:** Solved questions with step-by-step reasoning.
10. **Quick Revision Sheet & Self-Check Checklist:** Core takeaways and traps.

---

## 2. Mathematical Standards & Equation Formatting

- **KaTeX Equations:** Use LaTeX math blocks formatted cleanly:
  - Inline math: `$S(p) = \frac{T_1}{T_p}$`
  - Display equations on their own lines:
    $$\boxed{S(p) = \frac{1}{s + \frac{1 - s}{p}}}$$
- **Equation Requirements:**
  1. Define every single variable explicitly immediately after the formula.
  2. State what the formula measures in physical/computational terms.
  3. Provide full algebraic derivations when derived from first principles.
  4. Always box key formulas using `\boxed{...}`.
  5. Provide at least one fully worked-out numerical example with realistic numbers and complete step-by-step arithmetic.
  6. Highlight common examination mistakes (e.g., forgetting percentage conversion, units, or asymptotic limits).

---

## 3. Code Presentation & Analysis Standards

- Code must never be present merely for decoration.
- Every code example must include:
  1. **Objective:** What problem the code solves.
  2. **Syntax / Pragma Breakdown:** Purpose of directives, clauses, or function signatures.
  3. **Full C/C++ Code Snippet:** Clean, standards-compliant, and well-commented.
  4. **Execution Behavior:** What executes sequentially on the master thread and what executes concurrently across worker threads.
  5. **Thread Behavior & Memory Interactions:** Where variables live (shared heap vs thread-private stack), cache line movements, or synchronization wait points.
  6. **Expected Output & Nondeterminism:** Explain why output can appear in arbitrary order.
  7. **Common Traps:** Race conditions, missing private clauses, uninitialized variables, deadlock hazards.

---

## 4. Visual Diagram Standards

Every conceptual mechanism that involves thread timelines, memory hierarchies, or communication channels must feature an ASCII or text diagram. Examples:
- Fork-Join thread timelines
- Von Neumann & multi-core socket architectures
- Memory hierarchies with latency/bandwidth numbers
- Buffer states in synchronization problems
- Message passing networks and port handshakes

---

## 5. Standard Study File Structure

Each file in `notes_study/` must adapt the following consistent structure:

```markdown
# [Subtopic Name]

[Brief introductory paragraph explaining what this subtopic covers and how it logically connects to the previous material.]

---

## 1. Concept Name

### Basic Intuition
[Plain English explanation, real-world analogy]

### Why Do We Need It?
[Problem that existed before, hardware/software motivation]

### How Does It Work?
[Step-by-step mechanism, ASCII diagram, hardware/software behavior]

---

## 2. Subsequent Concepts...

---

## Important Formulas & Derivations
[Boxed formulas, variable definitions, derivations, physical interpretations]

---

## Code Implementations & Microarchitectural Flow
[Complete C/C++ programs, pragma breakdowns, thread behavior, memory layouts]

---

## Exam-Ready Definitions
[Precise, concise, word-for-word definitions ready for 2-mark and 5-mark university questions]

---

## Comparison Tables
[Structured difference tables comparing related constructs]

---

## Solved Numerical Problems
[Step-by-step worked numerical questions with complete intermediate steps]

---

## Conceptual Examination Questions & Model Answers
[High-yield university descriptive questions with thorough answers]

---

## Quick Revision Sheet
[Boxed summary formulas, core definitions, and must-remember points]

---

## What You Should Be Able to Do After This Topic
[Numbered checklist of questions and abilities the student should independently master]

---

## Source References
[Exact source PDF filenames and page ranges where the material originates]
```

---

## 6. Execution Protocol: Strict One-by-One Generation

- Study files must be generated **one by one, sequentially**.
- Never generate multiple study files in a single prompt or response.
- For each file:
  1. Retrieve exact source sections from `complete_content_extraction.md` and source extractions.
  2. Write the comprehensive, deep study document with zero compromises on quality.
  3. Run quality checks (completeness, intuition, mathematical correctness, exam readiness, code fidelity).
  4. Save the file to its designated path inside `notes_study/`.
  5. Update `PROGRESS.md` and `GENERATION_CONTEXT.md`.
