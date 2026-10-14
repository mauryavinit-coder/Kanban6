# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

A single-file IT PMO Kanban board (`index.html`) for a **fictitious** bank's internal IT PMO — a demo/training tool. All markup, CSS (`<style>`) and JS (`<script>`) live in that one file.

## Hard constraints (from the original spec — do not break)

- Vanilla HTML/CSS/JS only. No frameworks, libraries, build step, bundler or npm.
- Must stay a single `index.html` that runs by double-clicking it.
- **No persistence**: no localStorage, sessionStorage, IndexedDB or cookies. State is in memory and resets to seed data on refresh. The header note says so; that is intended behaviour.
- No external resources: no CDN scripts, web fonts or image files. Use the system font stack and Unicode/inline SVG icons.
- The only network call is the FormSubmit AJAX POST in `notifyNewTask()`. Don't send the user's email anywhere else.
- Branding: neutral "IT PMO" text wordmark and blue palette only. No real bank logos, trademarks or imitation of official systems. (The spec does require the `UOB-ITPM-####` ID format and the `[UOB IT PMO]` email subject.)
- No `alert()`/`confirm()` (delete uses an inline Yes/No toggle) and no `!important`. Every user-supplied string goes through `escapeHtml()` before it reaches `innerHTML`.

## Running / testing

There is no build, lint or test tooling. Open `index.html` in a browser.

- The board works from `file://`, but **FormSubmit rejects requests from pages opened as files** (it returns HTTP 200 with `success: "false"`). To test real email delivery, serve the folder, e.g. `python -m http.server 8000`.
- The endpoint is a placeholder (`YOUR_EMAIL@example.com`) until it is configured. FormSubmit also needs a one-time activation: the first submission sends a confirmation email to the address, and nothing is delivered until its link is clicked.
- Verification so far has been manual, in a browser: dispatch clicks/`DragEvent`s from the console and inspect `state`.

## Architecture (inside the `<script>`)

- **Config at top:** `FORMSUBMIT_ENDPOINT`, `ID_PREFIX`, `NOTIFY_TIMEOUT_MS`.
- **Single source of truth:** `state = { tasks, filters }`. Transient UI flags (`pendingDeleteId`, `openMenuId`, `sending`, `lastFocus`) are in a separate `ui` object. `nextId` drives `generateId()`.
- **Render-from-state:** all card DOM is produced by `renderBoard()` → `renderCard()` (HTML strings). Mutators `addTask()`, `moveTask()` and `deleteTask()` change `state` and then call `renderBoard()`. Don't mutate card contents anywhere else; the only direct DOM tweaks outside it are drag-state classes (`.dragging`, `.drag-over`). Summary counts use all tasks; column counts use the filtered set from `applyFilters()`.
- **Event delegation:** a single set of listeners on `#board` handles card buttons (via `data-action`: `delete`, `confirm-delete`, `cancel-delete`, `toggle-move`, `move-to`), Escape and native HTML5 drag-and-drop. Because re-rendering replaces card elements, handlers restore focus afterwards with `focusInCard(id, selector)`.
- **Add Task flow:** `handleTaskSubmit()` → `validateForm()` (inline errors via `showFieldError`) → optimistic `addTask()` → `setSending(true)` → `notifyNewTask()` in try/catch (failure shows the warning toast "Card added locally — email notification failed") → `finally` closes the modal.
- **Status columns** are fixed in the HTML (`data-status`). Element IDs come from `statusSlug()` (`list-in-progress`, `count-in-progress`, `sum-in-progress`, `col-title-in-progress`). Adding a status means updating `STATUSES`, the column markup, the summary `<li>` and the `--col-*` CSS token.
- **Categories & progress:** each category has a colour token (`--cat-*`) and a class from `CATEGORY_CLASS` (`.cat-appdev` etc. set `--cat-color`), used by the card tag and by the "Progress by category" cylinders. `renderProgress()` (called from `renderBoard()`) builds the cylinder markup once, then only updates `--pct` and text so the fill animates; it counts all tasks (Done ÷ total per category), ignoring filters. Adding a category means updating `CATEGORIES`, `CATEGORY_CLASS`, the `--cat-*` token and its `.cat-*` rule.
- **CSS:** dark theme (`color-scheme: dark`); tokens on `:root` (palette, priority colours, `--sp-1..6` spacing scale). Priority border/pill colours come from the `.prio-*` classes setting `--prio-color` and `--prio-bg`. Below 768px the layout drops to one column.
