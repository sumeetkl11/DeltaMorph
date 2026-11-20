# DeltaMorph: Autonomous Breaking-Change Healer

**Project Title:** DeltaMorph: Autonomous Breaking-Change Healer  
**Institution:** Dawood University of Engineering and Technology (DUET)  
**Department:** Computer Science  
**Developers:** Sumeet Kanhiya ([@sumeetkl11](https://github.com/sumeetkl11)) & Wahaj Khan  
**Documentation Version:** 2.3 (Production & Defense Ready — All Operational Landmines Addressed)  

---

### Project Documentation Index

1. [SYNOPSIS.md](file:///D:/codes/FYP/SYNOPSIS.md)
   - Official 2-page academic synopsis tailored for DUET CS Departmental Committee approval (Abstract, Problem Statement, Hypotheses, Methodology, Benchmark Summary, Timeline, References).
2. [01_PROJECT_PROPOSAL.md](file:///D:/codes/FYP/01_PROJECT_PROPOSAL.md)
   - Falsifiable Research Questions (H1, H2, H3), Literature Context with cited sources (Tidelift 2023, GitHub Octoverse 2023, Byam 2024, ChatRepair 2023), Competitor Comparison, Bounded Scope, Threats to Validity, and Faculty Defense Pitch.
3. [02_SYSTEM_REQUIREMENTS_SPECIFICATION_SRS.md](file:///D:/codes/FYP/02_SYSTEM_REQUIREMENTS_SPECIFICATION_SRS.md)
   - 8 Architectural Invariants, NTFS Directory Junctions for Windows, Multi-File Cascade Budget, Anti-Cheat Static Lint Gate (`@typescript-eslint/ban-ts-comment`), 15s subprocess timeout, and latency budgets.
4. [03_SYSTEM_ARCHITECTURE_AND_DESIGN.md](file:///D:/codes/FYP/03_SYSTEM_ARCHITECTURE_AND_DESIGN.md)
   - Complete Mermaid flowchart (35+ nodes), `.d.ts` diffing algorithm pseudocode, Alpha-Normalized Tabu State Machine, Prompt Sanitization pipeline, and Rich Terminal UX.
5. [04_IMPLEMENTATION_ROADMAP_AND_TECH_STACK.md](file:///D:/codes/FYP/04_IMPLEMENTATION_ROADMAP_AND_TECH_STACK.md)
   - 25 Benchmarks (15 production + 10 edge/held-out cases), Wilson score 95% CIs + Cohen's κ framework, 5-run ablation study design, baseline comparison matrix, MVP vs Stretch scope, and 6-month milestones.
6. [05_DEFENSE_LANDMINES_AND_VIVA_GUIDE.md](file:///D:/codes/FYP/05_DEFENSE_LANDMINES_AND_VIVA_GUIDE.md)
   - Detailed breakdown of all 6 technical landmines & solutions, Tropos vs DeltaMorph comparison matrix, and Top 10 Viva Interrogation Questions with word-for-word model answers.

---

### Version 2.3 Architectural Highlights

| Round 3 Gap Identified | Solution Implemented in DeltaMorph |
| :--- | :--- |
| Windows Git Worktree `node_modules` crash | NTFS Directory Junctions (`mklink /J` / `'junction'`) for 0ms non-admin linking |
| Multi-file cascade exhaustion | Two-tier repair loop (outer file queue + inner per-file retry budget) |
| LLM `@ts-ignore` & `any` shortcut cheats | Anti-Cheat Static Lint Gate via `@typescript-eslint/ban-ts-comment` |
| Subprocess hangs on promises/sockets | 15-second hard process timeout with atomic `SIGKILL` |
| Empty `.d.ts` delta on behavioral breaks | Formal fallback pipeline: stack-trace-only mode with `confidence: low` flagging |
| `ts-morph` fails on JS-only repos | Synthetic `tsconfig.json` auto-generation (`allowJs: true`, `checkJs: true`) |
| `.d.ts` lookup via `exports` map | Resolution chain: `exports` → `typesVersions` → `types` → `index.d.ts` |
| `.env` drift in git worktrees | Symlink forwarding of `.env*` files + `process.env` inheritance |
| Adversarial content in error messages | Prompt sanitization: regex stripping + XML delimiters + system prompt guard |
| `SIGINT` leaves stale `index.lock` | Trap handler: delete lock → force remove worktree → `process.exit(1)` |
| No inter-rater agreement metric | Cohen's Kappa (κ) with target $\kappa > 0.6$ |
| Flat Pass@k without confidence bounds | Wilson score 95% CIs + McNemar's paired test |
| No ablation study | 5-run component-disabled evaluation design |
| `ts-morph` cold-start unaccounted | Explicit latency budget: ≤5s initialization in total ≤40s local budget |
| Vague research hypotheses | 3 falsifiable RQs with specific numeric targets |
| No cited maintenance burden source | Tidelift 2023 Survey + GitHub Octoverse 2023 + Byam 2024 |
| No terminal UX design | Full CLI output mockup with error recovery messages (`deltamorph run`) |
| No `.d.ts` diffing algorithm | Complete pseudocode with Levenshtein rename detection |
| Search/Replace fragility | Two-tier matching (exact → whitespace-normalized) + multi-occurrence guard |
| No threats to validity | Internal, External, and Construct validity sections |
