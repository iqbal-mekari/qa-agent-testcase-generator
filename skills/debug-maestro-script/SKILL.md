---
name: debug-maestro-script
description: "Debug failing Maestro UI test scripts for Android/iOS mobile apps. Use when a Maestro flow exits with code 1, a testcase or scenario fails, an element is not found, a tap does not navigate, an assertion fails, or a coordinate-based selector breaks. Triggers: 'maestro fails', 'element not found', 'flow failed', 'debug maestro', 'maestro error', 'fix maestro test', 'maestro assertion failed'."
argument-hint: 'Provide the path to the failing YAML file and the error message if available.'
---

# Debug Maestro Script

Diagnose and fix failing Maestro flows for Android/iOS mobile apps. Follows a **read → run → screenshot → hierarchy → probe → fix → re-run** loop.

## When to Use

- A `maestro test` exits with code 1
- `Element not found` error in a testcase or scenario
- A `tapOn` doesn't navigate to the expected screen
- An `assertVisible` fails despite the element appearing on screen
- A coordinate-based `point:` selector breaks after a layout change
- A selector worked on one platform (Android/iOS) but not the other

## Prerequisites

- Maestro MCP tools are running (`list_devices` returns a connected device)
- The YAML file path is known

---

## Debugging Workflow

### Phase 1 — Context Gathering (do all in parallel)

1. **Read the failing YAML** — understand the full flow: `onFlowStart`, `runFlow` chain, selectors, assertions
2. **Read all referenced sub-flows** (`runFlow` dependencies) to understand the complete execution path
3. **List devices** — confirm the connected device ID via `list_devices`

### Phase 2 — Reproduce the Failure

4. **Run the scenario** via MCP:

   ```
   mcp_maestro_mcp_run  files=[failing_file.yaml]  env={APP_ID, LOCALE, ...}
   ```

   Capture the exact error: which command failed and the error message.

5. **Take a screenshot immediately after the run** — the emulator/simulator stays on the screen where the flow stopped. This is your primary debugging context.

### Phase 3 — Diagnose

6. **Inspect the view hierarchy**:

   ```
   mcp_maestro_mcp_inspect_screen
   ```

   Look for:
   - The actual `txt` or `a11y` value of the target element (never assume from screenshot)
   - Whether a dialog/overlay is blocking the target element
   - Whether the element exists but has different text than expected (hint text, concatenated values)
   - The `rid` (resource-id) if no stable text is available

7. **Identify the root cause** using the [Common Failure Patterns](./references/failure-patterns.md) table.

### Phase 4 — Probe the Fix Inline

8. **Test the corrected command** without editing the file:
   ```
   mcp_maestro_mcp_run  yaml="appId: <APP_ID>\n---\n- <corrected_command>"
   ```
   Confirm it succeeds before touching the YAML.

### Phase 5 — Apply and Verify

9. **Edit the YAML file** with the confirmed fix.
10. **Re-run the full scenario** end-to-end to check for cascading failures.
11. If it still fails, repeat from Phase 2 with the new error.

### Phase 6 — Record Unrecognized Failure Patterns

After a fix is confirmed, check if the root cause matches any entry in [failure-patterns.md](./references/failure-patterns.md).

**If the root cause is NOT already documented:**

1. Append a new entry to `references/failure-patterns.md` following the existing format:
   - **Error** — the exact Maestro error message or symptom
   - **Root cause** — why it happened
   - **Diagnosis** — how to identify it via screenshot/hierarchy
   - **Fix** — the corrected YAML with before/after example
2. Assign the next sequential pattern number.
3. Briefly mention the new entry in your final reply to the user.

This ensures the knowledge base grows with every debugging session and future failures are resolved faster.

### Exit Condition

Stop and report findings if:

- The same step fails 3 times with different attempted fixes
- The fix requires a Flutter code change (add `Semantics`, rebuild APK) — report exactly what change is needed
- The failure is in an external system (network, test account, backend data)

---

## Common Failure Patterns

See [failure-patterns.md](./references/failure-patterns.md) for the full reference table.

**Quick reference (most frequent):**

| Error                                        | Root Cause                                      | Fix                                                             |
| -------------------------------------------- | ----------------------------------------------- | --------------------------------------------------------------- |
| `Element not found: Text matching regex: X`  | Exact regex match fails on partial text         | Append `.*`: `text: 'X.*'`                                      |
| `Element not found: Text matching regex: X`  | Dialog/overlay blocking the element             | Dismiss the dialog first, then retry the step                   |
| `Element not found: Text matching regex: X`  | Hint text is `"X\n0/N"` (character counter)     | Use `text: 'X.*'`                                               |
| Tap does nothing / navigates wrong screen    | Duplicate text label (header + button)          | Add `index: 1` to target the button                             |
| `point: 'X%,Y%'` misses the button           | Layout shift / different screen density         | Replace with `tapOn: 'button_text'` using `a11y` from hierarchy |
| `assertVisible` fails despite text on screen | Label+value merged into one accessibility node  | Use `text: 'Label.*'` or `text: '.*Label.*'`                    |
| Flow passes but wrong screen                 | Navigation succeeded but assertion is too loose | Tighten assertion to screen-specific element                    |

---

## Selector Decision Tree

For the full selector priority hierarchy, decision tree, and accessibility node merging rules, see [shared-references/selector-rules.md](../shared-references/selector-rules.md).

**Quick reference:**

```
hierarchy element has `txt` (non-empty)?
  YES → use tapOn: 'exact_txt_value'  (or 'txt_value.*' if merged node)
  NO  → has `a11y` (non-empty)?
        YES → use tapOn: 'a11y_value'  (maps to text: in Maestro)
        NO  → has `rid` (resource-id)?
              YES → use tapOn: id: 'rid_value'
              NO  → same text appears multiple times?
                    YES → use tapOn: text: 'X' + index: N
                    NO  → add Semantics(identifier: 'name', container: true) to Flutter widget
                          rebuild APK, then use tapOn: id: 'name'
```

**Never use `point: 'X%,Y%'`** unless absolutely no other option exists. If a `point:` exists in the file and is failing, replace it first.

---

## Key Rules (from create-maestro-script skill)

- **`text:` is full-string regex** — `text: 'Foo'` only matches elements whose ENTIRE text equals "Foo". Use `text: 'Foo.*'` for prefix matches.
- **`a11y` → `text:` in selectors** — The `a11y` field in the hierarchy maps to Maestro's `text:` matcher. Never use `a11y` as a key.
- **Never use coordinates** — `point:` breaks across screen sizes/densities. The presence of a `# TODO` coordinate comment is a signal to replace it.
- **Rebuild required after Flutter code changes** — `Semantics` additions are not reflected until you rebuild and reinstall the app.
- **Timeout limits**: `waitForAnimationToEnd` max 600ms, `extendedWaitUntil` up to 3000ms.

---

## Tool Sequence (copy-paste reference)

````
1. list_devices                          → get device_id
2. run(files=[...], env={...})           → get first error
3. take_screenshot                       → see where flow stopped
4. inspect_screen                        → get actual txt/a11y/rid values
5. run(yaml="appId:...\n---\n- fix")     → probe the fix inline
6. edit the YAML file
7. run(files=[...], env={...})           → full scenario re-run8. append to `failure-patterns.md`         → if root cause was new```
````
