# Plan 001: Add Devin CLI as a first-class supported TUI agent

> **Executor instructions**: Follow this plan step by step. Run every
> verification command and confirm the expected result before moving to the
> next step. If anything in the "STOP conditions" section occurs, stop and
> report — do not improvise. When done, update the status row for this plan
> in `plans/README.md` to `DONE` — unless a reviewer dispatched you and told
> they maintain the index.
>
> **Drift check (Step 0)**: see Step 0 below. On mismatch with "Current state"
> excerpts, treat it as a STOP condition.

## Status

- **Priority**: P1
- **Effort**: M (was S — raised after review; six additional exhaustive-registration files)
- **Risk**: LOW
- **Depends on**: none
- **Category**: direction (new feature — adding a supported agent)
- **Planned at**: commit `dc8fdf65a`, 2026-06-12 (reviewed 2026-06-13)
- **Issue**: https://github.com/stablyai/orca/issues/5246

## Why this matters

Devin CLI is a prominent Rust-based AI coding agent by Cognition with a
growing user base. Adding it as a first-class supported agent lets Orca users
run Devin in parallel worktrees alongside Claude Code, Codex, Gemini, etc. —
a core Orca value proposition. The maintainer has explicitly welcomed a PR
(see issue comment from @AmethystLiang).

The issue's suggested config (`promptInjectionMode: 'argv'`) is **incorrect**.
The claim that `devin -- <prompt>` "pre-fills the input without submitting" is
wrong — per the official Devin CLI docs, `devin -- your prompt here` starts the
REPL and auto-submits the prompt. The correct mode is `'stdin-after-start'`,
which launches the REPL first, then pastes the prompt via bracketed paste so
the user can review before submitting. This matches how similar agents (aider,
goose, amp, kilo, crush, cline, grok) are configured in Orca.

## Current state

### Files and their roles

- `src/shared/types.ts` — defines the `TuiAgent` union type (L1933-1964).
  Every supported agent is a string literal in this union.
- `src/shared/tui-agent-config.ts` — defines `TUI_AGENT_CONFIG` as an
  **exhaustive** `Record<TuiAgent, TuiAgentConfig>` (L54-305). Also exports
  `isTuiAgent()` (L307), which checks membership via config keys.
- `src/shared/tui-agent-selection.ts` — defines `TUI_AGENT_AUTO_PICK_ORDER`
  (L6-38). Comment at L4: "Keep this order in sync with the desktop agent
  catalog." Also exports `pickTuiAgent()`.
- `src/shared/tui-agent-launch-defaults.ts` — defines `DEFAULT_TUI_AGENT_ARGS`
  (L9-32) with per-agent default CLI flags.
- `src/shared/agent-status-types.ts` — defines `WellKnownAgentType` union
  (L16-33) for agent status hook matching.
- `src/shared/agent-kind.ts` — maps every `TuiAgent` to a telemetry `AgentKind`
  via `TUI_AGENT_KIND_BY_AGENT` with `satisfies Record<TuiAgent, …>` (L16-48).
  **Compile fails if any TuiAgent is missing.**
- `src/shared/telemetry-events.ts` — closed `AGENT_KIND_VALUES` enum (L71-104).
  Every concrete agent kind in `agent-kind.ts` must appear here.
- `src/renderer/src/lib/agent-catalog.tsx` — desktop agent picker catalog.
  `getAgentCatalog()` order must match `TUI_AGENT_AUTO_PICK_ORDER` (enforced by
  `src/renderer/src/lib/quick-workspace-agent-selection.test.ts`).
- `src/renderer/src/lib/agent-status.ts` — `ICONABLE_AGENT_TYPES: Record<TuiAgent, true>`
  (L153-185) and `WELL_KNOWN_LABELS` (L113-130). **Compile fails if any
  TuiAgent is missing from the icon record.**
- `mobile/src/tasks/mobile-tui-agents.ts` — mobile mirror of auto-pick order,
  labels, and launch commands. Parity enforced by
  `mobile/src/tasks/mobile-agent-catalog.test.ts`.
