---
name: create-maestro-script
description: Creates Maestro UI test scripts (testcases and scenarios) for Flutter mobile applications. Use this when the user wants to write Maestro tests from a test plan, generate testcases for a screen, create end-to-end user journey scenarios, or automate mobile UI testing. Covers folder structure, selector strategies, localization patterns, and test execution.
license: Proprietary
---

# Create Maestro Scenario (Flutter)

Creates Maestro UI test scripts (testcases and scenarios) from a CSV test plan. For Flutter-based mobile applications.

## Trigger Keywords

Use this skill when the user mentions:

- "maestro", "Maestro"
- "UI test", "ui test", "UITest"
- "test automation", "test script"
- "test scenario", "test case", "testcases"
- "automate", "automation"
- Provides a CSV/spreadsheet of test cases
- Wants to create end-to-end mobile app tests

## When to Use

Use this skill whenever you want to:

- Write Maestro tests from a test case spreadsheet
- Generate testcases for a mobile screen
- Create test scenarios for a user journey
- Automate end-to-end UI testing with Maestro
- Build test suites from CSV/Jira test cases

## Core Concepts

### Testcase vs Scenario

| Aspect          | Testcase                                 | Scenario                                       |
| --------------- | ---------------------------------------- | ---------------------------------------------- |
| **Purpose**     | Atomic test — single action/verification | User journey — orchestrates multiple testcases |
| **Scope**       | Single screen, single interaction        | Multiple screens, complete flow                |
| **Structure**   | AAA (Arrange-Act-Assert)                 | Orchestrates testcases + state management      |
| **Reusability** | Reused across scenarios                  | Self-contained user journey                    |
| **Example**     | `tap_login_button.yaml`                  | `login_success_and_failure.yaml`               |

### Folder Structure

```
maestro/
├── testcases/           # Atomic tests (AAA pattern)
│   ├── login/
│   │   ├── verify_login_form_visible.yaml
│   │   └── tap_login_button.yaml
│   └── home/
│       └── verify_welcome_visible.yaml
├── scenarios/           # User journeys (orchestrates testcases)
│   └── login/
│       └── login_success_and_failure.yaml
├── commons/             # Shared flows (login, logout, app launch)
│   ├── app_launch_clean.yaml
│   └── login.yaml
└── utils/               # JS helpers & locale setup
    ├── label-<locale>.js  # One file per supported locale
    └── setup_platform_locale.yaml
```

## IMPORTANT: Skill Authority Rule

**When executing this skill, IGNORE all existing Maestro patterns in the codebase.** Only follow the patterns, folder structure, naming conventions, and rules defined in THIS skill document.

Existing test files may use legacy conventions (e.g., `topics/` folders, `_C<id>` suffixes, ordering numbers like `01_`, lifecycle hooks in testcases, `evalScript` completed tracking, `flows/` folder). These are **outdated** and must NOT be replicated. This skill document is the single source of truth.

## Prerequisites

Before using this skill, ensure you have:

1. **Maestro MCP tools available** — The MCP server must be running and accessible
2. **Maestro syntax reference** — Call `mcp_maestro_mcp_cheat_sheet` to get command reference
3. **Understanding of folder structure**:
   - `testcases/` — Atomic tests (single scenario, AAA + hooks)
   - `scenarios/` — User journeys (orchestrates testcases)
   - `commons/` — Shared flow
   - `utils/` — JS helpers

4. **Testcase naming format** — Start with action-based prefixes:
   - `tap_`, `verify_`, `check_`, etc.
   - **No ticket ID suffix** (e.g., `_C123456`)
   - **No ordering numbers** (e.g., `01_`, `02_`)

## Rules

### Rule 1: Testcase Scope = Screen Scope

Each `testcases/<feature>/` folder maps 1-to-1 with a screen (or a major section of a screen).

- A testcase folder covers everything a user can do on that screen: layout checks, interactions, error states
- Testcases are reused across multiple scenarios — never duplicated
- Folder name must match the screen/feature name:
  - `testcases/home/` → `home_screen.dart`
  - `testcases/<feature>/` → `<feature>_screen.dart`
