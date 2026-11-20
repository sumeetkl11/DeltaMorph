# Final Year Project (FYP) Synopsis

**Department of Computer Science**  
**Dawood University of Engineering & Technology (DUET), Karachi**  

---

### Project Title
**DeltaMorph: Compiler-Verified Breaking-Change Healer via TypeScript Compiler API & AST-Guided Program Repair**

### Student Details
* **Team Members:** Sumeet Kanhiya & Wahaj Khan
* **Degree Program:** B.S. Computer Science (Final Year Capstone)
* **Supervisor:** [Departmental Faculty Supervisor]
* **Document Version:** 3.2 (Empirically Grounded Pre-Defense Edition)

---

### 1. Project Abstract
Modern JavaScript and TypeScript applications depend heavily on rapidly evolving open-source npm packages. Empirical studies of the npm ecosystem show that major dependency upgrades routinely break downstream builds via removed default exports, narrowed parameter contracts, renamed symbols, or altered runtime behavior. Automated dependency bots (e.g., Dependabot, Renovate) only increment version numbers in `package.json`, delegating migration work to human developers. Conversely, general-purpose LLM coding assistants operate on large, uncurated file contexts without structured knowledge of the exact API delta between library versions, leading to high token consumption, hallucinated APIs, silent type-safety regressions (`// @ts-ignore`, `as any`), and cyclic retries.

This project proposes **DeltaMorph**, a **compiler-verified automated program repair (APR)** CLI tool engineered for TypeScript and JavaScript repositories. DeltaMorph separates **deterministic ground-truth analysis and verification** from **stochastic LLM patch synthesis**. It acquires prior and upgraded package declaration trees (`.d.ts`) via a lightweight tarball cache (`pacote`), computes a structured Symbol Delta Table using the TypeScript Compiler API (`ts-morph` `TypeChecker`), extracts a compact **3-Region AST Slice** (relevant `import` declarations, enclosing function scope, and referenced local types), and validates multi-hunk candidate patches inside an isolated `git worktree` using a **Unified In-Memory `ts-morph` Anti-Cheat Gate**, a **Binding-Aware Alpha-Normalized Tabu State Machine**, an immutable `Vitest` runner, and atomic rollback (with optional `--save-partial-patch` export).

---

### 2. Problem Statement & Research Motivation
Upgrading major npm dependencies routinely triggers TypeScript compiler diagnostics and unit-test failures across downstream codebases. Existing solutions exhibit three core bottlenecks:
1. **No Ground-Truth API Delta:** Dependency bots detect version updates but perform zero API remediation, while generic LLM agents must guess or search for breaking changes across opaque library code.
2. **Context Bloat vs. Import Blindness:** Prompting with entire source files wastes tokens and dilutes attention, whereas naively slicing only a failing function body strips out top-of-file `import` declarations and referenced local types required for a valid fix.
3. **Unchecked Synthesis Loops:** Naive LLM retry loops waste executions proposing semantically duplicate patches or silently bypass compiler checks using `// @ts-ignore` or `as any`.

---

### 3. Research Questions & Falsifiable Hypotheses
DeltaMorph evaluates three falsifiable research questions, each mapped directly to a mandatory MVP subsystem:

