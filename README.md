# DeltaMorph: Autonomous Breaking-Change Healer

**Project Title:** DeltaMorph: Autonomous Breaking-Change Healer  
**Institution:** Dawood University of Engineering & Technology (DUET), Karachi  
**Department:** Computer Science  
**Project Team:** Sumeet Kanhiya ([@sumeetkl11](https://github.com/sumeetkl11)) & Wahaj Khan  
**Supervisor:** [Departmental Faculty Supervisor]  
**Documentation Version:** 3.2 (Empirically Grounded Pre-Defense Edition — Appendix A Verified)  

---

### Project Documentation Index

1. [SYNOPSIS.md](./SYNOPSIS.md)
   - Official 2-page academic synopsis formatted for DUET CS Departmental Committee approval (Abstract, Problem Statement, RQ1–RQ3, Methodology, 15-Benchmark Evaluation Protocol, Compute/Cost Ceiling, Team Split, Timeline, 12 IEEE/ACM References).
2. [01_PROJECT_PROPOSAL.md](./01_PROJECT_PROPOSAL.md)
   - Falsifiable Research Questions (RQ1–RQ3), 14-Paper Literature Positioning across APR, Breaking Dependency Studies, and Program Slicing, Bounded Scope, Threat Model, Threats to Validity, Pre-Defense Evidence Plan, Team Role Split, and Defense Pitch.
3. [02_SYSTEM_REQUIREMENTS_SPECIFICATION_SRS.md](./02_SYSTEM_REQUIREMENTS_SPECIFICATION_SRS.md)
   - 8 Engineering Invariants, Cross-Platform Git Worktree + Windows NTFS Junction Safety, Dual-Version `.d.ts` Tarball Cache (`pacote`), 3-Region Import-Aware AST Slicing, Unified In-Memory `ts-morph` Anti-Cheat Gate, Binding-Aware Tabu State Machine, Atomic Rollback + `--save-partial-patch` Policy, and Formal Exit-Code & Failure-Mode Matrix.
4. [03_SYSTEM_ARCHITECTURE_AND_DESIGN.md](./03_SYSTEM_ARCHITECTURE_AND_DESIGN.md)
   - End-to-End Mermaid Flowchart, `TypeChecker` Multi-File `.d.ts` Diffing Algorithm, 3-Region Slicer, Binding-Aware Alpha-Normalization, Strict Data Contracts (`SymbolDelta`, `ThreeRegionSlice`, `PatchHunk`, `DeltaMorphReport`), Exact LLM Prompt Template & Multi-Hunk Parser Grammar, and Multi-File Atomicity Flow.
5. [04_IMPLEMENTATION_ROADMAP_AND_TECH_STACK.md](./04_IMPLEMENTATION_ROADMAP_AND_TECH_STACK.md)
   - Lean MVP vs. Stretch Scope (100% aligned with RQ1–RQ3), Reproducible Benchmark Manifest Schema (`benchmarks.manifest.json`), 5 Real-World Breaking API Diff Examples, 15 Curated Benchmarks, Exact Experiment Compute/Rate-Limit/API Cost Budget, Statistical Protocol (Wilson CIs, McNemar's test, Cohen's $\kappa$), 6-Month Two-Developer Roadmap, and Appendix A (Pre-Defense PoC & First-5-Repo Capture Template).
6. [05_DEFENSE_LANDMINES_AND_VIVA_GUIDE.md](./05_DEFENSE_LANDMINES_AND_VIVA_GUIDE.md)
   - 10 Implementation Landmines & Verified Solutions (including Partial Multi-File Atomicity and Malformed Hunk Parsing), Honest Industry Tool Comparison (Aider / Claude Code / OpenRewrite vs. DeltaMorph), 12 Examiner Trap Questions with A+ Answers, and Defense Checklist.

---

### Team Responsibilities

| Team Member | Core Ownership |
| :--- | :--- |
| **Sumeet Kanhiya** | Compiler & Program Analysis Core — Dual-version `.d.ts` tarball cache (`pacote`), `ts-morph` `TypeChecker` symbol delta engine, 3-Region AST slicer (imports + scope + local types), unified in-memory `ts-morph` anti-cheat static gate, and binding-aware alpha-normalized Tabu hasher. |
| **Wahaj Khan** | Execution Sandbox, Patch Engine & Evaluation — Git worktree & Windows NTFS junction lifecycle (`SIGINT`-safe), nonce-delimited prompt sanitizer, multi-hunk Search/Replace parser & patcher, atomic rollback / `--save-partial-patch`, Vitest runner (20s timeout), `commander` CLI, and 15-repo benchmark + ablation harness. |

---

### Version 3.1 Specification-Complete Upgrades

| Enhancement | Document(s) Updated |
| :--- | :--- |
| **Strict Data Contracts** (`SymbolDelta`, `ThreeRegionSlice`, `PatchHunk`, `StaticGateResult`, `DeltaMorphReport`) | `03_SYSTEM_ARCHITECTURE_AND_DESIGN.md` |
| **Exact LLM Prompt Template & Multi-Hunk Parser Grammar** (fence stripping, prose rejection, malformed hunk recovery) | `03_SYSTEM_ARCHITECTURE_AND_DESIGN.md`, `05_DEFENSE_LANDMINES_AND_VIVA_GUIDE.md` |
| **Formal Exit-Code & Failure-Mode Recovery Matrix** (Exit codes `0`–`7` & `130`) | `02_SYSTEM_REQUIREMENTS_SPECIFICATION_SRS.md` |
| **Multi-File Atomicity & Opt-In `--save-partial-patch` Policy** | `02_SRS`, `03_Architecture`, `05_Defense` |
| **Reproducible Benchmark Manifest Schema + 5 Concrete Real-World Breaking API Diffs** | `04_IMPLEMENTATION_ROADMAP_AND_TECH_STACK.md` |
| **Exact Experiment Call Math, Rate-Limit Pacing & API Cost Budget (`≤336` calls, `<$1–$2`)** | `SYNOPSIS.md`, `04_Roadmap`, `05_Defense` |
| **Expanded 14-Paper IEEE/ACM Literature Base** (APR, Breaking npm Studies, Program Slicing) | `SYNOPSIS.md`, `01_PROJECT_PROPOSAL.md` |
| **Appendix A: Pre-Defense PoC & First-5-Repo Evidence Capture Protocol** | `01_PROJECT_PROPOSAL.md`, `04_Roadmap`, `05_Defense` |
