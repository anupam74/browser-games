# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository

**GitHub**: https://github.com/anupam74/browser-games  
**Branch**: `main`

After any meaningful change, commit and push:
```
git add <files>
git commit -m "..."
git push
```

## How to Run

All games are single HTML files — open directly in a browser, no server or build step needed:
- `tictactoe.html` — Tic Tac Toe
- `shooter.html` — Sector Zero (top-down shooter)
- `tetris.html` — Tetris with 10 difficulty levels

## Project Pattern

Every game is a **single self-contained `.html` file** with inline `<style>` and `<script>`. No external dependencies, no build system, no npm. This is a strict constraint — do not introduce separate JS/CSS files or package managers.

Color theme across all games: dark background (`#1a1a2e` / `#0d0d1a`).

---

## shooter.html — Sector Zero

**Canvas**: fixed 800×600, `cursor: none` (crosshair drawn manually).

**Game state machine** — top-level `gameState` string dispatches to separate update/render functions:
```
MENU → PLAYING → LEVEL_TRANSITION → PLAYING (loop)
               → GAME_OVER → MENU
```

**Player controls**: WASD / arrow keys to move; mouse to aim; hold left-click to shoot; `R` to reload manually.

**Code sections** (in order inside `<script>`):
1. `CONSTANTS` — all tunable numbers live here (`PIXEL`, speeds, `ENEMY_STATS`, etc.)
2. `CANVAS SETUP`
3. `GAME STATE` — `gameState` string + `currentLevelIdx` / `currentWaveIdx`
4. `AUDIO` — Web Audio API synthesized SFX; `ensureAudio()` must be called on first user gesture
5. `INPUT` — `keys{}` map (keydown/keyup) + `mouse{x,y,down,clicked}`; `mouse.clicked` is a single-frame flag consumed by whichever state handler runs first
6. `SCREEN SHAKE` — `triggerShake(dur, intensity)` → translates ctx randomly each frame
7. `PALETTE + SPRITES` — `PAL{}` color map; sprites are 9×9 string arrays; `drawSprite(sprite, cx, cy, angle)` scales by `PIXEL=4` and rotates around center; all sprites face RIGHT (angle=0)
8. `PARTICLES` — flat `particles[]` array + `scorePopups[]`; `spawnParticles(x,y,count,colors,minSpd,maxSpd,life)`
9. `PLAYER` — single `player` object; `updatePlayer(dt)` handles movement, aiming, shooting, reload, invincibility timer
10. `PLAYER BULLETS` — `bullets[]`; collision vs. enemies checked in `updateBullets(dt)`
11. `ENEMY BULLETS` — `enemyBullets[]`; only TANKs fire; collision vs. player in `updateEnemyBullets(dt)`
12. `ENEMIES` — `enemies[]`; `spawnEnemy(type, hpScale, speedScale)`; AI per type:
    - `GRUNT`: straight pursuit
    - `DASHER`: state machine (`CIRCLING` → `DASHING` → `CIRCLING`)
    - `TANK`: pursuit + fires projectiles on `shootTimer`
13. `WAVE / LEVEL SYSTEM` — `LEVELS[]` array (3 hand-crafted); `generateLevel(idx)` for level 4+; `activeSpawners[]` gives each enemy type its own independent spawn timer; wave clears when all spawners empty AND `enemies.length === 0`
14. `LEVEL TRANSITION` — 3.2s screen, skippable on click
15. `MENU` / `GAME OVER` — overlay screens
16. `HUD` — drawn last in screen space (after `ctx.restore()` undoes shake)
17. `GAME LOOP` — `requestAnimationFrame`; `dt` capped at 50ms

**Adding a new enemy type**: add an entry to `ENEMY_STATS`, add a sprite to `SP`, add AI branch in `updateEnemies`, add render branch in `renderEnemies`, add the type to wave specs in `LEVELS`.

**Adding a new level**: append to `LEVELS[]` following the same shape: `{ waves, hpScale, speedScale, newThreat }`. Each wave is an array of `{ type, count, interval }` specs — one per enemy type, each with its own independent spawn timer.

**Tuning feel**: all gameplay numbers are in the `CONSTANTS` section at the top. Sprite pixel size is `PIXEL` (currently 4).

---

## tetris.html

