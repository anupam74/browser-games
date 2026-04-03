# Tic Tac Toe Project

## Overview
Single-file HTML/CSS/JS implementation of Tic Tac Toe (`tictactoe.html`). No build system, no dependencies.

## Structure
Everything lives in one file:
- **CSS**: Dark-themed UI (`#1a1a2e` background), responsive 3×3 grid using CSS Grid
- **HTML**: Board cells with `data-i` attributes (0–8), scoreboard, status line, reset button
- **JS**: Vanilla JavaScript — no frameworks

## Key JS Concepts
- `board`: flat array of 9 elements (`null | 'X' | 'O'`)
- `WINS`: hardcoded winning index triples
- `checkWinner()`: iterates `WINS`, returns `{ winner, line }` or `null`
- Score persists across games (reset only resets the board, not scores)

## How to Run
Open `tictactoe.html` directly in a browser — no server needed.
