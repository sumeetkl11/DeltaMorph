# Implementation Roadmap, Tech Stack, Evaluation Protocol & Benchmarks
## Project: DeltaMorph (Compiler-Verified Breaking-Change Healer)

**Institution:** Dawood University of Engineering & Technology (DUET), Karachi  
**Project Team:** Sumeet Kanhiya & Wahaj Khan  
**Document Version:** 3.2 (Empirically Grounded Pre-Defense Edition)  

---

### 1. Technology Stack (MVP vs. Stretch)

| Component | MVP Choice | Stretch Addition | Justification |
| :--- | :--- | :--- | :--- |
| **Runtime** | Node.js (v20+ LTS) & TypeScript | — | Native npm tooling, strict typing, async I/O |
| **Compiler, AST & Static Gate** | `ts-morph` (TS Compiler API) | — | Single unified engine for `.d.ts` `TypeChecker` diffing, 3-Region slicing, in-memory diagnostics, anti-cheat rules, and binding-aware Tabu |
| **Dual `.d.ts` Acquisition** | `pacote` (npm tarball extractor) | — | Extracts `package.json` + `**/*.d.ts` into local cache in `<1.5s` without `npm install` or lifecycle scripts |
| **Workspace Isolation** | Native `git worktree` + NTFS Junctions / Symlinks | Docker (`dockerode`, `--network=none`, `:ro`) | Zero working-tree pollution & zero-admin Windows support in MVP; OS security isolation in Stretch |
| **Test Runner** | `Vitest` (redirected `/tmp` cache, 20s hard kill) | `Jest` adapter | Fast TypeScript-native execution with predictable CLI exit codes |
| **LLM Orchestration** | `@google/genai` (pinned `gemini-2.5-flash`) | Groq Llama-3.3-70B failover | Low latency, low cost, structured output, single-provider simplicity for MVP |
| **CLI Interface** | `commander` (plain structured logs) | `ink` interactive TUI + GitHub Action | Focuses engineering effort on compiler accuracy before UI polish |

---

### 2. Curated Reproducible Benchmark Suite (15 Benchmarks)

#### 2.1 Reproducible Benchmark Manifest Schema (`benchmarks.manifest.json`)
Every benchmark is stored in a version-controlled JSON manifest so any examiner can reproduce the exact failure and repair run:

```json
{
  "id": "BM-01",
  "fixtureOrRepo": "benchmarks/BM-01 (pre-defense verified fixture)",
  "baselineCommitSha": "caf2f09b5b7f4047bdf75adae84790e8bd7dbe3f",
  "brokenCommitSha": "2282e811949df50bf1350580d818ca07f38c46ce",
  "package": "axios",
  "fromVersion": "0.27.2",
  "toVersion": "1.7.7",
  "deltaClass": "structural",
  "vitestCmd": "npx vitest run",
  "flakiness5xPassed": true,
  "baselineFailingTests": ["src/apiClient.test.ts"]
}
```

#### 2.2 Five Concrete Empirically Verified Breaking API Migrations
All five migrations below have been **reproduced end-to-end** against live npm packages (`pre-upgrade PASS`, `post-upgrade 5/5 deterministic FAIL`):

1. **`axios` (`v0.27.2` $\rightarrow$ `v1.7.7`, Structural — Interface & Generic Narrowing):**
   - *Break:* Request interceptors expect `InternalAxiosRequestConfig` instead of `AxiosRequestConfig`; `headers` narrowed to `AxiosHeaders | RawAxiosRequestHeaders` (`TS2345`). Tests **`TypeChecker.getDeclaredTypeOfSymbol()` interface diffing**.
2. **`date-fns` (`v2.30.0` $\rightarrow$ `v3.6.0`, Structural — Default Subpath Export Removal):**
   - *Break:* Subpath default export (`import format from 'date-fns/format'`) removed in favor of named export (`import { format } from 'date-fns/format'`, `TS2613`). Tests **Region 1 (Imports Header)** + **Multi-Hunk Search/Replace**.
3. **`zod` (`v3.23.8` $\rightarrow$ `v4.6.7`, Structural — Arity & Error Schema Removal):**
   - *Break:* `z.record(valueSchema)` 1-argument overload removed (`TS2554: Expected 2-3 arguments, but got 1`) and `ZodError.errors` removed in favor of `.issues` (`TS2339`). *(Verified live on npm: `zod` never published a stable `v2.x`.)*
