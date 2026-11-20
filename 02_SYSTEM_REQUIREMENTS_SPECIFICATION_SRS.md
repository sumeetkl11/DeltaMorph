# Software Requirements Specification (SRS)
## System Name: DeltaMorph (Compiler-Verified Breaking-Change Healer)

**Document Version:** 3.1 (Specification-Complete & Pre-Defense Ready)  
**Project Team:** Sumeet Kanhiya & Wahaj Khan  
**Academic Affiliation:** Dawood University of Engineering & Technology (DUET), Karachi  
**Target Environment:** Windows / Linux / macOS Developer Workstations (MVP Local Mode) & CI Runners (Stretch Docker Mode)  

---

### 1. Engineering Invariants
DeltaMorph enforces eight immutable engineering invariants:
1. **Unified Compiler Foundation:** All AST parsing, symbol resolution, `.d.ts` diffing, 3-Region slicing, anti-cheat static verification, and Tabu canonicalization use `ts-morph` (TypeScript Compiler API) exclusively.
2. **Deterministic Verification, Stochastic Synthesis:** Ground-truth `.d.ts` diffing and verification (`ts-morph` diagnostics + anti-cheat gate + test suite) are deterministic; LLM patch synthesis is explicitly modeled as guided stochastic search.
3. **Dual-Version `.d.ts` Acquisition Without `npm install`:** Prior-version declarations are extracted from npm registry tarballs (`pacote`) into a local declaration-only cache (`~/.DeltaMorph/dts-cache/`) with zero lifecycle script execution, eliminating the single-`node_modules` deadlock.
4. **Import-Aware 3-Region Slicing:** Every repair prompt includes relevant `import` declarations alongside the enclosing function scope and referenced local types so import-breaking upgrades are repairable.
5. **Working-Tree Safety, Windows Junction Hygiene & Atomicity:** Mutations run inside a detached `git worktree`; Windows `node_modules` linkage uses NTFS Directory Junctions, explicitly unlinked via non-recursive `fs.rmdirSync`/`fs.unlinkSync` prior to worktree removal; multi-file upgrades enforce all-or-nothing atomic rollback by default.
6. **Unified In-Memory Anti-Cheat Gate:** Static verification (`getPreEmitDiagnostics`, ban on new `@ts-ignore`/`@ts-nocheck`/`@ts-expect-error`, ban on newly introduced `any`, and `TypeChecker` floating-promise detection) runs inside the loaded `ts-morph` instance without spawning ESLint.
7. **Binding-Aware Anti-Oscillation:** Tabu hashing canonicalizes locally bound identifiers while preserving object destructuring property keys and external API symbols.
8. **Nonce-Delimited Prompt Security & Hard Process Guardrails:** Untrusted compiler/test output is `<`/`>` escaped and enclosed in per-run randomized XML nonces; every test execution is bounded by an atomic 20-second process-tree timeout.

---

### 2. Functional Requirements (FR)

#### FR-1: Workspace Isolation, Cross-Platform Linkage & Pre-Flight Triage
* **FR-1.1 (Clean-Tree Check & Detached Worktree):** Verify the host repository has no uncommitted changes (`git status --porcelain`). Spawn a detached scratchpad via `git worktree add --detach .DeltaMorph-scratch HEAD`.
* **FR-1.2 (Zero-Install `node_modules` Linkage):**
  - On Linux/macOS: Symlink host `node_modules` into `.DeltaMorph-scratch/node_modules`.
  - On Windows: Create an **NTFS Directory Junction** via `fs.symlinkSync(src, dest, 'junction')` (requires no Administrator privileges and completes in `<5ms`).
* **FR-1.3 (Safe Junction Teardown & `SIGINT` Trap):** On completion, failure, or `SIGINT`/`SIGTERM`:
  1. Remove any stale `.git/worktrees/.DeltaMorph-scratch/index.lock`.
  2. **Explicitly unlink** the `node_modules` junction/symlink using non-recursive `fs.rmdirSync` (Windows junction) or `fs.unlinkSync` (POSIX symlink)—**never** `fs.rmSync(..., { recursive: true })`—so host `node_modules` can never be deleted.
  3. Execute `git worktree remove --force .DeltaMorph-scratch`.
  4. Exit cleanly with the formal exit code defined in Section 4.
