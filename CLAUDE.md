# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

"Mathe-Held" — a German-language mental-arithmetic game for kids, built as a
single self-contained HTML file (inline CSS + vanilla JS, no build step, no
dependencies, no package manager). It is designed primarily as an iOS
add-to-home-screen PWA, so much of the CSS deals with safe areas, notches,
and per-device viewport tuning.

## Running

Open the HTML file directly in a browser, or serve the folder statically:

```
python3 -m http.server 8000   # then open http://localhost:8000/
```

There are no builds, linters, or tests to run.

## The two-file sync constraint (important)

`index.html` and `mathe-spiel.html` are **byte-identical and must be kept that
way**. `mathe-spiel.html` is the original; `index.html` is the copy GitHub
Pages serves from the repo root. Any change to the game must be applied to
**both** files in the same commit. Verify with `diff index.html
mathe-spiel.html` before committing.

## Architecture

Everything lives in one file. Layout:

- **CSS** (`<style>`): CSS-variable theme in `:root`, `clamp()`-based fluid
  sizing throughout, and a stack of `@media` blocks at the end that retune
  spacing for specific device classes (iPhone Pro Max, SE, landscape). Touch
  behavior is locked down globally (`user-select: none`, tap-highlight off).
- **`state` object**: the single source of truth for a play session
  (`maxNum`, `level`, `xp`/`xpNeeded`, `stars`, `streak`, `correct`/`total`,
  `questionNum`). `resetState()` reinitializes it for a new round.
- **Screen model**: three `.screen` divs (start / game / summary) plus a
  fixed level-up `.overlay`. Only one screen has the `.active` class at a
  time; `showScreen(name)` toggles it. There is no router — this class swap
  *is* the navigation.
- **Game loop**: `startGame()` → `nextQuestion()` → `handleAnswer()` →
  (after a 1600ms delay) back to `nextQuestion()`, until `questionsPerRound`
  is reached, then `showSummary()`. `generateQuestion()` produces add/subtract
  problems bounded by `state.maxNum` (results never go negative) plus three
  distractor answers.

When changing scoring, leveling, or question generation, the relevant logic is
concentrated in `handleAnswer` (XP/stars/streak rules, level-up trigger) and
`generateQuestion` (problem and distractor generation).

## Conventions

- UI strings are German; keep new user-facing text German and in the existing
  encouraging, kid-friendly tone (see the `PRAISE` / `ENCOURAGE` / `LEVEL_MSGS`
  arrays).
- DOM access goes through the `$ = id => document.getElementById(id)` helper.
- Keep it dependency-free — no frameworks, bundlers, or external scripts. The
  only external resource is the Nunito web font.
