# PR #1401 — land the Cursor CLI beta agent

**Ticket:** PR [#1401](https://github.com/RunMaestro/Maestro/pull/1401) — `feat(agents): add Cursor CLI as a first-class beta agent`
**Head:** `codex/cursor-cli-agent-rc` @ `d63e48e6d` (fork `guimpster/Maestro`)
**Base:** `rc`
**Round:** 1

## Problem

PR #1401 is feature-complete and reviewed, but it cannot merge. `gh pr view 1401` reports
`mergeable: CONFLICTING`, `mergeStateStatus: DIRTY`.

The cause is drift, not design. The PR last synced with `rc` on 2026-08-30. Since the merge
base (`cbb2901ef`), `rc` has advanced **580 commits** touching **1,872 files**. Three of those
files collide textually with the PR:

| File | Conflicts | Nature |
| --- | --- | --- |
| `src/main/process-manager/handlers/ExitHandler.ts` | 1 hunk (line ~116) | **Real ordering conflict.** Needs a decision, not a union. |
| `src/main/process-manager/handlers/StdoutHandler.ts` | 1 hunk (line ~595) | Mechanical union of two sibling methods. |
| `src/__tests__/main/process-manager/handlers/ExitHandler.test.ts` | 1 hunk (line ~92) | Import-line union. |

The CodeRabbit review is **not** a blocker. All 7 actionable findings plus the duplicate-`errors`
compile error were fixed in `6ec3e58d5` ("fix(agents): address Cursor integration review
findings"), which is an ancestor of the PR head, and each thread carries CodeRabbit's
"✅ Addressed in commit 6ec3e58" reply. The `CodeRabbit` check is **pass**. One caveat carries
forward — see "The ExitHandler ordering decision" below, where `rc` has since hardened the exact
contract that finding was about, so the *resolution* re-opens it.

## Approach

Merge `origin/rc` into the PR branch, resolve the three conflicts, and re-verify the Cursor
surface against the new `rc`. Do **not** rebase — the PR body commits to preserving the original
Cursor commits and @jSydorowicz21's authorship, and a 580-commit rebase would rewrite them.

### 1. `ExitHandler.test.ts` — import union (trivial)

`rc` widened the type imports. Take the `rc` side; it is a strict superset:

```ts
import type { AgentError, ManagedProcess } from '../../../../main/process-manager/types';
import type { AgentOutputParser, ParsedEvent } from '../../../../main/parsers';
```

Keep the PR's `new ExitHandler({ ..., dispatchParsedEvent })` construction at line ~157 — `rc`
still builds it as `new ExitHandler({ processes, emitter, bufferManager })`, and the PR adds the
fourth dependency. The same applies to the two production call sites
(`ChildProcessSpawner.ts:46`, `OpencodeServerSpawner.ts:99`), which already carry the PR's extra
argument and do not conflict.

### 2. `StdoutHandler.ts` — method union (mechanical)

The two branches changed adjacent things:

- `rc` added a new private method `resolveProvisionalError(...)` (holds/drops an in-turn API
  error notice based on the line that followed it).
- The PR changed `private handleParsedEvent(...)` to `handleParsedEvent(...)` (public) so
  `ExitHandler` can dispatch the trailing no-newline record through the same path.

Keep both: `rc`'s `resolveProvisionalError` in full, then the PR's public `handleParsedEvent`
with the PR's doc comment explaining *why* it is public. The `interruptedCursor` handling just
below the marker is PR-side and already merged cleanly.

### 3. `ExitHandler.ts` — the ordering decision (the real work)

This is the one hunk that needs judgment.

**What `rc` now does**, in order, inside `handleExit`:

1. `await this.awaitCopilotShutdown(sessionId, managedProcess)`
2. `if (this.isSuperseded(sessionId, managedProcess)) return;` — guarded by a long comment stating
   the check must sit above *every* step that emits into shared per-session state, and naming
   **"the stream-json remainder"** explicitly as one of them
3. `settleProvisionalAgentError(...)`
4. batch-mode exit
5. stream-json remainder

**What the PR does:** hoists the stream-json remainder block *above* `awaitCopilotShutdown`, so a
Copilot session ID that only ever appears in an unterminated last record is available to the
shutdown wait. That hoisted block clears `managedProcess.jsonBuffer`, and it emits `session-id`,
`agent-error`, and (via `dispatchParsedEvent`) usage/tool/result events.

Merging naively keeps the PR's position and puts emitting work above `rc`'s guard — a predecessor
draining at exit can push its remainder events into the successor's turn. That is exactly
CodeRabbit's Major finding at `ExitHandler.ts:107`; `6ec3e58d5` answered it against the *older*
`rc`, and the newer `rc` restates the contract more strongly.

**Resolution: split the hoisted block.** Above `awaitCopilotShutdown`, keep only a
**non-emitting** session-ID peek — parse the remainder *without* consuming it (do not clear
`jsonBuffer`), and if it yields a session ID, set `managedProcess.agentSessionId` and nothing
else. No `emit`, no `errorEmitted`, no buffer clear. Then leave the full remainder block in
`rc`'s position at step 5, below the guard, carrying the PR's enriched logic (error-envelope
routing through `agent-error`, otherwise `dispatchParsedEvent`). The later block emits
`session-id` as it always did, so nothing is lost by the peek staying silent.

This satisfies both sides: Copilot's shutdown wait gets its ID, and every emission stays below
the supersession guard.

Two details to preserve while resolving:

- The later block must be the **PR's** version, not `rc`'s. `rc`'s remainder block is the older,
  thinner one; the PR's replaces it. After the merge there must be exactly **one** remainder
  block that consumes `jsonBuffer`.
- Keep `emitDataBuffered(sessionId, remainingLine, managedProcess)` passing `managedProcess`
  explicitly (third argument). An exiting process may already be unregistered from the `processes`
  map, so resolving the buffer from the map instead would lose it. CodeRabbit flagged this; keep
  the fix.

Also re-check the PR's `code === 0` guard on the final stream-json flush (`ExitHandler.ts:266`
in the review) against `rc`'s current omp-hardening block — `6ec3e58d5` addressed it, but the
surrounding code moved.

### 4. Semantic drift beyond the three conflicts

These PR-touched files also changed on `rc` since the merge base and merged *textually* clean,
which is precisely where silent breakage hides. Read the merged result of each before trusting it:

```
src/cli/services/agent-spawner.ts
src/cli/services/batch-processor.ts
src/main/agents/capabilities.ts
src/main/process-manager/spawners/ChildProcessSpawner.ts
src/renderer/hooks/batch/inlineWizard/conversationActions.ts
src/renderer/hooks/wizard/useWizardHandlers.ts
src/shared/agentErrorPatterns.ts
src/shared/agentMetadata.ts
src/__tests__/cli/services/agent-spawner.test.ts
src/__tests__/cli/services/batch-processor.test.ts
src/__tests__/main/process-manager/handlers/StdoutHandler.test.ts
src/__tests__/renderer/hooks/useWizardHandlers.test.ts
CLAUDE.md, CLAUDE-AGENTS.md
```

Highest risk: `src/main/agents/capabilities.ts` and `src/shared/agentMetadata.ts`. If `rc` added
fields to the agent capability/metadata shape, the Cursor entry will compile-fail or, worse,
default silently. `npm run lint` (which is the TypeScript pass, see below) catches the first case;
`src/__tests__/main/agents/` catches the second.

Also confirm no agent registered on `rc` in the last 580 commits collides with the `cursor` agent
id or the generic `agent` binary probe in `src/main/agents/path-prober.ts`.

## Files to touch

Conflict resolution only — no new feature surface:

- `src/main/process-manager/handlers/ExitHandler.ts`
- `src/main/process-manager/handlers/StdoutHandler.ts`
- `src/__tests__/main/process-manager/handlers/ExitHandler.test.ts`

Plus whatever the drift review in step 4 turns up. If step 4 requires a change outside these
three, note it in the PR description — reviewers approved the pre-drift diff and will re-read
anything new.

## Verify

Run in the worktree, after `npm ci` (this repo builds with **npm**; pnpm produces ~28 phantom
`TS2307` errors):

```bash
# 1. TypeScript — this repo's `lint` script is the typecheck, and it is the
#    cheapest detector of capability/metadata drift.
npm run lint

# 2. Targeted suites: the conflicted handlers plus the whole Cursor surface.
npx vitest run \
  src/__tests__/main/process-manager/handlers/ExitHandler.test.ts \
  src/__tests__/main/process-manager/handlers/StdoutHandler.test.ts \
  src/__tests__/main/parsers/cursor-cli-output-parser.test.ts \
  src/__tests__/main/parsers/index.test.ts \
  src/__tests__/main/parsers/error-patterns.test.ts \
  src/__tests__/main/agents/ \
  src/__tests__/cli/services/agent-spawner.test.ts \
  src/__tests__/shared/pathUtils.test.ts \
  src/__tests__/renderer/components/Wizard/
```

`ExitHandler.test.ts` and `StdoutHandler.test.ts` are named explicitly because the exit-ordering
resolution touches per-session emission state shared across every agent, not just Cursor — those
two suites are what exercise the supersession guard and the remainder path.

The Cursor E2E suite (`src/__tests__/e2e/CursorCli*.e2e.test.ts`) needs a real authenticated
Cursor CLI on the host. Run it only if that is available; it is not a gate for the resolution
itself, since the conflicts are all in shared process-manager code the unit suites cover.

### CI owns

Read these from `gh pr checks 1401` rather than running them locally:

- `npm test` (full vitest suite)
- `npm run lint:eslint`
- `npm run format:check:all`
- `npm run build`
- Playwright E2E
- `Validate PR Title` (`.github/workflows/pr-linter.yml`)
- CodeRabbit re-review

Do not block on a missing local run of any of these.

## Done when

- [ ] `origin/rc` merged into `codex/cursor-cli-agent-rc`, three conflicts resolved as above
- [ ] Exactly one `jsonBuffer`-consuming remainder block in `ExitHandler.ts`, below the
      `isSuperseded` guard; the pre-shutdown peek emits nothing
- [ ] Step-4 drift files read and reconciled
- [ ] `npm run lint` clean; targeted vitest run green
- [ ] Pushed to `fork/codex/cursor-cli-agent-rc`; `gh pr view 1401` reports `mergeable: MERGEABLE`
- [ ] `gh pr checks 1401` green

## Assumptions

1. **"Solve this PR" means get #1401 to a mergeable, green state** — resolve the conflict with
   `rc` and re-verify. It does not mean re-litigating the Cursor design, which is already reviewed
   and CodeRabbit-passed.
2. **Worktree base.** The handoff template says branch from `dev`. This repo has no `dev` branch;
   `rc` is the PR's base. The worktree is branched from the **PR head** (`d63e48e6d`), not from
   `rc`, because the work *is* the merge — the implementer needs the PR's code in hand. Branch:
   `feature/pr1401-cursor-cli-agent`.
3. **Repo location.** The working directory `Personal Projects/Maestro` is not itself a git repo;
   the clone lives at `maestro-pr1401/`. The worktree is therefore at
   `maestro-pr1401/.claude/worktrees/pr1401-cursor-cli-agent`.
4. **No brief.** `docs/briefs/` does not exist in this repo. Proceeding without one. `docs/specs/`
   did not exist either and is created by this spec.
5. **Merge, not rebase** — preserves original Cursor commits and @jSydorowicz21's authorship, as
   the PR body promises.
6. **The CodeRabbit findings stay fixed.** `6ec3e58d5` is an ancestor of the head and all threads
   are marked addressed. The implementer should confirm the merge does not revert any of them,
   particularly in `ExitHandler.ts`, where the resolution rewrites the flagged region.
7. **Conflict counts are from a real trial merge** of `origin/rc` @ `44270d35b` into the PR head,
   run and aborted while writing this spec. If `rc` advances before the implementer starts, the
   set may grow; re-run `git merge --no-commit --no-ff origin/rc` and diff against the table above.
8. **PR title unchanged.** The existing title already satisfies Conventional Commits and describes
   the client-facing outcome; this work adds no new user-facing capability.
9. **This spec is `git add -f`'d.** `.gitignore:9` ignores `specs/` repo-wide, but the handoff
   protocol requires the spec committed or the Reviewer cannot see it. It lives only on
   `feature/pr1401-cursor-cli-agent` and must **not** be carried onto
   `codex/cursor-cli-agent-rc` — do not let it into the #1401 diff.

## PR title

```
feat(agents): add Cursor CLI as a first-class beta agent
```
