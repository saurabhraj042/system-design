# Connect Four

**Difficulty:** Easy | **Concepts:** Grid state, win detection, turn management, state machines

---

## Understanding the Problem

Connect Four is a two-player game on a 7-column, 6-row grid. Players alternate dropping discs into columns; each disc falls to the lowest available row. The first player to place four of their own discs in a line — horizontally, vertically, or diagonally — wins. If the grid fills with no winner, it's a draw.

---

## Requirements Clarification

Before designing, ask questions to pin down the spec.

**Core actions:**
- How do players interact? → Choose a column (0–6); disc falls to lowest available row
- What triggers game end? → Four in a row wins; full board with no winner is a draw
- What if someone drops in a full column? → Reject with false/error; don't corrupt state
- What if someone moves out of turn? → Reject the same way

**Scope:**
- Single game at a time (not concurrent games)
- Backend logic only — no UI support; no methods like `getValidMoves()` needed
- No move history, no undo, always 7×6

**Final Requirements:**
```
1. Two players take turns dropping discs into a 7-col, 6-row board
2. A disc falls to the lowest available row in the chosen column
3. Game ends when: four in a row (vertical/horizontal/diagonal) → WIN
                   board is full with no winner → DRAW
4. Invalid moves rejected: full column, wrong player's turn, game already over

Out of scope: UI, concurrent games, move history, undo, configurable board size
```

---

## Core Entities

Three nouns emerge from the requirements. Ask "does this thing maintain state or enforce rules?" to decide if it deserves its own class.

| Entity | Responsibility |
|--------|---------------|
| **Game** | Orchestrator. Holds the board, tracks whose turn it is, manages game state (IN_PROGRESS/WON/DRAW), and enforces turn-based rules. Single entry point for external code. |
| **Board** | Owns the 7×6 grid. Handles disc placement (gravity), full-column checks, and win detection. Doesn't know or care whose turn it is. |
| **Player** | Pure data holder: name and disc color. No game logic. |

**Key design insight:** Board encapsulates all physical rules (grid state, placement, win detection). Game handles orchestration (turn order, game lifecycle). Player is just data. This separation makes each class testable in isolation.

---

## Class Design

### Game (Orchestrator)

Derive state from requirements — what does Game need to remember to enforce each rule?

| Requirement | What Game must track |
|-------------|---------------------|
| "Two players take turns" | player1, player2, currentPlayer |
| "Game ends when..." | state: GameState (enum) |
| "A player gets four in a row → wins" | winner: Player? (nullable) |
| "Drop disc into a 7×6 board" | board: Board |

**Why use an enum for GameState instead of boolean flags?** Boolean flags like `isOver` and `hasWinner` allow impossible states: `isOver=true, hasWinner=false` (what does that mean — draw or in-progress?). An enum `{IN_PROGRESS, WON, DRAW}` makes exactly three valid states and no others. Every state is self-documenting.

```
class Game:
    - board: Board
    - player1: Player
    - player2: Player
    - currentPlayer: Player
    - state: GameState      // IN_PROGRESS, WON, DRAW
    - winner: Player?       // null if no winner yet or draw

    + Game(player1, player2)
    + makeMove(player, column) → bool
    + getCurrentPlayer() → Player
    + getGameState() → GameState
    + getWinner() → Player?
    + getBoard() → Board
```

`makeMove` is the only method that mutates state. Everything else is read-only. `winner` is nullable — in a draw there simply is no winner, which is cleaner than a sentinel "NONE" value.

### Board

| Requirement | What Board must track |
|-------------|----------------------|
| "7-column, 6-row board" | rows=6, cols=7 (fixed dimensions) |
| "Disc falls to lowest available row" | Grid state for each cell |
| "Board is full → draw" | Whether any column has space |
| "Four in a row" | Grid content to scan for contiguous matches |

Store `DiscColor?` not `Player` in the grid — simpler type, easier to test without mocking Player objects.

```
class Board:
    - rows: int = 6
    - cols: int = 7
    - grid: DiscColor?[rows][cols]

    + Board()
    + getRows() → int
    + getCols() → int
    + canPlace(column) → bool
    + placeDisc(column, color) → int    // returns row where disc lands; -1 if invalid
    + isFull() → bool
    + checkWin(row, column, color) → bool
    + getCell(row, column) → DiscColor?
```

