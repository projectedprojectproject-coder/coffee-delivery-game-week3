# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repo is

A Korean-language teaching exercise ("프롬프트 구조화 실습", week 3) built around a single-file browser
game, "달동네 커피" (Dal-dongne Coffee / Neighborhood Coffee): the player collects coffee cups at a cafe
and delivers them to neighbors around a small procedurally-rendered 3D village, rendered entirely with a
hand-rolled software 3D renderer onto a 2D `<canvas>`. There is no build step, no package manager, and no
external libraries or APIs — the whole game (HTML, CSS, game logic, renderer, DOM wiring) lives in
`index.html`. `README.md` is the student-facing lesson (five-slot prompt structure: 현재 상황 / 목표 /
구체적인 변경 / 유지할 것 / 완료 기준 / 결과물) and `WORKSHEET.md` is the submission log.

## Commands

- Run the game: open `index.html` directly in a browser (double-click or `file://`). No server needed.
- Run tests: `node --test tests/game.test.cjs`
- Run a single test: `node --test tests/game.test.cjs --test-name-pattern "<name substring>"`

There is no linter, formatter, or bundler configured for this project.

## Architecture (`index.html`)

The `<script>` block is a single IIFE that builds a `Daldongne` module (UMD-style: exports via
`module.exports` under Node, or `window.Daldongne` in a browser) and ends with page-level wiring that
mounts it onto the page's `<canvas>` and DOM controls. Three layers, in order:

1. **Pure game logic** (no DOM/canvas access) — `SITES` (village building data), `newGame`, `start`,
   `pause`, `interact`, `step`, `chooseTarget`, `advanceDay`, perk helpers (`pickPerks`, `applyPerk`,
   `choosePerk`). All state lives in one plain object returned by `newGame()`; every function takes that
   state `s` and mutates it in place. This layer is unit-testable in isolation (see Testing below).
2. **Rendering** — `makeScene` builds the static world geometry once per locale (cached in `sceneCache`)
   as a list of 3D polygons/labels; `draw(ctx, state)` re-projects the camera each frame, draws dynamic
   elements (player/neighbors/carried cups, HUD stat plate, compass, status overlays), and paints
   everything to the 2D canvas context. There is no WebGL/Three.js — `camera()` does the perspective
   projection by hand and polygons are painter's-algorithm sorted by `layer`/depth before filling.
3. **Mount / DOM wiring** — `mount(canvas, onUpdate)` owns the `requestAnimationFrame` loop, keyboard/
   pointer event listeners, and returns the public control API (`start`, `pause`, `reset`, `interact`,
   `perk`, `locale`, `target`, `pointer`, `destroy`) that the bottom-of-file page script wires up to the
   on-screen buttons (`#start`, `#pause`, `#reset`, `#action`, direction buttons, `#perk1`/`#perk2`) and
   keyboard shortcuts (arrows/WASD, E/Space, Enter, `1`/`2` for perk choice, `N` to toggle the controls
   panel).

### Roguelike day/perk loop

The delivery run is structured as repeating "days": `pickDayOrders()` randomizes which non-cafe buildings
need a delivery each day, `dayTimeLimit(day)` shrinks the time budget 10% per day from a 180s baseline,
and clearing a day (`status: 'dayclear'`) offers two random perks from `PERK_POOL` (speed/time/radius/
extra-life) via `choosePerk`/`advanceDay` before the next day starts. `status` drives both the canvas
overlay and which DOM controls are visible — valid values are `ready`, `playing`, `paused`, `dayclear`,
and `timeup`. Best-day progress persists across runs via `localStorage` (`loadBestDay`/`saveBestDay`).

### Testing harness quirk

`tests/game.test.cjs` loads the game by reading `index.html`, regex-extracting the `<script>` contents,
and slicing the string at `script.indexOf('const game=Daldongne.mount')` — everything *before* that marker
is run in a sandboxed Node `vm` context with no `document`/`window`. This means any page-wiring code that
touches the DOM (e.g. `document.querySelector`) must stay **after** the `const game=Daldongne.mount(...)`
line in the file, or the test loader throws `ReferenceError: document is not defined`. Currently the tests
are stale against the game logic itself too (they reference `g.ORDERS` and a `'won'` status that no longer
exist after the day/perk rework — the live API is `s.dayOrders`/`'dayclear'`) and a DOM-touching
`perksEl`/`syncPerks` block was added before the mount marker, so `node --test tests/game.test.cjs`
currently fails outright. Keep this constraint in mind before adding new top-of-file DOM code or trusting
a green test run.

## Pushing to GitHub

This folder is a real git repo (`git remote -v` → `origin` =
`https://github.com/projectedprojectproject-coder/coffee-delivery-game-week3`, tracking `main`). Use plain
`git add` / `git commit` / `git push` — pushing is a write action, so only do it when the user explicitly
asks.