4. **`uuid` (`v8.3.2` $\rightarrow$ `v9.0.0` with `@types/uuid` `8.3.4` $\rightarrow$ `9.0.8`, Structural — Return-Type Narrowing):**
   - *Break:* `parse(uuid: string)` return type narrowed from `ArrayLike<number>` (`OutputBuffer`) to `Uint8Array` (`TS2322`). Tests **DefinitelyTyped `@types/*` tarball resolution** + **return-type diffing**.
5. **`jsonwebtoken` (`v8.5.1` $\rightarrow$ `v9.0.2` with `@types/jsonwebtoken` `8.5.9` $\rightarrow$ `9.0.6`, Behavioral — Delta-Empty Fallback):**
   - *Break:* `tsc --noEmit` exits `0`, but `Vitest` fails at runtime with `Error: secretOrPrivateKey has a minimum key size of 2048 bits for RS256`. Tests **Behavioral Fallback Mode (`confidence: LOW`)**.

#### 2.3 Full 15-Benchmark Registry Plan

*Verification Rule:* The first **5 benchmarks** (`BM-01`, `BM-02`, `BM-03`, `BM-05`, `BM-09`) are already built, committed, and verified (`pre-upgrade PASS`, `5/5 post-upgrade deterministic FAIL`) in **Appendix A**; the remaining **10 benchmarks** are locked in Month 1 before repair experiments begin.

**Part A: Real-World Major Breaking-Change Repositories (10 Benchmarks)**

| ID | Target Package | Major Migration | Breaking-Change Mechanism | Expected Delta Class | Status |
| :--- | :--- | :--- | :--- | :--- | :--- |
| BM-01 | **Axios** | `0.27.2` → `1.7.7` | `InternalAxiosRequestConfig` / `AxiosHeaders` narrowing (`TS2345`) | Structural (Delta-Rich) | ✅ Verified (Appendix A) |
| BM-02 | **Date-fns** | `2.30.0` → `3.6.0` | Default subpath export removal (`TS2613`) | Structural (Delta-Rich) | ✅ Verified (Appendix A) |
| BM-03 | **Zod** | `3.23.8` → `4.6.7` | `z.record` arity (`TS2554`) & `ZodError.errors` removal (`TS2339`) | Structural (Delta-Rich) | ✅ Verified (Appendix A) |
| BM-04 | **Commander.js** | `8.x` → `10.x` | Action handler argument unpacking & option typing | Structural (Delta-Rich) | Month 1 Lock |
| BM-05 | **Uuid** | `8.3.2` → `9.0.0` | `@types/uuid` `parse()` return narrowed to `Uint8Array` (`TS2322`) | Structural (Delta-Rich) | ✅ Verified (Appendix A) |
| BM-06 | **Helmet** | `4.x` → `7.x` | Middleware option schema & CSP directive restructuring | Structural (Delta-Rich) | Month 1 Lock |
| BM-07 | **Got** | `11.x` → `12.x` | Options/hooks signature & response typing updates | Structural (Delta-Rich) | Month 1 Lock |
| BM-08 | **Supertest** | `6.x` → `7.x` | Agent/request typings & response contract updates | Structural (Delta-Rich) | Month 1 Lock |
| BM-09 | **JsonWebToken** | `8.5.1` → `9.0.2` | Runtime 2048-bit minimum RS256 key enforcement (`tsc` exits 0) | **Behavioral (Delta-Empty)** | ✅ Verified (Appendix A) |
| BM-10 | **Pino** | `6.x` → `7.x` | Transport/stream runtime execution model change | **Behavioral (Delta-Empty)** | Month 1 Lock |

**Part B: Targeted Stress, Held-Out & Triage Benchmarks (5 Benchmarks)**

