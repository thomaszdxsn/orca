# Plan 003: Devin sleeping-agent resume (`-c` / `-r`)

> **Executor instructions**: Follow this plan step by step. Run every
> verification command and confirm the expected result before moving to the
> next step. Honor STOP conditions. Update `plans/README.md` when done.
>
> **Drift check**: `git diff --stat 002b92d23..HEAD -- src/shared/agent-session-resume.ts src/shared/workspace-session-schema.ts`

## Status

- **Priority**: P2
- **Effort**: M
- **Risk**: MED (session id extraction must be correct)
- **Depends on**: `plans/002-devin-managed-agent-hooks.md` (recommended — session metadata from hooks)
- **Category**: direction
- **Planned at**: commit `002b92d23`, 2026-06-13

## Why this matters

Devin supports `devin -c` (continue last session in cwd) and `devin -r <SESSION_ID>`
([Commands & Flags](https://docs.devin.ai/cli/reference/commands)). Orca’s **sleeping
agent** flow persists `providerSession` and rebuilds argv via `getAgentResumeArgv`
for agents in `RESUMABLE_TUI_AGENTS`. Devin is absent today, so “resume Devin session”
from Orca UI does nothing.

## Current state

```5:13:src/shared/agent-session-resume.ts
export const RESUMABLE_TUI_AGENTS = [
  'claude',
  'codex',
  'gemini',
  'antigravity',
  'opencode',
  'droid',
  'grok'
] as const satisfies readonly TuiAgent[]
```

Grok resume argv (pattern to mirror):

```154:155:src/shared/agent-session-resume.ts
    case 'grok':
      return providerSession.key === 'session_id' ? ['grok', '--resume', id] : null
```

Devin resume argv (long form; matches grok-style `--resume`):

- Resume by id: `devin --resume <SESSION_ID>` (short `-r` in CLI docs)
- `devin -c` (continue in cwd) — **not** used for sleeping-agent revive (explicit `session_id` from hooks)

`src/shared/workspace-session-schema.ts` uses `z.enum(RESUMABLE_TUI_AGENTS)` for sleeping records — adding `devin` updates schema automatically.

## Commands you will need

| Purpose | Command | Expected |
|---------|---------|----------|
| Tests | `pnpm test -- agent-session-resume` | exit 0 |
| Typecheck | `pnpm typecheck` | exit 0 |

## Scope

**In scope:**

- `src/shared/agent-session-resume.ts`
- `src/shared/agent-session-resume.test.ts` (create or extend if missing)
- `src/shared/agent-hook-listener.ts` — only if session id extraction for `devin` is not already stored in `providerSession` after plan 002

**Out of scope:**

- AI Vault resume commands (`buildAiVaultResumeCommand`) — plan 005
- Changing `stdin-after-start` launch semantics

## Steps

### Step 1: Session id from hooks (after plan 002)

Devin hooks are Claude-compatible; payload should include `session_id`. In `extractAgentProviderSession`:

```typescript
case 'devin': {
  const id = readSessionId(payload, ['session_id', 'sessionId'])
  return id ? { key: 'session_id', id } : null
}
```

**STOP** only if plan 002 Step 0 proves otherwise.

### Step 2: Add `devin` to resumable agents

In `src/shared/agent-session-resume.ts`:

1. Append `'devin'` to `RESUMABLE_TUI_AGENTS` (after `'grok'`).
2. Add `case 'devin'` in `extractAgentProviderSession` (Step 1).
3. In `getAgentResumeArgv`:

```typescript
case 'devin':
  return providerSession.key === 'session_id'
    ? ['devin', '--resume', id]
    : null
```
**Do not** use `-c` for sleeping resume without explicit product spec.

**Verify:**

```bash
pnpm test -- agent-session-resume
```

### Step 3: Wire providerSession from hooks

If plan 002’s normalizer does not yet call `extractAgentProviderSession` for devin,
add the same call path Grok uses when persisting sleeping metadata
(search `extractAgentProviderSession` usages in listener / main process).

**Verify:** Manual — resume launches `devin --resume <id>` in PTY.

## Test plan

- `getAgentResumeArgv('devin', { key: 'session_id', id: 'abc12345' })` → `['devin', '--resume', 'abc12345']`
  - invalid key → `null`
- `isResumableTuiAgent('devin')` → true

## Done criteria

- [ ] `devin` ∈ `RESUMABLE_TUI_AGENTS`
- [ ] Tests above pass
- [ ] `pnpm typecheck` exit 0
- [ ] Manual QA resume once (documented in PR)
- [ ] `plans/README.md` → 003 DONE

## STOP conditions

- No session id in hook payloads after plan 002
- Resume argv breaks Windows quoting — use existing `buildAgentStartupPlan` / shell quoting paths, do not invent new launcher

## Maintenance notes

- If Devin changes flag spelling, update only `getAgentResumeArgv` and tests.
- `-c` vs `-r`: product may later add “resume last in worktree” without stored id.