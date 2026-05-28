# Preview Copy Button Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add a secondary copy button next to the preview title so users can copy the currently displayed `auth.json` again without re-fetching data.

**Architecture:** Keep the existing primary flow unchanged: the main button still fetches and auto-copies. Extend the popup UI with a secondary button inside the preview header area, and extend the popup script to toggle that button with preview visibility and reuse the existing clipboard helper to copy the current preview text.

**Tech Stack:** Chrome Extension (Manifest V3), plain HTML, inline CSS, vanilla JavaScript

---

## File Structure

- Modify: `popup.html`
  - Add a preview header layout that keeps the existing preview label and places a secondary copy button after it.
  - Add minimal CSS for the preview header row and the secondary button state.
- Modify: `popup.js`
  - Query the new button element.
  - Hide/show it alongside the preview.
  - Reuse `copyToClipboard()` for copying the current preview text.
  - Update status messaging for the secondary copy action.
- Verify manually in Chrome/Edge extension popup
  - No automated test harness exists in this repo.

### Task 1: Add preview copy button markup and styles

**Files:**
- Modify: `popup.html:42-47`
- Modify: `popup.html:81-103`
- Modify: `popup.html:144-147`

- [ ] **Step 1: Update the preview header markup**

Replace the current preview block:

```html
    <details id="preview" hidden>
      <summary>预览 auth.json (含敏感信息)</summary>
      <pre id="previewContent"></pre>
    </details>
```

With this markup:

```html
    <details id="preview" hidden>
      <summary>
        <span>预览 auth.json (含敏感信息)</span>
        <button id="copyPreviewButton" type="button" hidden>复制</button>
      </summary>
      <pre id="previewContent"></pre>
    </details>
```

- [ ] **Step 2: Add layout styles for the preview header**

Insert these CSS rules near the existing `details` and `details summary` styles:

```css
    details summary {
      display: flex;
      align-items: center;
      justify-content: space-between;
      gap: 8px;
      cursor: pointer;
      user-select: none;
      color: #5f6368;
    }

    #copyPreviewButton {
      flex: 0 0 auto;
      padding: 4px 10px;
      border: 1px solid #dedee3;
      border-radius: 6px;
      background: #ffffff;
      color: #202123;
      font-size: 12px;
      font-weight: 600;
      cursor: pointer;
    }
```

- [ ] **Step 3: Add dark mode styling for the secondary button**

Insert this rule inside the existing `@media (prefers-color-scheme: dark)` block:

```css
      #copyPreviewButton {
        background: #2f3037;
        color: #f7f7f8;
        border-color: #565869;
      }
```

- [ ] **Step 4: Visually inspect the HTML changes**

Read `popup.html` and confirm:
- the preview title still says `预览 auth.json (含敏感信息)`
- the new button id is exactly `copyPreviewButton`
- the button starts with the `hidden` attribute
- the summary styles still preserve the existing preview section layout

Expected result: the preview header now supports a title on the left and a hidden copy button on the right.

### Task 2: Wire the preview copy button behavior

**Files:**
- Modify: `popup.js:4-8`
- Modify: `popup.js:15-18`
- Modify: `popup.js:53-82`

- [ ] **Step 1: Query the new button element**

Update the DOM references at the top of `popup.js` to include the new button:

```javascript
  const grabButton = document.getElementById('grabButton');
  const copyPreviewButton = document.getElementById('copyPreviewButton');
  const status = document.getElementById('status');
  const preview = document.getElementById('preview');
  const previewContent = document.getElementById('previewContent');
  const builder = window.CodexCredentialBuilder;
```

- [ ] **Step 2: Add a helper to keep preview UI state in sync**

Insert this function after `setBusy()`:

```javascript
  function setPreviewVisible(isVisible) {
    preview.hidden = !isVisible;
    copyPreviewButton.hidden = !isVisible;
  }
```

- [ ] **Step 3: Hide the preview copy button while a new fetch is starting**

