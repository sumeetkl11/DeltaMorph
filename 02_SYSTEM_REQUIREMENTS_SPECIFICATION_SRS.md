# Software Requirements Specification (SRS)
## System Name: DeltaMorph (Autonomous Breaking-Change Healer)

**Document Version:** 2.3 (Production Hardened — All Operational Landmines Resolved)  
**Architects:** Sumeet Kanhiya ([@sumeetkl11](https://github.com/sumeetkl11)) & Wahaj Khan  
**Academic Affiliation:** Dawood University of Engineering & Technology (DUET)  
**Target Environment:** Linux / macOS / Windows Developer Workstations & GitHub Actions  

---

### 1. Architectural Invariants
DeltaMorph operates under eight immutable engineering invariants:
1. **Unified AST Foundation:** All parsing, slicing, type-checking, and code transformation use `ts-morph` exclusively.
2. **Compiler-Verified Ground-Truth Extraction:** Breaking changes extracted via `.d.ts` AST diffing with formal fallback for behavioral breaks. (No claims of end-to-end "determinism" — synthesis is stochastic, verification is 100% deterministic).
3. **Safe Sandbox Separation:** Test assertions are immutable; runner caches dynamically redirected to ephemeral `/tmp`.
4. **Zero Dirty-Tree Pollution & Windows Safety:** Mutations execute inside a detached `git worktree` using NTFS Directory Junctions for Windows compatibility.
5. **Alpha-Normalized Anti-Oscillation:** Tabu search canonicalizes local identifiers before hashing.
6. **Prompt Sanitization:** All compiler/runtime output is sanitized before LLM prompt assembly to prevent indirect prompt injection.
7. **Strict Anti-Cheat Lint Gate:** Patches with `@ts-ignore`, `@ts-nocheck`, or unjustified `any` are rejected by static analysis before testing.
8. **Hard Subprocess Guardrails:** All test runner executions are bound by an atomic 15-second process timeout with `SIGKILL` cleanup.

---

### 2. Functional Requirements (FR)

#### FR-1: Environment & Workspace Isolation
* **FR-1.1 (Git Worktree Scratchpad):** On invocation, verify repository is clean (`git diff --quiet`). Then spawn a detached working copy via `git worktree add --detach .deltamorph-scratch HEAD`. All file edits, test executions, and git operations execute inside this sandbox.
* **FR-1.2 (Windows & Linux Cross-Platform Linkage):** 
  - On Linux/macOS: Link `node_modules` and config files using standard filesystem symlinks (`ln -s`).
  - On Windows: Do NOT use symlinks (requires elevated admin privileges). Instead, dynamically create **NTFS Directory Junctions** via Node.js:
    ```javascript
    fs.symlinkSync(sourceNodeModules, targetNodeModules, 'junction');
    ```
    This avoids `EPERM: operation not permitted` on Windows non-admin developer workstations.
* **FR-1.3 (Clean Teardown with SIGINT Safety):** On successful repair, merge the scratch commit cleanly. On failure, `SIGINT`, or `Ctrl+C`:
  1. Delete any stale `index.lock` file at `.git/worktrees/.deltamorph-scratch/index.lock`.
  2. Remove junction/symlinks cleanly to avoid deleting original `node_modules`.
  3. Execute `git worktree remove --force .deltamorph-scratch`.
  4. Call `process.exit(1)`.
* **FR-1.4 (Environment & Config Forwarding):** Before test execution, forward `process.env` and junction `.env` files to prevent `ECONNREFUSED` during integration tests.

#### FR-2: Unified `ts-morph` AST & Type Delta Engine
* **FR-2.1 (Declaration Entry Point Resolution):** Resolve `.d.ts` paths via the following priority chain:
  1. `package.json` → `"exports"` conditional map (types subpath).
  2. `package.json` → `"typesVersions"` field.
  3. `package.json` → `"types"` or `"typings"` field.
  4. Fallback: `node_modules/<pkg>/index.d.ts` or `@types/<pkg>`.
* **FR-2.2 (Project Configuration & Alias Inheritance):** When initializing `ts-morph` `Project`:
  - Dynamically read the repository's `tsconfig.json`.
  - Preserve path aliases (`compilerOptions.paths`, e.g., `@/*` mappings).
  - Explicitly set `compilerOptions.jsx = ts.JsxEmit.Preserve` to prevent syntax parse failures in React/TSX files.
* **FR-2.3 (Structural Delta Extraction):** Using `ts-morph`, compute the Symbol Delta Table:
  - Exported identifiers added, removed, or renamed.
  - Parameter list alterations (added required params, reordered arguments).
  - Return type inversions (synchronous value $\rightarrow$ `Promise<T>`).
* **FR-2.4 (Behavioral Break Fallback Pipeline):** When the Symbol Delta Table is **empty** (behavioral breaking change with identical type signatures):
  1. Classify the break as a "Delta-Empty Behavioral Change."
  2. Switch to **Stack-Trace-Only Repair Mode:** extract context purely from filtered stack trace + test assertion failure message.
  3. Query the library's semver release tag diff for behavioral change hints.
  4. Flag the repair as `confidence: low` in the output report.
* **FR-2.5 (JS-Only Repository Support):** If no `tsconfig.json` exists in the project root, auto-generate a minimal synthetic config (`allowJs: true`, `checkJs: true`, `noEmit: true`).

#### FR-3: Surgical Context Slicing & Token Budgeting
* **FR-3.1:** Using `ts-morph` ancestor navigation, extract the enclosing function/method declaration and immediate caller call-sites (up to 2 hops).
* **FR-3.2 (Context Budget Guard):** Code slice strictly bounded between 200 and 400 tokens (vs 2,500+ tokens for full-file dumps).
* **FR-3.3 (Stack Frame Filtering):** Strip internal runner boilerplate (`node_modules/vitest`, `node:internal/*`). Retain only top application frame and root assertion message. Total error payload: **<100 tokens**.

#### FR-4: Prompt Sanitization & Security
* **FR-4.1 (Adversarial Content Stripping):** Before injecting compiler/runtime output into LLM prompt:
  1. Strip non-printable and control characters.
  2. Truncate lines to 200 characters.
  3. Strip injection patterns (`/(ignore|disregard).*(previous|above)/i`).
  4. Wrap in `<UNTRUSTED_COMPILER_OUTPUT>` boundaries.

#### FR-5: Structured Patching & Strict Anti-Cheat Lint Gate
* **FR-5.1 (Search/Replace Block Syntax):** LLM outputs replacement blocks rather than unified diffs.
* **FR-5.2 (Two-Tier Matching Strategy):** Tier 1: exact match. Tier 2: whitespace-normalized match. Multi-match rejected with `AMBIGUOUS_REPLACE_TARGET`.
* **FR-5.3 (Anti-Cheat Static Lint Gate):** Before running any test runner, the patched file is analyzed with ESLint AST rules:
  - `@typescript-eslint/ban-ts-comment`: Prohibits `// @ts-ignore` and `// @ts-nocheck`.
  - `@typescript-eslint/no-explicit-any`: Prohibits `any` casting unless explicitly present in original AST.
  - `@typescript-eslint/no-floating-promises`: Requires all async calls to be properly handled.
  If any anti-cheat rule triggers, the patch is rejected immediately with penalty score without invoking the test suite.

#### FR-6: Immutable Test Suites & Cache Redirection
* **FR-6.1 (Test Code Immutability):** Test files (`*.test.ts`, `*.spec.ts`) are marked read-only. Modifying tests is disallowed.
* **FR-6.2 (Cache Redirection):** Test runner flags inject cache isolation (`--cacheDirectory=/tmp/...`).
* **FR-6.3 (Subprocess Execution Guard):** Test runner subprocess spawned with 15-second hard timeout via `AbortController`. If process does not exit in 15s, fire `SIGKILL` to prevent hanging on unresolved Promises or background listeners.

#### FR-7: Multi-File Cascade Budget & Anti-Oscillation
* **FR-7.1 (Two-Tier Repair Budget):**
  - **Outer Loop (File Queue):** Identifies all broken files via `tsc --noEmit` diagnostic list (capped at max 5 files).
  - **Inner Loop (Per-File Repair):** Max 2 repair attempts per file before backtracking or escalating. Global loop capped at 6 total LLM calls.
* **FR-7.2 (Alpha-Normalized Tabu State):** Hash semantic patch AST ($H_{semantic} = \text{SHA256}(\text{Canonicalize}(\text{AST}(\text{patch})))$). Reject duplicate semantic proposals.
* **FR-7.3 (Termination Gates):** Loop terminates on: test suite pass, identical error across 2 rounds, or global budget exhaustion.

---

### 3. Non-Functional Requirements (NFR)

* **NFR-1 (Portability):** Zero dependency on Docker Desktop in Local Mode.
* **NFR-2 (Repair Latency Budget):**
  | Phase | Local Mode Budget | Docker Mode Budget |
  | :--- | :--- | :--- |
  | `ts-morph` cold-start & project load | ≤5s | ≤5s |
  | `.d.ts` delta extraction | ≤2s | ≤2s |
  | AST slicing per file | ≤200ms | ≤200ms |
  | LLM API call (round-trip) | ≤8s | ≤8s |
  | Static lint + `tsc --noEmit` | ≤5s | ≤5s |
  | Test runner execution | ≤20s | ≤45s |
  | **Total per iteration** | **≤40s** | **≤65s** |
  | **Total (3 iterations max)** | **≤120s** | **≤195s** |
* **NFR-3 (API Quota Resilience):** Failover chain: Gemini Flash → Groq Llama-3-70B → exponential backoff (max 3 retries, max wait 60s). User notification on each fallback event.
* **NFR-4 (Correctness Gate):** A fix is accepted if and only if: (1) all baseline tests pass, (2) `tsc --noEmit` exits 0, (3) zero test files modified, and (4) no `AMBIGUOUS_REPLACE_TARGET` errors.

---

### 3. Non-Functional Requirements (NFR)

* **NFR-1 (Portability):** Zero dependency on Docker Desktop in Local Mode.
* **NFR-2 (Repair Latency Budget):**
  | Phase | Local Mode Budget | Docker Mode Budget |
  | :--- | :--- | :--- |
  | `ts-morph` cold-start & project load | ≤5s | ≤5s |
  | `.d.ts` delta extraction | ≤2s | ≤2s |
  | AST slicing per file | ≤200ms | ≤200ms |
  | LLM API call (round-trip) | ≤8s | ≤8s |
  | Static lint + `tsc --noEmit` | ≤5s | ≤5s |
  | Test runner execution | ≤20s | ≤45s |
  | **Total per iteration** | **≤40s** | **≤65s** |
  | **Total (3 iterations max)** | **≤120s** | **≤195s** |
* **NFR-3 (API Quota Resilience):** Failover chain: Gemini Flash → Groq Llama-3-70B → exponential backoff (max 3 retries, max wait 60s). User notification on each fallback event.
* **NFR-4 (Correctness Gate):** A fix is accepted if and only if: (1) all baseline tests pass, (2) `tsc --noEmit` exits 0, (3) zero test files modified, and (4) no `AMBIGUOUS_REPLACE_TARGET` errors.
