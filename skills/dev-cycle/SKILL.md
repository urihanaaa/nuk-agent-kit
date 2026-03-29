---
description: "Orchestrate full dev cycle for a User Story — plan, implement, review, finalize (v1-lite)"
disable-model-invocation: true
allowed-tools: Bash, Read, Write, Edit, Glob, Grep, Agent, Skill, TodoWrite, AskUserQuestion
---

# /dev-cycle — User Story Development Orchestrator (v1-lite)

## Purpose

Orchestrate toàn bộ dev cycle cho 1 User Story qua 5 stages: Preflight → Plan → Implement → Review/Fix → Finalize. Tự động gọi Codex review, enforce model policy, handle failures, và persist state cho cross-session resume.

## Arguments

- `/dev-cycle <User Story description>` — Start new dev cycle
- `/dev-cycle continue` — Resume interrupted dev cycle
- `/dev-cycle --force-sonnet <User Story>` — Start with Sonnet override (bypass Opus requirement)

Nếu không có argument → ask user: "Describe the User Story you want to implement."

## Stage Contracts

| Stage | Entry Conditions | Allowed Skip | Outputs | Next Stage |
|-------|-----------------|--------------|---------|------------|
| 0 Preflight | `/dev-cycle` invoked | Never | Tool inventory, degradation flags | → 1 (or resume target if `continue`) |
| 1 Plan | Stage 0 done, Opus active (or `--force-sonnet`) | Fast path: AI detects trivial US + user confirms → single implicit chunk → Stage 2 | Locked plan file with chunk manifests | → 2 (first chunk) |
| 2 Implement | Plan locked, current chunk manifest exists, prev chunk approved (if any) | Never | Chunk code + docs + tests pass + verification pass | → 3 |
| 3 Review/Fix | Stage 2 verification pass | Never | Chunk APPROVED + triage resolved | → 2 (next chunk) or → 4 (last chunk) |
| 4 Finalize | All chunks approved | Opus skip: 1-chunk → Sonnet finalize | Suggested commit, US complete | Done |

## Opus Model Policy (Single Authority)

**Detection:** Check conversation context for model indicator. Nếu không detect được → ask user: "Are you on Opus? (y/n)".

| Stage | Requirement | Override | How Recorded |
|-------|-------------|----------|--------------|
| 0 Preflight | Any | — | Log model in plan header |
| 1 Plan | 🔒 Opus | `--force-sonnet`: user must confirm "I accept reduced planning quality" | `OVERRIDE:force-sonnet:stage1` in plan header |
| 2 Implement | Sonnet (default) | Escalation → PAUSE + ask user to switch to Opus | `ESCALATION:opus-required:stage2:{reason}` in YAML header `escalations` array |
| 3 Review/Fix | Sonnet + Codex | Same escalation as Stage 2 | `ESCALATION:opus-required:stage3:{reason}` in YAML header `escalations` array |
| 4 Finalize (multi-chunk) | 🔒 Opus | Same as Stage 1: `--force-sonnet` + explicit confirm | `OVERRIDE:force-sonnet:stage4` in plan header |
| 4 Finalize (single-chunk) | Any | — | — |

> **Platform limitation:** Claude Code không có API chính thức detect model. Detection = heuristic (conversation context) + user confirmation.

---

## Stage 0 — Preflight

**Entry:** User invokes `/dev-cycle`.

### Actions:
1. **Tool inventory:** Check availability:
   ```
   codex: command -v codex || npx @openai/codex --version
   git_clean: git status --porcelain (empty = clean)
   tests_pass: run project test command if defined
   ```
2. **Skill inventory:** Check external skill availability (see `references/external-skills.md`):
   ```
   codex_plan_review: check if /codex-plan-review skill is loadable
   codex_impl_review: check if /codex-impl-review skill is loadable
   codex_security_review: check if /codex-security-review skill is loadable
   codex_think_about: check if /codex-think-about skill is loadable
   ```
   - For each missing REQUIRED skill → WARN (see `external-skills.md` Fallback column)
   - For each missing OPTIONAL skill → set degradation flag (e.g., `degrade_codex_plan_review: true`)
3. **Build inventory object:**
   ```
   Tool Inventory: {codex: true/false, git_clean: true/false, tests_pass: true/false/unknown}
   Skill Inventory: {codex_plan_review: true/false, codex_impl_review: true/false, codex_security_review: true/false, codex_think_about: true/false}
   ```
