# Final Year Project (FYP) Proposal (Version 2.3 — Publication-Ready)
## Project Title: DeltaMorph: Autonomous Breaking-Change Healer

**Department:** Computer Science  
**Institution:** Dawood University of Engineering and Technology (DUET)  
**Developers:** Sumeet Kanhiya ([@sumeetkl11](https://github.com/sumeetkl11)) & Wahaj Khan  
**Academic Year:** 6th - 8th Semester (Capstone Project)  

---

### 1. Executive Summary
Modern software engineering relies heavily on third-party open-source dependencies. When libraries release major semantic versioning updates or deprecate APIs, dependent codebases break with runtime `TypeError`, altered function signatures, or changed parameter contracts. Tools like GitHub Dependabot only increment version numbers in `package.json`, immediately breaking CI builds and leaving developers with hours of manual refactoring.

**DeltaMorph** is an autonomous, compiler-verified program repair tool built for the TypeScript/JavaScript ecosystem. By unifying context extraction entirely on the **TypeScript Compiler API (`ts-morph`)**, DeltaMorph eliminates dual-AST impedance mismatches, extracts mathematical type deltas directly from `.d.ts` declarations (with a formal fallback pipeline for behavioral breaks), executes candidate repairs inside isolated ephemeral git worktrees and sandboxes, and eliminates regression oscillations via an **Alpha-Normalized Tabu Search** state machine.

---

### 2. Falsifiable Research Questions & Hypotheses

**RQ1: Does AST-guided context slicing improve automated repair success over full-file prompting?**
- **H1:** AST-sliced prompts (<350 tokens) achieve **Pass@1 ≥ 55%** compared to unbounded full-file prompts (>2000 tokens) achieving **Pass@1 ≤ 45%**, tested across 25 real npm breaking-change PRs.
- **Metric:** Pass@1, Pass@3 with Wilson score 95% confidence intervals.

**RQ2: Does Alpha-Normalized Tabu state detection reduce wasted repair iterations?**
- **H2:** Tabu state memory reduces mean iterations-to-fix from **≥2.5 to ≤1.8** compared to naive retry loops (no memory).
- **Metric:** Mean iteration count with and without Tabu memory, paired t-test significance.

**RQ3: Does `.d.ts` structural diffing outperform stack-trace-only prompting on delta-rich breaking changes?**
- **H3:** On the subset of benchmarks producing non-empty Symbol Delta Tables (~60%), `.d.ts`-augmented prompts yield **≥20% higher Pass@1** than stack-trace-only prompts.
- **Metric:** Ablation study comparing full pipeline vs. pipeline-without-delta.

---

### 3. Literature Context & Baseline Positioning
DeltaMorph builds on and extends the following established research:

| Prior Work | Contribution | DeltaMorph's Differentiation |
| :--- | :--- | :--- |
| **SWE-agent (Yang et al., 2024)** | Autonomous agent with terminal + file editing | No type-aware context slicing; monolithic file retrieval |
| **Aider (Gauthier, 2023)** | AST-aware diffs with LLM generation | No deterministic sandbox verification loop |
| **ChatRepair (Xia & Zhang, 2023)** | Conversational APR with test feedback | No `.d.ts` ground-truth delta extraction |
| **OpenRewrite / Moderne** | Deterministic AST codemods | Rule-based only; cannot generalize to undocumented breaks |
| **GenProg (Le Goues et al., 2012)** | Genetic programming for APR | Search-based; no LLM synthesis capability |

The maintenance burden statistic ("up to 40% of developer time on dependency management") is sourced from the **Tidelift 2023 Open Source Maintainer Survey** and corroborated by the **GitHub Octoverse 2023 Report** on dependency update frequency.

---

### 4. Comparison with Existing Industry & Academic Tools

| Capability | Renovate / Dependabot | Cursor / Copilot Agent | SWE-agent (Academic) | DeltaMorph (Ours) |
| :--- | :--- | :--- | :--- | :--- |
| **Breaking Change Healing** | ❌ Version bump only | ⚠️ Interactive / Prompt-dependent | ⚠️ Monolithic multi-file | ✅ Autonomous compiler-verified |
| **Ground-Truth Delta Source** | ❌ None | ⚠️ Web search / user prompt | ❌ None | ✅ `.d.ts` AST diffing + behavioral fallback |
| **AST Engine** | N/A | Naive file search | External grep/ctags | ✅ Unified `ts-morph` |
| **Anti-Oscillation Guard** | ❌ | ❌ | ❌ Retry limit only | ✅ Alpha-Normalized Tabu |
| **Working Tree Safety** | N/A | Modifies active files | Container-only | ✅ Isolated `git worktree` |
| **Prompt Sanitization** | N/A | ❌ | ❌ | ✅ Adversarial content stripping |

---

### 5. Bounded Project Scope

* **In Scope (Strict Depth):**
  - Language: TypeScript & JavaScript (Node.js LTS). JS-only repos supported via synthetic `tsconfig.json` with `allowJs: true`.
  - AST & Type Engine: Unified 100% on `ts-morph` (TypeScript Compiler API).
  - Test Runners: `Vitest` and `Jest` (with redirected `--cacheDirectory`).
  - Target Libraries: High-churn API and utility dependencies.
  - Sandboxing: Dual-mode (Docker containers + local git worktree fallback).
* **Explicitly Out of Scope:**
  - Multi-paradigm framework rewrites (Angular to React, Vue Options to Composition).
  - Database schema alterations and ORM data migrations.
  - Full ESM/CJS module system conversions (flagged as Architectural Migration for human review).

---

### 6. Threats to Validity

#### Internal Validity
- **Selection Bias:** Benchmarks are manually curated. Mitigation: Include both popular (axios) and niche (pino, helmet) libraries; split into "delta-rich" and "delta-empty" subsets.
- **Flaky Tests:** Non-deterministic tests could inflate Pass@k metrics. Mitigation: Run baseline tests 5× on unchanged code to identify and exclude flaky tests.
- **LLM Non-Determinism:** Even with `temperature=0`, API providers do not guarantee identical outputs. Mitigation: Pin model versions, log all prompts/responses with checksums, report ensemble consistency.

#### External Validity
- **Ecosystem Generalization:** Results are scoped to the npm/TypeScript ecosystem. Python, Rust, and Go are explicitly deferred to future work.
- **LLM Provider Variance:** Gemini Flash evolves monthly. Mitigation: Pin exact model version (e.g., `gemini-1.5-flash-001`).

#### Construct Validity
- **Pass@k Definition:** A "pass" requires all baseline tests to pass but does not fully verify semantic correctness beyond tests. Mitigation: Manual expert patch inspection on all successful repairs with Cohen's Kappa (κ) inter-rater agreement reporting.

---

### 7. Faculty Defense Pitch (DUET Evaluation Committee)
> *"Sir, DeltaMorph is an **Applied Software Engineering, Program Analysis, and Compiler-Verified Program Repair** project. We solved the dual-AST impedance mismatch by standardizing on `ts-morph`, bypassed fragile changelogs via `.d.ts` declaration diffing with a formal fallback pipeline for behavioral breaks, and designed an Alpha-Normalized Tabu State Machine to mathematically eliminate cyclic repair loops. Every candidate fix is verified through static lint gates and isolated ephemeral test runners. Our evaluation uses Wilson score confidence intervals and Cohen's Kappa inter-rater agreement on 25 real-world npm breaking changes."*
