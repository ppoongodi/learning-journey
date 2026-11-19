# Playwright Auto-Wait Internals

Playwright Auto-Wait is an in-browser execution mechanism that synchronizes test scripts with the live DOM by continuously verifying element readiness before performing actions.

Unlike Selenium-style polling that repeatedly communicates with the browser, Playwright injects an execution engine directly into the browser context and performs readiness checks on every animation frame.

---

# 1. Internal Architecture & Execution Flow

Playwright operates across two distinct layers:

- **Test Runner Layer** (Node.js / Python / Java / .NET)
- **Browser Engine Layer** (Chromium / Firefox / WebKit)

These layers communicate through a persistent WebSocket connection.

```text
[Test Script: Node.js]                 [Browser Process: Injected Script]
         │                                             │
         ├── 1. locator.click() ──────────────────────►│ (Single WebSocket Command)
         │                                             │
         │                                    ┌────────▼────────┐
         │                                    │  2. Resolve DOM │
         │                                    │ (Strict Mode)   │
         │                                    └────────┬────────┘
         │                                             │
         │                                    ┌────────▼────────┐
         │                                    │ 3. Actionability│
         │                                    │    Checks       │
         │                                    │  • Attached     │
         │                                    │  • Visible      │
         │                                    │  • Stable       │
         │                                    │  • Receives Evt │
         │                                    │  • Enabled      │
         │                                    └────────┬────────┘
         │                                             │
         │                                    ┌────────▼────────┐
         │                                    │ 4. Dispatch CDP │
         │                                    │ Native Event    │
         │                                    └────────┬────────┘
         │                                             │
         ◄──────────────────── 5. Action Completed ────┘
```

## Key Advantages

### Zero Network Overhead

Unlike Selenium, which often sends multiple requests to:

- Find element
- Check visibility
- Check clickability
- Perform action

Playwright sends **one command**, while all actionability checks execute directly inside the browser process.

### Frame-Synced Polling

Playwright uses:

```javascript
requestAnimationFrame()
```

to inspect page state before the browser paints the next frame.

This ensures:

- Accurate layout calculations
- Reliable visibility detection
- Stable action execution
- Reduced flaky tests

---

# 2. Deep Dive into Actionability Checks

Before executing an action, Playwright verifies a set of conditions.

The action proceeds only when **all required checks pass**.

---

## 2.1 Attached

Ensures the element is still connected to the DOM.

```javascript
element.isConnected === true
```

Checks:

- Element exists
- Element is attached to Document
- Element is attached to Shadow DOM (if applicable)

---

## 2.2 Visible

Ensures the element is actually visible on screen.

Playwright evaluates:

```javascript
const rect = element.getBoundingClientRect();
```

Requirements:

```javascript
rect.width > 0
rect.height > 0
```

And:

```javascript
getComputedStyle(element).visibility !== 'hidden'
```

---

## 2.3 Stable

Prevents clicks during animations and transitions.

Playwright captures the element's bounding box across consecutive animation frames.

Example:

```javascript
Frame 1:
x=100, y=50

Frame 2:
x=120, y=50
```

Since the position changed, the element is considered unstable.

Checks continue until:

```javascript
x, y, width, height
```

remain unchanged across frames.

---

## 2.4 Receives Events (Hit Target Validation)

Ensures no overlay blocks the target element.

Playwright calculates the center point:

```javascript
const centerX = ...
const centerY = ...
```

Then verifies:

```javascript
document.elementFromPoint(centerX, centerY)
```

returns the target element.

Common blockers:

- Loading spinners
- Modals
- Invisible overlays
- Sticky banners
- Tooltips

---

## 2.5 Enabled

Checks:

```html
<button disabled>
```

and

```html
aria-disabled="true"
```

Playwright also evaluates disabled parent fieldsets.

Action continues only when the control is enabled.

---

## 2.6 Editable

Specific to input-based operations such as:

```javascript
fill()
type()
clear()
```

Requirements:

- Enabled
- Not readonly

Example:

```html
<input readonly />
```

will cause the editable check to fail.

---

# 3. Actionability Matrix

Different Playwright methods require different actionability checks.

| Action | Attached | Visible | Stable | Receives Events | Enabled | Editable |
|----------|----------|----------|----------|----------|----------|----------|
| `locator.click()` | ✅ | ✅ | ✅ | ✅ | ✅ | ❌ |
| `locator.dblclick()` | ✅ | ✅ | ✅ | ✅ | ✅ | ❌ |
| `locator.fill()` | ✅ | ✅ | ❌ | ❌ | ✅ | ✅ |
| `locator.check()` | ✅ | ✅ | ✅ | ✅ | ✅ | ❌ |
| `locator.uncheck()` | ✅ | ✅ | ✅ | ✅ | ✅ | ❌ |
| `locator.hover()` | ✅ | ✅ | ✅ | ✅ | ❌ | ❌ |
| `locator.selectOption()` | ✅ | ✅ | ❌ | ❌ | ✅ | ❌ |
| `locator.focus()` | ✅ | ❌ | ❌ | ❌ | ❌ | ❌ |

