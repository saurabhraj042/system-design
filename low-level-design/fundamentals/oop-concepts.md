# OOP Concepts

## The Core Idea

Object-oriented programming is built on four concepts: encapsulation, abstraction, polymorphism, and inheritance. These aren't just vocabulary — they're design tools. Each one solves a specific class of problem. Knowing when to reach for each one, and when not to, is what separates a candidate who understands OOP from one who just memorized the definitions.

---

## Encapsulation

### What It Is

Encapsulation means keeping an object's internal data private and exposing only behavior through public methods. The object owns its state and is the only one allowed to modify it.

```python
# Without encapsulation — external code controls the object's state
class BankAccount:
    balance = 0  # public

account = BankAccount()
account.balance = -500  # no validation, no control

# With encapsulation — the object controls its own invariants
class BankAccount:
    def __init__(self):
        self._balance = 0
    
    def deposit(self, amount):
        if amount <= 0:
            raise ValueError("Deposit must be positive")
        self._balance += amount
    
    def get_balance(self):
        return self._balance
```

### Why It Matters

Encapsulation enforces invariants. The `BankAccount` knows that balance can never go negative (if you add the right checks). External code can't reach in and set it to -500.

This is sometimes called "Tell, Don't Ask." Instead of asking an object for its data and then computing something outside it, tell the object to do the work itself. The logic that depends on the state stays with the state.

**Example of Tell vs. Ask:**
```python
# Ask style — external code queries state and makes decisions
if account.get_balance() >= amount:
    account.set_balance(account.get_balance() - amount)  # BAD

# Tell style — behavior belongs with the data
account.withdraw(amount)  # account decides if it can
```

### Collections

Return copies or unmodifiable views of collections, not the internal reference:

```python
# BAD: caller can modify your internal list
def get_items(self):
    return self._items

# GOOD: return a copy so callers can't corrupt your state
def get_items(self):
    return list(self._items)
```

---

## Abstraction

### What It Is

Abstraction means hiding implementation details behind an interface. The caller knows *what* an object can do, not *how* it does it.

```python
from abc import ABC, abstractmethod

class PaymentProcessor(ABC):
    @abstractmethod
    def process_payment(self, amount: float, user_id: str) -> bool:
        pass

class StripeProcessor(PaymentProcessor):
    def process_payment(self, amount, user_id):
        # Stripe-specific API calls...
        return True

class PayPalProcessor(PaymentProcessor):
    def process_payment(self, amount, user_id):
        # PayPal-specific API calls...
        return True
```

`OrderService` calls `PaymentProcessor.process_payment()`. It doesn't know and doesn't care whether the underlying implementation is Stripe, PayPal, or a mock in tests.

### The Right Level of Abstraction

The interface should expose what the caller actually needs, at the level of the concept the caller is thinking about — not at the level of the implementation.

A caller that wants to "process a payment" should see `process_payment(amount, user_id)`. If the interface forced callers to pass a `StripeChargeRequest` object, the abstraction is leaking implementation details. If it exposed internal batch parameters, it's too detailed.

**Right level:** What does the caller conceptually want to do?

### Abstraction vs. Encapsulation

These are related but distinct. Encapsulation is about hiding state. Abstraction is about hiding implementation. Together, they prevent callers from depending on details that might change.

---

## Polymorphism

### What It Is

Polymorphism means many implementations behind one interface. Write code against the interface once; each concrete type handles itself.

```python
# Without polymorphism — type-checking everywhere
def calculate_area(shape):
    if shape.type == "circle":
        return math.pi * shape.radius ** 2
    elif shape.type == "rectangle":
        return shape.width * shape.height
    # Every new shape requires modifying this function

# With polymorphism — each shape handles itself
class Shape(ABC):
    @abstractmethod
    def area(self) -> float:
        pass

class Circle(Shape):
    def __init__(self, radius):
        self.radius = radius
    def area(self):
        return math.pi * self.radius ** 2

class Rectangle(Shape):
    def __init__(self, width, height):
        self.width = width
        self.height = height
    def area(self):
        return self.width * self.height

# Client code never changes when you add new shapes
def print_areas(shapes: list[Shape]):
    for shape in shapes:
        print(shape.area())
```

### When to Use It

Reach for polymorphism when:
- Behavior varies by type
- You want to add new types without modifying existing code
- You're currently writing `if isinstance(x, Foo)` or `switch on type`

### The Trade-off

Polymorphism makes code easier to extend but harder to trace. With an explicit if/else, you can read the whole decision tree in one place. With polymorphism, following the code requires looking up which implementation is active at runtime. For a codebase with 20 implementations of `PaymentProcessor`, finding where a payment actually goes requires knowing which processor was injected.