- `src/shared/tui-agent-startup.test.ts` — tests `buildAgentStartupPlan` shapes.
- `README.md` — lists supported agents with logos (L170-204). Order here is
  **independent** of `TUI_AGENT_AUTO_PICK_ORDER` (marketing layout).

### Exemplar: how Grok was registered (commit `e7f3bb603`)

Grok is the most recent analogous addition (no hook relay in the minimal path).
Match its pattern for shared + catalog + telemetry registration:

```typescript
// src/shared/agent-kind.ts — append before closing brace
  grok: 'grok'
// becomes:
  grok: 'grok',
  devin: 'devin'

// src/shared/telemetry-events.ts — append before 'other'
  'grok',
// becomes:
  'grok',
  'devin',

// src/renderer/src/lib/agent-catalog.tsx — after hermes entry, before openclaw
  {
    id: 'devin',
    label: translate('auto.lib.agent.catalog.<KEY>', 'Devin'),
    cmd: 'devin',
    faviconDomain: 'devin.ai',
    homepageUrl: 'https://devin.ai/cli'
  },
```

### Current TuiAgent type (src/shared/types.ts:1933-1964)

```typescript
export type TuiAgent =
  | 'claude' // Claude Code
  // ... (omitted for brevity)
  | 'copilot' // GitHub Copilot CLI
  | 'grok' // xAI Grok CLI
```

### Current TUI_AGENT_AUTO_PICK_ORDER tail (src/shared/tui-agent-selection.ts:33-38)

```typescript
  'qwen-code',
  'rovo',
  'hermes',
  'openclaw'
] as const satisfies readonly TuiAgent[]
```

### Current DEFAULT_TUI_AGENT_ARGS tail (src/shared/tui-agent-launch-defaults.ts:28-32)

```typescript
  rovo: '--yolo',
  hermes: '--yolo',
  copilot: '--yolo',
  grok: '--permission-mode bypassPermissions'
}
```

### Repo conventions

- **Comments**: explain *why*, not *what*. 1-2 lines max. Reference upstream
  docs/PRs when the behavior is non-obvious.
- **Catalog ↔ auto-pick sync**: `TUI_AGENT_AUTO_PICK_ORDER` and
  `getAgentCatalog()` entry order must be identical. Insert `devin` after
  `hermes`, before `openclaw` in both.
- **README order**: independent marketing layout — insert Devin badge after
  Hermes Agent (L187), before Goose (L188).
- **Localization**: agent catalog labels use `translate('auto.lib.agent.catalog.<KEY>', 'Devin')`.
  Add `'Devin'` to `NEVER_TRANSLATE_VALUES` in
  `config/scripts/locale-translation-policy.mjs`, then run
  `pnpm run sync:localization-catalog` to populate `en.json` and repair locale
  parity.
- **Type declarations**: `.ts` not `.d.ts` (project rule in AGENTS.md).

## Commands you will need

| Purpose        | Command                              | Expected on success       |
|----------------|--------------------------------------|---------------------------|
| Install        | `pnpm install`                       | exit 0                    |
| Typecheck      | `pnpm typecheck`                     | exit 0, no errors         |
| Agent tests    | `pnpm test -- tui-agent`             | all pass                  |
| Startup test   | `pnpm test -- tui-agent-startup`     | all pass                  |
| Lint           | `pnpm lint`                          | exit 0                    |
| Full test      | `pnpm test`                          | all pass                  |
| Localization   | `pnpm run sync:localization-catalog` | exit 0, keys added        |

## Scope

**In scope** (the only files you should modify):

- `src/shared/types.ts`
- `src/shared/tui-agent-config.ts`
- `src/shared/tui-agent-selection.ts`
- `src/shared/tui-agent-launch-defaults.ts`
- `src/shared/agent-status-types.ts`
- `src/shared/agent-kind.ts`
- `src/shared/telemetry-events.ts`
- `src/renderer/src/lib/agent-catalog.tsx`
- `src/renderer/src/lib/agent-status.ts`
- `mobile/src/tasks/mobile-tui-agents.ts`
- `config/scripts/locale-translation-policy.mjs`
- `src/renderer/src/i18n/locales/*.json` (via `sync:localization-catalog` only)
- `src/shared/tui-agent-startup.test.ts` (append one new `it` block only)
- `README.md`