- **Shared screens** (e.g., `testcases/home/`) may already have testcases from other features. Always scan existing files before creating new ones — reuse if an equivalent test exists.
- **Before creating a new testcase**, scan existing folders to check if an equivalent test already exists. Reuse existing testcases — do not duplicate.

**IMPORTANT: Testcases are atomic and simple.**

**runFlow policy for testcases:**

- ❌ **NEVER** use `runFlow` to call other testcases
- ✅ **CAN** use `runFlow` with `when: visible:` for conditional command execution (e.g., dismiss dialogs that may or may not appear)
- ✅ **CAN** use `runFlow` with `repeat:` for looping patterns

**runFlow policy for scenarios:**

- ✅ **PRIORITIZE** using existing testcases via `runFlow` — compose user journeys from reusable atomic tests
- ✅ **USE `commons/`** for shared multi-step flows (login, logout, app launch)
- ⚠️ **AVOID** duplicating testcase logic with inline commands — only use commands when no testcase exists

### Rule 2: Never Use Point Coordinates

> **Full reference:** See [shared-references/selector-rules.md](../../shared-references/selector-rules.md) for the complete selector decision tree, accessibility node merging rules, and timeout conventions.

**Selector Priority Hierarchy**

Before writing selectors, gather context from:

1. **Live UI tree** — `mcp_maestro_mcp_inspect_screen` for runtime text/IDs/states
2. **Screen source code** — Read Flutter screen file for widget keys (`Key('...')`), `Semantics(identifier: '...')`, and stable string constants

Then follow this priority order:

1. `text:` — visible, stable label on the element
2. `id:` — Semantics `identifier:` (always paired with `container: true` in Flutter)
3. `text + index:` — when the same text appears in header AND button
4. Relative positioning (`below:`, `above:`) — last resort before code change
5. Add `Semantics(identifier: '...', container: true)` to Flutter source if nothing works — **never use coordinates**

**Common patterns:**

```yaml
# Text selector
- tapOn: 'Login'

# Semantic identifier
- tapOn:
    id: 'email_field'

# Index for duplicate labels (title + button)
- tapOn:
    text: '${output.localization.submitAction}'
    index: 1  # 0 = title/header, 1 = button

# Relative positioning
- tapOn:
    text: 'Submit'
    below: 'Terms and Conditions'
```

**IMPORTANT:** After adding `Semantics` or any code changes to enable selectors, you MUST rebuild and reinstall the app before re-running scenarios:

```bash
# Rebuild and install the Flutter app
# Each project may have different build commands and flags
flutter build

```

Maestro tests the **compiled APK/IPA** on the device, not your source code. Changes to Flutter code (including Semantics) are not reflected until you rebuild and reinstall.

### Rule 3: Always Verify Copy Against Screen and ARB

#### Accessibility Node Merging

Flutter widgets that render a label + value in a `Row` (e.g., a list tile with caption + value) often combine both texts into a single accessibility node. The resulting `accessibilityText` is a multi-line string like:

```
Field label
Field value
```

This means `assertVisible: 'Field label'` **fails**, because the element's actual text is `Field label\nField value` — an exact text match won't work.

**Fix: always append `.*` to label-only assertions** on such screens:

```yaml
# WRONG — exact match fails when value is merged into same node
- assertVisible: '${output.localization.fieldLabel}'

# CORRECT — regex match succeeds
- assertVisible: '${output.localization.fieldLabel}.*'
```

**Route announcement nodes (parent container merging):**

Some parent containers emit a combined `accessibilityText` that includes the screen route + all child values, e.g.:

```
/feature/detail
Section heading
Section value
```

In this case `Section heading.*` still fails because the text starts with `/feature/detail\n`. Use a **leading wildcard** as well:

```yaml
# WRONG — text doesn't start with the label
- assertVisible: '${output.localization.sectionHeading}.*'

# CORRECT — leading .* covers any route prefix emitted by parent
- assertVisible: '.*${output.localization.sectionHeading}.*'
```

