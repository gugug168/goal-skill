# Codex /goal Research — Knowledge Bank

**Source:** `openai/codex` repo, `continuation.md`, `budget_limit.md`, issue #18076, `gpt_5_2_prompt.md`, Codex Autoresearch docs.
**Session:** 2025-05-12, Hermes Agent v2.1.0 goal skill upgrade.

---

## Codex /goal Architecture

### Core Files

| File | Role |
|------|------|
| `continuation.md` | Main loop prompt template injected each turn |
| `budget_limit.md` | Wrap-up steering when budget exhausted |
| `goal_autoresearch.md` | Autoresearch loop protocol |
| `update_plan` tool | Plan tracking with single in_progress rule |
| `update_goal` tool | Mark complete / create / get goals |
| `goal` feature flag | Gates goal tools in tool registry |

### 4-Document External Memory System (Codex Standard)

```
WORKDIR/
├── SPEC.md          # Goal + non-goals + hard constraints + deliverables + done-when
├── PLANS.md         # Milestones + acceptance criteria + verification commands
├── IMPLEMENT.md     # Execution runbook (references PLANS.md)
└── DOCUMENTATION.md # Real-time status + decisions + known issues
```

### Token Budget System

- **Soft stop:** Budget exhausted → mark goal `budget_limited` → inject wrap-up steering → continue current turn doing cleanup → stop new substantive work.
- **NOT a hard kill.** The turn after budget hits 100% still runs but only wraps up.
- Budget tracking: `.goal-budget.json` with `used_tokens`, `status: active|warning|paused|complete`.
- Token usage checkpoint on every state transition (especially important when replacing goal objective).

### Goal State Machine (Codex Core Runtime, issue #18076)

```
active → paused (user interrupt / idle interrupt)
active → budget_limited (token/time exhausted)
paused → active (user resumes)
budget_limited → active (user adds budget)
active → complete (update_goal + completion audit)
active → failed (3 pivots still failing)
```

**P1 bugs discovered in Codex implementation:**
1. `paused` goal crossing budget doesn't become `budget_limited` (SQL only handles `active → budget_limited`) → paused goals can exceed budget undetected.
2. `thread_goal_set` app-server path skips in-flight usage snapshot → undercounts tokens.
3. Single-turn abort doesn't emit goal event → goal state inconsistent.
4. `replace_thread_goal` resets persisted `tokens_used` but doesn't reset per-turn accounting checkpoint → next turn applies pre-goal usage to new goal.

**Rule:** Budget exhaustion detection must handle BOTH `active` AND `paused` states.

### Completion Audit (Codex continuation.md)

```
Before marking goal complete, treat completion as UNPROVEN.
Verify against actual current state for EVERY explicit requirement:

For each item determine:
  ✅ PROVES completion — evidence shows requirement satisfied
  ❌ CONTRADICTS completion — evidence shows NOT satisfied
  ⏳ INCOMPLETE — partial work
  ❓ MISSING — no evidence found
  ⚠️ TOO WEAK — indirect evidence doesn't support the scope of the claim

Evidence types (ranked by authority):
  1. Runtime behavior / cmd output
  2. Test results (only if test covers the specific requirement)
  3. File contents / code inspection
  4. Manifest / README (secondary)
  5. Conversation memory / "I believe it's done" (NOT evidence)

Key rules:
- Tests passing ≠ requirement met (tests must cover it specifically)
- "Looks good" ≠ requirement met
- Uncertainty → keep working, don't assume
- Do NOT mark complete merely because budget exhausted or stopping
```

### Hard Rules (from Codex Autoresearch)

```
Rule 1: Ask-Before-Act
  所有问题 → Phase 0-1 (before launch)
  User says "go" / "start" / "launch" → LOOP is fully autonomous
  After launch: NO pausing to ask / confirm / request permission
  If ambiguous → use best practice + record reasoning in TASK-过程记录.md

Rule 2: Single-Focus Change
  One hypothesis → one change → one verification → record → next
  NOT: 5 changes at once then can't attribute

Rule 3: Dirty Worktree Guard
  git status --porcelain before each agent run
  Clean → proceed
  Has changes + belongs to current task → continue, stage changes
  Has changes + unrelated → STOP, notify user

Rule 4: Pivot After 3 Failures (not brute retry)
  3 failures → switch agent, 2 more attempts → still failing → PIVOT
  Change approach, don't repeat same approach

Rule 5: Discard <1% Improvements
  If improvement <1% AND code complexity significantly increases
  → discard, record "discard: 收益 <1%", continue

Rule 6: Scope Fidelity (禁止缩小目标)
  User says "implement X"
  DON'T substitute X with "simpler/safer/narrower version of X"
  "Easier to complete" ≠ "要求的版本完成了"

Rule 7: Alignment = End State Movement
  An edit is aligned ONLY if it makes the requested final state more true
  "Useful behavior that preserves a different end state" = misalignment

Rule 8: Verify Before Assumption
  Before deciding next action → inspect current state
  git log / file contents / cmd output FIRST
  Don't assume code hasn't changed since last seen

Rule 9: update_plan Discipline (single in_progress)
  Exactly ONE item in_progress at a time
  Complete current → immediately mark complete → then mark next in_progress
  NEVER batch-complete multiple items after the fact
  Scope pivot → update plan FIRST, then continue
  Plan update is NOT a substitute for doing work
```

