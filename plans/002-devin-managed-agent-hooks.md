# Plan 002: Wire Devin CLI into Orca managed agent hooks

> **Executor instructions**: Follow this plan step by step. Run every
> verification command and confirm the expected result before moving to the
> next step. If anything in the "STOP conditions" section occurs, stop and
> report — do not improvise. When done, update the status row for this plan
> in `plans/README.md`.
>
> **Drift check (run first)**: `git diff --stat 002b92d23..HEAD -- src/main/devin src/shared/agent-hook-types.ts src/shared/agent-hook-relay.ts src/main/agent-hooks/managed-agent-hook-controls.ts`
> If in-scope files changed since this plan was written, compare "Current state"
> excerpts against live code; on mismatch, STOP.

## Status

- **Priority**: P1
- **Effort**: L
- **Risk**: MED (hook wire format must match Devin CLI; Windows config path)
- **Depends on**: none (baseline Devin TUI registration landed in plan 001 / PR #5282)
- **Category**: direction
- **Planned at**: commit `002b92d23`, 2026-06-13

## Why this matters

Without managed hooks, Orca cannot show Devin **working / waiting / tool** state,
completion notifications, or sleeping-agent session metadata. Devin CLI supports
**Claude Code–compatible** lifecycle hooks ([Hooks overview](https://docs.devin.ai/cli/extensibility/hooks/overview));
Orca already installs analogous scripts for Grok and Claude. This plan closes the
largest gap between “can launch Devin” and “Devin feels like other hook-backed agents.”

**Critical (explore 2026-06-13):** Installing hooks only into `~/.claude/settings.json` is **insufficient** — Devin would POST to `/hook/claude` and show as Claude in the UI. Orca must install a **dedicated** managed script posting to **`/hook/devin`** and write hooks under **`~/.config/devin/config.json`** (Windows: `%APPDATA%\devin\config.json`). Implement **`normalizeDevinEvent`** (clone of Claude normalizer with `agentType: 'devin'`), not reuse `normalizeClaudeEvent` without agent type override.

## Current state

- `src/shared/agent-hook-types.ts:6-19` — `AGENT_HOOK_TARGETS` has no `devin`.
- `src/main/agent-hooks/managed-agent-hook-controls.ts:21-34` — no devin installer.
- `src/main/agent-hooks/remote-managed-hook-installers.ts:19-29` — no remote devin.
- `src/shared/agent-hook-relay.ts:34-48` — `AgentHookSource` has no `devin`.
- `src/shared/agent-hook-listener.ts:2945-2959` — `HOOK_SOURCE_BY_PATHNAME` has no `/hook/devin`.
- `src/main/agent-hooks/installer-utils.ts:101-107` — `buildWindowsAgentHookPostCommand(source)` requires `AgentHookSource`; extend when adding `devin`.

**Exemplar (Grok — dedicated hooks JSON file):**

```42:47:src/main/grok/hook-service.ts
function getConfigPath(): string {
  // Why: Grok loads trusted global hook files from ~/.grok/hooks/*.json. Keep
  // Orca's managed entries in a dedicated file so user-authored hook files stay
  // untouched and project-level trust is not required for status reporting.
  return join(homedir(), '.grok', 'hooks', 'orca-status.json')
}
```

**Exemplar (Claude — settings.json + shared applyManagedHooks):**

```17:20:src/main/claude/hook-settings.ts
export const CLAUDE_HOOK_SETTINGS: ClaudeCompatibleHookSettings = {
  configDirName: '.claude',
  scriptBaseName: 'claude-hook'
}
```

```27:30:src/main/claude/hook-settings.ts
export const CLAUDE_EVENTS = [
  { eventName: 'UserPromptSubmit', definition: { hooks: [{ type: 'command', command: '' }] } },
  { eventName: 'Stop', definition: { hooks: [{ type: 'command', command: '' }] } },
```

**Devin official config (installer target):**

- User: `~/.config/devin/config.json` (Linux/macOS); Windows `%APPDATA%\devin\config.json` per [Configuration File](https://docs.devin.ai/cli/reference/configuration/config-file).
- Project hooks also exist (`.devin/hooks.v1.json`); Orca should use **user config `"hooks"` key** (same pattern as Claude `settings.json`) so SSH remote install has one stable path under `$HOME`.
- Devin events to mirror for status UI (align with Grok): `SessionStart`, `UserPromptSubmit`, `Stop`, `SessionEnd` ([lifecycle hooks](https://docs.devin.ai/cli/extensibility/hooks/lifecycle-hooks)).

**Managed script:** copy structure from `src/main/claude/hook-service.ts` `getManagedScript()` (posts to Orca hook server with `source=devin`) or reuse `buildWindowsAgentHookPostCommand('devin')` once `AgentHookSource` includes `devin`.

## Commands you will need

| Purpose   | Command | Expected on success |
|-----------|---------|---------------------|
| Typecheck | `pnpm typecheck` | exit 0 |
| Hook tests | `pnpm test -- hook-service agent-hook` | exit 0 |
| Lint | `pnpm lint` | exit 0 |

(Node 24 per `package.json` `engines`.)

## Scope

**In scope:**

- `src/main/devin/hook-settings.ts` (new)
- `src/main/devin/hook-service.ts` (new)
- `src/main/devin/hook-service.test.ts` (new, model after `src/main/grok/hook-service.test.ts`)
- `src/shared/agent-hook-types.ts`
- `src/shared/agent-hook-relay.ts`
- `src/shared/agent-hook-listener.ts` (routing + pathname only in this plan)
- `src/main/agent-hooks/managed-agent-hook-controls.ts`
- `src/main/agent-hooks/remote-managed-hook-installers.ts`
- `src/main/ipc/agent-hooks.ts`
- `src/main/ipc/agent-hooks.test.ts`
- `src/preload/api-types.ts`, `src/preload/index.ts`
- `src/renderer/src/web/web-preload-api.ts` (if other agents expose `*Status` there)

**Out of scope:**

- `src/shared/agent-session-resume.ts` — optional small addition in 002 only if hooks team wants session_id in same PR; else plan 003
- Renderer hook status preload — match `grokStatus` / `claudeStatus` pattern in `preload/index.ts`

## Git workflow

- Branch: `feat/devin-managed-hooks` from `feat/add-devin-agent` or `main` after PR merge
- Commits: `feat(hooks): install Devin managed status hooks`

## Steps

### Step 0: Confirm hook stdin (lightweight)

Docs state Claude Code hook compatibility. Expect `hook_event_name`, `session_id`, `prompt`, `tool_name` / `tool_input`. One manual capture still recommended before merging listener logic.

**STOP** only if payload shape diverges from Claude.

### Step 1: Add `devin` to shared hook registries

1. Append `'devin'` to `AGENT_HOOK_TARGETS` in `src/shared/agent-hook-types.ts`.
2. Append `| 'devin'` to `AgentHookSource` in `src/shared/agent-hook-relay.ts`.
3. Add `'/hook/devin': 'devin'` to `HOOK_SOURCE_BY_PATHNAME` in `src/shared/agent-hook-listener.ts`.

**Verify:** `pnpm typecheck` → errors until installers exist; proceed to Step 2.

### Step 2: Implement `devinHookService`

Create `src/main/devin/hook-service.ts` by copying **`src/main/claude/hook-service.ts`** structure:

- Config: `join(homedir(), '.config', 'devin', 'config.json')` (POSIX); `%APPDATA%\devin\config.json` (Windows).
- Script: `devin-hook.sh` / `devin-hook.cmd` under `~/.orca/agent-hooks/`.
- Managed script POST URL: **`/hook/devin`** via `buildWindowsAgentHookPostCommand('devin')`.
- Reuse `CLAUDE_EVENTS` + `applyManagedHooks` / `removeManagedHooks` from `claude/hook-settings.ts` on **`config.hooks`** (top-level Devin config).
- `installRemote`: `$HOME/.config/devin/config.json` + `$HOME/.orca/agent-hooks/devin-hook.sh`.

**Verify:** `pnpm test -- src/main/devin/hook-service.test.ts` → pass (status partial/installed paths).

### Step 3: Listener — `normalizeDevinEvent`

In `src/shared/agent-hook-listener.ts`:

- `HOOK_SOURCE_BY_PATHNAME`: `'/hook/devin': 'devin'`
- `isNewTurnEvent`: `case 'devin': return eventName === 'UserPromptSubmit'`
- `extractToolFields`: `case 'devin': return extractClaudeToolFields(...)`
- Add **`normalizeDevinEvent`** beside `normalizeClaudeEvent` — same state mapping (`UserPromptSubmit`/`PreToolUse` → working, `PermissionRequest` → waiting, `Stop`/`SessionEnd` → done) with **`agentType: 'devin'`**
- Dispatch: `case 'devin': payload = normalizeDevinEvent(...)`

**Verify:** `pnpm test -- agent-hook-listener` → pass.

### Step 4: Register installers and IPC

- `managed-agent-hook-controls.ts`: import `devinHookService`, add to install/remove/status arrays.
- `remote-managed-hook-installers.ts`: add `['devin', (sftp, home) => devinHookService.installRemote(sftp, home)]`.
- `ipc/agent-hooks.ts`: `agentHooks:devinStatus` handler (copy `grokStatus` pattern).
- Preload + `web-preload-api.ts`: `devinStatus()` if parity with grok.

**Verify:**

```bash
pnpm typecheck && pnpm test -- managed-agent-hook agent-hooks
```

### Step 5: Manual QA (local + SSH if available)

1. Enable agent status hooks in Settings (default on).
2. Run “reinstall hooks” or restart Orca; confirm `devin` shows **installed** in hook status.
3. Launch Devin in a worktree PTY; confirm status bar / agent dot transitions on prompt + stop.
4. On SSH worktree: confirm remote install writes `~/.config/devin/config.json` hooks and events reach desktop UI.

**Verify:** Record results in PR or plan status; no automated e2e required in this plan.

## Test plan

- New `src/main/devin/hook-service.test.ts` — install/remove/status on temp config dir (mock homedir or inject path if existing tests use temp dirs).
- Extend `src/main/ipc/agent-hooks.test.ts` mock for `devinHookService`.
- Optional: unit test that `HOOK_SOURCE_BY_PATHNAME['/hook/devin'] === 'devin'`.

## Done criteria

- [ ] `pnpm typecheck` exit 0
- [ ] `pnpm test -- devin/hook-service agent-hooks managed-agent` exit 0
- [ ] `AGENT_HOOK_TARGETS` includes `devin`
- [ ] Local `devinHookService.install()` writes managed command into user Devin config without deleting unrelated keys
- [ ] `plans/README.md` row for 002 → DONE

## STOP conditions

- Step 0 shows Devin hook stdin is **not** Claude-compatible → stop before Step 3; file issue with payload sample.
- Devin config on Windows cannot be resolved in CI/dev → stop and document; do not ship Linux-only without maintainer approval.
- `readHooksJson` cannot parse Devin config (JSON with comments) → check whether Devin uses JSONC; if yes, use same reader as other agents or add parser per maintainer direction.

## Maintenance notes

- Devin `read_config_from.claude` may duplicate hooks if users also have Claude settings; Orca only owns entries matching `devin-hook.*` managed matcher.
- Bump `ORCA_HOOK_PROTOCOL_VERSION` only if managed script POST body changes (same policy as `agent-hook-types.ts` comment).
- Follow-up: plan 003 (resume) depends on session id in hook payloads or terminal metadata.