| ID | Benchmark Case | What It Scientifically Validates |
| :--- | :--- | :--- |
| BM-11 | **Held-Out Custom / Obscure Package (`<100k` downloads)** | Proves `.d.ts` `TypeChecker` diffing generalizes with **zero** LLM training-data memorization. |
| BM-12 | **Cyclic Oscillation + Destructuring Trap** | Stress-tests Binding-Aware Alpha-Normalized Tabu (`{ data }` $\rightarrow$ `{ data: _v0 }`) for 0% duplicate test runs. |
| BM-13 | **Anti-Cheat Compiler-Bypass Trap** | Complex generic mismatch tempting `@ts-ignore` / `as any`; validates 100% rejection by in-memory `ts-morph` gate. |
| BM-14 | **Multi-File Cascade + Import Rewrite (3–4 Files)** | Validates dynamic diagnostic queue, 3-Region import slicing, multi-hunk patching, and atomic rollback / `--save-partial-patch`. |
| BM-15 | **Pre-Flight Triage & Stability Suite** | Validates clean detection of `ERR_REQUIRE_ESM` (Exit `3`), `ERESOLVE` (Exit `2`), 20s `SIGKILL` (Exit `6`), and 5$\times$ flakiness filter. |

*(BM-01 through BM-14 measure repair `Pass@1`/`Pass@3`; BM-15 validates safety/triage assertions.)*

---

### 3. Quantitative & Qualitative Evaluation Protocol

#### 3.1 Primary Metrics (Empirically Measured in Month 5)
1. **Pass@1 & Pass@3:** Reported overall ($n=14$) and split by Delta-Rich vs. Delta-Empty (Target: Delta-Rich `Pass@1 ≥ 55%`, `Pass@3 ≥ 75%`).
2. **Token Efficiency:** Median prompt tokens for 3-Region Slicing (target `<450 tokens`) vs. measured Full-File baseline ($\ge 70\%$ reduction target; PoC measured `77.1%` reduction on sample client).
3. **Mean Time to Repair (MTTR):** Wall-clock latency per repair (target `≤35s`/iteration in Local Mode).
4. **Gate & Tabu Telemetry:** Tabu duplicates blocked, anti-cheat rejections (`@ts-ignore`, new `any`, floating promises), and malformed-hunk recoveries.
5. **Exact API Cost per Fix:** Logged directly from `@google/genai` `usageMetadata`.

#### 3.2 Statistical Rigor
* **Confidence Intervals:** **Wilson Score 95% CIs** on all `Pass@1` and `Pass@3` proportions.
* **Paired Significance Testing:** **McNemar's test** ($p < 0.05$) comparing `A0 vs. A1` (**RQ2**) and `A0 vs. A2` (**RQ1**).
* **Flakiness Control:** 5$\times$ unchanged baseline run per benchmark before evaluation.

#### 3.3 Qualitative Patch Correctness Audit
Two independent reviewers (Faculty Supervisor + Senior Industry/Alumni TypeScript Engineer) blindly classify every passing patch (`Correct` / `Plausible-Suspect` / `Incorrect`), reporting **Cohen's Kappa ($\kappa > 0.6$)**.

#### 3.4 Ablation Study Design

| Run ID | Configuration | Scope | Isolates |
| :--- | :--- | :--- | :--- |
| **A0 (Full System)** | `.d.ts` Delta + 3-Region Slice + Tabu + `ts-morph` Anti-Cheat Gate | **MVP Core** | Full DeltaMorph performance |
| **A1 (No `.d.ts` Delta)** | Disable `.d.ts` `TypeChecker` diff (stack-trace only) | **MVP Core** | Value of `.d.ts` ground-truth delta (**RQ2**) |
| **A2 (Full-File Baseline)** | Replace 3-Region AST slice with full source file(s) | **MVP Core** | Token & accuracy impact of 3-Region slicing (**RQ1**) |
| **A3 (No Tabu)** | Disable Binding-Aware Tabu state machine | **Stretch** | Iteration reduction from anti-oscillation |
| **A4 (No Anti-Cheat Gate)** | Disable `@ts-ignore` / `any` / floating-promise checks | **Stretch** | Rate of silent type-safety bypasses prevented |

#### 3.5 Exact Experiment Compute, Rate-Limit & API Cost Budget
* **Core MVP Evaluation Calls (`A0`, `A1`, `A2` across 14 repairable benchmarks):**
  - Minimum (all Pass@1): $14 \times 3 \times 1 = 42\text{ LLM calls}$
  - Expected average (~3 calls/run): $14 \times 3 \times 3 = \mathbf{126\text{ LLM calls}}$
  - Worst-case ceiling (all hit global cap of 8): $14 \times 3 \times 8 = \mathbf{336\text{ LLM calls}}$
