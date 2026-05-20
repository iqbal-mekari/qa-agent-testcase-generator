# QA Agent UI Automation — Copilot Instructions

This workspace contains VS Code Copilot **agent definitions** and **skills** for mobile UI test automation on Flutter apps using the [Maestro](https://maestro.mobile.dev/) framework. No application source code lives here.

## Critical Rules (read before any task)

1. **Read the relevant `SKILL.md` first.** It is the authoritative source — it overrides any legacy patterns found elsewhere.
2. **Never write Maestro YAML without probing inline first.** Use `mcp_maestro_mcp_run` with inline `yaml:` to validate each command before saving a file.
3. **Never hardcode credentials.** Always use env vars in YAML.
4. **Never use `point:` coordinates** as a Maestro selector — they are brittle.
5. **Always call `mcp_maestro_mcp_cheat_sheet`** at the start of any Maestro authoring or debugging task for live syntax reference.
6. **Sub-agents are not user-invocable.** `maestro-testcase-writer`, `maestro-scenario-composer`, and `maestro-selector-debugger` are spawned by orchestrators only.

## Agents

| Agent | Role | User-invocable |
|---|---|---|
| `qa-test-case-generator` | Generates mobile UI test cases from Jira/PRD/Figma | ✅ |
| `maestro-script-creator` | Orchestrates full Maestro YAML production from a CSV | ✅ |
| `maestro-script-debugger` | Autonomous loop to debug failing Maestro flows | ✅ |
| `maestro-testcase-writer` | Writes one atomic testcase YAML | sub-agent |
| `maestro-scenario-composer` | Composes scenario YAML from testcases | sub-agent |
| `maestro-selector-debugger` | Diagnoses and fixes one failing selector | sub-agent |

## Skills

| Skill | When to invoke |
|---|---|
| [`create-maestro-script`](skills/create-maestro-script/SKILL.md) | Writing Maestro testcase or scenario YAML |
| [`debug-maestro-script`](skills/debug-maestro-script/SKILL.md) | Debugging a failing Maestro flow |
| [`create-test-cases`](skills/create-test-cases/SKILL.md) | Generating new mobile UI test cases |
| [`regenerate-test-cases`](skills/regenerate-test-cases/SKILL.md) | Updating test cases from code diffs or PRs |

## Maestro Conventions

**Folder layout** (in the target app repo, not here):
```
maestro/
├── testcases/<screen>/    ← atomic testcase YAMLs
├── scenarios/<feature>/   ← orchestrating scenario YAMLs
├── commons/               ← shared flows (login, logout, launch)
└── utils/                 ← label-<locale>.js, setup_platform_locale.yaml
```

**File naming:** `<verb>_<target>.yaml` — e.g. `tap_login_button.yaml`. No numeric prefixes, no ticket-ID suffixes.

**Selector priority (highest → lowest):**
1. `text:` (visible label)
2. `id:` (Semantics `identifier:` → Android `resource-id`)
3. Index (last resort)
4. If nothing works → add `Semantics(identifier: "...", container: true)` to Flutter source, rebuild

**Localization:** All UI strings via `${output.localization.<camelCaseKey>}`. Keys live in `maestro/utils/label-<locale>.js`.

**Semantics rule:** Always pair `identifier:` with `container: true`. Any change requires `flutter build` + reinstall before Maestro can detect it.

**`.*` suffix:** Append to any `text:` assertion where the element may be a merged label+value accessibility node.

## Test Case Conventions

- IDs: `TC001`, `TC002`, … (sequential)
- Titles: `User able to …` (happy path) / `User not able to …` (negative)
- Categories: `Smoke` (P0 core happy path) or `Regression` (everything else)
- Output path: `/test-cases/{epic_key}_{short_desc}_test_cases.csv` + smoke-only variant

## Reference Files

- [Failure patterns](skills/debug-maestro-script/references/failure-patterns.md) — living knowledge base; check before debugging, append after fixing
- [Flutter Semantics guide](skills/create-maestro-script/references/flutter-semantics.md) — how to add `identifier:` to Flutter widgets
- [Testcase YAML template](skills/create-maestro-script/references/testcase_template.yaml)
- [Scenario YAML template](skills/create-maestro-script/references/scenario_template.yaml)

## MCP Tools Available

| Tool | Use |
|---|---|
| `mcp_maestro_mcp_run` | Run a YAML file or inline yaml — use to probe before saving |
| `mcp_maestro_mcp_inspect_screen` | Dump view hierarchy for selector discovery |
| `mcp_maestro_mcp_take_screenshot` | Screenshot for visual debugging |
| `mcp_maestro_mcp_cheat_sheet` | Live Maestro syntax reference |
| `mcp_maestro_mcp_list_devices` | List connected/emulated devices |
| Atlassian MCP | Read Jira tickets, post comments, read Confluence pages |
| Figma MCP | Fetch design context, screenshots, component metadata |

## Scope Limits

- **Maestro / mobile UI only.** Do not generate API tests, backend tests, or web tests.
- **No web, no desktop.** Scope is Flutter Android/iOS.
- **`create-test-cases` SKILL.md** gates test case structure — do not invent new fields.
