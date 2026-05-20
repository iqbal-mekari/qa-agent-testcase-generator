# QA Agent UI Automation

Custom GitHub Copilot agents and skills for mobile UI test case generation, Maestro script authoring, and automated test debugging — scoped to Flutter-based Android/iOS apps.

## Agents

### `qa-test-case-generator`

Senior Mobile QA Engineer agent. Generates exhaustive, automation-ready UI test cases from Jira tickets, PRDs, Figma designs, or free-text feature descriptions. Outputs results as Jira comments and consolidated CSV files in `/test-cases/`.

**Scope:** Mobile app UI only (screen states, interactions, navigation, visual assertions). Does not generate API, web, or backend test cases.

**Trigger phrases:** `create test cases`, `generate test cases`, `mobile test cases`, `ui test cases`, `visual test cases`, `self-test`, `test case from ticket`, `test case from PRD`, `regenerate test cases`, `update test cases`, `test cases from Figma`

---

### `maestro-script-creator`

Orchestrator agent for producing production-ready Maestro UI test scripts from a test case file (CSV, spreadsheet, Jira export). Handles planning, screen mapping, triage (automate / needs setup / skip), mapping table confirmation, testcase YAML authoring, and scenario composition.

**Trigger phrases:** `maestro`, `UI test`, `test automation`, `test script`, `test scenario`, `automate`, `CSV test cases`, `Maestro script`, `create Maestro tests`, `generate maestro`

**Delegates to:** `maestro-testcase-writer`, `maestro-scenario-composer`, `maestro-selector-debugger`

---

### `maestro-script-debugger`

Autonomous debugging orchestrator for failing Maestro flows. Runs the failing flow, captures screenshots and view hierarchy, diagnoses the root cause, probes fixes inline, applies them, re-runs to verify, and records new failure patterns in the knowledge base.

**Trigger phrases:** `debug maestro`, `maestro fails`, `maestro error`, `element not found`, `flow failed`, `fix maestro test`, `maestro assertion failed`, `maestro broken`

**Delegates to:** `maestro-selector-debugger`, `maestro-testcase-writer`, `maestro-scenario-composer`

---

### `maestro-testcase-writer` _(sub-agent, not user-invocable)_

Writes a single atomic Maestro testcase YAML file. Handles selector discovery from live UI and Flutter source, localization key mapping, syntax validation, and inline probing via Maestro MCP before saving. Invoked by `maestro-script-creator`.

---

### `maestro-scenario-composer` _(sub-agent, not user-invocable)_

Composes a Maestro scenario YAML that orchestrates atomic testcases into end-to-end user journeys. Handles `onFlowStart` lifecycle hooks, `runFlow` orchestration, state resets between independent error paths, and end-to-end validation. Invoked by `maestro-script-creator`.

---

### `maestro-selector-debugger` _(sub-agent, not user-invocable)_

Diagnoses and fixes failing Maestro selectors. Takes a screenshot, inspects the view hierarchy, probes corrected selectors inline, and returns a verified fix (selector update or `Semantics` change). Invoked by `maestro-script-debugger` or `maestro-script-creator`.

---

## Skills

| Skill                   | Purpose                                                                                                                                                                         |
| ----------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `create-test-cases`     | Generate structured mobile UI test cases from Jira tickets, PRDs, Figma designs, or feature descriptions. Outputs as Jira comments and CSV in `/test-cases/`.                   |
| `regenerate-test-cases` | Update existing mobile UI test cases when code changes are detected (git diffs, PRs, or existing CSVs). Adds new cases, modifies affected ones, and flags obsolete cases.       |
| `impact-analysis`       | Identify impacted modules and test cases from a PR or branch diff using [tomo_search](https://github.com/timiwinibiti/tomo_search) (Dart AST-based scope detection) + full semantic AI matching. Outputs structured JSON consumed by `regenerate-test-cases`. |
| `create-maestro-script` | Create Maestro testcase and scenario YAML files for Flutter apps from a CSV test plan. Covers folder structure, selector strategies, localization patterns, and test execution. |
| `debug-maestro-script`  | Debug failing Maestro flows (exit code 1, element not found, assertion failures, broken selectors). Follows a read → run → screenshot → hierarchy → probe → fix → re-run loop.  |

## Structure

```
agents/
├── qa-test-case-generator.md              ← Test case generation agent
├── maestro-script-creator.agent.md        ← Maestro script orchestrator
├── maestro-script-debugger.agent.md       ← Maestro debugging orchestrator
├── maestro-testcase-writer.agent.md       ← Atomic testcase writer (sub-agent)
├── maestro-scenario-composer.agent.md     ← Scenario composer (sub-agent)
└── maestro-selector-debugger.agent.md     ← Selector diagnosis (sub-agent)

skills/
├── create-test-cases/
│   └── SKILL.md
├── regenerate-test-cases/
│   └── SKILL.md
├── impact-analysis/
│   └── SKILL.md                           ← tomo_search-powered PR/branch impact analysis
├── create-maestro-script/
│   ├── SKILL.md
│   └── references/
│       ├── flutter-semantics.md
│       ├── scenario_template.yaml
│       └── testcase_template.yaml
└── debug-maestro-script/
    ├── SKILL.md
    └── references/
        └── failure-patterns.md
```

## How to Use

### Generate test cases

```
Create test cases for https://jurnal.atlassian.net/browse/QON-12345

Generate test cases from this PRD: https://jurnal.atlassian.net/wiki/...

Create UI test cases for a contact form with name, phone, and email fields
```

### Regenerate test cases after code changes

```
Regenerate test cases for branch feature/new-contact-form

Update test cases based on this PR: https://github.com/.../pull/123

Refresh test cases against /test-cases/QON_contact_test_cases.csv
```

### Analyze PR/branch impact before regenerating

```
Run impact analysis on PR #42

What modules are impacted by branch feature/attendance-clock-in?

Which test cases need reassessment for this PR: https://github.com/.../pull/88
```

> **Recommended flow for large PRs:** Run `impact-analysis` first to get a precise list of affected test cases, then pass its JSON output to `regenerate-test-cases` for targeted updates.

### Create Maestro UI test scripts

```
Create Maestro tests from test-cases/login_cases.csv

Generate Maestro testcases for the login screen

Automate the checkout flow using this CSV
```

### Debug a failing Maestro flow

```
Debug maestro — maestro/testcases/login/tap_login_button.yaml fails with element not found

Fix maestro test: scenarios/checkout/checkout_full_journey.yaml exits with code 1
```

## Output

| Agent                     | Output                                                      |
| ------------------------- | ----------------------------------------------------------- |
| `qa-test-case-generator`  | Jira comments + CSV in `/test-cases/`                       |
| `maestro-script-creator`  | YAML files in `maestro/testcases/` and `maestro/scenarios/` |
| `maestro-script-debugger` | Fixed YAML files + updated `failure-patterns.md`            |

### Skill: `impact-analysis` output

```json
{
  "analysis_metadata": { "source": "PR #42", "target_branch": "develop" },
  "impact_summary": { "total_test_cases_affected": 7 },
  "impacted_modules": [ { "module": "attendance", "scopes": ["AttendanceService.clockIn"] } ],
  "recommended_test_cases": [
    { "tc_id": "TC005", "confidence": "HIGH", "recommended_action": "MODIFY" },
    { "tc_id": "NEW",   "confidence": "HIGH", "recommended_action": "CREATE" }
  ]
}
```

4. GitHub Copilot will automatically use the updated configuration

---

_Managed by: GitHub Copilot with custom agent skills_
