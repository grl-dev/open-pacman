# AGENTS.md

Vanilla Pac-Man in `src/`. No build step, no tests, no linter, no package manager.

## Run

Open `src/index.html` directly in a browser, or serve `src/` with any static server (e.g. `python -m http.server -d src`).

## Layout

- `src/index.html` — entrypoint; loads scripts in order: `maze.js` → `game.js` → `render.js` → `main.js`.
- `src/js/maze.js` — `MAZE` grid and start positions (`PACMAN_START`, `GHOST_STARTS`, `TUNNEL_ROW`).
- `src/js/game.js` — `createGame()` factory, `update(game)` rules, dot/ghost/pacman state.
- `src/js/render.js` — `draw(ctx, game, frame)`.
- `src/js/main.js` — rAF loop, keyboard, overlay/screens.
- `src/css/style.css`.

Modules communicate via globals (`createGame`, `update`, `draw`, `MAZE`, …). There is no bundler or module system — keep it that way unless asked.

## Workflow: spec-driven

This repo uses two skills in `.agents/skills/`. **Do not write code directly for a non-trivial change** — write a spec first.

- `/spec <feature>` (skill `spec`) — produces `specs/NN-slug.md` in `Draft` state. Asks clarifying questions; never writes code.
- `/spec-impl <spec>` (skill `spec-impl`) — implements an approved spec.

Specs go in `specs/` at the repo root, numbered `NN-kebab-slug.md`. Do not skip the clarifying-question phase; if the user insists, record it in the spec's decisions section.

## Conventions

- Reply in the same language the user wrote in (skills default to matching the prompt language; this repo's README is Spanish).
- Spec language must match the language of the most recent existing specs in `specs/`.
- No frameworks, no npm, no transpilation. Plain ES5-style script tags with string concatenation are used throughout `main.js` — match the existing style when editing.