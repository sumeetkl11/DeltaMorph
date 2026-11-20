# DeltaMorph: Defense Landmines & Viva Interrogation Guide

**Author:** Uday Sharma ([@udaysharmadev](https://github.com/udaysharmadev))  
**Academic Institution:** Dawood University of Engineering & Technology (DUET), Department of Computer Science  
**Project:** DeltaMorph — Compiler-Verified Breaking-Change Healer via TypeScript Compiler API & AST-Guided Program Repair  
**Document Purpose:** Master guide to ace the proposal defense, internal evaluations, and final external viva. Covers technical flaws, deep architectural comparisons, and rigorous defense strategies.

---

## 1. The 6 Implementation Landmines & Their Concrete Solutions

### Landmine 1: The Windows & Git Worktree `node_modules` Ghost Trap
* **The Vulnerability:**  
  When executing `git worktree add --detach .deltamorph-scratch HEAD`, git creates an isolated working copy containing only tracked repository files. Because `node_modules/` is git-ignored, the scratchpad directory has zero installed dependencies!
  - Running `npm install` inside the worktree wastes 30–90 seconds per run, destroying our `<30s` latency target.
  - Using standard filesystem symlinks (`ln -s` or `fs.symlinkSync(..., 'file')`) on Windows fails with `EPERM: operation not permitted` unless the developer runs the terminal as Administrator or has Windows Developer Mode enabled.
* **The DeltaMorph Solution:**  
  DeltaMorph detects the OS platform at runtime:
  - On Linux / macOS: Creates standard symlinks.
  - On Windows: Uses native **NTFS Directory Junctions** (`fs.symlinkSync(src, dest, 'junction')` or `mklink /J`). Junctions do NOT require administrator privileges on Windows and link instantaneously (0ms overhead).

---

### Landmine 2: Multi-File Cascade Failure (Cycle Exhaustion)
* **The Vulnerability:**  
  A major dependency update (e.g., Axios v0.27 $\rightarrow$ v1.6 changing `AxiosResponse<T>` generics) often breaks 3 to 6 files across a repository simultaneously. If the repair budget is a flat "max 3 iterations", the LLM spends all 3 attempts fixing `src/api/user.ts` and runs out of budget before ever seeing `src/api/auth.ts` or `src/api/products.ts`.
* **The DeltaMorph Solution:**  
  A **Two-Tier Repair Loop Architecture**:
  1. **Outer Loop (File Queue):** Identifies all broken files via `tsc --noEmit` diagnostic output and queues them (capped at 5 files).
  2. **Inner Loop (Per-File Repair):** Dedicated budget of max 2 iterations per file. If a file succeeds, its patch is staged, and the engine moves to the next file in the queue.
  - Global ceiling: 6 LLM calls total across all files.

---

### Landmine 3: `@ts-ignore` & `as any` Cheat Mode (LLM Shortcut Exploits)
* **The Vulnerability:**  
  When an LLM is prompted to make `tsc --noEmit` exit 0, it frequently discovers the "lazy escape hatch": inserting `// @ts-ignore`, `// @ts-nocheck`, or casting `(data as any).property`. The code "compiles", but type safety is destroyed and production runtime errors will occur.
* **The DeltaMorph Solution:**  
  **Anti-Cheat Static Lint Gate:**  
  Before running any compiler verification or test runner, the patched AST is passed through strict AST inspection rules:
  - `@typescript-eslint/ban-ts-comment`: Rejects any patch introducing `ts-ignore` or `ts-nocheck`.
  - `@typescript-eslint/no-explicit-any`: Compares AST nodes before and after. If an `AnyKeyword` was added that did not exist in the original AST, the patch is rejected immediately with a penalty score.

---

### Landmine 4: Subprocess Hanging on Unresolved Promises & Open Sockets
* **The Vulnerability:**  
  If a broken migration alters an asynchronous return type (e.g., an Express middleware callback that should return `next()` or an unresolved Promise in Vitest), the test runner subprocess hangs indefinitely. The CLI freezes, holding git lock files.
* **The DeltaMorph Solution:**  
  **Hard Subprocess Guardrails with `AbortController`:**  
  Every child process (`vitest`, `jest`, `tsc`) is wrapped in an atomic 15-second timeout. If the subprocess does not exit within 15 seconds:
  1. A `SIGKILL` (or `taskkill /F /T` on Windows) is dispatched immediately to the entire process tree.
  2. The failure is classified as `EXECUTION_TIMEOUT_ERROR`.
  3. The worktree is restored to its clean state without leaving stale `index.lock` files.

---

### Landmine 5: The "Deterministic" Word Trap in Academic Defense
* **The Vulnerability:**  
  If you claim in your defense that DeltaMorph is a "100% Deterministic Autonomous Healer", an external examiner with a background in formal methods will destroy your proposal: *"LLMs use temperature sampling and probabilistic next-token generation. How can you claim determinism?"*
* **The DeltaMorph Solution:**  
  **Honest, Falsifiable Scientific Phrasing:**  
  - Ground-truth extraction (`.d.ts` AST diffing) is **100% Deterministic**.
  - Verification (`tsc --noEmit` + test suite) is **100% Deterministic**.
  - Patch generation is **Guided Stochastic Program Repair**.
  DeltaMorph is academically defined as:  
  *"A Compiler-Verified Breaking-Change Healer via TypeScript Compiler API & AST-Guided Program Repair."*

---

### Landmine 6: Dynamic `tsconfig.paths` & JSX Alias Resolution
* **The Vulnerability:**  
  Modern full-stack TypeScript projects (Next.js, Vite) use path aliases like `@/components/Button` or `@/lib/db`. If `ts-morph` parses sliced files in isolation without loading `baseUrl` and `paths` mappings, it fails to resolve imported types, resulting in false positive compiler diagnostics.
* **The DeltaMorph Solution:**  
  DeltaMorph loads the project's root `tsconfig.json` dynamically into `ts-morph`, inheriting path aliases and compiler flags, and enforces `compilerOptions.jsx = ts.JsxEmit.Preserve` so JSX/TSX syntax does not cause AST parser crashes.

---

## 2. Comparative Analysis: Tropos vs. DeltaMorph

| Dimension | Tropos (Conceptual/Planning Agent) | DeltaMorph (Active Compiler-Verified Healer) |
| :--- | :--- | :--- |
| **Primary Nature** | Architecture & reasoning assistant (read/advise) | Active, autonomous software repair tool (mutate/verify) |
| **Execution Environment** | Host OS terminal / conversational prompt | Isolated `git worktree` & ephemeral container |
| **AST Technology** | Tree-sitter / regex pattern search | TypeScript Compiler API (`ts-morph`) with full type-checker |
| **Ground-Truth Source** | Natural language release notes / general LLM memory | Structural diff of compiler `.d.ts` declaration trees |
| **Repair Loop** | Single-shot prompt or open-ended chat | Closed-loop state machine with Alpha-Normalized Tabu hashing |
| **Safety Invariants** | Relies on user reviewing generated code | Anti-cheat AST linting, read-only test suite, 15s hard timeout |
| **Verification Gate** | Developer manual review | Dual-gate: `tsc --noEmit` (static) + Vitest/Jest (runtime) |
| **Academic Novelty** | Conversational workflow assistant | Bounded AST-guided APR (Automated Program Repair) |

---

## 3. Top 10 Viva Interrogation Questions & Model Answers

### Q1: "Why not just use GitHub Copilot Workspace or Cursor Composer?"
> **Model Answer:**  
> "Commercial agents like Copilot and Cursor dump full repository files into huge LLM context windows (2,000–8,000 tokens) and rely on stochastic next-token guessing. They lack ground-truth knowledge of what changed in the upgraded package's binary interface.  
> DeltaMorph is fundamentally different: it extracts the mathematical delta between old and new `.d.ts` declaration files using the TypeScript Compiler API (`ts-morph`). It provides the LLM with a 200–350 token surgical AST slice containing only the affected identifier and its callers. Furthermore, DeltaMorph enforces an automated, closed-loop compiler and test verification gate that prevents `@ts-ignore` cheating and infinite oscillation."

### Q2: "What if the breaking change is behavioral, where type definitions didn't change at all?"
> **Model Answer:**  
> "This is explicitly addressed by our **Behavioral Break Fallback Pipeline (FR-2.4)**. When `.d.ts` AST diffing produces an empty Symbol Delta Table (e.g., `jsonwebtoken` rejecting insecure algorithms at runtime), the system switches to Stack-Trace-Only Repair Mode. It sanitizes and filters the runtime assertion trace down to under 100 tokens, queries the package's semver git tag release diff, and flags the generated patch with `confidence: low` in the final engineering report."

### Q3: "Tree-sitter is faster and supports multiple languages. Why did you choose `ts-morph` over Tree-sitter?"
> **Model Answer:**  
> "Tree-sitter is a fast Concrete Syntax Tree (CST) parser, but it has zero type semantics. It cannot resolve whether `response.data` is an `AxiosResponse` or a generic `any`, and it cannot query symbol tables or run `tsc --noEmit`. Mixing Tree-sitter for slicing with `tsc` for verification creates a 'dual-AST coordinate mismatch' where byte offsets do not align. Consolidating 100% on `ts-morph` gives us access to TypeScript's internal type-checker, symbol resolution, and surgical AST manipulation in a single unified coordinate space."

### Q4: "How do you prove your system isn't just memorizing common fixes from training data?"
> **Model Answer:**  
> "Our evaluation benchmark contains 25 curated test cases: 15 open-source production migrations and 10 synthetic edge cases. Crucially, our roadmap includes 5 held-out obscure npm packages with fewer than 100,000 total downloads, ensuring their migration paths do not exist in LLM pre-training data. Furthermore, we run a 5-configuration component ablation study (Full system vs. No AST slice vs. No Symbol Delta vs. Full File vs. Raw LLM) to isolate the exact contribution of each component."

### Q5: "What prevents the LLM from hallucinating an infinite loop or flipping back and forth between two wrong fixes?"
> **Model Answer:**  
> "We implement an **Alpha-Normalized Tabu State Machine (FR-7)**. Before any candidate patch is tested, its AST is parsed, and all local variable names are normalized to canonical tokens (`_v0`, `_v1`). The resulting normalized AST is hashed via SHA-256. If that semantic hash has been visited in previous iterations, the patch is rejected immediately without wasting test execution time. The engine jitters the temperature parameter and appends the failed patch to the prompt's negative constraint memory."

### Q6: "Can an attacker compromise your system by putting malicious prompts inside package error messages?"
> **Model Answer:**  
> "No. DeltaMorph features a dedicated **Prompt Sanitization Layer (FR-4)**. All compiler output and error traces pass through regex filters that strip non-printable characters, truncate lines to 200 characters, and remove known prompt injection signatures. The sanitized diagnostic data is wrapped in strict `<UNTRUSTED_COMPILER_OUTPUT>` XML tags with explicit system instructions to treat it as passive diagnostic data rather than executable instructions."

### Q7: "Why use Git Worktrees instead of just editing files directly or using Docker for everything?"
> **Model Answer:**  
> "Editing files directly risks polluting the developer's working tree or creating unrecoverable merge conflicts if the repair fails or the user cancels with `Ctrl+C`. Docker Desktop introduces significant startup latency (15–30 seconds) and often fails on student workstations. Git Worktrees provide an isolated, ephemeral scratchpad that takes milliseconds to create, leaving the developer's working files 100% untouched while allowing our lightweight Local Mode to operate without Docker."

### Q8: "How do you evaluate success scientifically?"
> **Model Answer:**  
> "We report Pass@1 and Pass@3 metrics calculated with Wilson Score 95% Confidence Intervals. For statistical significance between DeltaMorph and baselines, we compute McNemar's test for paired binary outcomes ($p < 0.05$). For qualitative patch review by human senior developers, we compute Cohen's Kappa inter-rater agreement ($\kappa > 0.6$ required for validity)."

### Q9: "What is your fallback if the LLM API goes down or hits rate limits?"
> **Model Answer:**  
> "DeltaMorph implements a multi-provider failover chain. The primary synthesizer is Google Gemini 1.5 Flash (high speed, cost-effective). If an API rate limit (HTTP 429) or outage occurs, the system automatically falls back to Groq Llama-3-70B. If both fail, exponential backoff retries up to 3 times before generating a clean diagnostic report for the developer."

### Q10: "What are the limitations of DeltaMorph?"
> **Model Answer:**  
> "Being honest about limitations is vital:  
> 1. DeltaMorph is scoped to JavaScript and TypeScript repositories; it does not repair Python, Java, or Go.  
> 2. It requires an existing test suite or type checker to verify correctness; if a repository has zero tests and `noImplicitAny: false`, DeltaMorph cannot guarantee semantic validity.  
> 3. Complex multi-file architectural refactors involving more than 5 interdependent files exceed our token and cycle budget and are escalated to human developers."

---

## 4. Proposal Defense Checklist for DUET Panel

- [ ] Print copies of `SYNOPSIS.md` and `01_PROJECT_PROPOSAL.md`.
- [ ] Memorize the 3 core research questions (H1: AST Slicing Latency, H2: Delta-Guided Pass@1, H3: Tabu Anti-Oscillation).
- [ ] Understand the difference between `ts-morph` and Tree-sitter (semantic type-checker vs syntax-only CST).
- [ ] Practice explaining NTFS Junctions vs Symlinks for Windows.
- [ ] Emphasize that DeltaMorph is **Compiler-Verified Program Repair**, not a chat bot or basic Copilot wrapper.
- [ ] Keep the 5-configuration ablation study slide ready to prove scientific rigor.