4. **Set degradation flags** (xem Failure Handling Matrix):
   - Codex not found → `degrade_codex: true` — **cascade:** also set `degrade_codex_plan_review: true`, `degrade_codex_impl_review: true`, `degrade_codex_security_review: true`, `degrade_codex_think_about: true` (CLI missing = all Codex-dependent skills unavailable)
   - Dirty worktree policy (depends on mode):
     - **New run (`/dev-cycle <US>`):** PAUSE — "Uncommitted changes detected. Please commit or stash before starting dev-cycle."
     - **Resume (`/dev-cycle continue`):** Dirty-worktree check is DEFERRED to Cross-Session Resume (after active plan is identified — see below).
   - Tests fail (pre-existing) → WARN + continue

5. **If `/dev-cycle continue`:** Jump to Cross-Session Resume flow (see below).

**Exit:** Tools verified, degradation flags set.

---

## Stage 1 — Plan (🔒 Opus Required)

**Entry:** Stage 0 done, Opus active (or `--force-sonnet`).

### Opus Check:
- Detect current model
- If NOT Opus AND no `--force-sonnet`:
  > ⚠️ **Planning requires Opus reasoning.**
  > Switch to Opus with `/model opus`, or re-run with `/dev-cycle --force-sonnet <US>` to override.
  - PAUSE — do not proceed until user switches or overrides

### Actions:
1. **Analyze User Story:** Break down requirements, identify scope, assess complexity
2. **Fast path check:** If trivial US (≤1 file, clear implementation, no architectural decisions):
   - Suggest: "This looks like a simple change. Skip detailed planning and go straight to implementation with 1 chunk?"
   - If user confirms:
     1. Create plan file (same format as step 8) with single chunk + `plan_status: locked`
     2. Set `current_stage: 2` in YAML header
     3. → go to Stage 2
   - **Note:** Fast path skips steps 3-7 but MUST still create the plan file (SSOT for resume + state tracking).
3. **Design approach:** Architecture decisions, file impact, risk assessment
4. **Define chunks** using manifest schema (see `references/chunk-manifest.md`):
   - Each chunk = independently implementable + reviewable unit
   - Define verification command per chunk
5. **Optional — Architecture debate:** If uncertainty detected AND `degrade_codex_think_about: false`, suggest:
   > "Would you like to run `/codex-think-about` to debate the architecture approach?"
   If degraded → skip suggestion, decide approach internally.
6. **Plan review:** If `degrade_codex_plan_review: false` → gọi `/codex-plan-review` (Skill tool). If degraded → self-review: re-read plan with fresh eyes, check for consistency/gaps.
   - Triage Codex issues: FIX / REBUT
   - Apply fixes to plan
7. **User confirmation:** Present final plan → user confirms → set `plan_status: locked`
8. **Create plan file:** `docs/plans/dev-cycle-{story-slug}-{YYYYMMDD}.md` with YAML header:
   ```yaml
   ---
   dev_cycle:
     story: "<US title>"
     plan_status: locked
     current_stage: 2
     current_chunk: 1
     model_overrides: []
     escalations: []
     chunks:
       - id: 1
         status: pending
         review_round: 0
         unresolved_issues: []
   ---
   ```
   > **Note:** `current_stage` is set to `2` (not `1`) because plan is already locked — next action is implementation. This ensures resume jumps to the correct stage.

**Exit:** Plan file locked with chunk manifests, `current_stage: 2`.

---

## Stage 2 — Implement Chunk (Sonnet)

**Entry:** Plan locked, current chunk manifest exists, prev chunk approved (if any).

### Actions:
1. **Update state:** Set `current_stage: 2` + `chunks[N].status: implementing` in plan file YAML header. Also capture **baseline diff snapshot**: record `git diff --name-only` + `git diff --cached --name-only` as `chunks[N].baseline_files` in YAML header (used by scope drift check in step 8 to distinguish pre-existing residue from new edits).
2. **Scope boundary reminder:** Output chunk scope to conversation + TodoWrite before coding:
   > 📋 **Chunk {N} scope:**
   > - **Goal:** {chunk.Goal}
   > - **Files:** {chunk.Files list}
   > - **Non-goals:** {chunk.Non-goals}
   > - ⚠️ New files outside this list will be flagged by the filename-level scope drift check (step 8).
3. **Implement:** Code + docstrings + inline comments in 1 pass
4. **Write tests:** Unit/integration tests for the chunk
5. **Run verification:** Execute verification command from chunk manifest
   - Pass → continue
   - Fail → auto-retry 1x → if still fails, PAUSE + ask user
6. **Security scan:** Pattern match against sensitive patterns:
   - SQL queries, user input handling, auth/crypto code, secrets/env vars
   - If match found AND `degrade_codex_security_review: false` → suggest: "Security-sensitive code detected. Run `/codex-security-review`?"
   - If degraded → WARN: "Security-sensitive patterns detected. Manual review recommended (Codex security review unavailable)."
