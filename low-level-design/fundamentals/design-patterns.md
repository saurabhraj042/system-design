# Design Patterns

## The Core Idea

Design patterns are named, reusable solutions to recurring design problems. They're not code you copy-paste — they're structures that communicate intent. When you name a pattern in an interview ("I'd use the Strategy pattern here"), you're telling the interviewer you recognize the category of problem and know a principled solution.

Most LLD designs use zero to two patterns. Over-applying them is as bad as not knowing them. The question to ask is always: "Does the problem have the shape that this pattern solves, or am I forcing a pattern onto a problem that doesn't need it?"

---

## Pattern Categories

Patterns fall into three families based on what problem they solve:

- **Creational** — How objects get created
- **Structural** — How objects are composed into larger structures
- **Behavioral** — How objects communicate and distribute responsibility

---

## Creational Patterns

### Factory Pattern

**Problem it solves:** You need to create objects of different types, but the type isn't known until runtime, and you don't want the caller to know the construction details.

**Structure:** A factory method or class takes a type identifier and returns the appropriate concrete object.

```python
class ShapeFactory:
    @staticmethod
    def create(shape_type: str) -> Shape:
        if shape_type == "circle":
            return Circle()
        elif shape_type == "rectangle":
            return Rectangle()
        elif shape_type == "triangle":
            return Triangle()
        raise ValueError(f"Unknown shape: {shape_type}")

# Caller doesn't know or care how shapes are built
shape = ShapeFactory.create("circle")
shape.area()
```

**When to reach for it:**
- Multiple concrete types with a common interface
- Object creation logic is non-trivial (multiple parameters, initialization steps)
- You want to decouple callers from concrete implementations

**Classic interview appearances:** Notification service (create email vs. SMS vs. push notification sender), payment processor (create Stripe vs. PayPal processor), logger (create file vs. console vs. remote logger).

**When NOT to use it:** If you only have one concrete type, a factory just adds indirection. If construction is trivial, calling the constructor directly is cleaner.

---

### Builder Pattern

**Problem it solves:** Object construction requires many parameters, and not all are required. Without a builder, you end up with long, error-prone constructors or scattered setter calls.

**Structure:** A Builder class accumulates configuration through method calls, then constructs the final object with `build()`.

```python
class QueryBuilder:
    def __init__(self):
        self._table = None
        self._conditions = []
        self._limit = None
        self._order_by = None
    
    def from_table(self, table: str) -> 'QueryBuilder':
        self._table = table
        return self  # enables chaining
    
    def where(self, condition: str) -> 'QueryBuilder':
        self._conditions.append(condition)
        return self
    
    def limit(self, n: int) -> 'QueryBuilder':
        self._limit = n
        return self
    
    def build(self) -> Query:
        if not self._table:
            raise ValueError("Table is required")
        return Query(self._table, self._conditions, self._limit, self._order_by)

# Clean, readable construction
query = (QueryBuilder()
    .from_table("users")
    .where("age > 18")
    .where("active = true")
    .limit(100)
    .build())
```

**When to reach for it:**
- Object has more than ~4 configuration options
- Some parameters are optional and have defaults
- You want construction to be readable (each parameter named at the call site)

**Classic interview appearances:** HTTP request builder, SQL query builder, email message builder, configuration object builder.

---

### Singleton Pattern

**Problem it solves:** You need exactly one instance of a class globally — for a logger, a configuration object, or a connection pool — and you want controlled access to it.

```python
class Logger:
    _instance = None
    
    @classmethod
    def get_instance(cls) -> 'Logger':
        if cls._instance is None:
            cls._instance = Logger()
        return cls._instance
    
    def log(self, message: str):
        print(f"[LOG] {message}")

# Always returns the same instance
logger = Logger.get_instance()
```

**When to reach for it:** Rarely. The cases where Singleton is genuinely the right answer are narrow: a resource-expensive object that must be shared (connection pool), a global registry (plugin registry), or configuration that's loaded once and read many times.

