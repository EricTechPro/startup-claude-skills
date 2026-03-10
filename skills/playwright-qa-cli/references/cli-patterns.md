# Playwright CLI Best Practices & Patterns

Detailed patterns for robust `playwright-cli` usage. Referenced from SKILL.md Phase 4.

## Table of Contents

1. [Waiting Strategies](#waiting-strategies)
2. [Snapshot-First Interaction](#snapshot-first-interaction)
3. [Multi-Step User Flows](#multi-step-user-flows)
4. [Form Interactions](#form-interactions)
5. [File Uploads](#file-uploads)
6. [Dialogs & Modals](#dialogs--modals)
7. [Scrolling & Lazy Content](#scrolling--lazy-content)
8. [Error Detection](#error-detection)
9. [Auth State Management](#auth-state-management)
10. [Command Chaining](#command-chaining)

---

## Waiting Strategies

The biggest source of flaky QA is waiting too little (missing content) or too much (slow runs). Use the right strategy for each situation.

### When to use `eval` waits vs timed waits

| Situation | Strategy | Why |
|-----------|----------|-----|
| After navigation (`goto`, `click` that changes URL) | `eval` — wait for a specific DOM element | Navigation triggers full page load; waiting for a known element confirms the new page is ready |
| After login/auth redirect | Timed wait (3s) | Auth redirects involve server round-trips + client-side redirect chains with no single reliable DOM signal |
| After clicking a button that triggers API call | `eval` — wait for loading state to clear | The response time varies; a DOM signal (spinner gone, data appeared) is more reliable than guessing |
| After form submission | `eval` — wait for success indicator or error | Server response time varies |
| After file upload | Timed wait (2-5s depending on file size) + `eval` check | Upload progress is hard to observe from DOM; follow with a DOM check |

### Wait for element to appear

```bash
# Wait for an element with specific text to appear (max 10s)
playwright-cli eval "() => new Promise((resolve, reject) => {
  const timeout = setTimeout(() => reject('Timeout'), 10000);
  const check = () => {
    if (document.querySelector('[data-testid=\"receipt-list\"]')) {
      clearTimeout(timeout); resolve('found');
    } else setTimeout(check, 200);
  }; check();
})"
```

### Wait for loading to finish

```bash
# Wait for no elements with aria-busy or loading spinners
playwright-cli eval "() => new Promise((resolve, reject) => {
  const timeout = setTimeout(() => reject('Timeout'), 10000);
  const check = () => {
    const busy = document.querySelector('[aria-busy=\"true\"]');
    const spinner = document.querySelector('.animate-spin');
    if (!busy && !spinner) { clearTimeout(timeout); resolve('ready'); }
    else setTimeout(check, 300);
  }; check();
})"
```

### Simple timed wait (when appropriate)

```bash
# Use eval-based setTimeout — cleaner than shell sleep
playwright-cli eval "() => new Promise(r => setTimeout(r, 2000))"
```

This is preferable to shell `sleep` because it keeps the timing within the browser context and pairs naturally with other `eval` commands.

---

## Snapshot-First Interaction

Element refs (`e93`, `e104`, etc.) are assigned per-snapshot and change whenever the DOM changes significantly. This is the most common source of "element not found" errors.

### Rules

1. **Always snapshot before interacting with a new page** — after `goto`, after login redirect, after any navigation
2. **Re-snapshot after significant DOM changes** — dialog open/close, tab switch, form section expand, data load
3. **Don't re-snapshot for same-page interactions** — filling sequential form fields on the same page, clicking buttons that don't change the DOM structure
4. **Read the snapshot output carefully** — look for the element's role, name, and ref. Match by semantic meaning (e.g., `button "Save"` or `textbox "Email"`) not just ref number

### Pattern: Navigate + Snapshot + Act

```bash
playwright-cli goto "{url}"
playwright-cli eval "() => new Promise(r => setTimeout(r, 2000))"
playwright-cli snapshot                    # ← get fresh refs
playwright-cli screenshot --filename=...   # ← visual record
# Now read snapshot output to find refs
playwright-cli click {ref}                 # ← use refs from THIS snapshot
```

### Pattern: After dialog opens

```bash
playwright-cli click {trigger-button-ref}
playwright-cli eval "() => new Promise(r => setTimeout(r, 500))"
playwright-cli snapshot                    # ← dialog content has new refs
# Read snapshot to find dialog elements
playwright-cli fill {dialog-input-ref} "value"
```

---

## Multi-Step User Flows

For flows that span multiple pages or involve several interactions (e.g., "upload item → fill details → submit → verify in list"):

### Structure

```
1. Navigate to starting page
2. Snapshot + screenshot (baseline)
3. For each step:
   a. Perform interaction
   b. Wait for result (eval or timed)
   c. Snapshot (if DOM changed significantly)
   d. Screenshot (labeled by step)
   e. Verify expected state
4. Final snapshot + screenshot (end state)
```

### Example: Upload and verify

```bash
# Step 1: Navigate to target page
playwright-cli goto "{base_url}/dashboard/items"
playwright-cli eval "() => new Promise(r => setTimeout(r, 2000))"
playwright-cli snapshot
playwright-cli screenshot --filename=.../step-01-items-page.png

# Step 2: Click upload button (ref from snapshot)
playwright-cli click {upload-button-ref}
playwright-cli eval "() => new Promise(r => setTimeout(r, 500))"
playwright-cli snapshot  # dialog opened — new refs

# Step 3: Upload file
playwright-cli upload "/absolute/path/to/test-file.png"
playwright-cli eval "() => new Promise(r => setTimeout(r, 3000))"  # processing time
playwright-cli snapshot
playwright-cli screenshot --filename=.../step-02-after-upload.png

# Step 4: Verify result
playwright-cli console error  # check for JS errors
```

---

## Form Interactions

### Fill vs Type

| Command | Use When |
|---------|----------|
| `fill {ref} "text"` | Standard form inputs — clears existing value first, then sets new value. Works with `<input>`, `<textarea>`, `[contenteditable]` |
| `type "text"` | Need to simulate keystroke-by-keystroke typing into the currently focused element. Useful for autocomplete/search fields that respond to each keypress |

### Form flow pattern

```bash
# 1. Snapshot to get field refs
playwright-cli snapshot

# 2. Fill fields (order doesn't matter for independent fields)
playwright-cli fill {name-ref} "Test Business"
playwright-cli fill {email-ref} "test@example.com"
playwright-cli select {country-ref} "US"
playwright-cli check {terms-ref}

# 3. Submit
playwright-cli click {submit-ref}

# 4. Wait for response
playwright-cli eval "() => new Promise(r => setTimeout(r, 2000))"

# 5. Re-snapshot to check result (success message, validation errors, redirect)
playwright-cli snapshot
playwright-cli screenshot --filename=.../form-submitted.png
```

### Handling validation errors

After submit, check the snapshot for error indicators:
- Elements with `role="alert"` or text containing "error", "required", "invalid"
- Red-colored elements (check class names in snapshot)
- Form fields that gained error styling

---

## File Uploads

`playwright-cli upload` requires an **absolute path**. It works with the browser's file chooser that appears after clicking an upload button or drag area.

### Pattern

```bash
# 1. Click the upload trigger (button or dropzone)
playwright-cli click {upload-ref}

# 2. Upload the file (absolute path required)
playwright-cli upload "/absolute/path/to/test-file.png"

# 3. Wait for processing (uploads can be slow)
playwright-cli eval "() => new Promise(r => setTimeout(r, 3000))"

# 4. Snapshot to verify upload result
playwright-cli snapshot
playwright-cli screenshot --filename=.../after-upload.png
```

For multiple files, pass multiple paths:
```bash
playwright-cli upload "/path/to/file1.png" "/path/to/file2.pdf"
```

---

## Dialogs & Modals

### Browser-native dialogs (alert, confirm, prompt)

These are JavaScript `window.alert()`, `window.confirm()`, `window.prompt()` dialogs — NOT custom UI modals.

```bash
# Accept (OK / Yes)
playwright-cli dialog-accept

# Accept prompt with text
playwright-cli dialog-accept "my response text"

# Dismiss (Cancel / No)
playwright-cli dialog-dismiss
```

### Custom UI modals (shadcn Dialog, Sheet, etc.)

These are regular DOM elements. Interact via snapshot + click/fill:

```bash
# 1. Trigger modal open
playwright-cli click {trigger-ref}
playwright-cli eval "() => new Promise(r => setTimeout(r, 500))"

# 2. Snapshot to get modal content refs
playwright-cli snapshot

# 3. Interact with modal content
playwright-cli fill {modal-input-ref} "value"
playwright-cli click {modal-submit-ref}

# 4. Wait for modal to close / action to complete
playwright-cli eval "() => new Promise(r => setTimeout(r, 1000))"
playwright-cli snapshot
```

### Closing modals

- Click the X button (find in snapshot)
- Press Escape: `playwright-cli press Escape`
- Click overlay (if the modal supports it)

---

## Scrolling & Lazy Content

### Scroll down

```bash
# Scroll down 500px
playwright-cli mousewheel 0 500

# Scroll down a full viewport
playwright-cli mousewheel 0 900

# Scroll up
playwright-cli mousewheel 0 -500
```

### Loading more content

For infinite scroll or "load more" patterns:

```bash
# 1. Scroll to bottom
playwright-cli mousewheel 0 2000
playwright-cli eval "() => new Promise(r => setTimeout(r, 1000))"

# 2. Snapshot to see newly loaded content
playwright-cli snapshot
playwright-cli screenshot --full-page --filename=.../scrolled-content.png
```

### Full-page screenshot

Captures the entire scrollable page, not just the viewport:

```bash
playwright-cli screenshot --full-page --filename=.../full-page.png
```

---

## Error Detection

### JavaScript console errors

```bash
# Show only errors (most useful for QA)
playwright-cli console error

# Show warnings too
playwright-cli console warn
```

Run this after each major interaction and at the end of testing. Console errors often reveal issues invisible in the UI.

### Network request failures

```bash
# Show all network requests (excludes static resources by default)
playwright-cli network

# Include static resources too
playwright-cli network --static
```

Look for failed requests (4xx/5xx status codes) in the output.

### Visual error indicators

In the snapshot, look for:
- `role="alert"` elements — error banners, toast notifications
- Text containing "error", "failed", "something went wrong"
- Empty states where data should be present
- Loading spinners that never resolve

---

## Auth State Management

Logging in takes 3-5 seconds every time. For multi-page testing, save and restore auth state to avoid repeated logins.

### Save after login

```bash
playwright-cli state-save playwright-qa-screenshots/{test-name}/auth-state.json
```

### Restore on session loss

```bash
playwright-cli state-load playwright-qa-screenshots/{test-name}/auth-state.json
playwright-cli reload  # refresh to apply restored state
```

### When to use

- Testing multiple unrelated pages in one session
- Session expired mid-test (rare but happens)
- Resuming a test after browser restart

---

## Command Chaining

### Independent commands — run separately

Each `playwright-cli` command is a standalone bash invocation. Don't chain with `&&` because each command needs its own PATH export and error handling.

```bash
# Good — separate commands
export PATH="$HOME/.npm-global/bin:$PATH" && playwright-cli snapshot
export PATH="$HOME/.npm-global/bin:$PATH" && playwright-cli screenshot --filename=...

# Bad — chained (if first fails, second is skipped)
export PATH="$HOME/.npm-global/bin:$PATH" && playwright-cli snapshot && playwright-cli screenshot --filename=...
```

### Rapid sequences — use eval for delays

When you need a pause between actions (e.g., fill then submit), use `eval` rather than shell `sleep`:

```bash
export PATH="$HOME/.npm-global/bin:$PATH" && playwright-cli fill {ref} "text"
export PATH="$HOME/.npm-global/bin:$PATH" && playwright-cli eval "() => new Promise(r => setTimeout(r, 300))"
export PATH="$HOME/.npm-global/bin:$PATH" && playwright-cli click {submit-ref}
```

### Running Playwright code directly

For complex interactions that are awkward as individual CLI commands, use `run-code`:

```bash
playwright-cli run-code "async (page) => {
  await page.waitForSelector('[data-testid=\"item-list\"]', { timeout: 10000 });
  const count = await page.locator('[data-testid=\"item-row\"]').count();
  return 'Found ' + count + ' items';
}"
```

This is a powerful escape hatch — use it when `eval` plus basic commands aren't enough, like waiting for specific selectors or doing complex assertions.