7. **Escalation triggers → PAUSE + require Opus:**
   - Codex flags "architectural concern" during any review
   - Sonnet detects "approach uncertainty" (e.g., conflicting requirements, unclear API behavior)
   - User request
8. **Scope drift check** (automated — objective enforcement, không dựa vào self-discipline):
   - Collect **expected files** = union of:
     - Current chunk manifest `Files` list (files this chunk is ALLOWED to modify)
     - Plan file itself (`docs/plans/dev-cycle-*.md`) — always expected in diff (state mutations)
   - Collect **known residue** = files from all previously approved chunks (chunks where `status: approved`). These appear in `git diff` but are NOT modifiable by current chunk.
   - Run: `git diff --name-only` + `git diff --cached --name-only` (union staged + unstaged)
   - For each file in diff:
     - File ∈ expected files → ✅ OK (current chunk scope)
     - File ∈ known residue → check if file has NEW changes since chunk N started. Heuristic: if file was NOT in `git diff` at Stage 2 entry but IS in diff now → **SCOPE DRIFT** (current chunk modified an approved file without declaring it). If file was already in diff before → ✅ OK (pre-existing residue, not new edits).
     - File ∉ expected files AND ∉ known residue → **SCOPE DRIFT**
   - **Baseline capture:** At Stage 2 step 1, record `git diff --name-only` + `git diff --cached --name-only` as `baseline_files` in plan YAML header for current chunk. Use this to distinguish pre-existing vs new changes.
   - **Known limitation:** Baseline is filename-only, not content-aware. If an approved chunk file was already dirty at chunk start AND the current chunk adds new edits to the same file, the drift check treats it as pre-existing residue (false negative). This is accepted as rare in practice (chunks typically touch disjoint files) and avoids the complexity of per-file blob snapshots.
   - If drift detected → PAUSE:
     > ⚠️ **Scope drift:** Files changed outside expected scope:
     > - `{file_path}` (not in current or approved chunk manifests)
     >
     > Options:
     > 1. **Revert** these changes (recommended if truly out of scope)
     > 2. **Add to manifest** (if legitimately needed — update current chunk Files list + document reason in plan)
   - If no drift → continue to Stage 3 silently

**Exit:** Chunk code + docs + tests pass + verification pass + scope checked (best-effort, filename-level).

---

## Stage 3 — Review/Fix Chunk (Sonnet + Codex)

**Entry:** Stage 2 verification pass.

### Actions:
1. **Update state:** Set `current_stage: 3` + `chunks[N].status: reviewing` in plan file YAML header
2. **Invoke review:** If `degrade_codex_impl_review: false` → gọi `/codex-impl-review` (Skill tool) with **scope-aware SESSION_CONTEXT** injected from chunk manifest:
   ```
   Constraints: Changes SHOULD only touch files in scope (best-effort at filename granularity).
     Current chunk files (modifiable): {chunk.Files list}
     Plan file (expected mutations): {plan file path}
     Known residue (approved chunks, read-only): {list of Files from approved chunks}
     Non-goals: {chunk.Non-goals}
   Assumptions: File outside all listed scope = scope violation (ISSUE).
     Logic implementing non-goals items = scope creep (ISSUE).
     Known residue files appear in diff from prior chunks — only flag if NEW changes
     were introduced by THIS chunk (compare against baseline_files snapshot).
     Note: scope enforcement is filename-level. Edits to already-dirty approved files
     may not be detected (see SKILL.md known limitation).
   Tech stack: {infer from codebase}
   Acceptance criteria:
     1. No regressions, no new bugs, maintainable code (code quality)
     2. No new files outside current chunk scope + plan file (scope compliance — filename-level)
     3. No logic implementing non-goals items (scope compliance)
   ```
   If degraded → self-review: re-read diff, check scope compliance + code quality manually.
3. **Triage each issue:**
   - **FIX:** Apply fix immediately
   - **REBUT:** Explain why the issue is invalid, provide evidence
   - **DEFER:** Accept issue but defer to future chunk/US (document in plan)
   - **ESCALATE:** Flag as needing Opus — PAUSE
4. **Fix loop:** Max 5 rounds. After each round:
   - Update `chunks[N].review_round: N`
   - Update `chunks[N].unresolved_issues: [...]`
5. **Escalation triggers → PAUSE + require Opus:**
   - ESCALATE triage after 3 rounds non-convergence
   - Critical security findings
6. **On APPROVE:**
   - Set `chunks[N].status: approved`

**Exit:** Chunk approved, state updated in plan file.

**Next:** If more chunks → update `current_chunk` + `current_stage: 2` → back to Stage 2. If last chunk → set `current_stage: 4` → Stage 4.

---

## Stage 4 — Finalize (🔒 Opus Required for multi-chunk)

**Entry:** All chunks approved.

