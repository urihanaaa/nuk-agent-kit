# External Skills (Dependencies)

`/dev-cycle` gọi các skills bên ngoài qua `/skill` invocation (loose coupling). Các skills này phải được install riêng — KHÔNG bundled trong nuk-agent-kit.

## Optional Skills (Codex Integration)

| Skill | Purpose | Expected Install Path | Fallback if Missing |
|-------|---------|----------------------|---------------------|
| `/codex-plan-review` | Adversarial plan review (Stage 1) | `.claude/skills/codex-plan-review/` | DEGRADE: self-review only |
| `/codex-impl-review` | Code review + scope compliance via Codex (Stage 3) | `.claude/skills/codex-impl-review/` | DEGRADE: self-review only |
| `/codex-security-review` | Security audit on demand | `.claude/skills/codex-security-review/` | DEGRADE: skip security review |
| `/codex-think-about` | Architecture debate (Stage 1) | `.claude/skills/codex-think-about/` | DEGRADE: skip debate, decide internally |

## Dependencies

| Tool | Purpose | Check Command | Required? |
|------|---------|--------------|-----------|
| `codex` CLI | Run Codex reviews | `command -v codex \|\| npx @openai/codex --version` | Optional (DEGRADE if missing) |
| `git` | Version control, clean check | `git --version` | Required |

## Installation

Các skills trên đến từ **codex-review skill pack:** `/codex-plan-review`, `/codex-impl-review`, `/codex-security-review`, `/codex-think-about`

Xem README.md ở root repo để biết cách install/symlink.

> **Design note:** Loose coupling = `/dev-cycle` chỉ biết skill name + invocation contract. Nếu skill internal thay đổi, `/dev-cycle` không cần update. Fallback behavior đảm bảo `/dev-cycle` vẫn chạy được (degraded) khi thiếu optional skills.

## Invocation Contracts

### `/codex-impl-review` — Stage 3 Scope-Aware Review

`/dev-cycle` injects chunk manifest data into `SESSION_CONTEXT` khi gọi `/codex-impl-review`:

```
Constraints: Changes SHOULD only touch files in scope (best-effort, filename-level).
  Current chunk files (modifiable): {chunk.Files list}
  Plan file (expected mutations): {plan file path}
  Known residue (approved chunks, read-only): {approved chunks' Files}
  Non-goals: {chunk.Non-goals}
Assumptions: File outside all listed scope = scope violation (ISSUE).
  Logic implementing non-goals = scope creep (ISSUE).
  Known residue files appear in diff from prior chunks — only flag if NEW changes
  were introduced by THIS chunk (compare against baseline_files snapshot).
  Note: scope enforcement is filename-level; edits to already-dirty approved files
  may not be detected.
Acceptance criteria:
  1. Code quality (correctness, regressions, maintainability)
  2. No new files outside current chunk scope + plan file (filename-level scope)
  3. No non-goal logic (scope compliance)
```

**Expected behavior:** `/codex-impl-review` reports scope violations as standard `ISSUE-{N}` blocks alongside code quality findings. Không yêu cầu skill hiểu "scope" — chỉ cần follow `SESSION_CONTEXT` constraints/assumptions như với mọi review khác.
