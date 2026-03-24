# External Skills (Dependencies)

`/dev-cycle` gọi các skills bên ngoài qua `/skill` invocation (loose coupling). Các skills này phải được install riêng — KHÔNG bundled trong nuk-agent-kit.

## Required Skills

| Skill | Purpose | Expected Install Path | Fallback if Missing |
|-------|---------|----------------------|---------------------|
| `/memload` | Load 4-layer memory context | `.claude/skills/memload/` | WARN: skip memory load, continue without context |
| `/memsave` | Save session + daily memory | `.claude/skills/memsave/` | WARN: manual save needed. Flag `memsave_failed` |

## Optional Skills (Codex Integration)

| Skill | Purpose | Expected Install Path | Fallback if Missing |
|-------|---------|----------------------|---------------------|
| `/codex-plan-review` | Adversarial plan review (Stage 1) | `.claude/skills/codex-plan-review/` | DEGRADE: self-review only |
| `/codex-impl-review` | Code review via Codex (Stage 3) | `.claude/skills/codex-impl-review/` | DEGRADE: self-review only |
| `/codex-security-review` | Security audit on demand | `.claude/skills/codex-security-review/` | DEGRADE: skip security review |
| `/codex-think-about` | Architecture debate (Stage 1) | `.claude/skills/codex-think-about/` | DEGRADE: skip debate, decide internally |

## Dependencies

| Tool | Purpose | Check Command | Required? |
|------|---------|--------------|-----------|
| `codex` CLI | Run Codex reviews | `command -v codex \|\| npx @openai/codex --version` | Optional (DEGRADE if missing) |
| `git` | Version control, clean check | `git --version` | Required |

## Installation

Các skills trên đến từ 2 repos:
- **nuk-memory-kit:** `/memload`, `/memsave`
- **codex-review skill pack:** `/codex-plan-review`, `/codex-impl-review`, `/codex-security-review`, `/codex-think-about`

Xem README.md ở root repo để biết cách install/symlink.

> **Design note:** Loose coupling = `/dev-cycle` chỉ biết skill name + invocation contract. Nếu skill internal thay đổi, `/dev-cycle` không cần update. Fallback behavior đảm bảo `/dev-cycle` vẫn chạy được (degraded) khi thiếu optional skills.
