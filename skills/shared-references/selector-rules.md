# Maestro Selector Rules (Shared Reference)

This is the single source of truth for Maestro selector strategies across all skills and agents.

## Selector Priority Hierarchy

Before writing any selector, gather context from:

1. **Live UI tree** — `mcp_maestro_mcp_inspect_screen` for runtime text/IDs/states
2. **Screen source code** — Read Flutter screen file for widget keys (`Key('...')`), `Semantics(identifier: '...')`, and stable string constants

Then follow this priority order:

### 1. Text selector (highest priority)

Element has visible, stable text:

```yaml
- tapOn: 'Login'
- assertVisible: 'Welcome'
```

### 2. Semantic identifier

Element has no visible text but has resource ID/semantic tag:

```yaml
- tapOn:
    id: 'email_field'
- inputText: 'test@example.com'
```

**Flutter Semantics rule:** Always pair `identifier:` with `container: true`:

```dart
Semantics(
  identifier: 'widget_name',
  container: true,
  child: YourWidget(),
)
```

### 3. Index (when duplicate text exists)

When the same text appears multiple times (e.g., header + button):

```yaml
- tapOn:
    text: 'Submit'
    index: 1  # 0 = header, 1 = button
```

### 4. Relative positioning (last resort before code change)

```yaml
- tapOn:
    text: 'Submit'
    below: 'Terms and Conditions'
```

### 5. Add Semantics (when nothing else works)

If no selector works, add `Semantics(identifier: 'widget_key', container: true)` to the Flutter widget, rebuild, and use `id:` selector.

**⛔ NEVER use `point:` coordinates** — they break across screen sizes/densities.

---

## Accessibility Node Merging

Flutter widgets that render label + value in a `Row` often merge into a single accessibility node:

```
Field label
Field value
```

**Fix:** Append `.*` to label-only assertions:

```yaml
# WRONG — exact match fails
- assertVisible: '${output.localization.fieldLabel}'

# CORRECT — regex match
- assertVisible: '${output.localization.fieldLabel}.*'
```

**Route prefix merging:** Parent containers may emit route + child values:

```yaml
# Use leading .* when element text is prefixed by route path
- assertVisible: '.*${output.localization.sectionHeading}.*'
```

**Detection:** Use `mcp_maestro_mcp_inspect_screen` and check the `a11y` field. If it starts with something other than the label (e.g., a route path), add a leading `.*`.

---

## Selector Decision Tree

```
hierarchy element has `txt` (non-empty)?
  YES → use tapOn: 'exact_txt_value'  (or 'txt_value.*' if merged node)
  NO  → has `a11y` (non-empty)?
        YES → use tapOn: 'a11y_value'  (maps to text: in Maestro)
        NO  → has `rid` (resource-id)?
              YES → use tapOn: id: 'rid_value'
              NO  → same text appears multiple times?
                    YES → use tapOn: text: 'X' + index: N
                    NO  → add Semantics(identifier: 'name', container: true)
                          rebuild APK, then use tapOn: id: 'name'
```

---

## Timeout Conventions

| Action | Max timeout |
|--------|-------------|
| `waitForAnimationToEnd` | 600ms |
| `extendedWaitUntil` | 3000ms |
| Screen transition wait | Use `assertVisible` with default timeout |

---

## Rebuild Requirement

After adding `Semantics` or any Flutter code changes:

1. Rebuild the app (`flutter build`)
2. Reinstall on device/emulator
3. Only then re-run Maestro tests

Maestro tests the **compiled APK/IPA**, not source code. Changes are not reflected until rebuild + reinstall.