This is usually worth it — but it's a real cost, not zero.

---

## Inheritance

### What It Is

Inheritance lets one class derive state and behavior from a parent class. The child gets everything the parent has and can add or override.

```python
class Animal:
    def __init__(self, name):
        self.name = name
    
    def breathe(self):
        pass  # all animals breathe
    
    def eat(self, food):
        pass  # all animals eat

class Dog(Animal):
    def bark(self):
        return "Woof"
```

### When It's Appropriate

Inheritance makes sense when the child is genuinely a specialized form of the parent — when it satisfies LSP (could replace the parent without surprises). The classic guidance is the "is-a" test: a `Dog` is an `Animal`. A `SavingsAccount` is a `BankAccount`.

**When inheritance is stable and correct:**
- The parent class contains logic all subclasses genuinely share
- The hierarchy is shallow (2-3 levels max)
- The parent's interface doesn't change frequently

### The Fragile Base Class Problem

When you change a parent class, every child class can be affected. If the parent changes a method's semantics — not its signature, just what it does — all children that relied on the old behavior break. This is the "fragile base class" problem.

At scale, this means changes to a base class require auditing every subclass. Teams working on different subclasses may not even know about each other. The coupling is invisible until something breaks.

### Prefer Composition

In most LLD interview contexts, you won't need inheritance beyond implementing interfaces. If you find yourself reaching for class inheritance, consider composition instead:

```python
# Inheritance: BirdOfPrey "is a" Bird
class BirdOfPrey(Bird):
    pass

# Composition: BirdOfPrey "has a" HuntingBehavior
class BirdOfPrey:
    def __init__(self):
        self.hunting = HuntingBehavior()
        self.flight = FlightBehavior()
```

Composition is more flexible. You can mix and match behaviors at runtime. You can change the `HuntingBehavior` without affecting anything else. The class isn't locked into the constraints of its parent.

**Rule of thumb for LLD interviews:** Implement interfaces, avoid inheriting from concrete classes.

---

## How the Four Concepts Work Together

Think of building an e-commerce order system:

| Concept | How it applies |
|---------|---------------|
| Encapsulation | `Order` owns its item list; external code calls `addItem()` not `order.items.append()` |
| Abstraction | `PaymentProcessor` interface hides whether it's Stripe or PayPal |
| Polymorphism | `order.total()` delegates to each `LineItem.subtotal()` without type-checking |
| Inheritance | `DigitalOrder` and `PhysicalOrder` are separate implementations of an `Order` interface — or two classes that both use composition with a `FullfillmentStrategy` |

In practice:
- **Encapsulation** is always present — every class should protect its state
- **Abstraction** appears wherever you communicate between layers
- **Polymorphism** appears wherever behavior varies by type
- **Inheritance** appears rarely — only for stable, genuinely shared implementation

---

## Common Interview Mistakes

| Mistake | What It Signals | Fix |
|---------|----------------|-----|
| Making all fields public for convenience | No encapsulation | Add methods; validate before mutating |
| Returning `self._items` directly | Missing data protection | Return a copy or unmodifiable view |
| `isinstance()` checks in client code | Missing polymorphism | Add a method to the interface that each type implements |
| Inheritance for code reuse across unrelated classes | Misusing inheritance | Use composition or move shared logic to a helper |
| Building a 5-level deep inheritance hierarchy | Fragile base class | Flatten to interfaces + composition |
| Creating a method on a parent class that some children override to throw | LSP violation | Remove the method from the base; use composition |

---

## Practice Questions

1. You have a `Game` class with `private List<Player> players`. An interviewer asks "how do you ensure nobody modifies this list from outside the class?" What do you return from `getPlayers()` and why?
2. Your `ParkingLot` class has `if vehicle instanceof Motorcycle` and `if vehicle instanceof Car` scattered through several methods. Which OOP concept eliminates these checks, and what does the refactored design look like?
3. You've designed a `Rectangle` class that `Square` extends. The `Rectangle` has `setWidth()` and `setHeight()` methods. Explain why `Square.setWidth()` violates LSP and what you'd do instead.
4. When should you prefer composition over inheritance for adding a new behavior? Give an example from a parking lot system.
5. Design the class structure for a notification service that can send emails, SMS, and push notifications. Show how encapsulation, abstraction, and polymorphism each appear in the design.

## One-Line Summary

Encapsulation protects state, abstraction hides implementation, polymorphism dispatches by type without type-checking, and inheritance (used sparingly) shares stable implementation — together they isolate change and enable clean extension.