* **Estimated Token Volume (Worst-Case 336 Calls):**
  - `A0` & `A1` (sliced, ~650 input + ~250 output tokens/call): $\le 224 \times 900 \approx 201{,}600\text{ tokens}$
  - `A2` (full-file baseline, ~3,000 input + ~300 output tokens/call): $\le 112 \times 3{,}300 \approx 369{,}600\text{ tokens}$
  - **Combined Worst-Case Volume:** $< 0.6\text{ million total tokens}$ (including development/debugging buffer: $< 2\text{ million tokens}$).
* **Estimated Total Budget (`gemini-2.5-flash`):** **$< \$1.50\text{ USD}$** total across all development and final ablation runs, paced with a `4s` inter-call delay inside the evaluation harness to avoid HTTP 429 throttling.

---

### 4. Empirical Baseline Comparison Table (Populated in Month 5)

| Metric | Full-File LLM Baseline (`A2`) | Stack-Trace-Only Sliced (`A1`) | **DeltaMorph Full (`A0`)** |
| :--- | :--- | :--- | :--- |
| **Pass@1 (Delta-Rich)** | *Measured in Month 5* | *Measured in Month 5* | *Target: ≥55% (≥+20pp vs A1)* |
| **Pass@3 (Delta-Rich)** | *Measured in Month 5* | *Measured in Month 5* | *Target: ≥75%* |
| **Pass@1 / Pass@3 (Delta-Empty)** | *Measured in Month 5* | *Measured in Month 5* | *Reported separately* |
| **Median Prompt Tokens** | *Measured in Month 5* | *Measured in Month 5* | *Target: <450 (PoC: 177 vs 773)* |
| **Anti-Cheat Bypass Rate** | *Measured in Month 5* | *Measured in Month 5* | *Target: 0% (PoC: 3/3 blocked)* |
| **Mean Wall-Clock Latency** | *Measured in Month 5* | *Measured in Month 5* | *Target: ≤35s/iter* |

---

### 5. Lean MVP vs. Stretch Feature Matrix (100% Aligned with RQ1–RQ3)

| Feature | MVP (Must-Have) | Stretch (Post-MVP) | Owner |
| :--- | :---: | :---: | :--- |
| Git Worktree + Windows NTFS Junctions + `SIGINT` Cleanup | ✅ | — | Wahaj Khan |
| Pre-Flight Triage (`ERESOLVE` + `ERR_REQUIRE_ESM`) & Exit Codes | ✅ | — | Wahaj Khan |
| `pacote` Dual-Version `.d.ts` Cache + `TypeChecker` Delta Engine | ✅ | — | Sumeet Kanhiya |
| 3-Region Import-Aware AST Slicer + Multi-Hunk Parser/Patcher | ✅ | — | Sumeet & Wahaj |
| Unified In-Memory `ts-morph` Anti-Cheat Static Gate | ✅ | — | Sumeet Kanhiya |
| Binding-Aware Alpha-Normalized Tabu State Machine | ✅ | — | Sumeet Kanhiya |
| Atomic Rollback + `--save-partial-patch` Export | ✅ | — | Wahaj Khan |
| Nonce-Delimited Sanitizer + Pinned `gemini-2.5-flash` | ✅ | Groq Llama-3.3-70B failover | Wahaj Khan |
| `Vitest` Guarded Runner (20s Process-Tree Timeout) | ✅ | `Jest` adapter | Wahaj Khan |
| Plain `commander` CLI + `.DeltaMorph-report.json` | ✅ | `ink` TUI + GitHub Action | Wahaj Khan |
| 15 Reproducible Benchmarks + 3-Run Core Ablation (`A0–A2`) | ✅ | Extra Ablations (`A3–A4`) | Sumeet & Wahaj |
| Docker Sandbox (`dockerode`, `--network=none`, `:ro`) | — | ✅ | Wahaj Khan |

---

### 6. Six-Month Two-Developer Milestone Roadmap

#### Month 1: Proposal Approval & Remaining 10 Benchmarks
* **Completed Pre-Defense (Appendix A):** Standalone `pacote` + `ts-morph` PoC verified + first 5 benchmark fixtures locked (`BM-01`, `BM-02`, `BM-03`, `BM-05`, `BM-09`).
* **Wahaj:** Push the 5 verified fixtures and lock the remaining 10 benchmarks (`BM-04`, `BM-06`–`BM-08`, `BM-10`–`BM-15`) in `benchmarks.manifest.json` with 5$\times$ flakiness runs.
* **Joint:** Defend Synopsis & Proposal before DUET CS committee; finalize 14-paper literature review chapter.

