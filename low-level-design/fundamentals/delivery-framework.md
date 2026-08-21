# LLD Interview Delivery Framework

## The Core Idea

An LLD interview gives you roughly 35 minutes to clarify requirements, model your entities, design class APIs, and walk through core logic. Most candidates fail not because they lack knowledge but because they mismanage the time — diving into code before the structure is clear, or getting lost in details before the interviewer understands the big picture.

A structured framework keeps you grounded when nerves kick in and prevents you from drifting into the weeds.

## Mental Model / Analogy

Think of it like building a house. You don't start hammering nails before you have a blueprint. And you don't spend your first hour picking door knobs. You agree on the layout (requirements), sketch the rooms (entities), draw detailed plans for each room (class design), build the critical load-bearing walls (implementation), then discuss what it'd take to add a second floor (extensibility). Each phase unlocks the next.

---

## The Framework: 5 Stages

```
Requirements → Entities → Class Design → Implementation → Extensibility
    ~5 min       ~3 min      ~10-15 min       ~10 min          ~5 min
```

### Stage 1: Requirements (~5 minutes)

Every LLD prompt is intentionally minimal — "Design Tic Tac Toe" or "Design a parking lot system." Your first job is turning a sentence into a spec you can actually design around.

Ask questions along four themes:

- **Primary capabilities** — What operations must this system support?
- **Rules and completion** — What defines success, failure, or state transitions?
- **Error handling** — What happens when inputs or actions are invalid?
- **Scope boundaries** — What's explicitly in scope (core logic) and what's explicitly out (UI, networking, storage, concurrency)?

Write the spec and out-of-scope list on the whiteboard. Confirming scope prevents the interviewer from adding surprises mid-design, and writing "Out of Scope: networked multiplayer" shows mature engineering judgment.

**Example for Tic Tac Toe:**
```
Requirements:
1. Two players alternate placing X and O on a 3x3 grid
2. A player wins by completing a row, column, or diagonal
3. The game ends in a draw if all nine cells are filled with no winner
4. Invalid moves should be rejected (occupied cell, acting after game over)
5. The system should support querying state and resetting the game

Out of Scope:
- UI/rendering layer
- AI opponent or move suggestions
- Networked multiplayer
- Variable board sizes (NxN)
- Undo/redo functionality
```

---

### Stage 2: Entities and Relationships (~3 minutes)

Scan your requirements and pull out the meaningful nouns. These are the "things" that need to exist in your system — but not every noun. Apply a simple filter:

- If something maintains changing state or enforces rules → it deserves its own entity
- If it's just data attached to something else → it's probably a field on another class

Then define relationships:
- Which entity is the orchestrator (drives the main workflow)?
- Which entities own durable state?
- How do they depend on each other? (has-a, uses, contains)
- Where should specific rules live?

Don't overthink notation. Simple boxes and arrows showing ownership are enough. You're communicating structure, not drawing a formal diagram.

**Example for Tic Tac Toe:**
```
Entities:
- Game
- Board
- Player

Relationships:
- Game → Board (owns one)
- Game → Player (owns two: playerX and playerO)
```

This tells your interviewer the components, the ownership hierarchy, and the separation of responsibilities before you've written a line of code.

---

### Stage 3: Class Design (~10-15 minutes)

Go entity by entity, top-down (orchestrator first, then supporting entities). For each entity, answer two questions:

1. **State** — What does this class need to remember to enforce the requirements?
2. **Behavior** — What operations does the outside world need to perform on it?

**Deriving State:** For each entity, map each requirement to what that class must track.

| Requirement | What `Game` must track |
|-------------|----------------------|
| "Two players alternate turns" | The two players, whose turn it is, the Board |
| "Game ends when someone wins or board is full" | Game state (IN_PROGRESS, WON, DRAW) and winner (if any) |

**Deriving Behavior:** Ask what operations are needed and which requirements they satisfy. Aim for a small, focused API where each method corresponds to a real action or question.

| Need | Method on `Game` |
|------|----------------|
| Players need to make moves | `makeMove(player, row, col) → bool` |
| Ask whose turn it is | `getCurrentPlayer() → Player` |
| Check game state | `getGameState() → GameState` |
| See who won | `getWinner() → Player?` |

**Key principle:** Keep rules with the entity that owns the relevant state. Workflow rules (can this operation run now?) belong in the orchestrator. Data-specific rules (is this cell occupied?) belong in the entity that owns that data. This is encapsulation — sometimes called "Tell, Don't Ask."

