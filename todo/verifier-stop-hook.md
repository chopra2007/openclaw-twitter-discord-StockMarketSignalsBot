# Auto-run a separate verifier when work is claimed done (Stop hook)

**Status:** DONE 2026-07-07

**Created:** 2026-07-06

**CURRENT STATUS (2026-09-07):** SHIPPED + LIVE. A deterministic Stop hook is installed globally
(gated to this project + its worktrees) that re-runs the *affected* tests whenever a turn leaves
uncommitted code changes and blocks the stop on a regression. No LLM / no per-turn cost (user chose
"free test-checker only"). Proven with a 9/9 behavior harness + a live end-to-end block on a real
worktree regression. See **SHIPPED (2026-07-07)** below. On 2026-09-07 the hook was repaired: its 180s test
budget was shorter than the 186s the affected suite actually took, so it timed out and re-ran every
turn without ever producing a verdict; it now skips an unchanged tree via a code fingerprint and has a
300s budget. The original 3-phase plan is kept as history.

---

## SHIPPED (2026-07-07)

**What went live**
- `/root/.claude/hooks/verify-on-done.py` — the hook (Python). On each turn end it exits in
  milliseconds unless the turn left **uncommitted changes to code paths** (`consensus_engine/ scripts/
  tests/ config/`) inside this project; then it re-runs only the **affected** tests and, if a test that
  was green before now fails (a regression vs `.test-baseline`), blocks the stop with a short,
  cooperative `{"decision":"block","reason":...}`. Otherwise silent.
- `/root/.claude/settings.json` → `hooks.Stop` runs `python3 $HOME/.claude/hooks/verify-on-done.py`
  (timeout 240). Prior settings backed up in `/root/.claude/backups/settings.json.pre-verify-hook.*`.
- The hook is **global user config** (like `openclaw-digest.sh`), intentionally NOT in this repo. The
  user's Claude Code runs as ROOT using `/root/.claude/` (confirmed `/home/openclaw/.claude/` doesn't
  exist; auth via OAuth in `.credentials.json`).

**Decisions made (resolving the Open questions below)**
- *Scope:* global install **gated to this project + all worktrees** via the shared git dir
  (`realpath(git rev-parse --git-common-dir) == /home/openclaw/.openclaw/workspace/.git`). Chosen over
  pure project-level because the git-ignored `.claude/settings.json` doesn't reach worktrees, where a
  lot of coding happens. Silent in every other repo.
- *Blocking vs advisory:* **blocking**, but only on real regressions vs baseline (fast, deterministic,
  low-flake — the research's bar for a hard block). Message worded cooperatively (models ignore hostile
  hook text).
- *Which verifier:* **deterministic affected-test run, no LLM.** The lean LLM path (`claude -p --bare`)
  can't authenticate here (bare mode never reads OAuth); the authenticated non-bare path costs ~$0.076
  + ~4s/call (loads ~70k ctx), which fights the "don't slow every turn" requirement. User chose the
  free test-checker.
- *No duplication of pre-push:* this runs the **affected subset at COMPLETION time**; `scripts/pre-push`
  still runs the **full suite at PUSH time**.

**Safety / robustness**
- Loop guard (`stop_hook_active`) + Claude's built-in 8-block cap + a `VERIFY_ON_DONE_ACTIVE` recursion
  sentinel. Fails **open** on any error/timeout (never traps the agent). Runs pytest with scratch routed
  to /tmp (`PYTHONDONTWRITEBYTECODE=1`, `--basetemp`, `no:cacheprovider`) so a root-run hook can't leave
  root-owned cache files in the repo (verified: zero pollution).
- Affected-test mapping: module name as a whole word-token of the test filename (`db`→`test_db.py`, not
  `test_feedback.py`), else a whole-word content grep; capped at 40 files.

**Proof:** 9/9 behavior harness (real pytest + real git worktree) covering silent-on-clean,
silent-on-docs-only, block-on-regression (main + worktree), loop guard, sentinel, baseline, and
cross-repo gating; plus a live end-to-end block on a real-repo worktree regression, cleaned up with no
`.git` ownership residue. Latency: ~6s typical (narrator), ~28s for db.py (52 tests).

**To enable the optional AI-review layer later** (user declined for now): add a second stage that runs
ONLY after the free tests pass — non-bare `claude -p --model haiku --disallowedTools "Edit Write"
--output-format json` on the diff, with `VERIFY_ON_DONE_ACTIVE=1` set so it can't recurse. Cost ~$0.10
+ several seconds per clean code-turn.

**To change/remove:** edit or delete `/root/.claude/hooks/verify-on-done.py`; remove the `Stop` block
from `/root/.claude/settings.json` (or restore the backup) to turn it off entirely.

---

## Original plan (history — 2026-07-06)

**Goal** = a Claude Code **Stop hook** that automatically hands work to a *separate* read-only verifier
whenever an agent claims it's done, so verification isn't left to the same agent that did the work
(which does it poorly). Web research first, then code it thoughtfully. User's explicit ask: coded "very
thoughtfully and carefully" — fast, targeted, and not slowing every turn.

## Why (context from the 2026-07-06 session)
- Repeated problem: when asked to verify, the SAME agent that did the work also verifies it, and does
  it badly (re-reads its own reasoning instead of independently re-running; biased to pass its own work).
