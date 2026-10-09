# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project

A beginner's week-1 project: static web pages in plain HTML/CSS/JS, **no external libraries, no build step, no server, no package.json**. Each page is a single self-contained file (inline `<style>` and `<script>`), opened directly via `file://` in Chrome. Keep it that way. Don't add CDNs, `<link>`/`<script src>`/`<img>` to remote hosts, or tooling.

- `index.html`: "하루 플래너", a daily planner modeled on the Mo:tmo:t (모트모트) paper planner. Has a date nav with a popup calendar and hover tooltips, a Mon–Sun week strip, a D-DAY box, a daily memo ("COMMENT"), a to-do list, and a 10-minute-grid timetable painted by dragging.
- `about.html`: self-introduction page (hero, 3 interest cards, contacts). Name and contacts are placeholders marked with `✏️` comments. The email appears in three places (visible text, `mailto:`, `data-copy`).
- `index.backup-*.html`: frozen backups. Don't edit them.

The user is Korean and not a programmer. Reply in Korean, and explain in plain language. UI text and code comments are in Korean. Match that.

## Rules

- **Do only what the user asked.** Don't use your own judgment to add work outside the requested scope: no extra features, refactors, redesigns, file changes, or "while I'm here" fixes. If you notice something else worth doing, mention it briefly in the reply and wait for the user to ask.

## Running

```powershell
Start-Process chrome "C:\Users\anjfh\todo-app\index.html"
```

There is no test suite or linter. Verification has been done ad hoc with Node (v24, built-in `WebSocket`/`fetch`) driving headless Chrome over the DevTools Protocol:

- Launch `chrome.exe --headless=new --remote-debugging-port=<port> --user-data-dir=<fresh temp profile>`.
- Send real `Input.dispatchMouseEvent` / `Input.dispatchKeyEvent` input, then assert with `Runtime.evaluate`.
- Take screenshots with `Page.captureScreenshot`.

Use a fresh profile each run so localStorage starts empty and sample seeding runs. The user declined installing the Claude-in-Chrome extension, so don't suggest it.

## Architecture (index.html)

- **State and persistence:** module-level arrays and objects are loaded once from localStorage through `read(key, fallback)` / `write(key, value)`. Both wrap JSON and swallow storage errors. Mutate the state, call `saveTodos()`/`saveDays()`, then `render()`.

  | Key | Contents |
  |---|---|
  | `today-todos` | `[{id, date, text, done, color}]` |
  | `planner-days` | `{ 'YYYY-MM-DD': { memo, time: { slotIndex: colorIdx } } }` |
  | `planner-dday` | `{title, date}` |
  | `planner-pen` | color index, `-1` = eraser |
  | `today-todos-seeded`, `planner-seeded` | one-time sample-data flags |

- **Rendering:** `render()` rebuilds every section from state (`renderDate/Week/Todos/Memo/Timetable/Dday`, plus the calendar when it is open). There is no diffing. Insert user text only via `textContent`.
- **Dates:** local-time `'YYYY-MM-DD'` string keys throughout (`toKey`/`fromKey`/`shiftKey`). Don't use `toISOString()`, which shifts to UTC.
- **Timetable:** 24 rows × 6 cells, starting at `START_HOUR = 6` and running to 5 AM the next day. `slotIndex = (hour - 6) * 6 + tenMinuteColumn`. Total time is (painted cells × 10 min). Mouse drags paint. Touch paints one cell per tap, so the page still scrolls.
- **Samples:** `seedSamples()` inserts about 10 to-dos relative to today, plus a timetable, memo and D-DAY. Each flag makes this run once. Legacy to-dos without `date` are migrated to today on load.
- **Shortcuts:** ←/→ change day, `T` jumps to today, `D` toggles the theme, `Esc` closes popups. All are ignored while typing in an input.

## Theme (shared by both pages)

- Colors are CSS custom properties. The light palette lives on `:root`. The dark palette is defined twice:
  - under `@media (prefers-color-scheme: dark) { :root:not([data-theme="light"]) }`
  - under `:root[data-theme="dark"]`

  Add every new color token to all three places.
- An inline `<script>` at the very top of `<head>` applies the saved theme before first paint, to avoid a flash of the wrong theme.
- The choice is stored in the `planner-theme` key as a **raw string** `'light'`/`'dark'`, via `localStorage.setItem` directly. Don't use `write()`: it would JSON-quote the value and break the head script's comparison.
- Both pages share this key because they have the same `file://` origin, so toggling on one page carries over to the other. With no saved choice, the pages follow the OS setting.
- The toggle is a fixed top-right `role="switch"` button with `aria-checked`. The top-left corner holds the cross-link to the other page.
