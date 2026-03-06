# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Game Rules

Full rule reference: [RULES.md](./RULES.md)

## Project Overview

**서울 시티 인베스트 (Seoul City Invest)** — A Korean Monopoly-style browser board game set in Seoul. The entire project lives in a single file: `burumabul.html`.

## Running the Project

Open `burumabul.html` directly in a modern browser. No build step, no npm, no dependencies. Works fully offline.

## Architecture

The file is structured in three sections:

1. **HTML (lines 1–662):** Board layout, player info panels, modal containers, and inline `<style>` CSS
2. **CSS (lines 7–662, inside `<style>`):** Board grid, animations, CSS custom properties (`--bg`, `--cell-w`, `--corner`, color palette), flexbox/grid layout
3. **JavaScript (lines 771–3228, inside `<script>`):** All game logic

### Game State

A single global `state` object holds all runtime data:
```js
state = {
  players: [{ id, name, pos, money, loan, jailTurns, laps, job, stats }],
  props: { cellId: [{owner, buildings}] },  // multi-ownership array per cell
  currentPlayer, phase, doubleCount, log, totalTurns,
  _marketChanges, _lastFluctuationLap
}
```

`phase` drives the turn FSM: `'turn_announce' → 'roll' → 'moving' → 'action' → 'on_my_prop' → 'gameover'`

### Turn Flow

```
announceTurn() → rollDice() → movePlayer() → landOn() → handleCell() → endTurn()
```

- `movePlayer()` uses recursive `stepOne()` with 220ms per cell
- `landOn()` dispatches to cell-specific handlers (`handleProp`, `drawChance`, `handleTransport`, etc.)
- `endTurn(isDouble)` handles re-roll on doubles, 3-doubles-to-jail, and lap completion logic
- `aiTurn()` runs the AI player's full turn including strategic building upgrades and loan decisions

### Key Constants

```js
START_MONEY = 100_000_000     // 1억원
WIN_NET_WORTH = 10_000_000_000  // 100억원 — instant win
MAX_LAPS = 50                   // fallback win condition
LOAN_LTV = 0.7                  // 70% max borrow against property value
LOAN_RATE = 0.045               // 4.5% annual interest, charged at lap completion
```

### Property System

`getBld(cellId)` returns `{ cost: [villa, apt, building], rent: [...], sell: [...] }` for each cell. Properties can be owned at 3 building levels. `getNetWorth(pi)` = cash + sum of sell prices − loan.

Market fluctuates every 5 laps: all property values shift −40% to +60% (stored in `state._marketChanges`).

### Job System

`getRandomJob()` returns a weighted random job from 20 options. Lower-salary jobs are 10× more likely. Salary is paid when passing Start (lap completion). `job.salary` ranges from 0 (취준생) to 1,000,000,000 (금수저, 1/100 chance).

### UI / Modal System

`showModal(title, content, buttons)` is the single generic modal. All game events (purchase, chance cards, game over) use it. No external UI library.

### Chance Cards

32 hardcoded chance events in the `CHANCE_CARDS` array. `drawChance(playerIdx, isDouble)` picks randomly and applies the effect inline.