#### Month 2: Core Analysis & Isolation Engines
* **Sumeet:** Build production `TypeChecker` symbol delta engine (re-exports, overloads, `@types/*`, Delta-Empty fallback) + 3-Region AST slicer.
* **Wahaj:** Build `git worktree` manager with non-recursive Windows NTFS junction cleanup, `SIGINT` trap, synthetic `tsconfig`, and pre-flight triage (`Exit 2/3`).

#### Month 3: Verification Gate, Tabu & Repair Loop
* **Sumeet:** Build unified in-memory `ts-morph` anti-cheat gate (`getPreEmitDiagnostics`, `@ts-ignore`/`any`/floating-promise checks) + binding-aware Tabu hasher.
* **Wahaj:** Build nonce-delimited sanitizer, `@google/genai` client (`gemini-2.5-flash`), strict multi-hunk parser/patcher, 20s `AbortController` Vitest runner, and atomic rollback (`--save-partial-patch`).

#### Month 4: End-to-End MVP Integration & Hardening
* **Joint:** Wire dynamic cascade queue (`≤4 files`, `≤3 attempts/file`, `≤8 calls`) into `commander` CLI, validate all exit codes (`0`–`7`, `130`), and debug end-to-end across all 15 benchmarks.

#### Month 5: Empirical Evaluation, Ablations & Human Audit
* **Wahaj:** Run full `A0–A2` harness with `4s` rate-limit pacing and archived JSON telemetry logs.
* **Sumeet:** Compute Wilson 95% CIs, McNemar's tests, and coordinate blind 2-reviewer patch quality audit (Cohen's $\kappa$).

#### Month 6: Stretch Polish, Thesis & Final Viva
* **Joint:** Implement Stretch items strictly if Month 5 evaluation is complete; finalize thesis chapters, research paper draft, and live CLI demo.

---

### Appendix A: Empirically Measured Pre-Defense PoC & First-5-Benchmark Evidence
*(Executed live on Node.js v22.23.2, npm 10.9.8, TypeScript, `pacote`, `tar`, `ts-morph`, and `vitest` — 100% real measured telemetry.)*

#### A.1 Standalone `pacote` + `ts-morph` PoC Measurements (Owner: Sumeet Kanhiya)

| Measurement | Target / Workload | Empirically Measured Result | Budget Target | Status |
| :--- | :--- | :--- | :---: | :---: |
| `pacote` `.d.ts`-only tarball extraction (cold / warm cache) | `axios@0.27.2` (2 files: `package.json` + `index.d.ts`, zero JS/scripts) | **`146 ms` cold / `0.29 ms` warm** | `≤2,500 ms` / `≤200 ms` | ✅ Pass |
| `ts-morph` Project init + `TypeChecker` symbol graph diff | `axios@0.27.2 → 1.7.7` (30 old exports vs. 59 new exports) | **`2,094 ms`** (**63 total deltas**, **15 focused deltas**) | `≤5,000 ms` | ✅ Pass |
| 3-Region Import-Aware AST Slicing vs. Full-File | 106-line `EnterpriseApiClient` (`axios` interceptor client) | **`177 tok` vs. `773 tok` (`77.1%` reduction)** in **`2.23 ms`** | `<450 tok` (`≥70%` cut) | ✅ Pass |
| Unified In-Memory `ts-morph` Anti-Cheat Gate | Cheat patch (`@ts-ignore` + `as any` + floating promise) vs. clean patch | **3/3 cheats blocked** (`TS_IGNORE_ADDED`, `EXPLICIT_ANY_ADDED`, `FLOATING_PROMISE`); **`2.19 ms` warm** (`607 ms` cold init) | `≤300 ms` warm | ✅ Pass |
| Binding-Aware Alpha-Normalized Tabu Hasher | `{ data }` vs. `{ data: body }` (duplicate) vs. `{ status }` (distinct) | Duplicate hash match **`a63104c5c964c783` = `true`**; distinct property hash **`ff6bbb24096a2c3d` ≠ `a63104c5c964c783`** | `100%` precision | ✅ Pass |

