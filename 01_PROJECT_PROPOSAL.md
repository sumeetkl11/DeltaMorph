# Final Year Project (FYP) Proposal (Version 3.2 — Empirically Grounded Pre-Defense Edition)
## Project Title: DeltaMorph: Compiler-Verified Breaking-Change Healer via TypeScript Compiler API & AST-Guided Program Repair

**Department:** Computer Science  
**Institution:** Dawood University of Engineering & Technology (DUET), Karachi  
**Project Team:** Sumeet Kanhiya & Wahaj Khan  
**Supervisor:** [Departmental Faculty Supervisor]  
**Academic Period:** Final Year Capstone (6 Months)  

---

### 1. Executive Summary
Modern software engineering depends heavily on third-party open-source packages. Empirical analyses of the npm ecosystem demonstrate that breaking major updates frequently manifest as compile-time signature mismatches, removed default exports, narrowed generic contracts, or runtime behavioral regressions in client repositories. Dependency bots such as GitHub Dependabot and Renovate merely bump version strings in `package.json`, breaking CI builds and pushing repetitive refactoring onto developers.

**DeltaMorph** is a **compiler-verified automated program repair (APR)** CLI engineered for TypeScript and JavaScript repositories. We explicitly separate **deterministic analysis and verification** from **stochastic LLM patch synthesis**:
1. **Deterministic Dual-Version `.d.ts` Extraction:** Fetches prior and upgraded declaration trees into a lightweight local cache (`pacote`, extracting only `package.json` and `**/*.d.ts` without executing install scripts) and diffs exported symbol graphs via `ts-morph`'s `TypeChecker`.
2. **Import-Aware 3-Region Context Slicing:** Bounds LLM context to `<450 tokens` while preserving top-of-file `import` declarations, the enclosing function scope, and referenced local type definitions.
3. **Unified In-Memory Static Gate, Binding-Aware Tabu Search & Atomic Rollback:** Verifies candidate patches inside `ts-morph` in `<300ms` (blocking `@ts-ignore`, `@ts-nocheck`, `@ts-expect-error`, newly added `any`, and floating promises), rejects semantically duplicate patches via binding-aware alpha-normalization, validates behavior inside an isolated `git worktree` running `Vitest`, and enforces all-or-nothing atomic rollback (with opt-in `--save-partial-patch`).

---

### 2. Falsifiable Research Questions & Hypotheses

Every research question maps directly to a mandatory **MVP** subsystem:

**RQ1 (Context Efficiency & Import-Aware Slicing):** Does 3-Region AST Slicing (imports header + enclosing scope + referenced local types) reduce prompt token overhead without sacrificing repair accuracy compared to full-file prompting?
* **H1:** 3-Region AST-sliced prompts (target median `<450 tokens`) reduce median prompt tokens by **$\ge 70\%$** relative to full-file prompting while achieving **equal or higher `Pass@1`** across our reproducible benchmark suite.
* **Metrics:** Median prompt tokens, `Pass@1`, `Pass@3` (Wilson Score 95% CIs), and McNemar's paired test ($p < 0.05$).

**RQ2 (Ground-Truth `.d.ts` Delta Impact):** On structural (delta-rich) breaking upgrades, does conditioning LLM synthesis on a `TypeChecker`-extracted Symbol Delta Table outperform stack-trace-only repair?
* **H2:** On the delta-rich subset of benchmarks, `.d.ts`-augmented prompts achieve **$\ge 20$ percentage points higher `Pass@1`** than stack-trace-only prompts (Ablation `A0` vs. `A1`).
* **Metrics:** Subset `Pass@1` / `Pass@3` delta and McNemar's paired test ($p < 0.05$).

