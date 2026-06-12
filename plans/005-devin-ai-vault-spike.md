# Plan 005: Devin AI Vault integration

> **Executor instructions**: Phase A spike is **complete** (2026-06-13). Implement Phase B.
> Re-verify transcript paths on your machine if `DEVIN_HOME` or OS differs.
> Update `plans/README.md` when done.
>
> **Drift check**: `git diff --stat 002b92d23..HEAD -- src/shared/ai-vault-types.ts src/main/ai-vault/`

## Status

- **Priority**: P2
- **Effort**: M
- **Risk**: MED (Windows transcript path; optional SQLite enrichment)
- **Depends on**: none (can parallel 002/003)
- **Category**: direction
- **Planned at**: commit `002b92d23`, 2026-06-13 (spike updated 2026-06-13)

## Why this matters

AI Vault lists local agent sessions and builds resume commands. Devin stores
sessions on disk; users expect parity with Grok/Codex in the Vault panel.

## Spike results (Phase A — complete)

| Asset | Path |
|-------|------|
| Transcripts (ATIF v1.4 JSON) | `~/.local/share/devin/cli/transcripts/<session_id>.json` |
| Enrichment (optional) | `~/.local/share/devin/cli/sessions.db` → `sessions` (`id`, `working_directory`, `title`, `last_activity_at`, `hidden`) |
| Config / hooks | `~/.config/devin/config.json` (see plan 002) |
| Override | `DEVIN_HOME` env (mirror `GROK_HOME`) |

Transcript JSON: `session_id`, `agent.model_name`, `steps[]` with `metadata.created_at`, `metadata.metrics.*`, `metadata.is_user_input`, `tool_calls[]`.

Vault resume command: `cd '<cwd>' && devin --resume '<id>'` — add `devin` to the claude/grok arm in `buildAgentResumeInvocation` (space-separated `--resume`, not copilot's `--resume=`).

Repo: **no** existing devin fixtures in `tests/` or `src/main/ai-vault/`.

## Current state

`AI_VAULT_AGENTS` in `src/shared/ai-vault-types.ts` has no `devin`. Scanner has no
`DEVIN_TRANSCRIPTS_DIR` or `parseDevinSessionFile`. Model: `session-scanner-grok-parser.ts`.

## Commands you will need

| Purpose | Command | Expected |
|---------|---------|----------|
| Tests | `pnpm test -- session-scanner ai-vault` | exit 0 |
| Typecheck | `pnpm typecheck` | exit 0 |

## Scope

**In scope:**

- `src/shared/ai-vault-types.ts`
- `src/main/ai-vault/session-scanner-types.ts` — `devinTranscriptsDir?`, optional `devinSessionsDb?`
- `src/main/ai-vault/session-scanner-devin-parser.ts` (new)
- `src/main/ai-vault/session-scanner.ts`
- `src/main/ai-vault/session-scanner.test.ts` — minimal ATIF fixture in tmpdir

**Out of scope:**

- New native SQLite dependency solely for enrichment — use existing dep if present, else transcript-only v1
- SSH remote vault unless other agents already scan remote Devin home (match grok pattern if added later)

## Steps

### Step 1: Register agent in vault types

Add `'devin'` to `AI_VAULT_AGENTS` and `devin: 'Devin'` to `AI_VAULT_AGENT_LABELS`.
In `buildAgentResumeInvocation`, include `'devin'` in the `--resume <id>` switch arm with grok/claude.

**Verify:** `pnpm typecheck`

### Step 2: Discovery in session-scanner.ts

```typescript
const DEVIN_TRANSCRIPTS_DIR = join(
  process.env.DEVIN_HOME?.trim() || join(homedir(), '.local', 'share', 'devin', 'cli'),
  'transcripts'
)
```

Add `discoverFiles` for `agent: 'devin'`, `extensions: ['.json']` in the parallel discovery block.

**Verify:** compile

### Step 3: Parser (session-scanner-devin-parser.ts)

Mirror grok accumulator pattern: parse whole JSON, set `sessionId`, `model`, walk `steps[]`,
sum tokens from `metadata.metrics`, preview messages, `messageCount`, timeline from `created_at`.
Title from first user step text. cwd/title from transcript only unless Step 4 adds DB.

**Verify:** unit test with minimal fixture:

```json
{
  "session_id": "abc",
  "agent": { "model_name": "swe-1-6-fast" },
  "steps": [{ "metadata": { "created_at": "2026-01-01T00:00:00Z", "is_user_input": true, "metrics": { "input_tokens": 1, "output_tokens": 2 } } }]
}
```

### Step 4 (optional): sessions.db enrichment

If `better-sqlite3` (or project SQLite reader) already used elsewhere, join `sessions.id`
for `working_directory`, `title`, skip `hidden = 1`. Otherwise defer.

### Step 5: Integration test

Extend `session-scanner.test.ts` like grok: temp root, one transcript file, assert
`buildAiVaultResumeCommand({ agent: 'devin', sessionId: 'abc', cwd: '/r', platform: 'linux' })`
→ `cd '/r' && devin --resume 'abc'`.

## Done criteria

- [ ] `devin` in `AI_VAULT_AGENTS`
- [ ] Scanner discovers and parses fixture session
- [ ] Resume command test passes
- [ ] `pnpm test -- session-scanner` exit 0
- [ ] `plans/README.md` → 005 DONE

## STOP conditions

- Transcript path missing on executor machine — document OS-specific path from `devin` docs before blocking
- ATIF schema drift — update parser + fixture, do not guess fields

## Maintenance notes

- Read-only scan; never write Devin stores
- Windows: confirm `%LOCALAPPDATA%` vs `.local/share` for Devin data dir on maintainer machine