**Caution:** Singletons are global mutable state in disguise. They make code harder to test (you can't inject a different implementation), can cause subtle ordering bugs (who initializes it first?), and introduce concurrency issues if the initialization isn't thread-safe.

In interviews: If asked about logging or configuration, you can mention Singleton and also note that dependency injection is generally preferred in production systems for testability.

---

## Structural Patterns

### Decorator Pattern

**Problem it solves:** You want to add behavior to an object without modifying its class or creating a subclass for every combination of behaviors.

**Structure:** A wrapper class that implements the same interface as the wrapped object, adds some behavior, and delegates to the wrapped object for the rest.

```python
class TextProcessor(ABC):
    @abstractmethod
    def process(self, text: str) -> str:
        pass

class PlainText(TextProcessor):
    def process(self, text: str) -> str:
        return text

class UpperCaseDecorator(TextProcessor):
    def __init__(self, processor: TextProcessor):
        self._processor = processor
    
    def process(self, text: str) -> str:
        return self._processor.process(text).upper()

class TrimDecorator(TextProcessor):
    def __init__(self, processor: TextProcessor):
        self._processor = processor
    
    def process(self, text: str) -> str:
        return self._processor.process(text).strip()

# Stack decorators to compose behavior at runtime
processor = TrimDecorator(UpperCaseDecorator(PlainText()))
result = processor.process("  hello world  ")  # "HELLO WORLD"
```

**When to reach for it:**
- You want to mix and match behaviors without a subclass explosion
- Behaviors should be stackable or composable
- You want to add cross-cutting concerns (logging, caching, auth) without touching core logic

**Classic interview appearances:** Logger with multiple output sinks (file + console), cache that wraps a slow data source, auth middleware that wraps a handler, retry logic around an external API client.

**The key insight:** Each decorator doesn't know or care what it's wrapping — it just adds one layer of behavior. This makes combinations infinitely composable without code changes.

---

### Facade Pattern

**Problem it solves:** A subsystem has multiple classes and methods, but clients only need a simple, unified entry point for common operations. The Facade hides the complexity.

```python
class HomeTheaterFacade:
    def __init__(self):
        self._projector = Projector()
        self._speakers = Speakers()
        self._dvd_player = DVDPlayer()
        self._lights = SmartLights()
    
    def watch_movie(self, movie: str):
        self._lights.dim(30)
        self._projector.on()
        self._projector.set_input("HDMI")
        self._speakers.set_volume(70)
        self._dvd_player.insert(movie)
        self._dvd_player.play()
    
    def end_movie(self):
        self._dvd_player.stop()
        self._projector.off()
        self._speakers.mute()
        self._lights.full()
```

**When to reach for it:**
- Hiding a complex or messy internal API behind a simple external API
- Orchestrating multiple subsystems for a common workflow
- You're building a library and want a simple entry point for the most common use cases

**Classic interview appearances:** The main `Game` class in game designs (hides Board, Player, RuleEngine complexity), `OrderService` (hides payment, inventory, shipping subsystems), `NotificationService` (hides email, SMS, push subsystems).

---

## Behavioral Patterns

### Strategy Pattern

**Problem it solves:** An algorithm's behavior varies, and you want to switch between variants at runtime without if/else chains.

**Structure:** Define an interface for the algorithm. Each variant is a separate class that implements it. The context is given the strategy from outside (injected) rather than picking it via if/else.

```python
class SortStrategy(ABC):
    @abstractmethod
    def sort(self, data: list) -> list:
        pass

class QuickSort(SortStrategy):
    def sort(self, data: list) -> list:
        # quicksort implementation
        return sorted(data)

class MergeSort(SortStrategy):
    def sort(self, data: list) -> list:
        # mergesort implementation
        return sorted(data, key=lambda x: x)

class DataProcessor:
    def __init__(self, strategy: SortStrategy):
        self._strategy = strategy
    
    def process(self, data: list) -> list:
        return self._strategy.sort(data)

# Strategy selected at construction, swappable at runtime
processor = DataProcessor(QuickSort())
result = processor.process([3, 1, 4, 1, 5])
```

**When to reach for it:**
- Multiple algorithms for the same operation (sort, payment, discount, validation)
- Behavior that should be swappable at runtime or configurable from outside
- You're writing `if type == A: do this; elif type == B: do that` anywhere in your code

**Classic interview appearances:** Discount strategies (VIP, Premium, Default), payment processors (Stripe, PayPal, Crypto), elevator scheduling algorithms (SCAN, LOOK, SSTF), rate limiting algorithms (token bucket, sliding window).

---

### Observer Pattern

**Problem it solves:** When one object changes state, other objects need to be notified without the first object knowing about all of them. You want to decouple event producers from event consumers.

**Structure:** A Subject holds a list of subscribers. When its state changes, it calls a method on each subscriber. Subscribers implement a common interface.

```python
class EventEmitter:
    def __init__(self):
        self._subscribers: dict[str, list[Callable]] = {}
    
    def subscribe(self, event: str, callback: Callable):
        self._subscribers.setdefault(event, []).append(callback)
    
    def unsubscribe(self, event: str, callback: Callable):
        if event in self._subscribers:
            self._subscribers[event].remove(callback)
    
    def emit(self, event: str, data=None):
        for callback in self._subscribers.get(event, []):
            callback(data)

# Game using observer to notify UI and logger without knowing about them
class ChessGame:
    def __init__(self):
        self._emitter = EventEmitter()
    
    def on(self, event: str, callback: Callable):
        self._emitter.subscribe(event, callback)
    
    def make_move(self, move):
        # ... game logic ...
        self._emitter.emit("move_made", move)

game = ChessGame()
game.on("move_made", lambda m: ui.render(m))
game.on("move_made", lambda m: logger.log(m))
```

**When to reach for it:**
- Multiple components need to react to the same event
- You want the event producer to not know about its consumers
- You have a publish/subscribe relationship between components

**Classic interview appearances:** Inventory system notifying multiple services on stock change, chat room broadcasting messages to all participants, game events (move made, game over) driving UI updates and logging.

---

### State Machine Pattern

**Problem it solves:** An object's behavior changes dramatically based on its current state, and transitions between states follow defined rules. Without a state machine, you end up with complex nested if/else checking the current state everywhere.

**Structure:** Each state is a class that implements a common interface. The context object delegates behavior to the current state object, which handles transitions.

```python
from enum import Enum

class ElevatorState(Enum):
    IDLE = "idle"
    MOVING_UP = "moving_up"
    MOVING_DOWN = "moving_down"
    DOORS_OPEN = "doors_open"

class Elevator:
    def __init__(self):
        self._state = ElevatorState.IDLE
        self._floor = 0
    
    def request_floor(self, floor: int):
        if self._state == ElevatorState.DOORS_OPEN:
            raise Exception("Can't move while doors are open")
        if floor > self._floor:
            self._state = ElevatorState.MOVING_UP
        elif floor < self._floor:
            self._state = ElevatorState.MOVING_DOWN
        # ... move to floor, then open doors ...
    
    def open_doors(self):
        if self._state not in (ElevatorState.IDLE, ElevatorState.MOVING_UP, ElevatorState.MOVING_DOWN):
            raise Exception("Invalid transition")
        self._state = ElevatorState.DOORS_OPEN
```

**When to reach for it:**
- Object has well-defined states with rules about which transitions are allowed
- Behavior at a method call depends heavily on current state
- You want to make invalid transitions impossible by design

**Classic interview appearances:** Elevator (idle → moving → doors open → idle), vending machine (idle → item selected → money inserted → dispensing), order lifecycle (placed → paid → shipped → delivered), game state (IN_PROGRESS → WON → DRAW).

**The simplest form:** An enum `GameState {IN_PROGRESS, WON, DRAW}` plus checks in each method is already a state machine — no complex class hierarchy required.

---

## How to Choose a Pattern

| If you see this... | Reach for... |
|-------------------|-------------|
| Object creation varies by type | Factory |
| Many optional constructor parameters | Builder |
| Need exactly one global instance | Singleton (carefully) |
| Add behavior without subclassing | Decorator |
| Simplify a complex subsystem | Facade |
| Algorithm varies at runtime | Strategy |
| One change triggers many updates | Observer |
| Behavior changes by current state | State Machine |

**Default posture in interviews:** Start without patterns. When you notice an if/else checking type, add Strategy. When you need composable behaviors, add Decorator. When you have a complex subsystem, a Facade emerges naturally from your orchestrator class. Let patterns emerge from the design problem, don't impose them upfront.

---

## Common Interview Mistakes

| Mistake | Fix |
|---------|-----|
| Using Singleton for everything shared | Use dependency injection; pass the shared object to constructors |
| Adding a Factory when there's only one concrete type | Just call the constructor |
| Four layers of Decorators when one wrapper would do | Start simpler; add layers only when you need composability |
| Strategy pattern when there's only one algorithm | YAGNI — an enum or a simple method works |
| Observer when there's only one listener | Direct method call is simpler |

---

## Practice Questions

1. A vending machine must handle coins, item selection, dispensing, and refunds. Which pattern helps you model the valid operations in each state? Sketch the states and transitions.
2. You're building a file exporter that outputs CSV, JSON, or PDF based on user preference. Which pattern do you use, and why is it better than `if format == "csv":` inside the export method?
3. You want to add request logging and response caching to an existing `APIClient` class without modifying it. Which pattern applies? Draw the class hierarchy.
4. A `NotificationService` needs to send a message via email, SMS, and push simultaneously whenever an order ships. Which pattern helps you avoid the service knowing about all three channels?
5. You need to build a query object that accepts table, conditions, ordering, limit, and offset — some optional. Why would a Builder be better than a 6-argument constructor?

## One-Line Summary

Factory centralizes type-based object creation, Builder handles complex optional configs, Singleton provides a single global instance (use rarely), Decorator stacks behavior without subclassing, Facade simplifies subsystems, Strategy replaces type-checking with polymorphism, Observer decouples event emitters from their listeners, and State Machine encapsulates state-specific behavior and transitions.