`placeDisc` returns the row so `Game` can pass `(row, column, color)` to `checkWin` without re-scanning.

### Player

Minimal data holder. No game logic lives here.

```
class Player:
    - name: string
    - color: DiscColor

    + Player(name, color)
    + getName() → string
    + getColor() → DiscColor

enum GameState:
    IN_PROGRESS
    WON
    DRAW

enum DiscColor:
    RED
    YELLOW
```

---

## Implementation

### `makeMove` (the core method)

**Happy path:** Place disc → check win → check draw → switch turns

**Edge cases (reject before touching state):**
1. Game already over (state is WON or DRAW)
2. Wrong player's turn
3. Column is full (delegated to Board via `placeDisc` return value)

```python
makeMove(player, column):
    if state != IN_PROGRESS:
        return false
    if player != currentPlayer:
        return false
    
    row = board.placeDisc(column, player.getColor())
    if row == -1:
        return false    # column out of bounds or full
    
    if board.checkWin(row, column, player.getColor()):
        state = WON
        winner = player
    else if board.isFull():
        state = DRAW
    else:
        currentPlayer = (player == player1) ? player2 : player1
    
    return true
```

Notice: Game does not check column bounds directly — it delegates all grid validation to Board. Board returns -1 for any invalid placement. This keeps concerns cleanly separated.

### `placeDisc` (gravity mechanic)

**Core logic:** Scan from bottom row upward; place disc in first empty cell.

```python
placeDisc(column, color):
    if column < 0 || column >= cols:
        return -1
    if !canPlace(column):
        return -1
    
    for row = rows - 1 down to 0:
        if grid[row][column] == null:
            grid[row][column] = color
            return row
    
    return -1

canPlace(column):
    if column < 0 || column >= cols:
        return false
    return grid[0][column] == null  // top row empty = column has space

isFull():
    for c = 0 to cols - 1:
        if canPlace(c):
            return false
    return true
```

**Optimization note:** You can maintain a `heights[cols]` array tracking the next free row per column for O(1) placement. For a 7×6 board, the simple scan is fine.

### `checkWin` (the tricky method)

**Key insight:** After placing at (row, col), you only need to check if the new disc creates four in a row. Check all four directions, counting in both directions from the placed position.

**Directions as vectors:**
- Horizontal: (dr=0, dc=1)
- Vertical: (dr=1, dc=0)
- Diagonal down-right: (dr=1, dc=1)
- Diagonal up-right: (dr=-1, dc=1)

For each direction, count same-color discs moving forward AND backward from (row, col). If the total (including the placed disc itself) reaches 4+, return true.

```python
checkWin(row, col, color):
    if row < 0 || row >= rows || col < 0 || col >= cols:
        return false
    if grid[row][col] != color:
        return false
    
    directions = [[0,1], [1,0], [1,1], [-1,1]]
    for dr, dc in directions:
        count = 1                                          // count the placed disc
        count += countInDirection(row, col, dr, dc, color)    // forward
        count += countInDirection(row, col, -dr, -dc, color)  // backward
        if count >= 4:
            return true
    
    return false

countInDirection(row, col, dr, dc, color):
    count = 0
    r = row + dr
    c = col + dc
    while inBounds(r, c) && grid[r][c] == color:
        count++
        r += dr
        c += dc
    return count

inBounds(row, col):
    return row >= 0 && row < rows && col >= 0 && col < cols
```

**Why not separate WinChecker classes?** Over-engineering. Four directions, one counting helper, one method. The direction vector approach handles all four cases in ~10 lines. Adding separate `HorizontalWinChecker`, `VerticalWinChecker`, etc. classes would triple the code and add indirection with zero benefit.

---

## Verification Trace

Start with partially filled board: Row 5 (bottom): [RED, YELLOW, RED, _, _, _, _], Row 4: [RED, YELLOW, _, _, _, _, _]