**How to detect which case you're in:** Use `inspect_screen` and look at the `a11y` of the element. If it starts with something other than the label (e.g., a route path), add a leading `.*`.

Every string in `tapOn`, `inputText`, `assertVisible`, or `assertNotVisible` MUST be verified against actual app text.

**Two-step verification (MANDATORY before writing YAML):**

1. **Step A** — Find the key in the screen source file
2. **Step B** — Look up the key in the ARB file

If the string is dynamic (parameterized), use regex to match the filled-in value, not the template literal.

**Always use `${output.localization.<camelCaseKey>}` in testcases and scenarios** for locale-agnostic text matching.

**Localization key naming in `label-id.js` / `label-en.js`:**

- Convert ARB snake_case keys to camelCase, stripping the module prefix and `_label` suffix
- Pattern: `<arb_prefix>_<name>_label` → `<prefix><Name>`
- Examples:
  - `login_email_label` → `loginEmail`
  - `login_submit_button_label` → `loginSubmitButton`
  - `settings_dark_mode_label` → `settingsDarkMode`
  - `error_invalid_input` → `errorInvalidInput`
- If a key is already in a locale label file, reuse it — do not duplicate

**Default locale** — Check your project's default locale. When verifying localization keys, check both the primary locale and any secondary locales in your ARB files.

Example mapping table (generic):

| ARB Key               | Label (en)      | Label (primary) | Usage in Testcases                         |
| --------------------- | --------------- | --------------- | ------------------------------------------ |
| `login_button_label`  | "Login"         | "Masuk"         | `${output.localization.loginButtonLabel}`  |
| `email_address_label` | "Email address" | "Alamat email"  | `${output.localization.emailAddressLabel}` |
| `confirm_action`      | "Confirm"       | "Konfirmasi"    | `${output.localization.confirmAction}`     |

### Rule 4: Never Hardcode Credentials

Use environment variables or test accounts — never commit real credentials.

### Rule 5: Running Maestro Tests

Always validate scenarios by running them through `maestro test` directly.

**When a flow fails:**

1. **Take screenshot FIRST** — Use `mcp_maestro_mcp_take_screenshot` to capture current screen state
2. **Then inspect view hierarchy** — Use `mcp_maestro_mcp_inspect_screen` to reveal real accessibilityText, resource-id, element bounds
3. **Trust screenshot/hierarchy as source of truth** — If screenshot/hierarchy show the previous screen but Maestro output indicates it's on the next screen, the navigation FAILED. Fix the navigation step before proceeding.
4. Probe inline with `mcp_maestro_mcp_run` (inline `yaml:`) to test a single corrected command before editing YAML
5. Fix the YAML file
6. Re-run full integration via `maestro test` to confirm end-to-end fix

**Navigation debugging:**

- If tapOn fails to navigate: Check for duplicate text labels (use `index:` to specify)
- If screen appears unchanged: Add `waitForAnimationToEnd` before the tap
- If element not found: Use `inspect_screen` to get the exact text/ID

**Premature failures (10–30s)** — retry up to 3 times.

### Rule 6: Correct inputText Pattern

Maestro's `inputText` only accepts a simple string. To input into a specific field:

**CORRECT:**

```yaml
- tapOn:
    id: 'email_field'
- inputText: 'test@example.com'
```

**INCORRECT:**

```yaml
- inputText:
    id: 'email_field'
    text: 'test@example.com' # This syntax is invalid!
```

### Rule 7: Timeout Limits

Timeout limits differ by command — do not mix them up:

**`waitForAnimationToEnd` and `scrollUntilVisible`** — maximum **600ms**:

```yaml
- waitForAnimationToEnd:
    timeout: 500 # Good: under 600ms

- scrollUntilVisible:
    element: 'Submit Button'
    timeout: 600 # Maximum allowed
```

