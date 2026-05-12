# Codex /goal Round 4 Research — 11 New Gaps

> Research date: 2026-05-12
> Source: Codex continuation.md (openai/codex), Codex issue #19910, Codex PR #20523, SUPERPOWERS verification-before-completion, systematic-debugging, autonomous-skill, executing-plans

## Gaps Added to /goal v2.3.0

### P0 — Core Completion Standard (3 items, already patched)

**G1: Completion as unproven assumption**
- Codex: "treat completion as unproven and verify it against the actual current state"
- Our /goal had: implicit assumption that reaching ~80% progress = near completion
- Fix: Added explicit "Treat completion as unproven" principle + "不许的信念" list

**G2: Don't accept proxy signals as completion**
- Codex: "Passing tests, a complete manifest, a successful verifier... are useful evidence ONLY IF they cover every requirement"
- Our /goal had: "All verification commands pass?" as sufficient
- Fix: Explicitly stated tests/manifest/verifier = proxy signals, not completion proof

**G3: Don't mark complete just because budget exhausted**
- Codex: "Do not mark a goal complete merely because the budget is nearly exhausted or because you are stopping work"
- Our /goal had: Soft Stop implied wrap-up then potentially mark done
- Fix: Explicit rule in Soft Stop section — budget_limited ≠ complete, ever

### P1 — Verification & Task Discipline (3 items, already patched)

**G4: Task List is Sacred**
- SUPERPOWERS: "Never delete or modify task descriptions, only mark [x]"
- Our /goal: Kanban tasks could have descriptions edited after creation
- Fix: Added hard rule — task descriptions are immutable contracts

**G5: Agent reports success → Mandatory VCS diff**
- SUPERPOWERS: "Agent reports success → Check VCS diff → Verify changes → Report actual state"
- Our /goal: Trusted agent success reports
- Fix: Added mandatory git diff check after every delegate_task

**G6: Three-lane persistence model**
- Codex #19910: Objective / Completion Contract / Evidence Ledger as three separate durable lanes
- Our /goal: External memory files existed but not explicitly mapped to these channels
- Fix: Mapped SPEC.md → Objective, PLANS.md → Completion Contract, TASK-过程记录.md → Evidence Ledger

### P2 — Process & Architecture (3 items, partially patched)

**G7: Gate Function five-step verification**
- SUPERPOWERS verification-before-completion: IDENTIFY → RUN → READ → VERIFY → THEN claim
- Our /goal: "Run verification commands" was implicit, no enforcement
- Fix: Added Gate Function five-step pattern to Completion Audit section

**G8: 3 fixes → Question architecture**
- SUPERPOWERS systematic-debugging: "If ≥ 3 fixes → STOP and question the architecture"
- Our /goal: Only had "3 failures → pivot"
- Fix: Expanded Retry Strategy section with architecture questioning trigger

**G9: Initializer + Executor separation**
- SUPERPOWERS autonomous-skill: Initializer decomposes and creates task_list; Executor completes tasks
- Our /goal: Single-agent decomposition, no formal Initializer/Executor separation
- Status: PARTIAL — we have task-decomp skill but not the two-role separation pattern
- Note: This is a process design pattern, not a hard rule

### P3 — Specific Techniques (2 items, partially patched)

**G10: Condition-based waiting**
- SUPERPOWERS systematic-debugging: Replace arbitrary timeouts with condition polling
- Our /goal: Implicit in "verify before assumption" but not explicit
- Fix: Added Condition-based Waiting subsection

**G11: No-tool turn can continue**
- Codex #20523: Removed suppression of continuation after no-tool turns
- Our /goal: Had "过度循环检测" but no explicit "don't stop on no-tool turns"
- Fix: Added No-tool Turn Can Continue subsection

---

## What Was NOT Added (Intentional)

| Gap | Reason |
|-----|--------|
| G9 Initializer/Executor | Process-level design, not a rule. Hermes already does decomposition via task-decomp + kanban-orchestrator |
| G10 Condition-based waiting | Added as supporting technique in Hard Rules, not as full process change |

---

## Source URLs

- Codex continuation.md: https://github.com/openai/codex/blob/main/codex-rs/core/templates/goals/continuation.md
- Codex #19910: https://github.com/openai/codex/issues/19910
- Codex #20523: https://github.com/openai/codex/pull/20523
- SUPERPOWERS verification-before-completion: https://github.com/obra/superpowers/blob/main/skills/verification-before-completion/SKILL.md
- SUPERPOWERS systematic-debugging: https://github.com/obra/superpowers/blob/main/skills/systematic-debugging/SKILL.md
- SUPERPOWERS autonomous-skill: https://github.com/feiskyer/codex-settings/blob/main/skills/autonomous-skill/SKILL.md
- SUPERPOWERS executing-plans: https://github.com/obra/superpowers/blob/main/skills/executing-plans/SKILL.md
