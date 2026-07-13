# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project

Tetris implemented in vanilla JavaScript with HTML5 Canvas and CSS. No dependencies, no build process, no package.json.

## Running the game

Open `index.html` directly in a browser, or serve it with any static server, e.g.:

```bash
python3 -m http.server 8000
# or
npx serve .
```

There is no build, lint, or test tooling in this repo — there's nothing to compile or check beyond opening the page and playing.

## Architecture

Three files, no modules/bundler — `game.js` is loaded as a single classic script from `index.html`.

- `index.html` — DOM structure: the `#board` canvas (300×600, i.e. `COLS × BLOCK` by `ROWS × BLOCK`), the `#next-canvas` preview canvas, HUD elements (`#score`, `#lines`, `#level`), and the pause/game-over `#overlay`.
- `style.css` — dark/retro arcade visual theme only.
- `game.js` — all game logic, structured around this flow:

```
init() → createBoard(), spawn() first piece, requestAnimationFrame(loop)
loop(ts) → accumulate dt; when dt ≥ dropInterval, advance the piece or lockPiece(); draw(); reschedule
keydown → move / tryRotate() / softDrop() / hardDrop() / togglePause()
```

Key mechanics, all in `game.js`:

- **Board model**: `ROWS × COLS` matrix; each cell is `0` (empty) or a color index `1–7` identifying the piece type.
- **Pieces**: defined as square matrices in `PIECES`; rotation (`rotateCW`) transposes + reverses rows.
- **Collision** (`collide`): checks board bounds and overlap with locked cells.
- **Wall kicks** (`tryRotate`): on rotation collision, retries at x offsets `[0, -1, 1, -2, 2]` before giving up.
- **Locking** (`lockPiece`): merges the piece into `board`, clears full lines, spawns the next piece.
- **Line clearing** (`clearLines`): scans bottom-up, splices full rows out and unshifts empty rows in; awards score via `LINE_SCORES` scaled by `level`.
- **Leveling/speed**: level increases every 10 lines; `dropInterval = max(100, 1000 - (level - 1) * 90)` ms.
- **Ghost piece** (`ghostY`): projects the current piece straight down to its landing row, drawn at low alpha.
- **Game over**: triggered in `spawn()` when a freshly spawned piece already collides.

Tunable constants live at the top of `game.js`: `COLS`, `ROWS`, `BLOCK`, `COLORS`, `LINE_SCORES`, `dropInterval`. If `COLS`/`ROWS`/`BLOCK` change, update the `#board` canvas `width`/`height` in `index.html` to match (`COLS × BLOCK`, `ROWS × BLOCK`).