- Best practice (human review + LLM "generator–critic"): a separate agent, with no stake in the answer,
  told to disprove, re-runs the actual behavior. See the global rule already in CLAUDE.md
  ("Never self-approve in the same active context; use code-reviewer or verifier for the approval pass")
  and the project regression-gate rule ("a separate agent, not the one that wrote the code, re-runs the suite").
- Claude Code supports the building blocks (subagents w/ isolated context, custom `.claude/agents/*.md`
  with restricted tools, `/code-review`, hooks) but has **no built-in "auto-verify with a different
  agent" toggle** — you assemble it. A Stop hook is how you make it happen automatically instead of
  hoping the working agent delegates.

## Phase 1 — Web research FIRST (the user's main requirement)
Research how developers using LLM coding agents (Claude Code, Cursor, Aider, etc.) implement hooks well.
Gather concrete, cited patterns on:
- **Context bloat:** keep what the hook feeds back to the model tiny + structured (PASS/FAIL + short
  findings), never dump full logs/test output into the transcript.
- **Effectiveness:** what actually catches defects vs. what's theater; adversarial framing; re-running
  behavior vs. re-reading code.
- **When to run vs. skip (the qualifier):** how others gate so the hook does NOT fire on every turn —
  changed-code detection (git diff), path filters, completion signals, debounce.
- **Avoiding per-turn slowdown:** short-circuit/early-exit cheaply, run heavy work async/background,
  cache, only-on-real-completion; measured latency impact others report.
- **Loop prevention + exit semantics:** `stop_hook_active`, exit 0 (allow) vs exit 2 (block + feed
  stderr back), blocking vs advisory.
- **Anti-patterns others hit:** hooks that spam, block forever, leak tokens, fire redundantly.
- Collect 3–5 real examples (repos/blog posts/docs) and distill a short "do / don't" list before coding.

## Phase 2 — Design (from the research + this session's discussion)
- **Event:** `Stop` (main agent finishing a turn). Consider `SubagentStop` too.
- **Qualifier (decides run vs skip):** only run when the turn left **uncommitted changes to CODE paths**
  (`consensus_engine/`, `scripts/`, `tests/`, `config/`) — skip docs-only and no-change turns. Mirrors
  the doc-only-vs-code split CLAUDE.md already uses. Objective (git diff), not wording-based.
- **Verifier:** a **read-only** agent (`.claude/agents/verifier.md`, `tools: Read, Grep, Glob`, no Write
  — structurally cannot self-approve by editing), adversarially prompted, that **re-runs the affected
  tests / exercises the real behavior**, not just re-reads. (Alt: invoke `/code-review`, or headless
  `claude -p`; research which is leanest.)
- **Enforcement:** exit 2 to block the stop + feed back a concise findings summary so the agent must
  fix; exit 0 to allow. Guard loops with `stop_hook_active`.
- **Keep fed-back output tiny** (structured verdict + top findings only).

## Phase 3 — Implement carefully + test
- Wire in `settings.json` (use the `update-config` skill). Script at `.claude/hooks/verify-on-done.sh`;
  agent at `.claude/agents/verifier.md`.
- Prove all four behaviors on real turns: (a) silent on a docs-only turn, (b) silent on a no-op/Q&A turn,
  (c) fires + blocks on a broken code change, (d) no infinite loop, (e) no noticeable per-turn slowdown.

## Files / paths involved
- `.claude/settings.json` (or `~/.claude/settings.json` if global) — hooks config.
- `.claude/hooks/verify-on-done.sh` — the qualifier + verifier-invocation script.
- `.claude/agents/verifier.md` — the read-only reviewer agent.
- Docs: code.claude.com/docs/en/hooks, /sub-agents, /code-review.

## Open questions
- **Blocking vs advisory:** block (exit 2, force fix) for code changes, or just warn? (research others' take)
- **Which verifier:** local read-only agent vs `/code-review` vs headless `claude -p` — leanest + most effective?
- **Scope:** this project only (`.claude/settings.json`) or all projects (`~/.claude/settings.json`)?
- **Don't duplicate the existing pre-push gate.** `scripts/pre-push` already runs pytest at PUSH time
  (git hook). This Stop hook is complementary — it runs a *reviewer agent* at COMPLETION time. Make sure
  they don't fight or double-run; decide the division of labor.

## Relationship to #68 (discover)
This is the **interactive-session** fix (normal coding work, and discover's Pass 5 build, which runs in
the main session). #68's per-pass checkpoint is the **in-pipeline** analog for discover Passes 0–4 (the
Workflow engine — a Stop hook does NOT reach inside those background passes). Same principle
(separate verification / fail-loud), two mechanisms for two environments.

### Session notes — 2026-09-07
- **Worked on:** The hook could never finish. The in-progress trade-alerts build left 23 uncommitted code files, which mapped to 23 affected test files (394 tests) taking 186.5s — against the hook's own 180s cap. Every turn it ran ~3 minutes, got killed 6s short, produced no verdict, and repeated. Fixed: added a code fingerprint (path+size+mtime of the changed files, plus the test list) recorded on ATTEMPT, so an unchanged tree skips instantly; raised `VERIFY_ON_DONE_TIMEOUT` 180→300 and the `settings.json` hook timeout 240→420 so a run can actually reach a verdict.
- **Decisions:** Record the fingerprint on attempt, not on success — a suite that times out must fail open once, not re-burn minutes every turn. Did not narrow the test selection (integration tests stay in scope); the cost problem was repetition, not breadth.
- **Next:** Nothing owed. Watch that the ~3-minute check now happens once per code change, not once per turn.
