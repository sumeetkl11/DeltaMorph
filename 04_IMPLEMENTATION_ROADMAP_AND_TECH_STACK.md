# Implementation Roadmap, Tech Stack, Evaluation Protocol & Benchmarks
## Project: DeltaMorph (Autonomous Breaking-Change Healer)

**Institution:** Dawood University of Engineering and Technology (DUET)  
**Developers:** Sumeet Kanhiya ([@sumeetkl11](https://github.com/sumeetkl11)) & Wahaj Khan  
**Document Version:** 2.3 (Production-Ready)  

---

### 1. Technology Stack

| Component | Technology | Justification |
| :--- | :--- | :--- |
| **Runtime** | Node.js (v20+ LTS) & TypeScript | Native npm integration, async IO, strict typing |
| **AST & Type Engine** | `ts-morph` (TS Compiler API) | Unified engine: slicing, type checking, `.d.ts` diffing |
| **Workspace Isolation** | Native Git Worktrees | Zero pollution, instant branch creation, `SIGINT`-safe |
| **Container Sandbox** | Docker via `dockerode` | Ephemeral `tmpfs`, `:ro` test isolation, zero egress |
| **Lint Gate** | ESLint (`@typescript-eslint`) | Sub-second floating promise and type error detection |
| **LLM Orchestration** | Gemini Flash + Groq Llama-3-70B | Free-tier, sub-second latency, failover chain |
| **CLI UI** | `commander` + `ink` | Rich interactive terminal, zero web dashboard |

---

### 2. Curated Benchmark Dataset (25 Repositories)

#### Part A: Real-World Production Repositories (15 Repos)

| # | Library | Migration | Breaking Change Class | Expected Delta Type |
| :--- | :--- | :--- | :--- | :--- |
| 1 | **Axios** | `v0.27` → `v1.x` | Config parameter reordering, AxiosError shape | Structural (Delta-Rich) |
| 2 | **Date-fns** | `v2.x` → `v3.x` | Default export removal, argument strictness | Structural |
| 3 | **JsonWebToken** | `v8.x` → `v9.x` | Runtime algorithm enforcement (signature unchanged) | **Behavioral (Delta-Empty)** |
| 4 | **Zod** | `v2.x` → `v3.x` | Chained validation syntax, transformer rename | Structural |
| 5 | **Commander.js** | `v8.x` → `v10.x` | Action argument unpacking | Structural |
| 6 | **Pino** | `v6.x` → `v7.x` | Sync stream → async worker threads | **Behavioral** |
| 7 | **Got** | `v11.x` → `v12.x` | Promise hook, options normalization | Structural |
| 8 | **Chai** | `v4.x` → `v5.x` | CJS → ESM module boundary | **Behavioral** |
| 9 | **Formik** | `v2.1` → `v2.4` | React 18 type compatibility | Structural |
| 10 | **Dotenv** | `v8.x` → `v16.x` | Multiline parsing, override options | Structural |
| 11 | **Bcrypt** | `v5.0` → `v5.1` | Async salt generator returns | Structural |
| 12 | **Uuid** | `v8.x` → `v9.x` | Named exports enforcement | Structural |
| 13 | **Supertest** | `v6.x` → `v7.x` | Agent cookie persistence | Structural |
| 14 | **Helmet** | `v4.x` → `v7.x` | CSP directive schema migration | Structural |
| 15 | **Morgan** | `v1.9` → `v1.10` | Stream interface types | Structural |

**Delta Distribution:** ~11 Delta-Rich (structural) + ~4 Delta-Empty (behavioral). Results reported separately per category.

#### Part B: Targeted Edge-Case Repositories (10 Repos)

| # | Edge Case | Purpose |
| :--- | :--- | :--- |
| 1 | Flaky Test Suite | Baseline stability filtering validation |
| 2 | Cyclic Oscillation Trap | Tabu state machine stress test |
| 3 | Floating Promise Bug | ESLint static gate validation |
| 4 | DefinitelyTyped Detached | `@types/<pkg>` resolution engine test |
| 5 | Monorepo Workspace | pnpm workspace hoisted dependency traversal |
| 6 | Multi-Occurrence Symbol | `AMBIGUOUS_REPLACE_TARGET` guard test |
| 7 | Long Stack Trace (600 lines) | Stack frame truncation (<100 tokens) test |
| 8 | Malformed Search Target | Two-tier fuzzy matching recovery test |
| 9 | Zero Test Coverage Module | Unverifiable function flagging test |
| 10 | Peer Dependency Deadlock | `npm explain` diagnostic halt test |

---

### 3. Quantitative Evaluation Protocol

#### 3.1 Primary Metrics
1. **Pass@1:** % of benchmarks fixed on first iteration.
   - Target: $\geq 55\%$ (Delta-Rich), $\geq 30\%$ (Delta-Empty).
2. **Pass@3:** % fixed within 3 iterations.
   - Target: $\geq 78\%$ (Delta-Rich), $\geq 50\%$ (Delta-Empty).
3. **Token Efficiency:** Median prompt tokens per repair attempt.
   - Unbounded baseline: ~3,800 tokens. AST-sliced target: <350 tokens (~90% reduction).
4. **Mean Time to Repair (MTTR):** End-to-end latency per successful fix.
   - Local mode target: <40s. Docker mode target: <75s.
5. **API Cost per Fix:** Total tokens consumed × price-per-token.

#### 3.2 Statistical Rigor
* **Confidence Intervals:** Wilson score 95% CIs on all Pass@k rates. With n=25, a measured 60% Pass@1 yields CI ≈ [40.7%, 76.6%].
* **Significance Testing:** McNemar's test comparing AST-sliced vs. full-file prompting on the same 25 benchmarks (paired binary outcomes).
* **Effect Size:** Report Cohen's d for iteration count reduction (Tabu vs. no-Tabu).

#### 3.3 Qualitative Evaluation (Patch Correctness)
* **Reviewers:** Faculty supervisor (Reviewer A) + one external software engineer / DUET alumni (Reviewer B).
* **Classification per Patch:**
  - `Correct` — Preserves business semantics, idiomatic code.
  - `Plausible but Suspect` — Tests pass but logic is questionable.
  - `Incorrect` — Test gap exploited or silent bug introduced.
* **Inter-Rater Agreement:** Report **Cohen's Kappa (κ)**.
  - Target: $\kappa > 0.6$ (substantial agreement).
  - If $\kappa < 0.4$, reviewers discuss disagreements and re-classify with documented rationale.

#### 3.4 Ablation Study Design
Systematically disable each component and re-run all 25 benchmarks:

| Ablation Run | Component Disabled | Measures |
| :--- | :--- | :--- |
| **A0 (Full System)** | None (baseline) | Pass@1, Pass@3, MTTR |
| **A1** | `.d.ts` Delta Engine (stack-trace only) | Isolates value of type diffing |
| **A2** | AST Slicing (full-file context) | Isolates value of bounded context |
| **A3** | Tabu State Machine (naive retries) | Isolates anti-oscillation benefit |
| **A4** | Static Lint Gate (no ESLint pre-check) | Isolates value of floating-promise detection |
| **A5** | Prompt Sanitization (raw error injection) | Measures prompt injection vulnerability rate |

---

### 4. Baseline Comparison Protocol

| Metric | Manual Human Fix | Cursor/Aider (Interactive) | Full-File LLM (No Slicing) | **DeltaMorph (Ours)** |
| :--- | :--- | :--- | :--- | :--- |
| Pass@1 | 100% (by definition) | ~60-65% | ~40-45% | **≥55%** |
| Time per fix | ~15 min | ~2 min | ~90s | **<40s** |
| Token cost | \$0 | ~\$0.02 | ~\$0.01 | **~\$0.003** |
| Deterministic | ✅ | ❌ | ❌ | ✅ (compiler-verified) |

---

### 5. Lean MVP vs. Stretch Features

| Feature | 🚨 MVP (Must-Have) | ⏸️ Stretch (If Time Permits) |
| :--- | :--- | :--- |
| **Git Worktree Isolation** | ✅ Core | — |
| **`ts-morph` Slicing + `.d.ts` Diffing** | ✅ Core | — |
| **Single LLM Provider (Gemini Flash)** | ✅ Core | Multi-provider failover (Groq) |
| **Vitest Runner** | ✅ Core | Jest support |
| **Search/Replace Block Patching** | ✅ Core | — |
| **Static Lint Gate** | ✅ Core | — |
| **Alpha-Normalized Tabu** | ⏸️ Stretch | — |
| **Prompt Sanitization** | ✅ Core | — |
| **GitHub Action Integration** | ⏸️ Stretch | — |
| **Ablation Study (5 runs)** | ⏸️ Month 5-6 | — |

---

### 6. Six-Month Academic Milestone Roadmap

#### Semester 6 — Months 1–2: Proposal, PoC & Benchmarks
* [x] Complete publication-ready documentation (v2.3).
* [ ] Build standalone `ts-morph` `.d.ts` differ PoC on Axios + Date-fns.
* [ ] Fork and prepare 5 initial benchmark repos with before/after branches.
* [ ] Submit formal Synopsis to DUET CS committee.
* [ ] Begin Related Work literature review (cite 15 papers: Byam, SWE-agent, Aider, ChatRepair, GenProg, APR surveys).

#### Semester 7 — Months 3–4: Core Engine Development
* [ ] Build `deltamorph` CLI with git worktree lifecycle management.
* [ ] Implement `ts-morph` ancestor slicer + `.d.ts` delta engine.
* [ ] Implement Search/Replace two-tier matching engine.
* [ ] Integrate Vitest runner with `--cacheDirectory` redirection.
* [ ] Implement prompt sanitization pipeline.
* [ ] Implement static lint gate (`tsc --noEmit` + `no-floating-promises`).
* [ ] Begin writing Methodology section of research paper (parallel with dev).

#### Semester 8 — Months 5–6: Evaluation, Paper & Final Defense
* [ ] Complete all 25 benchmark repos (15 production + 10 edge cases).
* [ ] Run full evaluation harness: Pass@1, Pass@3, MTTR, token efficiency.
* [ ] Run ablation study (5 component-disabled runs).
* [ ] Conduct blind patch review with 2 reviewers; compute Cohen's κ.
* [ ] Complete research paper draft: *"Compiler-Verified Program Repair for Dependency Breaking Changes via TypeScript Compiler AST Slicing"*.
* [ ] Prepare live CLI terminal demo + 15-slide defense deck.
* [ ] Submit to DUET external examination panel.