* **RQ1 (Import-Aware Context Efficiency):** Does **3-Region AST Slicing** (imports header + enclosing scope + referenced local types, target median `<450 tokens`) reduce median prompt token consumption by $\ge 70\%$ compared to full-file prompting while achieving equal or higher `Pass@1`?
* **RQ2 (Ground-Truth `.d.ts` Delta Impact):** On structural (delta-rich) breaking upgrades, does conditioning patch synthesis on a `TypeChecker`-extracted Symbol Delta Table increase `Pass@1` by $\ge 20$ percentage points over stack-trace-only repair (evaluated via McNemar's paired test, $p < 0.05$)?
* **RQ3 (Anti-Oscillation & Anti-Cheat Enforcement):** Does combining a **Binding-Aware Alpha-Normalized Tabu State Machine** with a **Unified In-Memory `ts-morph` Anti-Cheat Gate** prevent $100\%$ of semantically duplicate test executions and reject $100\%$ of newly introduced `@ts-ignore`, `@ts-nocheck`, `@ts-expect-error`, and `as any` compiler bypasses?

---

### 4. Proposed Methodology & System Architecture
DeltaMorph executes a 5-stage closed-loop repair pipeline inside an isolated `git worktree`:

1. **Workspace Isolation & Pre-Flight Triage:** Verifies a clean git tree, spawns `.DeltaMorph-scratch`, and links host `node_modules` instantaneously via **NTFS Directory Junctions** (Windows, no admin required) or POSIX symlinks (Linux/macOS), with non-recursive junction unlinking on `SIGINT`/teardown. Pre-flight detects out-of-scope failures (`ERESOLVE` peer deadlocks, `ERR_REQUIRE_ESM` module-boundary errors) and exits with structured error codes.
2. **Dual-Version `.d.ts` Acquisition & `TypeChecker` Delta Extraction:** Resolves the previous package version from `git show HEAD~1:package-lock.json` (or `--from`) and extracts only `package.json` and `**/*.d.ts` into `~/.DeltaMorph/dts-cache/<pkg>@<ver>` via `pacote` (`<1.5s`, zero install scripts). `ts-morph`'s `TypeChecker` traverses barrel re-exports, overloads, and interfaces to construct a structured `SymbolDeltaTable`. Empty deltas activate **Stack-Trace-Only Fallback Mode** tagged `confidence: low`.
3. **3-Region AST Slicing & Nonce-Delimited Sanitization:** Extracts relevant `import` lines, enclosing call-site scope, and referenced local types (`250–450 tokens`) plus filtered stack traces (`<100 tokens`). Untrusted output is `<`/`>` escaped and wrapped inside per-run cryptographic XML nonces (`<UNTRUSTED_COMPILER_OUTPUT_<nonce>>`).
4. **Multi-Hunk Synthesis & Binding-Aware Tabu Check:** Prompts pinned `gemini-2.5-flash` to emit 1–3 structured `SEARCH/REPLACE` hunks per file, validated with two-tier matching and ambiguity guards. Candidate patches undergo **Binding-Aware Alpha-Normalization** (expanding shorthand destructuring `{ data }` $\rightarrow$ `{ data: _v0 }` to preserve property semantics) and SHA-256 Tabu cycle rejection.
5. **Unified In-Memory `ts-morph` Anti-Cheat Gate, Runtime Verification & Atomicity:** Executes `getPreEmitDiagnostics()`, comment-suppression diff, `AnyKeyword` diff, and `TypeChecker` floating-promise checks in memory (`<300ms`), followed by `Vitest` (ephemeral `/tmp` cache, 20s `AbortController` + process-tree `SIGKILL`). The dynamic cascade queue allows up to 4 files, 3 attempts/file, and 8 global LLM calls, enforcing **all-or-nothing atomic rollback** by default (with opt-in `--save-partial-patch`).

---

### 5. Evaluation Protocol, Benchmarks & Compute Budget
DeltaMorph will be empirically evaluated on **15 reproducible benchmarks** locked in a `benchmarks.manifest.json` registry (GitHub URL, commit SHA, version pair, 5$\times$ flakiness verification):
* **10 Real-World Major Upgrade Benchmarks:** 8 structural (delta-rich: `axios`, `date-fns`, `zod`, `commander`, `uuid`, `helmet`, `got`, `supertest`) and 2 behavioral (delta-empty: `jsonwebtoken`, `pino`).
* **5 Stress, Held-Out & Triage Benchmarks:** Held-out custom/obscure package (`<100k` downloads), cyclic oscillation + destructuring trap, anti-cheat `@ts-ignore`/`as any` trap, multi-file cascade + import rewrite, and pre-flight triage (`ERR_REQUIRE_ESM`, `ERESOLVE`, 20s timeout, flakiness filter).

**Statistical Rigor & Cost Feasibility:**
* `Pass@1` / `Pass@3` split by Delta-Rich vs. Delta-Empty with **Wilson Score 95% CIs** and **McNemar's paired test** ($p < 0.05$) across 3 core ablations (`A0 Full`, `A1 No .d.ts Delta`, `A2 Full-File`).
* Blind 2-reviewer patch quality audit reporting **Cohen's Kappa ($\kappa > 0.6$)**.
* **Transparent Compute Ceiling:** 14 repair benchmarks $\times$ 3 ablation runs $\times$ max 8 LLM calls = **336 hard call ceiling (~126 expected calls)**, totaling `<$1–$2` on `gemini-2.5-flash` with rate-limit pacing.

---

### 6. Team Roles & Six-Month Milestone Timeline
* **Sumeet Kanhiya:** `pacote` `.d.ts` cache & `TypeChecker` differ, 3-Region AST slicer, unified in-memory `ts-morph` anti-cheat gate, binding-aware Tabu hasher.
* **Wahaj Khan:** Git worktree + Windows NTFS junction lifecycle, nonce-delimited sanitizer, multi-hunk parser/patcher, atomic rollback, guarded Vitest runner, CLI, and 15-repo benchmark/ablation harness.

| Timeline | Milestones |
| :--- | :--- |
| **Month 1** | Proposal defense backed by completed **Appendix A empirical proof** (`146 ms` `pacote`, `2,094 ms` `ts-morph` diff, `77.1%` token cut, 5/5 verified fixtures for `BM-01`/`02`/`03`/`05`/`09`); lock remaining 10 repos; complete literature review. |
| **Month 2** | Production `TypeChecker` symbol delta engine, 3-Region AST slicer, cross-platform git worktree/NTFS junction isolation, pre-flight triage. |
| **Month 3** | Multi-hunk parser/patcher, unified in-memory `ts-morph` anti-cheat gate, binding-aware Tabu state machine, guarded 20s Vitest runner, atomic rollback. |
| **Month 4** | End-to-end `commander` CLI integration, nonce sanitization, JSON telemetry report, full MVP validation across all 15 benchmarks. |
| **Month 5** | 5$\times$ flakiness screening, full `A0–A2` ablation study, Wilson CIs, McNemar's test, and blind 2-reviewer patch audit (Cohen's $\kappa$). |
| **Month 6** | Stretch modules (Docker sandbox / Jest / `A3–A4` if ahead of schedule), final thesis report, research paper draft, and live terminal viva defense. |