* **FR-1.4 (Opt-In Environment Forwarding):** Inherit minimal safe `process.env` by default. Only link `.env*` files into the worktree when the developer explicitly passes `--forward-env`, avoiding unintended secret exposure.
* **FR-1.5 (Pre-Flight Out-of-Scope Triage):** Before entering the repair loop:
  - Detect peer dependency deadlocks (`ERESOLVE`) via lockfile/resolution inspection and halt with `PEER_DEPENDENCY_CONFLICT` (Exit Code `2`).
  - Detect CJS/ESM module-boundary crashes (`ERR_REQUIRE_ESM`) during baseline test execution and halt cleanly with `UNSUPPORTED_MODULE_BOUNDARY` (Exit Code `3`).

#### FR-2: Dual-Version `.d.ts` Cache & `TypeChecker` Delta Engine
* **FR-2.1 (Dual-Version Acquisition):** Determine upgraded `(pkg, newVersion)` from current `package-lock.json` (or `--pkg`) and `oldVersion` from `git show HEAD~1:package-lock.json` (or explicit `--from <pkg>@<ver>`).
  - Resolve `new.d.ts` directly from linked `node_modules/<pkg>` (or `node_modules/@types/<pkg>`).
  - Fetch `old.d.ts` via `pacote` tarball stream into `~/.DeltaMorph/dts-cache/<pkg>@<oldVersion>/`, extracting **only** `package.json` and `**/*.d.ts` (zero JS execution, zero `postinstall` scripts, `<1.5s` cached).
* **FR-2.2 (Declaration Entry-Point Resolution Chain):** For both old and new trees, resolve entry `.d.ts` files via:
  1. `package.json` $\rightarrow$ `"exports"` conditional map (`types` condition).
  2. `package.json` $\rightarrow$ `"typesVersions"` matching the project's TypeScript version.
  3. `package.json` $\rightarrow$ `"types"` or `"typings"`.
  4. Fallback: `index.d.ts` or `@types/<pkg>` fetched via the same tarball cache.
* **FR-2.3 (`TypeChecker` Multi-File Symbol Delta Extraction):** Load old and new declaration projects into `ts-morph` and traverse `sourceFile.getExportSymbols()`, following barrel re-exports (`export * from`) via `TypeChecker.getAliasedSymbol()`. Extract removed/renamed exports, overload & parameter contract changes, return-type transitions (`T` $\rightarrow$ `Promise<T>`), interface property changes, and generic constraint narrowing into a structured `SymbolDeltaTable`.
* **FR-2.4 (Behavioral Break Fallback Pipeline):** When the `SymbolDeltaTable` is empty despite failing tests (delta-empty behavioral change):
  1. Classify the migration as `deltaKind: "behavioral"`.
  2. Switch to **Stack-Trace-Only Mode** using the filtered runtime assertion trace.
  3. Tag output and evaluation logs with `confidence: low`.
* **FR-2.5 (Project Config & JS-Only Support):** Inherit root `tsconfig.json` (preserving `baseUrl`, `paths` aliases like `@/*`, and forcing `jsx: ts.JsxEmit.Preserve`). If no `tsconfig.json` exists, synthesize an in-memory config with `allowJs: true`, `checkJs: true`, `noEmit: true`.

#### FR-3: 3-Region Import-Aware AST Slicing & Stack Filtering
* **FR-3.1 (3-Region Context Extraction):** For each broken diagnostic/stack site, construct a compact `ThreeRegionSlice`:
  1. **Region 1 — Imports Header:** Relevant `import` declarations referencing the upgraded package or symbols used in the failing scope.
  2. **Region 2 — Enclosing Scope:** Enclosing function, method, or class declaration containing the error line.
  3. **Region 3 — Referenced Local Types:** Local `interface` / `type` declarations referenced by the broken call-site.
* **FR-3.2 (Token Budget):** Target `250–450 tokens` per file slice (hard cap `600 tokens` for complex signatures; never whole-file dumps).
* **FR-3.3 (Stack Frame Filtering):** Strip runner internals (`node_modules/vitest`, `node:internal/*`), retaining top application frames and root assertion/diagnostic messages (`<100 tokens`).

#### FR-4: Nonce-Delimited Prompt Sanitization
* **FR-4.1 (Escaping & Nonce Wrapping):** Before inserting compiler diagnostics or test stderr into the LLM prompt:
  1. Strip non-printable ASCII control characters (except `\n`, `\t`) and truncate individual lines to 200 characters.
  2. Escape angle brackets (`<` $\rightarrow$ `&lt;`, `>` $\rightarrow$ `&gt;`) so untrusted output cannot inject or close XML tags.
  3. Wrap inside a per-run 64-bit cryptographic hex nonce (`<UNTRUSTED_COMPILER_OUTPUT_<nonce>> ... </UNTRUSTED_COMPILER_OUTPUT_<nonce>>`).