**Canvas**: 300×600 (`gc`, 10×20 grid, 30px cells). Two additional mini-canvases: `hc` (hold piece, 88×72) and `nc` (next piece, 88×72). All UI panels (score, level, lines, best) are DOM elements flanking the canvas — no framework.

**Game state** — `gameState` string:
```
'menu' → 'playing' ↔ 'paused'
'playing' → 'clearing' → 'playing'   (line-clear animation, 280ms)
'playing' → 'over' → restart → 'playing'
```

**Piece system**: the `T` object holds all 7 tetrominoes (I O T S Z J L). Each entry has `col` (fill), `drk` (shadow), and `sh[]` (array of rotation matrices). The active piece (`cur`) tracks `{ type, rotIdx, shape, x, y, col, drk }`. Pieces spawn at `y=-1`; cells above row 0 are skipped during rendering and placement.

**7-bag randomiser**: `nextBag()` fills `bag[]` with a shuffled copy of all 7 types and pops one per request — guarantees each piece appears once per cycle.

**Lock delay**: 500ms timer (`lockActive` / `lockAcc`). Moving or rotating resets it via `resetLock()`. Timer only starts fresh when the piece first touches a surface (`!lockActive` guard in the drop section).

**Code sections** (in order inside `<script>`):
1. `CONSTANTS` — `COLS`, `ROWS`, `CELL`, `LEVELS[]` (10 entries with `n`, `name`, `ms`, `req`), `LINE_PTS[]`, tetromino definitions in `T{}`
2. `DOM` — canvas refs, overlay refs, display element refs
3. `GAME STATE` — all `let` declarations; `gameState` + per-frame accumulators
4. `7-BAG RANDOMISER` — `nextBag()`
5. `PIECE CREATION` — `mkPiece(type)` → piece object at spawn position
6. `BOARD UTILITIES` — `mkBoard()`, `valid(shape, px, py)` (skips rows < 0)
7. `PIECE ACTIONS` — `rotate(dir)` with 5-offset wall kicks, `move(dx)`, `softDrop()`, `holdPiece()`, `resetLock()`
8. `GHOST` — `ghostY()` drops `cur` until invalid
9. `PLACE & CLEAR` — `place()` writes `cur` to board (skips `br<0`); `findLines()` triggers scoring + clears or spawns; `finishClear()` splices rows and calls `spawnPiece()`
10. `SPAWN` — `spawnPiece()` pulls from `nxt`, game-over if spawn position is invalid
11. `GAME LOOP` — single `loop(ts)` via `requestAnimationFrame`; handles `'clearing'` branch (animation only) then normal drop + lock logic; `dt` flows naturally (no cap)
12. `RENDERING` — `render()`, `drawBg()` (grid lines), `drawBoard()`, `drawGhost()`, `drawPieceMain()`, `drawClearFlash()` (sin-wave alpha), `cell()` (3-D bevel), `renderMini()` (shared hold/next renderer)
13. `UI` — `updateUI()` syncs all DOM displays + level-bar width + localStorage high score
14. `GAME STATES` — `initGame()`, `startGame()`, `pauseGame()`, `gameOver()`, `showLevelUp()` (1.8s overlay, non-blocking), `showOnly()`
15. `INPUT` — `held{}` keymap + per-key repeat timers (`REP_DELAY=160ms`, `REP_RATE=45ms`); `handleKey(code)` dispatches actions
16. `TOUCH CONTROLS` — swipe gestures on `gc`: tap=rotate, swipe-left/right=move, swipe-down=soft-drop, swipe-up=hold
17. `BOOT` — reads localStorage high score, sets `gameState='menu'`, draws initial empty grid

**Scoring**: `LINE_PTS = [0, 100, 300, 500, 800]` × current level number. Soft-drop adds 1 pt per cell. High score saved to `localStorage` key `tet_hi`.

**Level progression**: `LEVELS[lvIdx].req` is the cumulative lines threshold. Checked in `checkLevelUp()` after every clear. Level 10 (`req: Infinity`) never auto-advances.

**Adding a new level**: append to `LEVELS[]` with `{ n, name, ms, req }` — `ms` is the drop interval in milliseconds, `req` is cumulative lines to trigger the next level.

---

## tictactoe.html

Single `board[]` array (9 elements, `null | 'X' | 'O'`). `checkWinner()` iterates the hardcoded `WINS` triples. Score persists across `init()` calls (new game resets board only).