**`extendedWaitUntil`** — used for waiting on slow UI transitions (e.g., after app launch, login, navigation). Accepts values in **milliseconds with a higher limit**. Typical range: 1000–3000ms:

```yaml
# Wait up to 1 second for Sign In to appear after app cold start
- extendedWaitUntil:
    visible: 'Sign In'
    timeout: 1000

# Wait up to 3 seconds for a slow loading screen
- extendedWaitUntil:
    visible: 'Home Screen'
    timeout: 3000
```

## Common Patterns

### Text Input Pattern

Always tap the field first to focus it, then input text:

```yaml
- tapOn:
    id: 'email_field'
- inputText: 'test@example.com'
- tapOn:
    id: 'password_field'
- inputText: 'SecurePass123!'
- tapOn: 'Login'
```

### Conditional Execution

Run commands only when an element is visible:

```yaml
- runFlow:
    file: testcases/login/verify_login_form_visible.yaml
    when:
      visible: 'Login Button'
```

### State Reset Between Scenarios

Navigate back to a known state before independent error scenarios:

```yaml
# Happy path
- runFlow: testcases/feature/success.yaml

# Reset state
- runFlow: commons/navigate_to_home.yaml
- runFlow: commons/logout.yaml

# Error path
- runFlow: testcases/feature/error.yaml
```

### Repeat Commands

**Repeat while an element is visible:**

```yaml
- repeat:
    while:
      visible: 'Load More'
    commands:
      - tapOn: 'Load More'
      - waitForAnimationToEnd
```

**Repeat for tap retry (screen not ready):**

When a screen might not be ready for interaction immediately (e.g., home screen after login), use repeat to retry the tap:

```yaml
- evalScript: ${output.targetTapAttempt = 0}
- repeat:
    while:
      true: ${output.targetTapAttempt < 3}
    commands:
      - tapOn: 'Target Button'
      - runFlow:
          when:
            visible: 'Expected Next Screen Element'
          commands:
            - evalScript: ${output.targetTapAttempt = 3}
      - evalScript: ${output.targetTapAttempt = output.targetTapAttempt + 1}
```

This pattern retries the tap up to 3 times until the expected next screen element appears.

### Env Variable Passthrough

Pass test data from scenarios to testcases via `env:` in `runFlow`:

```yaml
# In scenario:
- runFlow:
    file: ../../testcases/<feature>/select_option.yaml
    env:
      OPTION_VALUE: '${OPTION_VALUE}'

# In testcase (select_option.yaml):
- tapOn: '${output.localization.optionDropdown}'
- inputText: '${OPTION_VALUE}'
- tapOn: '${OPTION_VALUE}'
```

Testcases that need external data should document required env vars in their ARRANGE comments.

## Lifecycle Hooks (Scenarios Only)

Scenarios support lifecycle hooks for setup/teardown:

```yaml
onFlowStart:
  # Launch app with clean state
  - launchApp:
      clearState: true
      clearKeychain: true
      stopApp: true
      permissions: { all: allow }

  # Setup platform/locale configuration
  - runFlow: ../../utils/setup_platform_locale.yaml
```

**Important:** Testcases do NOT use lifecycle hooks — only scenarios.

## Common Pitfalls

| Pitfall                                   | Why It's Wrong                                             | Correct Approach                                                                           |
| ----------------------------------------- | ---------------------------------------------------------- | ------------------------------------------------------------------------------------------ |
| Using coordinates                         | Fragile, breaks on different screen sizes                  | Use text/id selectors                                                                      |
| Hardcoded strings                         | May not match actual app text                              | Use `${output.localization.<key>}`                                                         |
| `inputText` with `id`/`text` map          | Invalid Maestro syntax                                     | Tap field first, then `inputText: "value"`                                                 |
| Duplicating testcases                     | Maintenance nightmare                                      | Reuse existing testcases                                                                   |
| Skipping state reset                      | Flaky tests due to leftover state                          | Reset between independent scenarios                                                        |
| Using `runFlow` in testcases              | Testcases must be atomic/simple                            | Scenarios handle orchestration; testcases only use `runFlow` for conditional/loop patterns |
| Not using index for duplicates            | Taps wrong element when text appears multiple times        | Use `index: 1` for second occurrence (common for header + button)                          |
| Not using `container: true` in Semantics  | Widget children not exposed to accessibility               | Always add `container: true` when adding Semantics for testing                             |
| Exact text match on label+value nodes     | Flutter Row widgets merge label+value into one node        | Append `.*` to the assertion: `'${output.localization.myLabel}.*'`                         |
| Exact label match fails on route nodes    | Parent container prepends route path to accessibility text | Use leading wildcard: `'.*${output.localization.myLabel}.*'`                               |
| Using `id:` for buttons without Semantics | `resource-id` doesn't exist — Maestro can't find element   | Use `text: + index:` selector instead                                                      |