#### FR-5: Multi-Hunk Patching & Unified In-Memory `ts-morph` Anti-Cheat Gate
* **FR-5.1 (Multi-Hunk Search/Replace Format):** LLM outputs `1` to `3` structured `SEARCH` / `REPLACE` hunks per file, stripped of outer markdown fences and validated by a strict parser. Malformed responses trigger one structured format-correction retry (counting against the global call ceiling).
* **FR-5.2 (Two-Tier Deterministic Matching):** Apply each hunk via Tier 1 (exact match) $\rightarrow$ Tier 2 (whitespace-normalized match). Zero matches re-prompt with the exact target slice; `>1` matches reject with `AMBIGUOUS_REPLACE_TARGET`.
* **FR-5.3 (Unified In-Memory `ts-morph` Static & Anti-Cheat Gate):** Immediately update the in-memory `ts-morph` `SourceFile` and run four checks without spawning external ESLint (`<300ms`):
  1. **Type Check:** `project.getPreEmitDiagnostics()` on affected files.
  2. **Comment Suppression Ban:** Reject any patch introducing new `// @ts-ignore`, `// @ts-nocheck`, `// @ts-expect-error`, or `eslint-disable` directives.
  3. **Explicit `any` Ban:** Reject any newly introduced `SyntaxKind.AnyKeyword` annotation or `as any` cast not present in the original AST.
  4. **Floating-Promise Check:** Reject modified `ExpressionStatement` nodes whose `TypeChecker` type is `PromiseLike<T>` without `await`, `return`, `.then()`, `.catch()`, or explicit `void`.

#### FR-6: Guarded Test Execution & Immutability
* **FR-6.1 (Test File Modification Guard):** Before accepting any patch, assert via `git diff --name-only` inside `.DeltaMorph-scratch` that zero test files (`*.test.*`, `*.spec.*`, `__tests__/**`) were modified.
* **FR-6.2 (Cache Redirection):** Invoke `Vitest` with an ephemeral cache directory under OS `os.tmpdir()`.
* **FR-6.3 (20-Second Hard Process-Tree Timeout):** Wrap test execution with an `AbortController` enforcing an atomic **20-second timeout**. On timeout, terminate the entire process tree (`SIGKILL` on POSIX / `taskkill /F /T` on Windows), classify as `EXECUTION_TIMEOUT_ERROR`, and roll back the candidate patch.

#### FR-7: Dynamic Cascade Queue, Binding-Aware Tabu & Atomic Rollback
* **FR-7.1 (Dynamic Cascade Budget):**
  - Re-query broken files dynamically from `ts-morph` diagnostics + test failures after every accepted per-file patch.
  - **Cascade Scope Cap:** Maximum **4 affected files** (`>4` halts immediately with `EXCEEDS_CASCADE_SCOPE`, Exit Code `4`).
  - **Per-File Retry Cap:** Maximum **3 attempts per file** (aligned with `Pass@1` and `Pass@3`).
  - **Global LLM Call Ceiling:** Maximum **8 total LLM calls** per `DeltaMorph run`.
* **FR-7.2 (Binding-Aware Alpha-Normalized Tabu State Machine):** Canonicalize locally bound variables (`_v0`, `_v1`, ...) using `ts-morph` symbol bindings—expanding shorthand destructuring `{ data }` $\rightarrow$ `{ data: _v0 }` to preserve property keys and leaving imported/external identifiers untouched—strip comments/whitespace, and compute `SHA-256`. Reject duplicate hashes before running tests, apply temperature jitter (`+0.15`), and append a negative constraint.
* **FR-7.3 (Termination & Acceptance Gate):** Exit `0` if and only if all baseline tests pass, final external `tsc --noEmit` exits `0`, zero test files changed, and zero anti-cheat violations occurred.
* **FR-7.4 (Multi-File Atomicity & Partial-Patch Policy):**
  - **Default Mode (CI-Safe Atomic Rollback):** If File 1 passes independently but File 2 exhausts the retry or global call budget, **zero** partial changes are merged into the developer's active branch. The worktree is pruned cleanly and `.DeltaMorph-report.json` records which files passed vs. failed.
  - **Opt-In Partial Export (`--save-partial-patch`):** If the developer passes `--save-partial-patch`, DeltaMorph runs `git diff` on individually verified files inside `.DeltaMorph-scratch` and writes `.DeltaMorph-partial.patch` to disk **before** rolling back and pruning the worktree, allowing manual inspection (`git apply .DeltaMorph-partial.patch`) without risking an unverified merge.

---

### 3. Non-Functional Requirements (NFR)

