# System Architecture & Design Document (Version 2.3)
## System Name: DeltaMorph (Compiler-Verified Breaking-Change Healer)

**Lead Architect:** Uday Sharma ([@udaysharmadev](https://github.com/udaysharmadev))  
**Academic Affiliation:** Dawood University of Engineering & Technology (DUET)  

---

### 1. Complete End-to-End System Flowchart

```mermaid
flowchart TD
    A["CLI: deltamorph run"] --> B["Register SIGINT Trap Handler & Dirty-Tree Check"]
    B --> C["Git Worktree: git worktree add --detach .deltamorph-scratch"]
    C --> D{"OS Platform?"}
    D -- "Windows" --> D1["Create NTFS Junctions (mklink /J / fs.symlink junction)"]
    D -- "Linux / macOS" --> D2["Create Filesystem Symlinks (ln -s)"]
    D1 --> E["Check tsconfig.json & Alias Paths (@/*)"]
    D2 --> E
    E -- "Missing" --> F["Generate Synthetic tsconfig (allowJs: true, preserve JSX)"]
    E -- "Present" --> G["Pre-Flight: Lockfile & Peer Dependency Check (npm explain)"]
    F --> G
    G --> H{"Peer Conflicts (ERESOLVE)?"}
    H -- "Yes" --> I["Output Diagnostic & Halt"]
    H -- "No" --> J["Baseline Test Run (Cache → /tmp, 15s SIGKILL Timeout)"]
    
    J --> K{"Tests Pass?"}
    K -- "Yes" --> L["No Breaking Change → Prune Worktree & Exit 0"]
    K -- "No" --> M["Stack Frame Filtering (<100 tokens)"]
    
    subgraph Type Delta & Context Extraction
        M --> N["Resolve .d.ts via exports → typesVersions → types → index.d.ts"]
        N --> O["ts-morph Declaration Diffing: old.d.ts vs new.d.ts"]
        O --> P{"Symbol Delta Table Empty?"}
        P -- "No (Delta-Rich)" --> Q["Generate Structural Delta Context"]
        P -- "Yes (Behavioral Break)" --> R["Activate Stack-Trace-Only Fallback Mode"]
        R --> S["Flag confidence: low in output"]
        M --> T["ts-morph Ancestor Slicing (Enclosing Scope + 2 Hops)"]
        Q --> U["Assemble Bounded Prompt (<350 tokens)"]
        S --> U
        T --> U
    end
    
    subgraph Prompt Security Layer
        U --> V["Sanitize: Strip Non-Printable, Injection Patterns, Truncate"]
        V --> W["Wrap in XML Delimiters (UNTRUSTED_COMPILER_OUTPUT)"]
    end
    
    subgraph Two-Tier Closed-Loop Healing Engine
        W --> X["LLM Synthesizer (Gemini Flash / Groq Llama-3)"]
        X --> Y["Output: Search/Replace Block"]
        Y --> Z["Two-Tier Match: Exact → Whitespace-Normalized"]
        Z -- "Match Failed" --> AA["Re-prompt with Actual File Content"]
        AA --> X
        Z -- "Matched" --> AB["Multi-Occurrence Check"]
        AB -- "Ambiguous (>1 match)" --> AC["Reject: AMBIGUOUS_REPLACE_TARGET"]
        AC --> X
        AB -- "Unique Match" --> AD["Alpha-Normalize Variables (_v0, _v1)"]
        AD --> AE["Tabu Check: SHA256(Canonical AST) in History?"]
        AE -- "Cycle Detected" --> AF["Reject → Jitter Temperature → Negative Constraint"]
        AF --> X
        AE -- "New State" --> AG["Anti-Cheat Static Gate: ban-ts-comment + no-explicit-any"]
        AG -- "Lint Cheat Detected" --> AH["Reject Patch: Prohibit @ts-ignore"]
        AH --> X
        AG -- "Pass" --> AI["Apply Patch in Worktree"]
        AI --> AJ["Run Tests (Cache→/tmp, Tests :ro, 15s Timeout)"]
        AJ --> AK{"All Tests Pass?"}
        AK -- "No & File Retries < 2" --> AL["Log Hash to Tabu Set → Extract New Trace"]
        AL --> U
        AK -- "No & File Retries >= 2" --> AM{"More Affected Files in Queue?"}
        AM -- "Yes" --> AN1["Advance to Next Broken File"]
        AN1 --> U
        AM -- "No" --> AO1["Rollback → Prune Worktree → Failure Report"]
        AK -- "Yes" --> AP["tsc --noEmit Final Project Verification"]
        AP -- "Pass" --> AQ["Merge Worktree Commit → Clean Exit 0"]
        AP -- "Fail" --> AL
    end
```

---

### 2. Architectural Subsystem Deep Dives

#### 2.1 The `.d.ts` Declaration Diffing Algorithm

```
Algorithm: ExtractSymbolDeltas(oldPkg, newPkg)
─────────────────────────────────────────────
Input:  oldPkg = resolved path to old .d.ts declarations
        newPkg = resolved path to new .d.ts declarations
Output: SymbolDelta[] — list of structural breaking changes

1. RESOLVE entry points:
   For each package, resolve .d.ts path via:
     exports map → typesVersions → types field → index.d.ts fallback
   If package uses @types/<pkg>, resolve from DefinitelyTyped entry.

2. LOAD into ts-morph:
   oldProject = new Project(); oldSource = oldProject.addSourceFileAtPath(oldPath)
   newProject = new Project(); newSource = newProject.addSourceFileAtPath(newPath)

3. EXTRACT exported symbols:
   oldSymbols = Map<name, {kind, signature, params[], returnType}>
   newSymbols = Map<name, {kind, signature, params[], returnType}>

4. MATCH symbols:
   For each oldSym in oldSymbols:
     a. Exact name match in newSymbols → paired
     b. If no exact match, compute Levenshtein distance against all newSymbols
        If min distance / max(len) < 0.3 → probable rename, pair with note
     c. If no match → SYMBOL_REMOVED

   For each newSym not yet paired → SYMBOL_ADDED

5. DIFF each paired symbol:
   Compare parameter lists (count, names, types, optionality)
   Compare return types (sync vs Promise<T>)
   Compare generic type parameters
   Classify: PARAM_ADDED | PARAM_REMOVED | PARAM_REORDERED |
             RETURN_TYPE_CHANGED | RENAMED | SIGNATURE_MODIFIED

6. RETURN deltas where changeKind is a breaking change
```

#### 2.2 Behavioral Break Fallback Pipeline (Delta-Empty Mode)
When the Symbol Delta Table is empty (identical `.d.ts` signatures but runtime behavior changed):

1. **Classification:** Mark the break as `deltaKind: "behavioral"`.
2. **Context Assembly:** Use only:
   - Filtered stack trace (top application frame + assertion message).
   - Git tag diff between old and new library versions (extracted via `git log --oneline v8.5.1..v9.0.0` on the library's repo).
3. **Confidence Flagging:** Output report explicitly states: `"⚠️ No structural type delta detected. Repair based on runtime error analysis only. Confidence: LOW."`
4. **Separate Metric Reporting:** Results for delta-empty repairs are reported in a separate evaluation column to avoid inflating overall Pass@k metrics.

#### 2.3 Alpha-Normalized Tabu State Machine

```
Algorithm: IsSemanticDuplicate(candidatePatch, tabuSet)
──────────────────────────────────────────────────────
Input:  candidatePatch = modified source text
        tabuSet = Set<string> of previously seen hashes
Output: boolean

1. PARSE candidatePatch into AST via ts-morph
2. EXTRACT all locally declared identifiers within patched scope
3. SORT identifiers by first-appearance position
4. REPLACE each identifier with canonical token:
     1st declared → _v0, 2nd → _v1, 3rd → _v2, ...
5. STRIP all comments (line + block)
6. COLLAPSE all whitespace runs to single space
7. COMPUTE hash:
     H = SHA256(canonicalizedString)
8. IF H ∈ tabuSet → RETURN true (duplicate detected)
9. ELSE → add H to tabuSet, RETURN false
```

#### 2.4 Prompt Sanitization Pipeline

```
Algorithm: SanitizeCompilerOutput(rawOutput)
────────────────────────────────────────────
Input:  rawOutput = raw stdout/stderr from test runner
Output: sanitizedPayload (<100 tokens)

1. SPLIT rawOutput into lines
2. FILTER: Remove lines matching node_modules/*, node:internal/*
3. RETAIN: Top application stack frame + assertion failure line
4. For each retained line:
   a. Strip non-printable characters (ASCII < 32 except newline)
   b. Truncate to 200 characters
   c. Remove if matches injection regex:
      /(ignore|disregard).*(previous|above).*(instruction|prompt)/i
      /^(SYSTEM|INST|ASSISTANT):/i
5. WRAP result:
   <UNTRUSTED_COMPILER_OUTPUT>
   ...sanitized lines...
   </UNTRUSTED_COMPILER_OUTPUT>
6. RETURN wrapped payload
```

#### 2.5 Terminal UX Design

The CLI renders a live, interactive terminal experience:
```
$ deltamorph run

  ✓ Isolating workspace (git worktree)...
  ✓ Linked .env → .deltamorph-scratch/.env (NTFS Junction)
  ✓ Installing dependencies...
  ⚙ Running baseline tests...
  ✗ 3/47 tests failed

  📋 Breaking changes detected:
    • jwt.verify() — PARAMETER_SIGNATURE_MODIFIED (src/auth.ts:42)
    • axios.get() — CONFIG_RESTRUCTURED (src/api.ts:15)
    △ Type delta: 2 structural changes found

  ⚙ Synthesizing patch 1/3...
    Token usage: 287 / 350
  ✓ Patch generated (8 lines)
  ✓ Static checks passed (tsc ✓, eslint ✓)
  ⚙ Re-running tests...
  ✗ 1/47 tests still failing (src/api.test.ts:55)

  ⚙ Synthesizing patch 2/3...
    Tabu check: new unique state ✓
  ✓ Patch generated (12 lines)
  ⚙ Re-running tests...
  ✓ All tests pass! (47/47)

  ✓ Merging fix to main branch...
  ✓ Worktree cleaned up

  📊 Summary:
    Iterations: 2 / 3
    Time: 34s
    API cost: $0.003
    Delta type: structural (confidence: HIGH)
    Tabu states rejected: 0
```

Error recovery UX:
```
  ✗ Failed to create worktree: You have unstaged changes in src/
    → Run `git stash` or commit your changes first.

  ✗ LLM API unavailable (HTTP 503). Retrying with Groq Llama-3...
    → Waiting 5s for rate limit reset...

  ✗ Max iterations exhausted. No verified fix found.
    → Worktree pruned. Your working tree is untouched.
    → Failure report saved to .deltamorph-report.json
```