**Out of scope** (do NOT touch, even though they look related):

- `src/shared/agent-session-resume.ts` — session resume requires hook relay
  integration to extract Devin session IDs from agent output. Devin supports
  `-c` (continue) and `-r <session_id>` (resume), but wiring this up is a
  separate follow-up.
- `src/shared/synthetic-agent-title.ts` — synthetic titles are only added
  after observing an agent's actual terminal title behavior.
- `src/shared/tui-agent-startup.ts` — no changes needed; existing
  `buildAgentStartupPlan` handles `stdin-after-start` generically.
- `src/shared/agent-hook-relay.ts`, `src/shared/agent-hook-listener.ts`,
  `src/main/**` — hook integration is a separate effort (see Grok commit for
  the full hook path; not required for basic launch/detection).
- `src/shared/ai-vault-types.ts` — Devin vault scanning is a separate follow-up.
- Any other renderer/UI files beyond those listed in scope.

## Git workflow

- Branch: `feat/add-devin-agent` (create/checkout before Step 1)
- Commit style: conventional commits — e.g. `feat: add Devin CLI as supported TUI agent`
- Do NOT push or open a PR unless the operator instructed it.

## Steps

### Step 0: Drift check

**Run**:

```bash
git diff --stat dc8fdf65a..HEAD -- \
  src/shared/types.ts \
  src/shared/tui-agent-config.ts \
  src/shared/tui-agent-selection.ts \
  src/shared/tui-agent-launch-defaults.ts \
  src/shared/agent-status-types.ts \
  src/shared/agent-kind.ts \
  src/shared/telemetry-events.ts \
  src/renderer/src/lib/agent-catalog.tsx \
  src/renderer/src/lib/agent-status.ts \
  mobile/src/tasks/mobile-tui-agents.ts \
  README.md
```

**Verify**: empty output (no diff). If any listed file changed, open it and
compare against the "Current state" excerpts above. On structural mismatch
(e.g. `TuiAgent` refactored, catalog moved), STOP and report.

### Step 1: Add `'devin'` to the `TuiAgent` union type

**File**: `src/shared/types.ts`

After `| 'grok' // xAI Grok CLI` (L1964), add:

```typescript
  | 'devin' // Devin CLI
```

**Verify**: `pnpm typecheck` → **non-zero exit expected** until Steps 2–6
complete (exhaustive records now reference a missing member). Do not proceed
past Step 6 until typecheck passes.

### Step 2: Add Devin config to `TUI_AGENT_CONFIG`

**File**: `src/shared/tui-agent-config.ts`

After the `grok` entry (ends L304), add:

```typescript
  devin: {
    detectCmd: 'devin',
    launchCmd: 'devin',
    expectedProcess: 'devin',
    // Why: `devin -- <prompt>` auto-submits the prompt (the issue's claim
    // that it pre-fills without submitting is incorrect per the official
    // docs at docs.devin.ai/cli/reference/commands). `stdin-after-start`
    // launches the REPL first, then pastes via bracketed paste so the
    // user can review before submitting — same as aider, goose, amp, etc.
    promptInjectionMode: 'stdin-after-start'
  }
```

### Step 3: Register Devin in telemetry mapping

**File**: `src/shared/telemetry-events.ts`

In `AGENT_KIND_VALUES`, add `'devin'` after `'grok'`, before `'other'`.

**File**: `src/shared/agent-kind.ts`

In `TUI_AGENT_KIND_BY_AGENT`, add `devin: 'devin'` after `grok: 'grok'`.

**Verify**: `pnpm test -- agent-kind` → all pass (maps every config key to a
concrete telemetry kind).

### Step 4: Add `'devin'` to auto-pick order (desktop + mobile)

**File**: `src/shared/tui-agent-selection.ts`

In `TUI_AGENT_AUTO_PICK_ORDER`, insert `'devin'` after `'hermes'`, before
`'openclaw'`:

```typescript
  'hermes',
  'devin',
  'openclaw'
```

**File**: `mobile/src/tasks/mobile-tui-agents.ts`

