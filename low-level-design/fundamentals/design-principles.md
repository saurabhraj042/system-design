# Design Principles

## The Core Idea

Design principles are rules of thumb that guide class and interface decisions. They're not laws — they're heuristics for writing code that's easy to reason about, change, and extend over time. Most principles address the same underlying problem from different angles: **how do you isolate the impact of change?**

The answer is always some form of: keep related things together and unrelated things apart. Every principle below is a specific application of that idea.

---

## General Principles

### KISS (Keep It Simple)

Start with the simplest solution that satisfies the requirements. Not the most general one, not the most extensible one — the simplest one. Complexity is a cost that must be justified by a corresponding benefit.

**How it shows up in interviews:**
- Reach for an enum before a strategy pattern
- Use a list before a priority queue if ordering isn't required
- Write the 3-line method before abstracting into a helper hierarchy

**Why it's easy to violate:** Clever code is satisfying to write. But code is read far more often than it's written. The person reading it (including future you) should be able to understand it in one pass.

---

### DRY (Don't Repeat Yourself)

When two pieces of code capture the same concept, a change to that concept should only need to happen in one place. Duplication means changes must be tracked down and applied everywhere — and humans reliably miss at least one.

**The subtlety:** DRY is about conceptual duplication, not textual similarity. Two pieces of code that look similar but represent different concerns should stay separate. Merging them creates an artificial coupling: changing one concern now requires you to untangle it from the other.

**Test:** Would a future change to this concept naturally change both occurrences, or might one of them actually be expressing something different?

---

### YAGNI (You Aren't Gonna Need It)

Build for what the requirements say now, not for what you imagine the requirements might say someday. Unused abstractions and extension points add complexity without adding value.

**In interviews:** This is especially important for LLD. Don't add a StrategyFactory when there's only one strategy. Don't add a plugin architecture when the interviewer didn't ask for extensibility. Build what was asked; structure the design so extensions could be added cleanly if needed.

**What it doesn't mean:** Don't confuse YAGNI with technical debt. Using a proper interface rather than a concrete class doesn't violate YAGNI — it costs almost nothing and makes your existing code easier to test. YAGNI warns against premature abstraction layers and features nobody asked for.

---

### Separation of Concerns

Different responsibilities belong in different places. The three concerns that almost always want separation:

- **Presentation layer** — what the user sees (UI, HTML, CLI output)
- **Business logic layer** — rules and decisions (validation, computation, state transitions)
- **Data layer** — storage and retrieval (database access, file I/O)

**Why it matters:** When presentation is mixed with business logic, you can't unit test the logic without rendering a UI. When business logic is mixed with data access, you can't change your database without modifying business rules. Separation allows each concern to evolve independently.

**What it looks like in practice:** A class like `OrderProcessor` should contain order validation, discount rules, and state transitions. It should call a `PaymentGateway` interface and an `OrderRepository` interface — but should not know whether the repository is PostgreSQL or DynamoDB.

---

### Law of Demeter (Don't Talk to Strangers)

An object should only communicate with its immediate collaborators — things it was directly handed. It shouldn't navigate through a chain of objects to get what it needs.

**Red flag:** Method chains like `order.getCustomer().getAddress().getCity()`.

**Why this matters:** Each `.` in that chain is a dependency on an intermediate object's internals. If `Address` restructures how it stores city, you now need to change code in `OrderProcessor` even though `OrderProcessor` doesn't work with addresses. The change radiates outward because you've coupled across three layers.

**The fix:** Ask for what you actually need. If `OrderProcessor` needs the city, give it a method `order.getShippingCity()` that extracts it internally. Or inject the city directly. Keep the navigation inside the class that knows the structure.

---

## SOLID Principles

### S — Single Responsibility Principle (SRP)

A class should have only one reason to change.

"Reason to change" means "who or what would cause this class to be modified?" If two different stakeholders (a UI team and a billing team) would each independently want to modify the same class, that class has more than one responsibility.

**Classic violation:** A `User` class that handles login logic, sends emails, and formats user data for export. Three different teams touch it. When the email format changes, you modify a class that also contains authentication code — and potentially break authentication while editing email templates.

**The fix:** `UserAuthenticator`, `UserNotificationService`, `UserExportFormatter` — each with one reason to change.

---

### O — Open/Closed Principle (OCP)

Software entities should be open for extension but closed for modification.

When new behavior is needed, you should be able to add it by writing new code, not by modifying existing code. Modifying existing code risks breaking everything that already works.

**How it works in practice:** Design around abstractions (interfaces). When a new behavior is required, add a new class that implements the interface. The calling code doesn't change — it was already written to the interface, not the concrete class.

**Example:** A discount calculator that uses `if (user.isPremium)` and `if (user.isVIP)` requires modification every time a new user tier is added. A `DiscountStrategy` interface with `PremiumDiscount`, `VIPDiscount`, and `DefaultDiscount` implementations lets you add a `LoyaltyDiscount` without touching existing code.