**RQ3 (Anti-Oscillation & Anti-Cheat Gate Effectiveness):** Does combining a Binding-Aware Alpha-Normalized Tabu State Machine with a Unified In-Memory `ts-morph` Anti-Cheat Gate eliminate redundant test runs and compiler-bypass patches?
* **H3:** The Tabu state machine blocks **$100\%$ of semantically equivalent repeat patches** prior to test execution, and the `ts-morph` anti-cheat gate rejects **$100\%$ of newly introduced `@ts-ignore`, `@ts-nocheck`, `@ts-expect-error`, and `as any` bypasses**.
* **Metrics:** Duplicate patch rejection count, anti-cheat rejection rate, mean iterations-to-fix, and blind human patch audit (Cohen's $\kappa > 0.6$).

---

### 3. Literature Review & Positioning (14-Paper Foundation)

DeltaMorph synthesizes insights across three established research pillars:

#### Pillar A: Empirical Studies of Breaking Dependency Changes
* **Cogo et al. (2019, *IEEE TSE*)** and **Venturini et al. (2023, *ACM TOSEM*)** empirically documented how breaking changes in npm packages cascade into client applications, showing that breaking updates frequently stem from exported symbol removals, changed signatures, and behavioral shifts that semver checks alone cannot remediate.
* **Brito et al. (2018, *IEEE SANER*)** and **Mujtaba et al. (2022, *EMSE*)** showed that migration documentation and changelogs are frequently incomplete or inconsistent—motivating DeltaMorph's extraction of ground-truth deltas directly from compiled TypeScript `.d.ts` declarations rather than relying on prose changelogs.
* **Tidelift (2023)** and **GitHub Octoverse (2023)** highlight ongoing developer maintenance pressure caused by high dependency churn in modern JavaScript/TypeScript repositories.

#### Pillar B: Automated Program Repair & LLM Coding Agents
* **GenProg (Le Goues et al., 2012, *IEEE TSE*)** pioneered test-suite-guided APR via genetic search, but search-based mutation cannot synthesize calls to newly renamed external library APIs.
* **ChatRepair (Xia & Zhang, 2023/2024)** and **RepairAgent (Bouzenia et al., 2024/2025, *ICSE*)** demonstrated that iterative test-feedback loops with LLMs improve repair rates, yet neither extracts dual-version library `.d.ts` API deltas or enforces a TypeScript anti-cheat compiler gate.
* **SWE-bench (Jimenez et al., 2024, *ICLR*)** and **SWE-agent (Yang et al., 2024, *NeurIPS*)** established autonomous repository-level issue resolution, while commercial CLIs (**Aider**, **Claude Code**) execute general multi-file edit-and-test loops—at high token volume and without specialized breaking-change `.d.ts` diffing or binding-aware Tabu deduplication.

#### Pillar C: Static Program Slicing & Rule-Based Codemods
* **Weiser (1981, *ICSE*)** formalized static program slicing to isolate code statements relevant to a failure criterion. DeltaMorph adapts this principle for LLM repair via **3-Region Import-Aware AST Slicing** using the **TypeScript Compiler API (`ts-morph`)**.
* Rule-based AST migration frameworks (**OpenRewrite**, **jscodeshift**, **GritQL**) execute deterministic rewrites with high precision, but require maintainers to hand-author version-specific migration recipes—leaving long-tail and undocumented breaking upgrades uncovered.

*Empirical Discipline:* All quantitative comparisons against baselines (`Full-File LLM` `A2`, `Stack-Trace-Only Sliced` `A1`) are **empirically measured on our own 15-repository benchmark harness** in Month 5; no unverified competitor pass rates are asserted prior to experimentation.

---

### 4. Qualitative Capability Matrix vs. Existing Tools

| Capability | Dependabot / Renovate | General Agentic CLIs (Aider / Claude Code) | Rule-Based Codemods (jscodeshift / GritQL) | **DeltaMorph (Ours)** |
| :--- | :--- | :--- | :--- | :--- |
| **Breaking-Change Code Repair** | ❌ Version bump only | ⚠️ Prompt/context dependent | ✅ Only if recipe exists | ✅ Compiler-verified closed loop |
| **Dual-Version `.d.ts` Delta** | ❌ None | ❌ Manual file inspection | ❌ Hand-coded rules | ✅ Automated `pacote` + `TypeChecker` diff |
| **Context Bounding** | N/A | Full files / repo map | Rule AST match | ✅ 3-Region slice (imports + scope + types) |
| **Anti-Cheat Type Safety Gate** | N/A | ❌ Can emit `as any` / `@ts-ignore` | ✅ Rule-constrained | ✅ Unified in-memory `ts-morph` gate |
| **Anti-Oscillation Guard** | ❌ | ❌ Retry limit only | N/A | ✅ Binding-aware Alpha-Normalized Tabu |
| **Working-Tree Safety** | PR-based | Edits active working tree | Edits active working tree | ✅ Detached `git worktree` + atomic rollback |

---

### 5. Bounded Project Scope & Explicit Threat Model

* **In Scope (MVP — Strict Depth):**
  - **Language:** TypeScript & JavaScript (Node.js v20+ LTS). JS-only repos supported via synthetic `tsconfig.json` (`allowJs: true`, `checkJs: true`).
  - **Compiler & AST Engine:** Unified 100% on `ts-morph` (TypeScript Compiler API) for `.d.ts` diffing, 3-Region slicing, in-memory static anti-cheat gating, and binding-aware Tabu canonicalization.
  - **Test Runner:** `Vitest` (MVP) with redirected ephemeral `/tmp` cache and 20s hard process-tree timeout.
  - **Cascade Scope:** Up to 4 affected source files per upgrade (`≤3` attempts/file, `≤8` global LLM calls) with all-or-nothing atomic rollback by default (`--save-partial-patch` opt-in).
  - **Workspace Isolation:** Local Mode detached `git worktree` with Windows NTFS junctions / POSIX symlinks.
* **Explicitly Out of Scope:**
  - Multi-paradigm framework migrations (Angular $\rightarrow$ React, Vue 2 Options $\rightarrow$ Vue 3 Composition).
  - Database schema migrations and ORM data backfills.
  - Full CommonJS $\leftrightarrow$ ESM module-system rewrites (`ERR_REQUIRE_ESM` is detected at pre-flight and escalated cleanly as `UNSUPPORTED_MODULE_BOUNDARY`).
* **Honest Threat Model (Local Mode vs. Stretch Docker Mode):**
  - **Local Mode (MVP)** guarantees **git working-tree cleanliness** and prevents accidental edits to user files; it is **not** an OS-level security sandbox against malicious npm packages or network egress. Forwarding host `.env` files is strictly opt-in (`--forward-env`).
  - **Docker Mode (Stretch)** provides true OS-level isolation (`--network=none`, `:ro` test mounts, ephemeral container filesystem) for untrusted CI environments.

---

### 6. Threats to Validity

#### Internal Validity
* **Benchmark Selection Bias / Pre-Training Memorization:** Popular packages may exist in LLM training data. *Mitigation:* Split reporting into structural (delta-rich) vs. behavioral (delta-empty) subsets, and include a held-out custom/obscure package (`<100k` downloads) absent from pre-training corpora.
* **Flaky Tests:** Non-deterministic tests could distort `Pass@k`. *Mitigation:* Run every benchmark baseline 5$\times$ prior to evaluation; quarantine flaky suites before recording results.
* **LLM Stochasticity:** Hosted LLM APIs are not bit-identical even at `temperature = 0`. *Mitigation:* Pin exact model version (`gemini-2.5-flash`), log full prompts, responses, token counts, and timestamps as reproducible JSON artifacts.

#### External Validity
* **Ecosystem Scope:** Findings apply to TypeScript/JavaScript (`npm` + `Vitest`) projects; Python, Java, and Go are deferred to future work.
* **Cascade Ceiling:** Upgrades breaking $>4$ interdependent files are intentionally escalated rather than partially patched; findings apply to localized-to-moderate API migrations.

#### Construct Validity
* **Test-Suite Overfitting (`Pass@k` $\neq$ Semantic Correctness):** Passing unit tests does not guarantee idiomatic, bug-free fixes. *Mitigation:* Enforce the in-memory `ts-morph` anti-cheat gate and conduct a blind manual review of every passing patch by two independent reviewers, reporting **Cohen's Kappa ($\kappa$)**.

---

### 7. Team Role Split (Two-Developer Plan)

* **Sumeet Kanhiya (Compiler & Program Analysis Lead):**
  - `pacote` dual-version `.d.ts` tarball fetcher & cache manager
  - `ts-morph` `TypeChecker` symbol delta engine (barrel re-exports, overloads, interface diffs)
  - 3-Region AST slicer (imports + enclosing scope + referenced local types)
  - Unified in-memory `ts-morph` anti-cheat static gate & binding-aware alpha-normalized Tabu hasher
* **Wahaj Khan (Sandbox, Patching, CLI & Evaluation Lead):**
  - Cross-platform `git worktree` + Windows NTFS junction / POSIX symlink lifecycle & `SIGINT` cleanup
  - Nonce-delimited prompt sanitization pipeline & `@google/genai` LLM orchestrator
  - Multi-hunk two-tier Search/Replace parser/patcher, dynamic cascade queue & atomic rollback (`--save-partial-patch`)
  - Guarded `Vitest` runner (20s timeout), `commander` CLI, and 15-repo benchmark + ablation harness

---

### 8. Verified Pre-Defense Empirical Proof (Summary of Appendix A)
To prove technical feasibility prior to our DUET proposal defense, Sumeet and Wahaj executed and verified all three pre-defense proof artifacts (detailed verbatim in **Appendix A of `04_IMPLEMENTATION_ROADMAP_AND_TECH_STACK.md`**):
1. **Verified Standalone `pacote` + `ts-morph` PoC (`axios@0.27.2 → 1.7.7`):** Measured `146 ms` cold / `0.29 ms` warm `.d.ts`-only extraction (2 files, zero scripts), `2,094 ms` `ts-morph` `TypeChecker` symbol graph diffing (`63` total / `15` focused interface deltas via `getDeclaredTypeOfSymbol()`), **`77.1%` token reduction** via 3-Region AST slicing (`177` vs. `773` tokens in `2.23 ms`), **`2.19 ms` warm** unified anti-cheat gate blocking 3/3 injected cheats (`@ts-ignore`, `as any`, floating promise), and verified binding-aware destructuring Tabu deduplication (`{ data }` $\equiv$ `{ data: body }` $\neq$ `{ status }`).
2. **First 5 Verified Reproducible Benchmark Fixtures:** Locked pre-upgrade passing SHAs and post-upgrade broken SHAs across `BM-01` (`axios`), `BM-02` (`date-fns`), `BM-03` (`zod 3.23.8→4.6.7`), `BM-05` (`uuid`), and `BM-09` (`jsonwebtoken`), each verified **100% passing pre-upgrade** and **5/5 deterministic failures post-upgrade**.
3. **Verified Experiment Cost & Quota Math:** Complete Month 5 `A0–A2` call ceiling (`≤336` calls, `~126` expected), `<0.6M` worst-case evaluation tokens, and `<$1.50 USD` total budget on `gemini-2.5-flash`.

---

### 9. Faculty Defense Pitch (DUET Evaluation Committee)
> *"DeltaMorph is an Applied Software Engineering and Program Analysis system for TypeScript dependency upgrades. Rather than relying on full-file LLM guessing or fragile changelog scraping, we extract exact API deltas from old and new `.d.ts` declaration trees via a lightweight tarball cache and the TypeScript `TypeChecker`. We assemble an import-aware 3-Region AST slice under 450 tokens, eliminate compiler-bypass cheating inside a unified in-memory `ts-morph` gate, block cyclic retries with a binding-aware Tabu state machine, and guarantee atomic rollback inside an isolated git worktree. Every claim is evaluated empirically against full-file and stack-trace-only baselines on 15 reproducible benchmarks using Wilson confidence intervals, McNemar's test, and Cohen's Kappa patch audits."*

---

### 10. References
1. Weiser, M. (1981). Program slicing. *Proc. 5th ICSE*, 439–449.
2. Le Goues, C., Nguyen, T., Forrest, S., & Weimer, W. (2012). GenProg: A generic method for automatic software repair. *IEEE TSE*, 38(1), 54–72.
3. Brito, A., Xavier, L., Hora, A., & Valente, M. T. (2018). Why and how Java developers break APIs. *IEEE SANER*, 255–265.
4. Cogo, F. R., Oliva, G. A., & Hassan, A. E. (2019). An empirical study of dependency downgrades in the npm ecosystem. *IEEE TSE*, 47(11), 2457–2470.
5. Mujtaba, H., et al. (2022). Understanding breaking changes and migration documentation in JavaScript libraries. *Empirical Software Engineering (EMSE)*.
6. Venturini, D., Cogo, F. R., Polato, I., Gerosa, M. A., & Wiese, I. S. (2023). I depended on you and you broke me: An empirical study of manifesting breaking changes in client packages. *ACM TOSEM*, 32(4).
7. Tidelift. (2023). *State of the Open Source Maintainer Report 2023*.
8. GitHub. (2023). *Octoverse: The State of Open Source Software*.
9. Xia, C. S., & Zhang, L. (2023). Automated program repair via conversation (*ChatRepair*). *ISSTA 2024* (`arXiv:2304.00385`).
10. Jimenez, C. E., et al. (2024). SWE-bench: Can language models resolve real-world GitHub issues? *ICLR 2024*.
11. Yang, J., et al. (2024). SWE-agent: Agent-computer interfaces enable software engineering language models. *NeurIPS 2024* (`arXiv:2405.15793`).
12. Bouzenia, I., Devanbu, P., & Pradel, M. (2024). RepairAgent: An autonomous, LLM-based agent for program repair. *ICSE 2025* (`arXiv:2403.17134`).
13. Microsoft TypeScript Team. (2024). *TypeScript Compiler API & TypeChecker Architecture*.
14. Sherret, D. (2024). *ts-morph: TypeScript Compiler API Wrapper*.
