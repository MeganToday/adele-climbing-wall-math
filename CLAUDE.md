# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project

Single-file browser game (`index.html`) for a kindergartner named Adele. No build step, no dependencies, no server required — open directly in any browser. Primary device is iPad (touch-first).

## Running the game

```bash
open index.html
```

Or drag `index.html` into a browser. There is no build, lint, or test tooling.

## Architecture

Everything lives in one file: HTML structure → `<style>` block → `<script>` block. The script is organized into clearly delimited sections (marked with `═══` banners):

1. **`STUFFY_DEFS`** — Data array defining all stuffies. `unlockLevel: 0` = available from start; `unlockLevel: N` = earned by completing level N. To swap in a real photo: set `img: 'filename.png'` (place file next to `index.html`) and `icon: ''`.

2. **`LEVELS`** — Data array defining each level. Each entry has `bg`, `rockColor`/`rockHighlight`/`rockGray` for theming, and a `generate()` function that returns `{ display, answer }`. Adding a new level = adding one object here.

3. **`state`** — Persistent session state (unlocked stuffies, names, star ratings, selected stuffy). Not saved between browser sessions.

4. **`game`** — Runtime state reset on each `startLevel()` call (wall, solve count, miss count, active rock).

5. **Wall generation** (`buildWall`, `WALL_ROW_SIZES`) — Creates a DAG of 30 rocks across 7 rows `[7,6,5,5,4,2,1]`. Rows are indexed bottom=0, top=6. Each rock stores `parentsUp` (IDs of the 2 closest rocks in the row above, by fractional position). Reachability propagates upward from the solved rock via `updateReachability` → `getReachableFrom`.

6. **Canvas rendering** (`drawWall`, `drawRock`, `computeRockPositions`) — Rocks are drawn as seeded irregular polygons (`getRockPoints` + `seededRng`). `rockPositions` maps rock id → `{x, y, r}` and is recomputed on resize. The stuffy emoji is drawn above `currentRockId`.

7. **Answer modal** — `openAnswerModal(rock)` captures `modalRock`. First wrong answer: freebie + hint button appears. Second wrong answer: show correct answer, close modal, call `grayOutRock()`. Two grayed rocks total → `triggerLevelFail()`. Correct: call `markRockSolved(rock)` which checks if the top rock was just solved → `triggerLevelComplete()`.

8. **Sound** — Web Audio API via `playSound('correct'|'wrong'|'bell')`. `audioCtx` is lazily created on first interaction (required by browsers).

## Git workflow

After every meaningful unit of work — a feature added, a bug fixed, a visual change — commit and push to GitHub. Do not batch multiple unrelated changes into one commit.

```bash
git add index.html
git commit -m "short description of what changed and why"
git push
```

Commit messages should be specific (e.g. `fix: grayed rock not blocking path after second miss`) not generic (e.g. `update game`). This ensures no work is ever lost and the history is readable.

## Key invariants

- Rock `state` values: `'inactive'` | `'reachable'` | `'solved'` | `'grayed'`
- Rows are stored bottom-first (`rows[0]` = bottom row, `rows[rows.length-1]` = top row with 1 rock)
- Win condition: the single rock in `rows[rows.length-1]` reaches state `'solved'`
- `game.missedRocks` counts grayed rocks (not individual wrong answers); fail triggers at `>= 2` in non-practice mode
- Practice mode (`state.practiceMode`) skips miss counting and the fail trigger entirely