**Verbatim Extracted `SymbolDelta[]` Sample (`axios@0.27.2 → 1.7.7`, `ts-morph` `TypeChecker.getDeclaredTypeOfSymbol()`):**
```json
[
  {
    "symbolName": "AxiosRequestConfig.headers",
    "deltaKind": "PROPERTY_CONTRACT_CHANGED",
    "oldSignature": "headers?: AxiosRequestHeaders | undefined",
    "newSignature": "headers?: AxiosHeaders | (Partial<RawAxiosHeaders & { Accept: AxiosHeaderValue; ... }>)"
  },
  {
    "symbolName": "AxiosRequestConfig.paramsSerializer",
    "deltaKind": "PROPERTY_CONTRACT_CHANGED",
    "oldSignature": "paramsSerializer?: ((params: any) => string) | undefined",
    "newSignature": "paramsSerializer?: CustomParamsSerializer | ParamsSerializerOptions | undefined"
  },
  {
    "symbolName": "AxiosRequestConfig.onUploadProgress",
    "deltaKind": "PROPERTY_CONTRACT_CHANGED",
    "oldSignature": "onUploadProgress?: ((progressEvent: any) => void) | undefined",
    "newSignature": "onUploadProgress?: ((progressEvent: AxiosProgressEvent) => void) | undefined"
  }
]
```

**Key Compiler Engineering Finding Validated During PoC:** TypeScript's `TypeChecker.getTypeOfSymbolAtLocation()` returns value-space types (functions/constants) and misses member properties on `interface`/`type` declarations (`AxiosRequestConfig`, `AxiosResponse`). Invoking `TypeChecker.getDeclaredTypeOfSymbol()` for type declarations lifted focused delta recall from `0` to `15` real interface contract changes on `axios`.

#### A.2 First 5 Verified Reproducible Benchmark Fixtures (Owner: Wahaj Khan)
*(All 5 fixtures initialized as standalone git repositories with pre-upgrade passing baseline commits and post-upgrade broken commits; ready to push verbatim to the team's GitHub benchmark archive.)*

| ID | Package Upgrade | Pre-Upgrade Passing SHA | Post-Upgrade Broken SHA | Pre-Pass | 5$\times$ Post-Fail | Verbatim Reproduced Compiler / Runtime Diagnostic |
| :--- | :--- | :--- | :--- | :---: | :---: | :--- |
| **BM-01** | `axios` `0.27.2 → 1.7.7` | `caf2f09b5b7f4047bdf75adae84790e8bd7dbe3f` | `2282e811949df50bf1350580d818ca07f38c46ce` | ✅ Pass | **5/5 Fail** | `TS2345: Argument of type '(config: AxiosRequestConfig) => ...' is not assignable to '(value: InternalAxiosRequestConfig) => ...'. Types of property 'headers' are incompatible.` |
| **BM-02** | `date-fns` `2.30.0 → 3.6.0` | `a9965de91ff4ed8fdd4c1e7fd872f7151c27cb3f` | `e57c1c922b90630a2e5e36a7743829da718cfca1` | ✅ Pass | **5/5 Fail** | `TS2613: Module '.../date-fns/format' has no default export. Did you mean to use 'import { format } from ...'?` |
| **BM-03** | `zod` `3.23.8 → 4.6.7` | `089beaeb46780e7a2b5cb6278edad03784c87d2b` | `c61a7ef3dedb296bb18144f6867a4f1d4b4d12ce` | ✅ Pass | **5/5 Fail** | `TS2554: Expected 2-3 arguments, but got 1` (`z.record`) & `TS2339: Property 'errors' does not exist on type 'ZodError<...>'` |
| **BM-05** | `uuid` `8.3.2 → 9.0.0` (`@types/uuid` `8.3.4 → 9.0.8`) | `8f296710c74b350e58f9cbb47e39613d9adb5eb0` | `cbdfb0e1f6928d09bc4ae17ee91878817ab1a84a` | ✅ Pass | **5/5 Fail** | `TS2322: Type '(_uuid: string) => number[]' is not assignable to type 'parse'. Type 'number[]' is missing properties from type 'Uint8Array'` |
| **BM-09** | `jsonwebtoken` `8.5.1 → 9.0.2` (`@types` `8.5.9 → 9.0.6`) | `3aa2151940b0170469636a3d1f50516365221c42` | `d83076d9138bf431848d7eda4691035a6fc3873a` | ✅ Pass | **5/5 Fail** | *(`tsc` exits 0; Vitest runtime behavioral break)* `Error: secretOrPrivateKey has a minimum key size of 2048 bits for RS256` |