---

# 4. Essential Commands & Patterns

## A. Standard Auto-Waiting Actions

### Click

Automatically waits for:

- Attached
- Visible
- Stable
- Receives Events
- Enabled

```javascript
await page
  .getByRole('button', { name: 'Submit' })
  .click();
```

### Fill

Automatically waits for:

- Attached
- Visible
- Enabled
- Editable

```javascript
await page
  .getByLabel('Username')
  .fill('admin_user');
```

---

## B. Web-First Assertions (Auto-Retry)

Traditional assertions execute only once.

```javascript
expect(value).toBe(true);
```

Web-first assertions continuously retry until:

- Condition passes
- Timeout expires

### Verify Text

```javascript
await expect(
  page.locator('.toast-message')
).toHaveText('Order Confirmed');
```

### Verify Hidden Element

```javascript
await expect(
  page.locator('.loading-spinner')
).toBeHidden();
```

Default assertion timeout:

```text
5000 ms
```

---

## C. Explicit Waiting

Useful when you need synchronization without performing an immediate action.

```javascript
await page
  .locator('#dynamic-modal')
  .waitFor({
      state: 'visible',
      timeout: 10000
  });
```

Supported states:

```text
attached
detached
visible
hidden
```

---

## D. Debugging and Overrides

### Trial Mode

Executes all checks without performing the action.

Useful for troubleshooting.

```javascript
await page
  .locator('#submit')
  .click({ trial: true });
```

---

### Force Click

Bypasses most actionability checks.

Use only when absolutely necessary.

```javascript
await page
  .locator('#custom-checkbox')
  .click({ force: true });
```

Typical use cases:

- Hidden controls
- Custom UI libraries
- Complex drag and drop widgets

---

### Custom Timeout

Override action timeout for a specific operation.

```javascript
await page
  .locator('#slow-report-btn')
  .click({
      timeout: 60000
  });
```

---

# 5. Global Timeout Configuration

Example `playwright.config.ts`

```typescript
import { defineConfig } from '@playwright/test';

export default defineConfig({
  // Maximum time for a single test
  timeout: 30 * 1000,

  expect: {
    // Maximum time for web-first assertions
    timeout: 5 * 1000,
  },

  use: {
    // Timeout for actions such as click, fill, check
    actionTimeout: 10 * 1000,

    // Timeout for navigation events
    navigationTimeout: 15 * 1000,
  },
});
```

---

# 6. What Auto-Wait Does NOT Cover

Playwright Auto-Wait is powerful, but it is not magic.

Understanding its limitations helps avoid flaky tests.

---

## API / Network Responses

Playwright does **not automatically wait** for background API calls triggered by a UI action.

Example:

```javascript
await page.getByRole('button', { name: 'Save' }).click();
```

If clicking the button triggers an API request, Playwright only waits for the click action itself.

Use:

```javascript
await page.waitForResponse(
  response =>
    response.url().includes('/orders') &&
    response.status() === 200
);
```

or follow with a web-first assertion:

```javascript
await expect(
  page.getByText('Order Saved')
).toBeVisible();
```

---

## Non-Retrying Methods

The following methods execute immediately against the current DOM snapshot:

```javascript
locator.count()
locator.allInnerTexts()
locator.allTextContents()
page.evaluate()
locator.evaluate()
```

Example:

```javascript
const count = await page.locator('.row').count();
```

If rows are still loading, Playwright will not wait automatically.

Prefer:

```javascript
await expect(
  page.locator('.row')
).toHaveCount(10);
```

---

# Best Practices

✅ Prefer `Locator` APIs over `page.$()` or `page.waitForSelector()`.

✅ Prefer web-first assertions (`toHaveText`, `toBeVisible`, `toHaveCount`).

✅ Let Playwright Auto-Wait handle synchronization whenever possible.

✅ Use `trial: true` when troubleshooting actionability failures.

✅ Use `force: true` only as a last resort.

✅ Wait for user-visible outcomes rather than arbitrary timeouts.

❌ Avoid `waitForTimeout()` for synchronization.

❌ Avoid hard-coded sleeps.

❌ Avoid forcing actions unless you understand why normal actionability checks are failing.

---

# Summary

Playwright's Auto-Wait mechanism works by executing element readiness checks directly inside the browser process on every animation frame. Before performing an action, it verifies element actionability through checks such as visibility, stability, event receivability, enabled state, and editability.

This architecture eliminates most synchronization issues, reduces flaky tests, and allows Playwright tests to operate closer to real user interactions than traditional polling-based automation frameworks.
