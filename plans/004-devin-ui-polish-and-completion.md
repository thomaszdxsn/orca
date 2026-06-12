# Plan 004: Devin UI polish (title, search, completion coordinator)

> **Executor instructions**: Follow step by step; update `plans/README.md` when done.
>
> **Drift check**: `git diff --stat 002b92d23..HEAD -- src/renderer/src/components/terminal-pane/title-agent-identity.ts src/renderer/src/components/settings/general-search.ts src/renderer/src/components/terminal-pane/agent-completion-coordinator.test.ts`

## Status

- **Priority**: P3
- **Effort**: S
- **Risk**: LOW
- **Depends on**: `plans/002-devin-managed-agent-hooks.md` (completion coordinator meaningful once hooks identify devin)
- **Category**: dx
- **Planned at**: commit `002b92d23`, 2026-06-13

## Why this matters

Devin is registered in catalog and `WellKnownAgentType`, but several **secondary**
registries still omit `devin`, causing weaker tab-title detection, Settings search
misses, and hook-based completion tests not covering Devin.

## Current state

```7:8:src/renderer/src/components/terminal-pane/title-agent-identity.ts
const TITLE_AGENT_TOKEN_RE =
  /(?<![\w./\\-])(claude|openclaude|codex|gemini|antigravity|agy|opencode|openclaw|aider|copilot|cursor-agent|cursor|droid|hermes|grok|pi)(?![\w./\\-])/i
```

`src/renderer/src/components/settings/general-search.ts` includes grok keyword
(`c61b14be7c`) but not Devin catalog key `fc80296033`.

`agent-completion-coordinator.test.ts` lists hook agents including `grok` but not `devin`.

## Scope

**In scope:**

- `src/renderer/src/components/terminal-pane/title-agent-identity.ts`
- `src/renderer/src/components/settings/general-search.ts`
- `src/renderer/src/components/terminal-pane/agent-completion-coordinator.test.ts`
- `src/renderer/src/components/sidebar/worktree-title-derived-agent-rows.ts` (if Devin label missing — grep `Grok:`)
- `src/renderer/src/lib/use-tab-agent.ts` (same pattern)

**Out of scope:** synthetic terminal titles (`synthetic-agent-title.ts`) until Devin title behavior observed in Orca.

## Steps

### Step 1: Title token

Add `devin` to `TITLE_AGENT_TOKEN_RE` alternation (word boundary safe).

**Verify:** `pnpm test -- title-agent-identity` if exists; else `pnpm typecheck`.

### Step 2: Settings search keywords

Add Devin translate key to the same array as grok in `general-search.ts`:

`translateSearchKeyword('auto.lib.agent.catalog.fc80296033', 'Devin')`

**Verify:** `pnpm run verify:localization-catalog` (if keys already synced in 001, only search wiring needed).

### Step 3: Completion coordinator test parity

Add `'devin'` to the `it.each([...])` hook agent list in `agent-completion-coordinator.test.ts` next to `grok`.

**Verify:** `pnpm test -- agent-completion-coordinator`

### Step 4: Derived agent rows (if needed)

If `worktree-title-derived-agent-rows.ts` maps display names → ids, add `Devin: 'devin'` next to `Grok: 'grok'`. Mirror in `use-tab-agent.ts`.

**Verify:** `pnpm typecheck`

## Done criteria

- [ ] `devin` in title regex
- [ ] Settings search finds “Devin”
- [ ] Completion coordinator test includes devin
- [ ] `pnpm test` for touched tests pass
- [ ] `plans/README.md` → 004 DONE

## STOP conditions

- None expected; if title regex causes false positives (e.g. substring “devin” in paths), narrow with boundaries or skip title change and document.

## Maintenance notes

- Revisit `synthetic-agent-title.ts` after observing real Devin terminal titles post-hooks.