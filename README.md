# nuk-agent-kit

Workflow orchestration skills cho Claude Code. Tách biệt với [nuk-memory-kit](../nuk-memory-kit/) (memory management).

## Skills

| Skill | Description | Status |
|-------|-------------|--------|
| `/dev-cycle` | Orchestrate full dev cycle cho 1 User Story (v1-lite) | ✅ Ready |

## Installation

### Option A: Symlink (recommended)

Symlink skill folder vào `.claude/skills/` của project:

**PowerShell (Admin) hoặc cmd:**
```cmd
mklink /D "e:\coding project\.claude\skills\dev-cycle" "e:\coding project\nuk-agent-kit\skills\dev-cycle"
```

**Git Bash:**
```bash
ln -s "/e/coding project/nuk-agent-kit/skills/dev-cycle" "/e/coding project/.claude/skills/dev-cycle"
```

### Option B: Copy

**PowerShell:**
```powershell
Copy-Item -Recurse "e:\coding project\nuk-agent-kit\skills\dev-cycle" "e:\coding project\.claude\skills\dev-cycle"
```

### Verify Installation

Sau khi install, `/dev-cycle` sẽ xuất hiện trong Claude Code skill list. Test:
```
/dev-cycle Add hello world endpoint
```

## Dependencies

`/dev-cycle` gọi các external skills (loose coupling). Xem `skills/dev-cycle/references/external-skills.md` để biết chi tiết.

**Required:**
- nuk-memory-kit: `/memload`, `/memsave`

**Optional (Codex integration):**
- codex-review skill pack: `/codex-plan-review`, `/codex-impl-review`, `/codex-security-review`, `/codex-think-about`
- `codex` CLI: `npm install -g @openai/codex` hoặc dùng `npx`

> Nếu thiếu optional dependencies, `/dev-cycle` vẫn chạy ở chế độ degraded (self-review thay vì Codex review).

## Repo Structure

```
nuk-agent-kit/
├── README.md                              # This file
├── .gitignore
└── skills/
    └── dev-cycle/
        ├── SKILL.md                       # Main skill — 5-stage orchestrator
        └── references/
            ├── chunk-manifest.md          # Chunk contract schema + field mapping
            └── external-skills.md         # External skill dependency contracts
```

## Design Principles

1. **Skill format:** SKILL.md = pure Markdown + YAML frontmatter. Không cần runtime code.
2. **Loose coupling:** Gọi external skills qua name, không import. Fallback khi thiếu.
3. **Plan file as SSOT:** Toàn bộ state persist trong plan file YAML header → cross-session resume.
4. **Graceful degradation:** Thiếu Codex → self-review. Thiếu Opus → `--force-sonnet`. Thiếu memsave → warn.
5. **Opus policy:** Planning (Stage 1) + Finalize (Stage 4 multi-chunk) require Opus. Override possible with explicit user consent.