**Example class sketch:**
```python
class Game:
    - board: Board
    - playerX: Player
    - playerO: Player
    - currentPlayer: Player
    - state: GameState (IN_PROGRESS, WON, DRAW)
    - winner: Player?

    + makeMove(player, row, col) → bool
    + getCurrentPlayer() → Player
    + getGameState() → GameState
    + getWinner() → Player?
    + getBoard() → Board
```

**On UML:** Skip it. UML was designed for an era when running code was expensive. Modern engineers design in code — they stub out classes and use interfaces. The notation above (state fields, then behavior methods) communicates everything UML does with a fraction of the ceremony.

---

### Stage 4: Implementation (~10 minutes)

**Ask the interviewer what they prefer** before starting — most want pseudocode for key methods; some want near-complete code in a specific language.

Focus on the methods that show:
- How classes cooperate
- How state transitions occur
- How edge cases are handled cleanly
- How logic is isolated in the right classes

**Start with the happy path** — the normal flow when everything goes right. Then add edge cases: invalid inputs, illegal operations, out-of-range values, calls that violate the current system state.

**Example — `makeMove` for Tic Tac Toe:**
```python
makeMove(player, row, col):
    if state != IN_PROGRESS:
        return false
    if player != currentPlayer:
        return false
    if !board.canPlace(row, col):
        return false
    
    board.placeMark(row, col, player.mark)
    
    if board.checkWin(row, col, player.mark):
        state = WON
        winner = player
    else if board.isFull():
        state = DRAW
    else:
        currentPlayer = (player == playerX) ? playerO : playerX
    
    return true
```

**Verification step (~2 minutes):** After implementing, trace through a concrete scenario tick by tick. This catches logical errors before the interviewer finds them and demonstrates the ability to verify your own code. Look for: forgot to switch turns, win detection doesn't trigger, state transitions in wrong order.

---

### Stage 5: Extensibility (~5 minutes, if time allows)

If there's time left, the interview often shifts to "what if" questions. The interviewer proposes a twist to see whether your design can evolve cleanly:

- "How would you add undo functionality?"
- "What if we wanted an AI player?"
- "How would you support an NxN board?"

You stay high-level here — you're not rewriting code, you're pointing to the parts of your design that make the change clean. If you isolated state mutations properly, undo becomes straightforward: add a command history stack. The rest of the system doesn't need to change.

**How much you get here depends on level:**
- Junior: little or no extensibility discussion
- Mid-level: one or two small follow-ups
- Senior: several "what if" questions in a row

---

## Pacing Notes

- **Don't dive into code** before your entities and relationships are clear — this is the most common failure mode
- **Don't over-specify every tiny detail** at the requirements stage — scope boundaries and major rules, not exhaustive edge cases
- **Follow the interviewer's lead** if they pull you off this framework to cover specific questions; gently guide back to cover important bits
- **Use your language of choice** — don't switch to an unfamiliar language to impress an interviewer; it usually backfires

---

## Key Trade-offs

| Stage | Mistake | Fix |
|-------|---------|-----|
| Requirements | Jumping straight to code | Spend 5 minutes on spec + out-of-scope |
| Entities | Over-modeling (too many micro-objects) | Only entities that maintain state or enforce rules |
| Class design | Exposing fields instead of methods | Return behavior, not data; keep rules with the owner |
| Implementation | Starting with edge cases | Happy path first, then enumerate failures |
| Extensibility | Rewriting code | Point to clean boundaries in your design |

## Practice Questions

1. You're given the prompt "Design a coffee machine that dispenses coffee and makes espresso." Walk through all 5 stages of the delivery framework. What requirements do you clarify? What entities do you identify?
2. In the class design stage, why should `Board.canPlace(row, col)` live on `Board` rather than on `Game`? What principle does this illustrate?
3. An interviewer asks you to "design a vending machine." Halfway through class design, they interrupt to ask about how you'd handle a coin jammed in the slot. How do you handle this while keeping the interview on track?
4. During the implementation phase, you realize your `makeMove` logic switches the player before checking for a win. Walk through how you catch this by doing a concrete scenario trace.
5. After you implement the core Tic Tac Toe logic, the interviewer asks "how would you add undo?" Sketch the change without rewriting your classes.

## One-Line Summary

Structure your LLD interview as five timed stages — requirements spec, entity identification, class design, implementation of key methods, and extensibility discussion — and you'll produce clean designs even under pressure.