Mirror the same insertion in `MOBILE_TUI_AGENT_AUTO_PICK_ORDER`. Also add to
the exhaustive records in the same file:

```typescript
// MOBILE_TUI_AGENT_LABELS
  devin: 'Devin',

// MOBILE_TUI_AGENT_FAVICON_DOMAINS (Partial — optional but recommended)
  devin: 'devin.ai',

// MOBILE_TUI_AGENT_LAUNCH_COMMANDS
  devin: 'devin',
```

### Step 5: Add Devin to desktop agent catalog and icon registry

**File**: `src/renderer/src/lib/agent-catalog.tsx`

Insert a new catalog entry **after the `hermes` entry, before `openclaw`**:

```typescript
  {
    id: 'devin',
    label: translate('auto.lib.agent.catalog.PLACEHOLDER', 'Devin'),
    cmd: 'devin',
    faviconDomain: 'devin.ai',
    homepageUrl: 'https://devin.ai/cli'
  },
```

Use any unique 10-character hex suffix for `PLACEHOLDER` (e.g. generate with
`openssl rand -hex 5`); `sync:localization-catalog` will register the key.

**File**: `src/renderer/src/lib/agent-status.ts`

Add to both maps:

```typescript
// WELL_KNOWN_LABELS
  devin: 'Devin',

// ICONABLE_AGENT_TYPES
  devin: true,
```

**Verify**: `pnpm test -- quick-workspace-agent-selection` → the catalog sync
test passes.

### Step 6: Add Devin default args and well-known status type

**File**: `src/shared/tui-agent-launch-defaults.ts`

In `DEFAULT_TUI_AGENT_ARGS`, after `grok: '--permission-mode bypassPermissions'`, add:

```typescript
  devin: '--permission-mode bypass'
```

Devin's `--permission-mode bypass` auto-approves all tool calls (see
docs.devin.ai/cli/reference/commands — "Bypass" mode).

**File**: `src/shared/agent-status-types.ts`

After `| 'hermes'`, add `| 'devin'`.

**Verify**: `pnpm typecheck` → exit 0

### Step 7: Localization and README

**File**: `config/scripts/locale-translation-policy.mjs`

Add `'Devin'` to `NEVER_TRANSLATE_VALUES` (alphabetically near other agent
product names, e.g. after `'Droid'`).

**Run**: `pnpm run sync:localization-catalog` → exit 0. This adds the new
`translate()` key to `src/renderer/src/i18n/locales/en.json` and repairs locale
parity in other locale files.

**File**: `README.md`

After the Hermes Agent entry (L187), before Goose (L188), add:

```html
  <a href="https://devin.ai/cli"><kbd><img src="https://www.google.com/s2/favicons?domain=devin.ai&sz=64" alt="Devin logo" width="16" valign="middle" /> Devin</kbd></a> &nbsp;
```

**Verify**: `pnpm run verify:localization-catalog` → exit 0

### Step 8: Add startup plan test

**File**: `src/shared/tui-agent-startup.test.ts`

Append inside the existing `describe('tui agent startup plans', …)` block (after
the last `it`, before the closing `})`):

```typescript
  it('launches Devin with stdin-after-start prompt delivery', () => {
    const plan = buildAgentStartupPlan({
      agent: 'devin',
      prompt: 'fix the tests',
      cmdOverrides: {},
      platform: 'linux'
    })
    expect(plan).toEqual({
      agent: 'devin',
      launchCommand: 'devin',
      expectedProcess: 'devin',
      followupPrompt: 'fix the tests'
    })
  })
```

**Verify**: `pnpm test -- tui-agent-startup` → all pass including the new test.

### Step 9: Full verification

**Run**:

```bash
pnpm typecheck && pnpm lint && pnpm test -- tui-agent && pnpm test
```

**Verify**: all exit 0.

**Run**: `git status --porcelain` and confirm only in-scope files are modified.

## Test plan

- **Existing tests must pass** after registration: `pnpm test -- tui-agent`
  covers selection, startup, agent-kind sync, and mobile catalog parity.