---

### L — Liskov Substitution Principle (LSP)

If S is a subtype of T, then objects of type T may be replaced with objects of type S without altering any of the desirable properties of the program.

More practically: subclasses should work wherever the parent class works, without surprising the callers.

**Classic violation:** A `Square` class that extends `Rectangle`. A `Rectangle` lets you set width and height independently. A `Square` must keep them equal. Code written for `Rectangle` that sets `width = 10, height = 20` will break when it receives a `Square` — the area will be 400, not 200. The `Square` subtype can't be substituted for a `Rectangle` without changing behavior.

**The test:** Could you swap in the subclass without the caller knowing? If any caller would need to special-case the subclass, LSP is violated.

---

### I — Interface Segregation Principle (ISP)

Clients should not be forced to depend on interfaces they don't use.

A fat interface — one with many unrelated methods — forces every implementor to implement methods they don't need (often as empty stubs or throwing exceptions). This creates noise and false coupling.

**Example:** An `Animal` interface with `eat()`, `sleep()`, `fly()`, and `swim()`. A `Dog` can eat and sleep but not fly. It has to implement `fly()` anyway, probably by throwing `UnsupportedOperationException`. Now callers of `Animal` have to know that some animals can't fly.

**The fix:** Split into `FlyingAnimal`, `SwimmingAnimal`, and `LandAnimal` interfaces. Classes implement only what they support. Callers only depend on the capabilities they use.

---

### D — Dependency Inversion Principle (DIP)

High-level modules should not depend on low-level modules. Both should depend on abstractions.

**The direction of code:** In typical code, business logic (`OrderProcessor`) calls a concrete implementation (`PostgresOrderRepository`). The business logic now knows about PostgreSQL — changing the database requires modifying business logic.

**Inverted:** `OrderProcessor` depends on an `OrderRepository` interface. `PostgresOrderRepository` and `DynamoOrderRepository` both implement it. The business logic is isolated from storage technology. You can swap implementations without touching business rules.

**Injection in practice:** The interface is typically injected through the constructor rather than created inside the class. `OrderProcessor` receives its `OrderRepository` from outside — the caller decides which implementation to use. This is called dependency injection, and it's how DIP manifests in most real code.

---

## How Principles Interact

The principles aren't independent — they reinforce each other and often appear together in well-designed code:

- **SRP** and **DIP** together: each class has one job, and it communicates with collaborators through interfaces rather than concrete classes
- **OCP** and **Strategy pattern:** new strategies implement the existing interface; old code never changes
- **ISP** and **DIP** together: you inject small, focused interfaces — not fat ones that carry methods you don't need
- **LSP** and **Inheritance:** every time you use inheritance, ask whether the subclass is truly substitutable for the parent, or whether you should use composition instead

In most LLD interviews, you won't name these principles explicitly unless asked. But interviewers will notice when you break them — the most common signals are: adding if/else chains to check type, modifying an existing class to add a new feature rather than adding a new class, and coupling business logic to concrete implementations.

---

## Common Interview Mistakes

| Mistake | Principle Violated | Fix |
|---------|-------------------|-----|
| Big if/else chain on object type | OCP, Polymorphism | Strategy or Factory pattern |
| Business class that also persists itself | SRP | Separate repository class |
| Class that creates its own dependencies | DIP | Constructor injection |
| Interface with methods some implementors can't support | ISP | Split into smaller interfaces |
| Subclass that throws NotImplemented for parent methods | LSP | Reconsider the inheritance |
| Caching logic mixed into domain logic | SRP, Separation of Concerns | Decorator pattern or separate cache layer |

---

## Practice Questions

1. You're designing a report generator. One class handles fetching data from the database, formatting it as HTML or CSV, and emailing it to recipients. Which principle is violated and how would you fix it?
2. You have a `Shape` class with an `area()` method and a function `printAreas(shapes)` that loops through shapes and prints each area. You need to add a new shape type. Which principle guides you to add a new class rather than modify `printAreas`? What does that design look like?
3. A `Bird` interface has methods `fly()`, `walk()`, `eat()`, and `sing()`. You need to implement a `Penguin` class. Which two SOLID principles does this design violate?
4. Your `OrderService` class creates a `new PostgresOrderRepository()` inside its constructor. Why does this violate DIP? What would the dependency-inverted version look like?
5. Two classes, `PremiumUserEmailSender` and `FreeUserEmailSender`, contain nearly identical email templating logic but different pricing copy. Applying DRY would merge them — but should you? What question do you ask to decide?

## One-Line Summary

Design principles are heuristics for isolating the impact of change: keep each class focused on one concern (SRP), extend by adding new code not modifying old code (OCP), ensure subclasses truly substitute for parents (LSP), split fat interfaces into focused ones (ISP), and depend on abstractions injected from outside (DIP).