---

### 7. Core References
1. Weiser, M. (1981). Program slicing. *Proceedings of the 5th International Conference on Software Engineering (ICSE)*, 439–449.
2. Le Goues, C., Nguyen, T., Forrest, S., & Weimer, W. (2012). GenProg: A generic method for automatic software repair. *IEEE Transactions on Software Engineering (TSE)*, 38(1), 54–72.
3. Brito, A., Xavier, L., Hora, A., & Valente, M. T. (2018). Why and how Java developers break APIs. *IEEE SANER 2018*, 255–265.
4. Cogo, F. R., Oliva, G. A., & Hassan, A. E. (2019). An empirical study of dependency downgrades in the npm ecosystem. *IEEE Transactions on Software Engineering (TSE)*, 47(11), 2457–2470.
5. Venturini, D., Cogo, F. R., Polato, I., Gerosa, M. A., & Wiese, I. S. (2023). I depended on you and you broke me: An empirical study of manifesting breaking changes in client packages. *ACM Transactions on Software Engineering and Methodology (TOSEM)*, 32(4).
6. Tidelift. (2023). *State of the Open Source Maintainer Report 2023*.
7. GitHub. (2023). *Octoverse: The State of Open Source Software*.
8. Xia, C. S., & Zhang, L. (2023). Keep the conversation going: Fixing 162 out of 337 bugs for $0.42 each using ChatGPT (*ChatRepair*). *arXiv:2304.00385 / ISSTA 2024*.
9. Jimenez, C. E., Yang, J., Wettig, A., Yao, S., Pei, K., Press, O., & Narasimhan, K. (2024). SWE-bench: Can language models resolve real-world GitHub issues? *ICLR 2024*.
10. Yang, J., et al. (2024). SWE-agent: Agent-computer interfaces enable software engineering language models. *NeurIPS 2024* (`arXiv:2405.15793`).
11. Bouzenia, I., Devanbu, P., & Pradel, M. (2024). RepairAgent: An autonomous, LLM-based agent for program repair. *ICSE 2025* (`arXiv:2403.17134`).
12. Microsoft TypeScript Team & Sherret, D. (2024). *TypeScript Compiler API Architecture* & *ts-morph Documentation*.