- **New test** (Step 8): `buildAgentStartupPlan` for `devin` with
  `stdin-after-start` — `launchCommand` is bare `devin`, prompt delivered via
  `followupPrompt`. Model after the Mistral Vibe test at L90-103 in
  `tui-agent-startup.test.ts`.
- **Catalog parity**: `quick-workspace-agent-selection.test.ts` and
  `mobile-agent-catalog.test.ts` enforce order/coverage — no manual test edits
  needed if Steps 4–5 are correct.
- **Verification**: `pnpm test -- tui-agent-startup` → all pass; `pnpm test`
  → full suite green.

## Done criteria

Machine-checkable. ALL must hold:

- [ ] `pnpm typecheck` exits 0
- [ ] `pnpm lint` exits 0
- [ ] `pnpm test` exits 0; the new Devin startup test exists and passes
- [ ] `pnpm test -- agent-kind` exits 0
- [ ] `pnpm test -- quick-workspace-agent-selection` exits 0
- [ ] `pnpm test -- mobile-agent-catalog` exits 0
- [ ] `grep -c "'devin'" src/shared/types.ts` returns `1`
- [ ] `grep "devin:" src/shared/tui-agent-config.ts` matches detectCmd, launchCmd, and expectedProcess lines
- [ ] `grep -c "'devin'" src/shared/tui-agent-selection.ts` returns `1`
- [ ] `grep -c "devin:" src/shared/tui-agent-launch-defaults.ts` returns `1`
- [ ] `grep -c "'devin'" src/shared/agent-status-types.ts` returns `1`
- [ ] `grep -c "devin:" src/shared/agent-kind.ts` returns `1`
- [ ] `grep -c "'devin'" src/shared/telemetry-events.ts` returns `1`
- [ ] `grep -c "'devin'" src/renderer/src/lib/agent-catalog.tsx` returns `1`
- [ ] `grep -c "devin:" src/renderer/src/lib/agent-status.ts` returns at least `2` (labels + iconable)
- [ ] `grep -c "'devin'" mobile/src/tasks/mobile-tui-agents.ts` returns `1`
- [ ] No files outside the in-scope list are modified (`git status --porcelain`)
- [ ] `plans/README.md` status row updated to `DONE`

## STOP conditions

Stop and report back (do not improvise) if:

- The `TuiAgent` union at `src/shared/types.ts:1933` has changed structure
  since this plan was written (refactored to a different pattern).
- After completing Steps 1–6, `pnpm typecheck` still fails — likely a new
  exhaustive `Record<TuiAgent, …>` file was added since this plan was written.
  Report the file; do not patch out-of-scope areas without approval.
- `pnpm typecheck` fails after Step 9 with errors unrelated to this change.
- The `TUI_AGENT_CONFIG` entry shape has new required fields not shown above.
- `pnpm run sync:localization-catalog` fails and cannot be resolved by adding
  the `'Devin'` never-translate entry.
- Devin CLI is installed and `devin --permission-mode bypass --help` reports an
  unrecognized flag. (If Devin is **not** installed, skip this check — it is
  informational only.)
- Manual testing shows `stdin-after-start` bracketed paste fails against Devin's
  REPL. Report observed behavior; do not switch to `argv` without approval.

## Maintenance notes

- **Session resume follow-up**: Devin supports `devin -c` and `devin -r <id>`.
  To add later: (1) `'devin'` in `RESUMABLE_TUI_AGENTS` in
  `src/shared/agent-session-resume.ts`, (2) `getAgentResumeArgv` case, (3) hook
  relay to extract session IDs.
- **Hook integration follow-up**: mirror `e7f3bb603` (Grok) for
  `agent-hook-listener.ts`, `src/main/grok/hook-service.ts` pattern if Devin
  exposes structured status hooks.
- **Synthetic agent titles**: add to `src/shared/synthetic-agent-title.ts` once
  Devin's terminal title behavior is observed in Orca.
- **Reviewers should check**: `stdin-after-start` is correct (not `argv`) per
  https://docs.devin.ai/cli/reference/commands; catalog order matches
  `TUI_AGENT_AUTO_PICK_ORDER`; telemetry kind `'devin'` round-trips.