## Guidelines

### Writing Testcases & Scenarios

**Step 1 · Parse the test case file**

When user provides a test case spreadsheet (CSV, Jira, etc.), extract for each case:

- **Title** — what is being tested
- **Priority** — P0 / P1 / P2
- **Section header** — the group (maps to a screen)
- **Preconditions** — required state before test
- **Steps** — user actions
- **Expected result** — what the app should show

**Step 2 · Separate by screen (Rule 1)**

Group test cases by the screen they exercise. Use section headers as primary signal; cross-reference with screen names in the Flutter codebase.

**Finding screen source code:** Look for files with `_screen.dart` suffix (e.g., `login_screen.dart`, `home_screen.dart`).

**Step 3 · Triage — decide what to skip**

Classify each test case:

- ✅ **Automate** — pure UI interaction, stable data, no external dependencies
- ⚠️ **Prompt developer** — automatable but requires account/data setup. Do NOT skip silently — ask for confirmation on env variables.
- ❌ **Skip for now** — hard dependency Maestro cannot satisfy. Add `# SKIP:` comment explaining why.

**Step 4 · Produce mapping table**

Before writing YAML, produce a mapping table of feasible cases. Present to developer for confirmation before proceeding.

Recommended columns: CSV Test Case, Priority, Automate?, Screen Folder, Testcase File, Notes.

**Step 5 · Write the testcases**

After mapping is confirmed, implement each file following testcase template:

1. **Do NOT include navigation** — testcases are atomic and assume they're already on the target screen
2. Use `mcp_maestro_mcp_inspect_screen` + source code to find selectors (Rule 3)
3. **Handle screens not ready for interaction** — Use repeat pattern to retry taps if needed
4. Write ARRANGE → ACT → ASSERT
5. Run via `mcp_maestro_mcp_run` (inline `yaml:`) to validate before saving

**Step 6 · Write the scenario**

Once all testcases exist, compose the scenario:

1. **PRIORITIZE existing testcases** — Use `runFlow:` to call testcases instead of duplicating commands
2. Only use inline commands when no testcase exists for the action
3. Group testcases into logical user journey (happy path first, then error paths)
4. Reset to known state between independent error scenarios (navigate back to home, then re-enter feature)
5. Wrap conditional testcases in `runFlow: when: visible:`
6. End with `assertVisible` on final screen route or distinctive landmark element

## Tool Usage

This skill uses the following Maestro MCP tools:

- `mcp_maestro_mcp_cheat_sheet` — Get Maestro syntax reference
- `mcp_maestro_mcp_inspect_screen` — Get UI element tree as compact JSON
- `mcp_maestro_mcp_run` — Execute Maestro commands: inline `yaml:`, specific `files:`, or a `dir:` with tag filters; syntax is validated automatically
- `mcp_maestro_mcp_take_screenshot` — Capture screen for debugging
- `mcp_maestro_mcp_list_devices` — List available devices

## References

Template files are available in the `references/` folder:

- `references/testcase_template.yaml` — AAA pattern template for atomic testcases
- `references/scenario_template.yaml` — User journey orchestration template
- `references/flutter-semantics.md` — Guide for adding `Semantics(identifier: '...')` to Flutter widgets
