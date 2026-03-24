# Chunk Manifest Schema

Reference schema cho dev-cycle chunk contracts. Mỗi chunk trong plan file phải follow format này.

## Schema

```markdown
### Chunk N: [tên ngắn gọn]

**Planning fields** (author viết khi planning):
- **Goal:** Mục tiêu cụ thể — chunk hoàn thành gì?
- **Files:** Danh sách files tạo mới hoặc sửa đổi
- **Non-goals:** KHÔNG làm gì trong chunk này (scope fence)
- **Dependencies:** Chunks phải xong trước (e.g., "Chunk 1")
- **Verification:** Lệnh test/check cụ thể (exact command, e.g., `npm test -- --grep "auth"`)
- **Done criteria:** Điều kiện cụ thể để pass (e.g., "all auth tests green + endpoint returns 200")
- **Risk class:** low | medium | high

**Runtime fields** (skill tự update trong quá trình thực thi):
- **Status:** pending | implementing | reviewing | approved
- **Review round:** N (current review iteration, starts at 0)
- **Unresolved issues:** [] (list of ISSUE-N IDs from current review round)
```

## Field Mapping — Manifest vs Plan YAML Header

Chunk state tồn tại ở 2 nơi. Bảng dưới xác định authority:

| Field | In Manifest (per chunk) | In Plan YAML Header | Authority |
|-------|------------------------|---------------------|-----------|
| Goal, Files, Non-goals, Dependencies, Verification, Done criteria, Risk class | ✅ Primary | ❌ | **Manifest** |
| Status | ✅ Quick reference | ✅ Canonical | **YAML Header** |
| Review round | ✅ Quick reference | ✅ Canonical | **YAML Header** |
| Unresolved issues | ✅ Quick reference | ✅ Canonical | **YAML Header** |
| Model overrides, escalations | ❌ | ✅ | **YAML Header** |

> **Rule:** Khi conflict → YAML header luôn wins. Manifest runtime fields là convenience copy, được update cùng lúc nhưng không phải SSOT.

## Example

```markdown
### Chunk 1: User authentication endpoint

**Planning fields:**
- **Goal:** Create POST /api/auth/login endpoint with JWT token generation
- **Files:** src/routes/auth.ts (new), src/middleware/auth.ts (new), tests/auth.test.ts (new)
- **Non-goals:** Registration, password reset, OAuth
- **Dependencies:** None (first chunk)
- **Verification:** `npm test -- --grep "auth"` + `curl -X POST localhost:3000/api/auth/login -d '{"email":"test@test.com","password":"test"}'`
- **Done criteria:** All auth tests green, login returns JWT, invalid credentials return 401
- **Risk class:** medium

**Runtime fields:**
- **Status:** pending
- **Review round:** 0
- **Unresolved issues:** []
```