```
currentPlayer = player1 (RED), state = IN_PROGRESS

Move 1: player1 → column 0
  placeDisc(0, RED) → lands at row 3
  checkWin(3, 0, RED): vertical count = 1 + 2 = 3 (rows 4,5 also RED)
  No win (count=3 < 4), currentPlayer = player2

Move 2: player2 → column 1  
  placeDisc(1, YELLOW) → row 3
  checkWin(3, 1, YELLOW): vertical count = 3 (rows 3,4,5)
  No win, currentPlayer = player1

Move 3: player1 → column 2
  placeDisc(2, RED) → row 4
  checkWin(4, 2, RED): no consecutive 4 in any direction
  currentPlayer = player2

Move 5: player1 → column 0 again
  placeDisc(0, RED) → row 2
  checkWin(2, 0, RED): vertical ↓ (3,0)=RED, (4,0)=RED, (5,0)=RED → count = 1 + 3 = 4 ✓
  state = WON, winner = player1

Move 6: player2 tries column 1
  state != IN_PROGRESS → returns false immediately
```

This trace verifies: disc placement, vertical win detection, state transitions, move rejection after game ends.

---

## Extensibility

### "How would you support different board sizes?"

Board hardcodes `rows=6, cols=7` right now. To make it configurable, pass dimensions to the Board constructor:

```
Board(rows, cols)
```

All placement and win logic already works with arbitrary dimensions — it uses `rows`, `cols`, and `inBounds` throughout. Game just needs to pass the right Board at construction time. This change is isolated to Board's constructor and Game's initialization.

### "How would you add undo?"

Undo belongs in Game because Game controls lifecycle and turn order. A move record needs player, row, and column — the row is already returned by `placeDisc`.

```
class Move:
    - player: Player
    - row: int
    - col: int

// Add to Game:
- moveHistory: Stack<Move>

// In makeMove, after successful placement:
moveHistory.push(Move(player, row, column))

// New method on Board:
+ clearCell(row, col) → grid[row][col] = null

// undoLastMove on Game:
undoLastMove():
    if moveHistory.isEmpty():
        return false
    last = moveHistory.pop()
    board.clearCell(last.row, last.col)
    currentPlayer = last.player
    state = IN_PROGRESS    // simplest approach: reset, recompute if needed
    winner = null
    return true
```

Board only needs one new method. Game controls the history stack. This change doesn't affect any other class.

### "How would you add a computer opponent?"

The key insight: game rules don't change. Game still enforces turns, Board still validates placements. A bot just picks a column.

```
class BotEngine:
    + chooseMove(game) → int    // returns column number

// Simple implementation (pick first valid column):
chooseMove(game):
    board = game.getBoard()
    for col = 0 to board.getCols() - 1:
        if board.canPlace(col):
            return col
    return -1

// Game loop:
game = Game(humanPlayer, botPlayer)
bot = BotEngine()
while game.getGameState() == IN_PROGRESS:
    current = game.getCurrentPlayer()
    if current == humanPlayer:
        column = /* read from input */
    else:
        column = bot.chooseMove(game)
    game.makeMove(current, column)
```

Board and Game are unchanged. The bot is just a decision-making component that picks which column to pass into `makeMove`. Player stays as pure data — no need to make it an interface.

---

## What Interviewers Look For

| Level | Bar |
|-------|-----|
| Junior | Three classes with sensible responsibilities; `placeDisc` gravity works; horizontal + vertical win detection; handles full column and out-of-turn moves |
| Mid-level | Clean separation: Game orchestrates, Board owns grid logic, Player is just data. `makeMove` validates before mutating. Win detection handles all four directions via direction vector approach (not four separate methods). Can discuss one extensibility scenario. |
| Senior | Design decisions proactively justified: why GameState enum over booleans, why win detection lives on Board not Game, why DiscColor not Player in the grid. `checkWin` uses direction vectors with a single `countInDirection` helper. Catches own edge cases. Can discuss networked multiplayer or spectator mode structure. |

---

## Practice Questions

1. Why is `DiscColor` stored in the grid rather than `Player`? What would break if you stored `Player` instead?
2. In `makeMove`, the column validity check is delegated to `Board.placeDisc`. Why not have `Game` call `board.canPlace(column)` first, then `board.placeDisc(column, color)` separately?
3. The win check only runs after each placed disc. Could there ever be a win that this misses? Why or why not?
4. How would your design change if you needed to support spectators watching the game in real time (observers that get notified on each move)?
5. A player drops a disc in column 3. It lands at row 4. `checkWin` scans the diagonal up-right direction. Trace through which cells it examines and why.

## One-Line Summary

Connect Four maps cleanly to three classes: Game orchestrates turns and lifecycle, Board owns the grid and all physical rules (placement, win detection), and Player is pure data — keep them separate and the whole design becomes a 10-minute interview problem.
