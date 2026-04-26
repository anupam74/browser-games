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

## tictactoe.html

Single `board[]` array (9 elements, `null | 'X' | 'O'`). `checkWinner()` iterates the hardcoded `WINS` triples. Score persists across `init()` calls (new game resets board only).