### update_plan Tool Semantics (Codex)

```
- 1-7 sentence steps, status per step: pending / in_progress / completed
- Exactly one in_progress at any time
- Mark completed steps before moving to next
- If all done → mark all completed
- If understanding changes → update plan with explanation, then continue
- Short steps (5-7 words each) — easy to verify as you go
```

### Fidelity Principle (Codex continuation.md)

```
WRONG:  Optimize for smallest stable-looking subset
        Substitute narrower/safer/easier solution because more likely to pass
RIGHT:  Keep full objective intact
        Make concrete progress toward real requested end state
        Temporary rough edges OK while work moves in right direction
        Completion requires requested end state to be true and verified
```

### Autoresearch Loop Protocol (from Codex)

```
1. Assess: inspect current state + identify best next action
2. Act: execute one focused change
3. Verify: prove the change worked with evidence
4. Log: record result in TASK-过程记录.md
5. Repeat

After each milestone:
  → Run verification commands
  → Fail → fix before continuing
  → Pass → move to next milestone
  NOT: hope it works, move on
```

---

## Ralph (openai-ralph-codex) — Third-Party Codex Enhancement

**Repo:** `github.com/JungyuOO/openai-ralph-codex`
**Purpose:** PRD-driven software delivery loop built on top of Codex.

### Architecture

```
.ralph/
├── prd.md           # Source-of-truth PRD
├── tasks.json       # Persisted task graph with retry state + context
├── state.json       # Phase + current task + next action + retry info
├── progress.md      # Append-only execution history
├── memory.json      # Distilled reusable lessons from loop turns
├── split-proposals.json  # Local split suggestions for blocked work
├── config.yaml      # Runner + verification + context settings
└── evidence/        # Per-task verification artifacts
```

### Ralph Commands

| Command | Action |
|---------|--------|
| `orc enable` | Opt current project into Ralph hook routing |
| `orc disable` | Opt out |
| `orc init` | Create `.ralph/` from templates |
| `orc plan` | Generate/regenerate task graph from prd.md |
| `orc run` | Execute next runnable task |
| `orc verify` | Run verification commands only |
| `orc status` | Show phase + current task + next action |
| `orc resume` | Re-queue blocked/interrupted work |

### Key Insights from Ralph

- **Intent classification:** Codex classifies prompt intent on loop entry → routes to `plan`/`run`/`verify`/`status` accordingly
- **Session latch:** Active loops use compact latched continuation routing (not full re-analysis each turn)
- **Verification as hard gate:** `orc verify` must pass before marking task complete
- **Evidence preservation:** `evidence/` dir stores verification artifacts per task — proof that requirements were met
- **Failure fingerprints:** `tasks.json` stores retry state + last failure reason — avoids repeating same failed approach
- **Split proposals:** When task is too broad, Ralph suggests local splits → user approves or redirects

---

## Comparison: Our /goal (Hermes v2.1) vs Codex /goal

| Feature | Hermes `/goal` | Codex `/goal` |
|---------|---------------|---------------|
| Trigger | `/goal [objective]` | `/goal [objective]` |
| Token Budget | ✅ Optional, 80%/100% thresholds | ✅ Built-in, soft stop |
| External Memory | 4 docs (SPEC/PLAN/IMPL/DOC) | Same (4 docs) |
| Single in_progress | ✅ update_plan discipline | ✅ update_plan tool |
| Completion Audit | ✅ Phase 5 | ✅ continuation.md |
| Goal State Machine | ✅ 5 states | ✅ Core runtime (more complete) |
| Scope Fidelity | ✅ Rule 6 | ✅ fidelity principle |
| Alignment rule | ✅ Rule 7 | ✅ alignment = end state movement |
| Verify Before Assumption | ✅ Rule 8 | ✅ inspect before act |
| Soft Stop | ✅ wrap-up steering | ✅ budget_limit.md |
| Dirty Worktree | ✅ git status check | Implicit |
| Pivot after 3 failures | ✅ | ✅ |
| ACP protocol | ❌ (roadmap) | N/A (native) |
| Concurrent task dispatch | ✅ leaf role三角并行 | Sequential (single in_progress) |
| Self-healing | Manual | `orc resume` |

---

## Priority Improvements for Future Sessions

1. **ACP protocol for Gemini CLI** — would enable Gemini CLI as a proper coding agent (not just via terminal)
2. **Evidence/ directory per task** — like Ralph's evidence/ — stores verification artifacts
3. **orc verify as hard gate** — `orc verify` must pass before task marked complete (Ralph pattern)
4. **Failure fingerprints** — store last failure reason in tasks.json to avoid repeating same approach
5. **Session latch for continuation** — compact routing on loop continuation (don't re-classify intent every turn)
