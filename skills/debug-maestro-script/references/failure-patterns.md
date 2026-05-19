# Maestro Failure Patterns Reference

Catalog of known failure root causes and their fixes. Derived from real debugging sessions.

---

## 1. Coordinate-Based Tap Misses Target

**Error:** Element not found — or tap silently lands on wrong widget.  
**Symptom:** A `point: 'X%,Y%'` step either does nothing, navigates to wrong screen, or partially works (e.g. opens a dialog but dismisses on wrong button).

**Root cause:** `point:` coordinates are calculated as a percentage of the screen's pixel dimensions. These shift when:

- Screen resolution or density differs from the device used when the test was written
- A parent layout wraps or expands (e.g. bottom sheet height changes)
- The same flow runs on a different OS version

**Diagnosis:**

1. Run `inspect_screen` — find the element's exact `b` (bounds): `[x1,y1][x2,y2]`
2. Compute center: `cx = (x1+x2)/2`, `cy = (y1+y2)/2`
3. Compare to `point: 'X%,Y%'` — check if `X%*screen_w ≈ cx` and `Y%*screen_h ≈ cy`
4. Off by more than ~10px → replace with a text/id selector

**Fix:** Replace `point:` with the element's `a11y` or `txt` value:

```yaml
# Before (fragile)
- tapOn:
    point: '89%,68%'

# After (stable)
- tapOn: 'Save' # from a11y="Save" in hierarchy
```

**Note:** A `# TODO` comment next to a `point:` in the file is a strong signal that this is a known fragile selector.

---

## 2. Exact Text Match Fails on Merged Accessibility Node

**Error:** `Element not found: Text matching regex: Field Label`  
**Symptom:** The text IS visible on screen (confirmed by screenshot) but `tapOn`/`assertVisible` still fails.

**Root cause:** Flutter `Row` widgets that combine a label + value into a single node produce an `accessibilityText` (mapped to `a11y` in the hierarchy) that is a multi-line string:

```
Field label
Field value
```

Maestro's `text:` uses full-string regex. `text: 'Field label'` requires an EXACT match — it won't match `"Field label\nField value"`.

**Diagnosis:** In the hierarchy, find the element and check its `a11y` value. If it contains `\n` with the field value appended, this is the cause.

**Fix:**

```yaml
# Wrong — exact match fails
- assertVisible: 'Field label'

# Correct — regex match covers the merged value
- assertVisible: 'Field label.*'
```

---

## 3. Route Prefix in Parent Container Node

**Error:** `assertVisible` fails even with `.*` suffix.  
**Symptom:** Same as pattern 2 but even `'Label.*'` doesn't work.

**Root cause:** Some parent containers emit `accessibilityText` that starts with the screen's route path:

```
/feature/detail
Section heading
Section value
```

Because the text STARTS with the route (not the label), `'Section heading.*'` still fails.

**Diagnosis:** In the hierarchy, look at the parent container's `a11y`. If it starts with a `/route` path, this is the cause.

**Fix:** Add a leading wildcard:

```yaml
# Wrong
- assertVisible: 'Section heading.*'

# Correct — leading .* covers the route prefix
- assertVisible: '.*Section heading.*'
```

---

## 4. Hint Text Contains Character Counter

**Error:** `Element not found: Text matching regex: Reason`  
**Symptom:** An `EditText` input field is visible but `tapOn: 'Reason'` fails.

**Root cause:** Android `EditText` hint text often includes a character counter appended by the framework: `"Reason\n0/200"`. Maestro's full-string regex `'Reason'` won't match `"Reason\n0/200"`.

**Diagnosis:** In the hierarchy, look for the field's `hint` value. If it's `"Label\n0/N"`, this is the cause.

**Fix:**

```yaml
# Wrong
- tapOn: 'Reason'

# Correct
- tapOn: 'Reason.*'
```

---

## 5. Dialog Blocking Target Element

**Error:** `Element not found: Text matching regex: Clock out`  
**Symptom:** The target element IS in the view hierarchy but a modal/bottom sheet is on top.

**Root cause:** A previous step (e.g. a time/date picker) was not properly dismissed. The dialog remains open and intercepts all touches, making elements behind it effectively unreachable for Maestro.

**Diagnosis:**

1. Take a screenshot — if a dialog/overlay covers the screen, this is the cause
2. Check the hierarchy — the dialog's elements will appear at the TOP of the tree (highest z-order last in the JSON)

**Fix:** Ensure the dismissal step actually works before proceeding:

```yaml
# If the dismiss button is a known text
- tapOn: 'Save' # verify via inspect_screen first

# If the dialog may or may not appear (conditional)
- runFlow:
    when:
      visible: 'Select time'
    commands:
      - tapOn: 'Save'
```

---

## 6. Duplicate Text Label (Header + Button)

**Error:** The wrong element gets tapped (e.g. the screen title instead of the CTA button).  
**Symptom:** Flow taps a label in the app bar when it should tap a button further down.

**Root cause:** Many screens show the same text in both the top bar title AND a button. `tapOn: 'Submit'` always picks the first match (index 0 = top of screen = the title).

**Diagnosis:** In the hierarchy, count how many elements share the same `txt` or `a11y`. More than one → use `index`.

**Fix:**

```yaml
# Wrong — taps the title (index 0)
- tapOn: 'Submit'

# Correct — taps the button (index 1)
- tapOn:
    text: 'Submit'
    index: 1
```

---

## 7. Missing Semantics on Flutter Widget

**Error:** No `rid` in hierarchy, no `a11y`, no visible `txt`.  
**Symptom:** The element exists visually but has nothing stable to select.

**Root cause:** The Flutter widget has no `Semantics` wrapper and no inherent text.

**Fix:** Add `Semantics` to the Flutter source:

```dart
Semantics(
  identifier: 'widget_name',   // Maps to id: in Maestro
  container: true,              // Always required
  child: YourWidget(),
)
```

Then use in Maestro:

```yaml
- tapOn:
    id: 'widget_name'
```

**Important:** Rebuild and reinstall the app after any Flutter code change. Maestro tests the compiled binary.

---

## 8. Time Picker / Date Picker Save Button Not Found

**Error:** Coordinate tap on time picker Save button does nothing or misses.  
**Root cause:** The MpTimePickerDialog (custom picker component) renders its Save button inside a bottom sheet. The button's position is fixed within the sheet, but the sheet's position varies by screen size.

**Fix:** Use the button's `a11y` text instead of coordinates:

```yaml
# Before (fragile — TODO in original code)
- tapOn:
    point: '89%,68%'

# After (stable — Save button has a11y="Save" in hierarchy)
- tapOn: 'Save'
```

---

## iOS-Specific Patterns

### 9. Element Found on Android but Not on iOS (Different Accessibility Tree)

**Error:** `Element not found` on iOS for a selector that works on Android.  
**Root cause:** Flutter renders differently on iOS vs Android. The accessibility tree structure, `a11y` (content-desc on Android / accessibilityLabel on iOS) values, and node grouping can differ between platforms even for the same Flutter widget.

**Diagnosis:** Run `inspect_screen` on both platforms and compare. Look for:

- Different `a11y` text (e.g. Android: `"Submit"` / iOS: `"Submit button"`)
- Different node grouping (merged vs split nodes)
- Elements present in one tree but absent in the other

**Fix:** Use platform-conditional steps when the selector must differ:

```yaml
- runFlow:
    when:
      platform: Android
    commands:
      - tapOn: 'Submit'
- runFlow:
    when:
      platform: iOS
    commands:
      - tapOn: 'Submit button'
```

Or add a shared `Semantics(identifier: 'submit_button', container: true)` to the Flutter widget and use `id: 'submit_button'` on both platforms.

---

### 10. `resource-id` Doesn't Exist on iOS

**Error:** `Element not found` when using `id:` selector on iOS.  
**Root cause:** `resource-id` is an Android-only attribute. On iOS, Maestro maps `id:` to the `accessibilityIdentifier` — which is only set when `Semantics(identifier: '...')` is applied to the Flutter widget.

**Diagnosis:** Run `inspect_screen` on iOS — `rid` field will be empty for all Flutter-rendered elements unless `Semantics(identifier:)` is present.

**Fix:** If both platforms need `id:` selectors, the Flutter widget MUST have:

```dart
Semantics(
  identifier: 'my_widget',  // accessibilityIdentifier on iOS, resource-id on Android
  container: true,
  child: YourWidget(),
)
```

Rebuild and reinstall after adding `Semantics`.

---

### 11. iOS Keyboard Blocks Element After Text Input

**Error:** `Element not found` on the next step after `inputText`.  
**Symptom:** Works on Android, fails on iOS. The software keyboard stays up and covers the target element.

**Root cause:** On iOS simulators, the software keyboard does not auto-dismiss after `inputText`. Elements below the fold become unreachable while the keyboard is visible.

**Fix:** Dismiss the keyboard explicitly before the next interaction:

```yaml
- tapOn:
    id: 'email_field'
- inputText: 'test@example.com'
- hideKeyboard # Dismiss iOS soft keyboard before next tap
- tapOn: 'Next'
```

---

### 12. iOS Secure Text Field Not Receiving Input

**Error:** `inputText` silently does nothing on a password field.  
**Root cause:** iOS secure text fields sometimes lose focus between `tapOn` and `inputText`, especially if an animation is in progress.

**Fix:** Add `waitForAnimationToEnd` before inputting, and ensure the field is explicitly focused:

```yaml
- tapOn:
    id: 'password_field'
- waitForAnimationToEnd:
    timeout: 500
- inputText: '${PASSWORD}'
```

---

### 13. iOS Back Navigation Fails

**Error:** Flow gets stuck on a screen — back button does nothing.  
**Root cause:** iOS uses a swipe-back gesture for navigation, not a hardware back button. `pressKey: Back` (Android) has no equivalent on iOS.

**Fix:** Use `back` command (Maestro handles platform routing) or tap the explicit back button element:

```yaml
# Cross-platform back
- back

# Or tap the back chevron by its a11y label (iOS-specific label)
- tapOn: 'Back' # or whatever the screen's back button a11y is
```

---

### 14. `clearKeychain` Required for Clean State on iOS

**Symptom:** iOS scenario picks up leftover auth tokens from a previous run. Login screen is skipped unexpectedly, or the wrong account is active.  
**Root cause:** iOS stores auth tokens in the Keychain, which persists across `launchApp` even with `clearState: true`. Android uses SharedPreferences/databases which `clearState` wipes completely.

**Fix:** Always include `clearKeychain: true` in `onFlowStart` for iOS:

```yaml
onFlowStart:
  - launchApp:
      clearState: true
      clearKeychain: true # Required on iOS; harmless on Android
      stopApp: true
      permissions: { all: allow }
```

---

### 15. iOS Simulator Animation Not Disabled

**Symptom:** `extendedWaitUntil` timeouts on iOS even when the screen loads quickly on Android.  
**Root cause:** `disableAnimations: true` in `config.yaml` works reliably on Android. On iOS simulators, it may not fully suppress all spring/transition animations in Flutter.

**Fix:** Increase `extendedWaitUntil` timeouts for iOS-sensitive transitions, and/or check that `config.yaml` has:

```yaml
platform:
  ios:
    disableAnimations: true
  android:
    disableAnimations: true
```

If animations are still causing flakiness, add `waitForAnimationToEnd` (max 600ms) before the affected tap.