At the start of `grabAndCopyAuth()`, replace:

```javascript
    preview.hidden = true;
```

With:

```javascript
    setPreviewVisible(false);
```

- [ ] **Step 4: Show the preview copy button after a successful fetch**

Inside the `try` block of `grabAndCopyAuth()`, replace:

```javascript
      preview.hidden = false;
```

With:

```javascript
      setPreviewVisible(true);
```

- [ ] **Step 5: Add a handler that copies the current preview only**

Insert this function above the final event listener registration:

```javascript
  async function copyPreviewAuth() {
    try {
      await copyToClipboard(previewContent.textContent);
      setStatus('已复制当前预览内容。', 'ok');
    } catch (error) {
      const message = error instanceof Error ? error.message : String(error);
      setStatus(message, 'error');
    }
  }
```

- [ ] **Step 6: Bind the new button click event**

Replace the single listener at the bottom:

```javascript
  grabButton.addEventListener('click', grabAndCopyAuth);
```

With:

```javascript
  grabButton.addEventListener('click', grabAndCopyAuth);
  copyPreviewButton.addEventListener('click', copyPreviewAuth);
```

- [ ] **Step 7: Verify the behavior by reading the updated script**

Read `popup.js` and confirm:
- `copyPreviewButton` is queried once
- `setPreviewVisible()` is used for both hiding and showing preview state
- `copyPreviewAuth()` uses `previewContent.textContent`
- `copyPreviewAuth()` does not call `builder.buildFromBrowser()`

Expected result: the new button only copies current preview content and never re-fetches data.

### Task 3: Manually verify the popup behavior in the browser

**Files:**
- Modify: none
- Verify: local unpacked extension loaded from `/Users/liuzan/Projects/Personal/github/codex-credentials-extension`

- [ ] **Step 1: Reload the unpacked extension**

In Chrome or Edge extensions page, reload the unpacked extension for this repo.

Expected result: the popup reflects the local `popup.html` and `popup.js` changes.

- [ ] **Step 2: Verify initial state**

Open the extension popup before clicking anything.

Expected result:
- the main button `抓取并复制 auth.json` is visible
- the preview section is hidden
- the preview copy button is not visible

- [ ] **Step 3: Verify successful fetch state**

Click the main button once while logged into `https://chatgpt.com`.

Expected result:
- the main button enters the busy state and then returns
- the preview section appears
- the preview copy button appears next to `预览 auth.json (含敏感信息)`
- the status area still shows the existing success details from the fetch flow

- [ ] **Step 4: Verify secondary copy behavior**

Click the new `复制` button in the preview header.

Expected result:
- no loading state appears on the main button
- no new network fetch is triggered by the popup script
- the current preview text is copied again
- the status area changes to `已复制当前预览内容。`

- [ ] **Step 5: Verify repeated use**

Click the main button again, then click the preview `复制` button again.

Expected result:
- the preview still updates to the latest generated content
- the preview copy button remains available after success
- the secondary copy action always copies the latest preview content

### Task 4: Review changes and prepare commit

**Files:**
- Modify: `popup.html`
- Modify: `popup.js`

- [ ] **Step 1: Review the diff**

Run:

```bash
git diff -- popup.html popup.js
```

Expected result: diff only shows the preview header/button UI changes and the minimal script changes needed to support them.

- [ ] **Step 2: Verify git status**

Run:

```bash
git status --short
```

Expected result: only intended files are modified for this feature.

- [ ] **Step 3: Commit the change**

Run:

```bash
git add popup.html popup.js
git commit -m "feat: add preview copy button"
```

Expected result: a new commit is created containing only the popup UI and popup script changes.

## Self-Review

- Spec coverage: covered UI placement, conditional visibility, no re-fetch behavior, status feedback, and manual verification.
- Placeholder scan: no TODO/TBD placeholders remain.
- Type consistency: `copyPreviewButton`, `setPreviewVisible()`, and `copyPreviewAuth()` are named consistently across all tasks.