### Opus Check (multi-chunk only):
- If 2+ chunks AND not Opus → same PAUSE as Stage 1
- If 1 chunk → any model OK

### Actions:
1. **Multi-chunk coherence review:** (Opus) Review full `git diff` across all chunks:
   - Check cross-chunk consistency
   - Verify no orphan imports/references
   - Ensure tests cover integration points
2. **Suggest commit:**
   > Suggested commit message:
   > `feat: <US summary>`
   >
   > Commit? (y/n/edit)
3. **Mark complete:** Set `plan_status: completed` in plan file

**Exit:** US complete.

---

## Cross-Session Resume

When `/dev-cycle continue` is invoked:

### Locate plan file:
1. Scan `docs/plans/dev-cycle-*.md` for files with `dev_cycle:` YAML header where `plan_status != completed`
2. **1 match:** Resume automatically
3. **>1 match:** List candidates → ask user to choose:
   > Found multiple active dev-cycle plans:
   > 1. dev-cycle-add-auth-20260325.md (Stage 3, Chunk 2/3 reviewing)
   > 2. dev-cycle-fix-api-20260324.md (Stage 2, Chunk 1/1 implementing)
   > Which one to resume?
4. **0 match:** ABORT — "No active dev-cycle plan found. Start a new one with `/dev-cycle <US>`."

### Resume:
1. Read plan file YAML header → extract `current_stage`, `current_chunk`
2. **Dirty-worktree check (resume-specific):** Now that the target plan is known, use the **same scope model as Stage 2/3**:
   - Collect **plan-owned files** = union of:
     - All chunk `Files` lists (all chunks, regardless of status)
     - Plan file itself (`docs/plans/dev-cycle-*.md`)
   - Run: `git diff --name-only` + `git diff --cached --name-only` (union staged + unstaged)
   - If all changed files ∈ plan-owned files → WARN + continue
   - If unrelated changes detected → PAUSE: "Uncommitted changes detected that are NOT part of the active plan. Please commit or stash unrelated changes."
3. Rebuild TodoWrite state from plan data
4. Re-verify stage entry conditions (tools available, etc.)
5. Jump to current stage + chunk
6. If `chunks[N].status: reviewing` but no active Codex thread → restart review from round 1

### Reconciliation rules:
- **Plan file = SSOT.** TodoWrite = display only (rebuild from plan state)

---

## Failure Handling Matrix

| Failure | Detection Point | User Message | Action | Resume Path |
|---------|----------------|--------------|--------|-------------|
| Opus unavailable | Stage 1/4 entry | "Opus required for {stage}. Switch with `/model opus`, or use `--force-sonnet` to override." | PAUSE | User switches model or uses override |
| Codex CLI not found | Stage 0 | "⚠️ Codex CLI not found. All Codex-dependent capabilities disabled." | DEGRADE: set `degrade_codex: true` → cascade to all per-skill flags (`degrade_codex_plan_review`, `degrade_codex_impl_review`, `degrade_codex_security_review`, `degrade_codex_think_about`) | Continue without Codex |
| Optional skill missing (`/codex-*`) | Stage 0 | "⚠️ Skill `{name}` not found. Using fallback." | DEGRADE: set `degrade_{skill}: true`, use fallback per `external-skills.md` | Continue degraded |
| Dirty worktree (new run) | Stage 0 | "Uncommitted changes detected. Please commit or stash before starting." | PAUSE | User resolves → re-run `/dev-cycle` |
| Dirty worktree (resume) | Cross-Session Resume (after plan identified) | "Uncommitted changes detected outside active plan." | WARN if plan-related changes; PAUSE if unrelated changes | User commits unrelated → re-run `continue` |
| Tests fail (pre-existing) | Stage 0 | "⚠️ Existing tests are failing. Proceeding with caution." | WARN + continue | — |
| Tests fail (new code) | Stage 2 verify | "Verification failed: `{cmd}` → `{output}`" | PAUSE: auto-retry 1x → ask user if still failing | Fix code → re-run Stage 2 verification |
| Codex review timeout | Stage 1/3 poll | "Codex timed out after {N}s." | DEGRADE: use partial results if review.md exists, else skip review | Continue with warning |
| Plan file missing | `/dev-cycle continue` | "Cannot find active dev-cycle plan file." | ABORT | User provides path or starts fresh |
| Network/API error | Any stage | "API error: {details}" | PAUSE: auto-retry 1x (30s wait) → ask user | User retries or skips |

---

## State Management Summary

| Store | Purpose | Canonical? |
|-------|---------|------------|
| **Plan file** (docs/plans/dev-cycle-*.md) | Full state: stages, chunks, overrides, escalations | ✅ SSOT |
| **TodoWrite** | Live display of current progress | ❌ Display only, rebuilt from plan |