* **NFR-1 (Portability):** MVP Local Mode requires only Node.js v20+ and Git—zero Docker Desktop dependency on Windows, macOS, or Linux.
* **NFR-2 (Latency Budget — Local Mode):**

| Stage | Frequency | Target Budget |
| :--- | :--- | :--- |
| Worktree creation + NTFS junction / symlink | Once per run | `≤1s` |
| `pacote` `.d.ts` tarball fetch (uncached / cached) | Once per run | `≤2.5s` / `≤0.2s` |
| `ts-morph` project init + `TypeChecker` `.d.ts` diff | Once per run | `≤5s` |
| **Per-Iteration Loop:** | | |
| 3-Region AST slicing + nonce sanitization | Per iteration | `≤0.3s` |
| LLM API round-trip (`gemini-2.5-flash`) | Per iteration | `≤8s` |
| Multi-hunk patch + Tabu hash + In-memory `ts-morph` gate | Per iteration | `≤0.5s` |
| `Vitest` test execution (hard kill at 20s) | Per iteration | `≤20s` |
| **Total per single repair iteration** | **Per iteration** | **`≤30s` (worst-case `≤35s`)** |
| **Final external `tsc --noEmit` verification** | Once on pass | `≤6s` |

* **NFR-3 (API Quota & Error Handling):** Pin `gemini-2.5-flash` via `@google/genai` with exponential backoff (up to 3 retries on HTTP 429/503, capped at 30s wait). Secondary provider failover (Groq Llama-3.3-70B) is a Stretch feature.
* **NFR-4 (Reproducibility & Telemetry Logging):** Every run emits a structured `.DeltaMorph-report.json` containing pinned model ID, exact input/output tokens, delta classification, Tabu rejections, anti-cheat rejections, exit code, and wall-clock latency breakdown.

---

### 4. Formal Exit-Code, Failure-Mode & Recovery Matrix

| Exit Code | Error Code / Status | Trigger Condition | Worktree & Cleanup Action | Artifact Saved | User Recovery Guidance |
| :---: | :--- | :--- | :--- | :--- | :--- |
| **`0`** | `HEAL_SUCCESS` / `NO_BREAK_DETECTED` | All tests + `tsc --noEmit` pass (or baseline already green) | Apply verified patch (if any), unlink junction, prune worktree | `.DeltaMorph-report.json` | Review `git diff` and commit. |
| **`1`** | `DIRTY_WORKING_TREE` | Uncommitted changes detected in host repo before start | No worktree created | None | Run `git stash` or commit changes before retrying. |
| **`2`** | `PEER_DEPENDENCY_CONFLICT` | Lockfile / `ERESOLVE` conflict detected at pre-flight | Unlink junction, prune worktree | `.DeltaMorph-report.json` | Resolve npm peer dependencies manually first. |
| **`3`** | `UNSUPPORTED_MODULE_BOUNDARY` | `ERR_REQUIRE_ESM` or CJS/ESM module crash detected | Unlink junction, prune worktree | `.DeltaMorph-report.json` | Perform CJS$\rightarrow$ESM migration manually (out of scope). |
| **`4`** | `EXCEEDS_CASCADE_SCOPE` | `>4` source files have compilation/test failures initially | Unlink junction, prune worktree | `.DeltaMorph-report.json` | Break migration into smaller PRs or refactor manually. |
| **`5`** | `REPAIR_BUDGET_EXHAUSTED` | Max per-file retries (`3`) or global calls (`8`) exhausted without full pass | Atomic rollback (export `.DeltaMorph-partial.patch` if flag set), unlink junction, prune worktree | `.DeltaMorph-report.json` (+ optional `.DeltaMorph-partial.patch`) | Inspect report diagnostics (and optional partial patch). |
| **`6`** | `EXECUTION_TIMEOUT_ERROR` | Test suite exceeds 20s hard timeout across all remaining retries | Process-tree `SIGKILL`/`taskkill`, rollback, unlink junction, prune worktree | `.DeltaMorph-report.json` | Check for unresolved Promises or open server sockets. |
| **`7`** | `LLM_API_UNAVAILABLE` | LLM API fails after 3 exponential backoff retries (429/503/network) | Rollback, unlink junction, prune worktree | `.DeltaMorph-report.json` | Verify `GEMINI_API_KEY`, quota, or network connectivity. |
| **`130`** | `USER_INTERRUPTED_SIGINT` | User presses `Ctrl+C` (`SIGINT`) or `SIGTERM` received | Trap handler deletes stale `index.lock`, unlinks junction, prunes worktree | None | Host repository guaranteed clean and untouched. |
