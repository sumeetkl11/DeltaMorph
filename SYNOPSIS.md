# Final Year Project (FYP) Synopsis

**Department of Computer Science**  
**Dawood University of Engineering & Technology (DUET), Karachi**  

---

### Project Title
**DeltaMorph: Autonomous Breaking-Change Healer**

### Student Details
* **Group Members:** Sumeet Kanhiya ([@sumeetkl11](https://github.com/sumeetkl11)) & Wahaj Khan
* **Degree Program:** B.S. Computer Science (Final Year)
* **Supervisor:** [Departmental Faculty Supervisor]

---

### 1. Project Abstract
Modern software systems spend up to 40% of their maintenance effort adapting application code to breaking changes introduced by third-party dependency updates. Existing automated tools such as Dependabot and Renovate merely bump version strings in configuration files, delegating the tedious, error-prone remediation of broken APIs, altered method signatures, and modified return types to human developers. Conversely, general-purpose LLM code assistants (e.g., GitHub Copilot, Cursor) feed entire files into large context windows without semantic ground truth, frequently hallucinating incompatible syntax or introducing subtle type-checking regressions.

This project proposes **DeltaMorph**, an autonomous, compiler-verified program repair tool specifically engineered for TypeScript and JavaScript ecosystems. DeltaMorph extracts mathematical deltas between library version declarations (`.d.ts`) using the TypeScript Compiler API (`ts-morph`), isolates call-site scopes to compact AST slices (200–350 tokens), and synthesizes candidate patches verified through a closed-loop dual-gate (static `tsc --noEmit` and runtime unit testing). By integrating an Alpha-Normalized Tabu State Machine to eliminate repair oscillation, safe Git Worktree scratchpads, and strict anti-cheat static linting, DeltaMorph achieves production-grade automated migration without risking working tree corruption.

---

### 2. Problem Statement & Research Motivation
When package maintainers release major semantic versions (e.g., Axios v0.27 $\rightarrow$ v1.6, Zod v2 $\rightarrow$ v3), downstream client applications encounter compiler diagnostic errors and test assertion failures. 
The current state of the art suffers from three fundamental bottlenecks:
1. **Lack of Ground-Truth Extraction:** Automated dependency bots identify *that* an update is available, but provide zero automated remediation for API incompatibilities.
2. **Context Window Inefficiency & Hallucination:** LLM-based assistants process complete multi-thousand-token files, leading to context dilution, high latency (>60s), and hallucinated API methods.
3. **Semantic Oscillation & Type Cheating:** Existing repair loops frequently cycle between two incorrect patches or introduce `// @ts-ignore` comments to bypass compiler gates.

---

### 3. Research Objectives & Falsifiable Hypotheses
The primary objective of DeltaMorph is to develop and evaluate a lightweight, compiler-grounded automated repair pipeline for breaking dependency upgrades. 

* **Hypothesis 1 (Context Efficiency):** Slicing source files to enclosing AST scopes via `ts-morph` reduces repair prompt token consumption by $\ge 75\%$ compared to full-file baselines while maintaining equivalent or superior patch accuracy.
* **Hypothesis 2 (Ground-Truth Superiority):** Conditioning LLM patch generation on structured `.d.ts` Symbol Delta Tables increases first-attempt repair success (Pass@1) by $\ge 25$ percentage points over stack-trace-only repair models.
* **Hypothesis 3 (Oscillation Elimination):** Alpha-Normalized Tabu hashing reduces cyclic patch generation from an empirical baseline of $18\%$ down to $0\%$.

---

### 4. Proposed Methodology & System Architecture

DeltaMorph executes a 5-stage pipeline:
1. **Workspace Isolation:** Spawns a detached working copy via `git worktree add --detach .deltamorph-scratch HEAD` and links dependencies using NTFS Directory Junctions (Windows) or symlinks (Linux/macOS), ensuring zero pollution of the developer's working directory.
2. **Deterministic Delta Extraction:** Compares prior and upgraded `.d.ts` declaration trees using `ts-morph` to extract added, removed, or modified symbols into a Symbol Delta Table.
3. **Surgical Context Slicing:** Locates broken call-sites from compiler diagnostics and extracts the enclosing function scope (200–350 tokens) and filtered runtime traces (<100 tokens).
4. **Prompt Sanitization & Patch Synthesis:** Wraps sanitized diagnostic data inside XML boundaries (`<UNTRUSTED_COMPILER_OUTPUT>`) and prompts a fast LLM (Gemini 1.5 Flash / Groq Llama-3-70B) to generate a Search/Replace patch block.
5. **Closed-Loop Verification & Tabu State Machine:** Applies patch to AST, verifies against an Anti-Cheat Lint Gate (`@typescript-eslint/ban-ts-comment`), and tests via `tsc --noEmit` and Vitest/Jest. If testing fails, the Alpha-Normalized AST hash is logged to the Tabu state set to prevent identical semantic proposals.

---

### 5. Evaluation Protocol & Curated Benchmarks
DeltaMorph will be empirically evaluated against a benchmark of **25 open-source repositories**:
* 15 production repositories exhibiting real-world migrations across major packages (`axios`, `date-fns`, `zod`, `chalk`, `commander`, `jsonwebtoken`).
* 5 synthetic stress-test repositories evaluating circular dependencies, multi-file renames, and behavioral breaks.
* 5 held-out obscure npm packages (<100k downloads) to guarantee zero training-data memorization bias.

**Quantitative Metrics:**
* Pass@1 and Pass@3 repair rates with Wilson Score 95% Confidence Intervals.
* Statistical significance via McNemar's paired binary test ($p < 0.05$).
* Token consumption and end-to-end latency per migration.
* Qualitative review of patch idiomaticity using Cohen's Kappa inter-rater agreement ($\kappa > 0.6$).

---

### 6. Project Timeline & Semester Milestones
* **Month 1:** Literature survey, formal defense presentation, setup of 25 benchmark repositories.
* **Month 2:** Development of `ts-morph` declaration differ and AST slicing engine.
* **Month 3:** Implementation of Git Worktree isolation, Windows NTFS junction handling, and test runner sandbox.
* **Month 4:** Integration of LLM synthesis, Anti-Cheat Static Lint Gate, and Alpha-Normalized Tabu State Machine.
* **Month 5:** Comprehensive ablation studies, baseline comparisons, and statistical evaluation.
* **Month 6:** Final thesis write-up, research paper draft submission, and final departmental viva defense.

---

### 7. Core References
1. Tidelift. (2023). *State of the Open Source Maintainer Report 2023*.
2. GitHub. (2023). *Octoverse: The State of Open Source Software*.
3. Xia, C. S., & Zhang, L. (2023). Keep the conversation going: Fixmind automated program repair with LLMs. *ACM FSE 2023*.
4. Yang, J., et al. (2024). SWE-agent: Agent-computer interfaces enable software engineering language models. *arXiv:2405.15793*.
5. He, J., et al. (2023). ChatRepair: Automated program repair via conversation with large language models. *IEEE ICSE 2023*.
6. TypeScript Compiler API Documentation. (2024). *Microsoft TypeScript Team*.
7. David, D. (2024). *ts-morph: TypeScript Compiler API Wrapper*.