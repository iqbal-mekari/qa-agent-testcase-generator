# Human-in-the-Loop Confirmation Gate

This document defines the explicit approval checkpoints where a human must confirm before the pipeline proceeds to the next phase. Gate 3 is now automatic and does not require human approval.

## Pipeline Flow with Gates

```
┌─────────────────────────────────────────────────────────────────────────┐
│                                                                         │
│  Phase 1: Test Case Generation                                          │
│  ─────────────────────────────────────────────────────────────────────  │
│                                                                         │
│  Input: PRD / Jira ticket / Figma / Codebase / Existing CSV             │
│       ↓                                                                 │
│  Agent: qa-test-case-generator                                          │
│       ↓                                                                 │
│  Output: CSV in /test-cases/ + Jira comment                             │
│                                                                         │
└──────────────────────────────┬──────────────────────────────────────────┘
                               │
                    ┌──────────▼──────────┐
                    │   🚦 GATE 1         │
                    │   Human reviews     │
                    │   generated test    │
                    │   cases CSV         │
                    │                     │
                    │   ✅ Approve        │
                    │   ✏️  Request edits  │
                    │   ❌ Reject         │
                    └──────────┬──────────┘
                               │ (only on ✅ Approve)
                               │
┌──────────────────────────────▼──────────────────────────────────────────┐
│                                                                         │
│  Phase 2: Automation Triage & Mapping                                   │
│  ─────────────────────────────────────────────────────────────────────  │
│                                                                         │
│  Agent: maestro-script-creator (planning stage)                         │
│       ↓                                                                 │
│  Output: Mapping table (CSV TC → Maestro file, automate/skip/setup)     │
│                                                                         │
└──────────────────────────────┬──────────────────────────────────────────┘
                               │
                    ┌──────────▼──────────┐
                    │   🚦 GATE 2         │
                    │   Human reviews     │
                    │   mapping table     │
                    │                     │
                    │   ✅ Confirm mapping │
                    │   ✏️  Adjust entries  │
                    │   ❌ Cancel          │
                    └──────────┬──────────┘
                               │ (only on ✅ Confirm)
                               │
┌──────────────────────────────▼──────────────────────────────────────────┐
│                                                                         │
│  Phase 3: Maestro Script Generation                                     │
│  ─────────────────────────────────────────────────────────────────────  │
│                                                                         │
│  Agent: maestro-script-creator → delegates to:                          │
│    • maestro-testcase-writer (per testcase)                             │
│    • maestro-scenario-composer (per scenario)                           │
│       ↓                                                                 │
│  Output: YAML files in maestro/testcases/ and maestro/scenarios/        │
│                                                                         │
└──────────────────────────────┬──────────────────────────────────────────┘
                               │
                               │
┌──────────────────────────────▼──────────────────────────────────────────┐
│                                                                         │
│  Phase 4: Execution & Self-Healing                                      │
│  ─────────────────────────────────────────────────────────────────────  │
│                                                                         │
│  Agent: maestro-script-debugger (if failures occur)                     │
│    • Runs tests automatically after YAML generation                     │
│    • Screenshots + hierarchy inspection                                 │
│    • Auto-fixes selectors (self-healing)                                │
│    • Re-runs to verify fix                                              │
│       ↓                                                                 │
│  Output: Fixed YAML + updated failure-patterns.md                       │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

## Gate Definitions

### Gate 1: Test Case Approval

**When:** After `qa-test-case-generator` produces the CSV output.

**What the human reviews:**
- Completeness — are all features/requirements covered?
- Correctness — do steps match actual app behavior?
- Priority — are Smoke vs Regression categories correct?
- Scope — no out-of-scope API/backend test cases?

**Agent behavior:**
1. Present the generated test cases summary (count by category, coverage map).
2. Explicitly ask: _"Please review the test cases in `{csv_path}`. Shall I proceed to Maestro script creation, or would you like changes?"_
3. Wait for user response before proceeding.

**If edits requested:** Regenerate affected test cases and re-present for approval.

---

### Gate 2: Mapping Table Confirmation

**When:** After `maestro-script-creator` produces the triage/mapping table.

**What the human reviews:**
- Which test cases will be automated vs skipped
- Screen → folder mapping correctness
- Testcase file naming
- Any "needs setup" items that require Flutter code changes

**Agent behavior:**
1. Present the mapping table in a readable format.
2. Explicitly ask: _"Please confirm this mapping table. Should I proceed with writing the YAML files?"_
3. Wait for user response before delegating to sub-agents.

**If adjustments:** Update mapping entries and re-present.

---

### Gate 3: Automatic Execution

**When:** After all testcase + scenario YAML files are generated.

**What happens:**
- The agent starts execution automatically.
- The human can review results after execution, but no approval is required to begin.

**Agent behavior:**
1. Present a summary of files created (paths, line counts, selector types used).
2. Run the generated scripts on the device.
3. If failures occur, invoke the debugging flow.

---

## Implementation Rules for Agents

1. **Never skip gates 1 and 2.** Those gates are mandatory — agents must pause and wait for human input.
2. **Present context.** Always show enough information at the gate for the human to make an informed decision (file paths, counts, key decisions made).
3. **No silent progression.** An agent must not proceed from Phase 1 to Phase 2 or Phase 2 to Phase 3 without the human explicitly saying "proceed", "confirm", "yes", or equivalent.
4. **Loop on edits.** If the human requests changes at gates 1 or 2, the agent applies them and re-presents for approval at the same gate.
5. **Record decisions.** After each gate approval, log the decision (what was approved and any modifications requested) in the session context for traceability.
