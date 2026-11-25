# LLD Notes: SOLID Principles

> Part of the [LLD Notes](index.md) series.

---

## 1. OOD Foundations

SOLID and design patterns build on a few basic ideas. Knowing them makes the rest of these notes much easier to follow.

### 1.1 The Four Pillars of OOP

| Pillar | Meaning | Example |
|---|---|---|
| **Encapsulation** | Bundle data with the methods that operate on it, and hide internal state behind a public interface | `BankAccount` keeps `balance` private; you can only change it through `deposit()` and `withdraw()`, which enforce the rules |
| **Abstraction** | Expose *what* an object does, hide *how* | You call `paymentGateway.charge(amount)` without knowing about HTTP calls, retries, or signatures |
| **Inheritance** | A class acquires the fields and behavior of a parent class ("is-a") | `SavingsAccount extends BankAccount` |
| **Polymorphism** | One interface, many implementations; the right behavior is chosen at runtime | `List<Shape>` where `shape.area()` runs the circle or square version automatically |

Polymorphism is the engine behind almost every design pattern: code written against an interface works with any implementation, including ones written later.

### 1.2 Coupling and Cohesion

These two measures describe the quality of a design better than any rule.

- **Cohesion** = how closely related the responsibilities *inside* one module are. **High cohesion is good**: a `PasswordHasher` that only hashes and verifies passwords is cohesive; a `Utils` class with date formatting, email sending, and PDF generation is not.
- **Coupling** = how much one module depends on the internals of another. **Low coupling is good**: if changing class A forces changes in B, C, and D, they're tightly coupled.

**Goal: high cohesion, low coupling.** Each SOLID principle is a specific technique for moving toward that goal.

### 1.3 Composition over Inheritance

Inheritance is the tightest form of coupling in OOP: a subclass depends on its parent's implementation details, and a change in the parent can silently break every child (the "fragile base class" problem). Inheritance hierarchies also explode when behaviors combine: `FlyingSwimmingDuck`, `FlyingNonSwimmingDuck`, ...

**Composition** means an object *has* collaborators that provide behavior, instead of *being* a subtype that inherits it:

```java
// Inheritance: behavior fixed at compile time
class RubberDuck extends Duck { /* override fly() to do nothing?? */ }

// Composition: behavior plugged in, swappable at runtime
class Duck {
    private FlyBehavior flyBehavior;
    Duck(FlyBehavior flyBehavior) { this.flyBehavior = flyBehavior; }
    void performFly() { flyBehavior.fly(); }
}
```

Use inheritance for genuine "is-a" relationships where the subtype honors the parent's full contract (see LSP). Use composition for "has-a" relationships and for combining or swapping behaviors. Strategy, Decorator, State, Composite, and many other patterns are all forms of composition.

### 1.4 Program to an Interface, Not an Implementation

```java
ArrayList<String> names = new ArrayList<>();   // tied to one implementation
List<String> names = new ArrayList<>();        // depends only on the List contract
```

Depending on abstractions means implementations can change without affecting callers. This idea underlies Open/Closed and Dependency Inversion.

---

## 2. SOLID Principles

SOLID is a set of five principles collected by Robert C. Martin ("Uncle Bob") in the early 2000s (the acronym was coined by Michael Feathers). Their shared goal is **code that is easy to change**: when requirements change, you should be able to make the change in one place, without breaking unrelated features.

| Letter | Principle | One-sentence summary |
|---|---|---|
| **S** | Single Responsibility | A class should have only one reason to change |
| **O** | Open/Closed | Open for extension, closed for modification |
| **L** | Liskov Substitution | Subtypes must be usable anywhere their base type is expected |
| **I** | Interface Segregation | Clients shouldn't depend on methods they don't use |
| **D** | Dependency Inversion | Depend on abstractions, not concrete implementations |

These are **principles, not laws.** Applying them everywhere produces over-engineered code full of tiny interfaces nobody needed. Apply them where change is likely or already happening.

---

### 2.1 S: Single Responsibility Principle (SRP)

#### Definition

> **A class should have one, and only one, reason to change.**

A more precise modern phrasing from Martin: *a module should be responsible to one, and only one, actor*, where an actor is a group of people (a stakeholder or role) who might request changes.

"One responsibility" doesn't mean "one method." It means the class's methods all serve the same purpose and change for the same reason.

#### The problem

```java
// BAD: one class, many reasons to change
class UserService {
    public void register(String name, String email) {
        // 1. Validation rules
        if (!email.contains("@")) throw new IllegalArgumentException("Invalid email");

        // 2. Persistence (SQL details)
        Connection conn = DriverManager.getConnection("jdbc:postgresql://...");
        conn.prepareStatement("INSERT INTO users ...").executeUpdate();

        // 3. Logging format and destination
        System.out.println("[" + LocalDateTime.now() + "] Registered " + email);

        // 4. Email content and SMTP details
        SmtpClient smtp = new SmtpClient("smtp.example.com", 587);
        smtp.send(email, "Welcome!", "Hi " + name + ", thanks for joining...");
    }
}
```

This class changes when:
- The **validation** rules change (product team).
- The **database** changes, e.g. moving from Postgres to DynamoDB (backend/infra team).
- The **logging** format or destination changes (ops team).
- The **welcome email** wording or provider changes (marketing team).

Consequences:
- **Hard to test:** to test validation, you need a database and an SMTP server (or complicated mocking).
- **Risky changes:** editing the email template means touching the class that saves users; a typo can break registration.
- **Merge conflicts:** four teams edit the same file.
- **No reuse:** another feature that needs to send emails can't reuse this logic without dragging the rest along.

#### The fix: separate responsibilities

```java
class UserValidator {
    void validate(User user) {
        if (!user.getEmail().contains("@")) throw new IllegalArgumentException("Invalid email");
    }
}

class UserRepository {
    void save(User user) { /* database code only */ }
}

class EmailService {
    void sendWelcomeEmail(User user) { /* email code only */ }
}

class Logger {
    void info(String message) { /* logging code only */ }
}

// Orchestrates the steps; its single responsibility is the registration workflow
class UserRegistrationService {
    private final UserValidator validator;
    private final UserRepository repository;
    private final EmailService emailService;
    private final Logger logger;

    UserRegistrationService(UserValidator v, UserRepository r, EmailService e, Logger l) {
        this.validator = v; this.repository = r; this.emailService = e; this.logger = l;
    }

    public void register(User user) {
        validator.validate(user);
        repository.save(user);
        emailService.sendWelcomeEmail(user);
        logger.info("Registered " + user.getEmail());
    }
}
```

Now each class has one reason to change. Switching email providers touches only `EmailService`. Each class can be tested alone, and `UserRegistrationService` can be tested with fakes for its collaborators.

Note that `UserRegistrationService` still *coordinates* several things. That's fine: coordinating the workflow is its one responsibility. SRP doesn't forbid classes that use other classes.

#### How to spot SRP violations

- The class name contains "And," "Manager," "Processor," "Utils," or "Helper" and does unrelated things.
- You struggle to describe the class in one sentence without using "and."
- The class imports from many unrelated areas (database, HTTP, email, file system, UI).
- Different groups of methods use different subsets of the fields (low cohesion).
- Unrelated changes keep landing in the same file.
- The class is very long (a symptom, not proof).

#### Benefits

- Easier to understand: small, focused classes.
- Easier to test: fewer dependencies per class.
- Safer changes: changes are localized.
- Better reuse: focused classes can be used elsewhere.

#### Pitfalls

- **Over-splitting:** one class per method creates a maze of trivial classes, and understanding a feature means jumping across 20 files. Group things that change together.
- "Responsibility" is about **reasons to change**, not about counting methods. A `Money` class with `add`, `subtract`, `multiply`, `format` has one responsibility (representing money).

---

### 2.2 O: Open/Closed Principle (OCP)

#### Definition

> **Software entities (classes, modules, functions) should be open for extension but closed for modification.** (Bertrand Meyer, 1988)

- **Open for extension:** you can add new behavior.
- **Closed for modification:** you add it *without editing existing, tested code*.

How is that possible? Through **abstraction and polymorphism**: existing code depends on an interface, and new behavior arrives as a new implementation of that interface.

#### The problem

```java
// BAD: every new payment method means editing this class
class PaymentProcessor {
    public void pay(String method, double amount) {
        if (method.equals("CREDIT_CARD")) {
            // card logic
        } else if (method.equals("PAYPAL")) {
            // PayPal logic
        } else if (method.equals("UPI")) {
            // UPI logic
        }
        // Adding Apple Pay → modify this method again
    }
}
```

Problems:
- Each addition edits a class that already works, risking regressions in existing payment methods.
- The method grows indefinitely; every branch must be retested.
- The same `if/else` or `switch` on type often appears in several places (fees, refunds, receipts), and all must be updated together. Missing one is a bug.

#### The fix: extend through new classes

```java
interface PaymentMethod {
    void pay(double amount);
}

class CreditCardPayment implements PaymentMethod {
    public void pay(double amount) { /* card logic */ }
}

class PayPalPayment implements PaymentMethod {
    public void pay(double amount) { /* PayPal logic */ }
}

class UpiPayment implements PaymentMethod {
    public void pay(double amount) { /* UPI logic */ }
}

class PaymentProcessor {
    public void process(PaymentMethod method, double amount) {
        method.pay(amount);     // never changes when new methods are added
    }
}

// New requirement: Apple Pay. Add a class; touch nothing else.
class ApplePayPayment implements PaymentMethod {
    public void pay(double amount) { /* Apple Pay logic */ }
}
```

#### Another classic example: shapes

```java
// BAD
double totalArea(List<Object> shapes) {
    double total = 0;
    for (Object s : shapes) {
        if (s instanceof Circle c) total += Math.PI * c.r * c.r;
        else if (s instanceof Rectangle r) total += r.w * r.h;
        // add Triangle → edit here
    }
    return total;
}

// GOOD
interface Shape { double area(); }
class Circle implements Shape { double r; public double area() { return Math.PI * r * r; } }
class Rectangle implements Shape { double w, h; public double area() { return w * h; } }

double totalArea(List<Shape> shapes) {
    return shapes.stream().mapToDouble(Shape::area).sum();
}
```

#### Mechanisms for achieving OCP

- **Interfaces + polymorphism** (above).
- **Strategy pattern:** inject the behavior.
- **Decorator pattern:** add behavior by wrapping.
- **Template Method:** subclasses fill in steps of a fixed algorithm.
- **Plugins and registries:** new handlers register themselves (e.g., a map from payment type to implementation, populated by dependency injection).
- **Configuration:** behavior driven by data rather than code changes.

#### Benefits

- New features add code rather than changing it, reducing regression risk.
- Existing tests keep protecting existing behavior.
- Teams can add features in parallel in separate classes.

#### Pitfalls

- **You can't be closed against every change.** Predicting all future variations leads to speculative abstractions that are never used. Martin's advice: when a kind of change happens for the first time, apply the abstraction so the *next* change of that kind is easy ("fool me once").
- Some `switch` statement is usually still needed somewhere, typically in a **factory** that maps a string from a request to the right implementation. The goal is to confine it to one place.
- Bug fixes obviously still modify code; OCP is about adding features.

---

### 2.3 L: Liskov Substitution Principle (LSP)

#### Definition

> **Objects of a superclass should be replaceable with objects of its subclasses without breaking the correctness of the program.** (Barbara Liskov, 1987)

Formally: if S is a subtype of T, then objects of type T may be replaced with objects of type S without altering any of the desirable properties of the program.

In practice: **a subclass must honor the full contract (the promised behavior) of its parent**, not just its method signatures. Code written against the parent must not need to know which subclass it received.

#### The classic example: birds

```java
// BAD
class Bird {
    void fly() { System.out.println("Flying"); }
}

class Sparrow extends Bird { }

class Penguin extends Bird {
    @Override
    void fly() { throw new UnsupportedOperationException("Penguins can't fly"); }
}

void makeBirdsFly(List<Bird> birds) {
    for (Bird b : birds) b.fly();     // crashes when it reaches a Penguin
}
```

`Bird` promised that every bird can fly. `Penguin` breaks that promise, so it can't safely substitute for `Bird`. Callers now need `if (!(b instanceof Penguin))` checks, which also violates OCP. The same problem applies to kiwis, ostriches, and emus.

**Fix: model the hierarchy around actual capabilities.**

```java
abstract class Bird {
    abstract void eat();
}

interface Flyable {
    void fly();
}

class Sparrow extends Bird implements Flyable {
    void eat() { /* ... */ }
    public void fly() { /* ... */ }
}

class Penguin extends Bird {
    void eat() { /* ... */ }
    void swim() { /* ... */ }
}

void makeBirdsFly(List<Flyable> flyers) {
    for (Flyable f : flyers) f.fly();      // only things that can fly are accepted
}
```

Now the type system prevents the mistake: you can't put a `Penguin` in a `List<Flyable>`.

#### The subtle example: Rectangle and Square

Mathematically, a square *is a* rectangle. But in code:

```java
class Rectangle {
    protected int width, height;
    void setWidth(int w)  { width = w; }
    void setHeight(int h) { height = h; }
    int area() { return width * height; }
}

class Square extends Rectangle {
    @Override void setWidth(int w)  { width = w; height = w; }   // keep sides equal
    @Override void setHeight(int h) { width = h; height = h; }
}

void resize(Rectangle r) {
    r.setWidth(5);
    r.setHeight(4);
    assert r.area() == 20;    // fails for Square: area is 16
}
```

`Rectangle`'s implicit contract is "setting the width doesn't change the height." `Square` breaks it. The lesson: **"is-a" in the real world doesn't guarantee behavioral substitutability in code.** Fixes: make shapes immutable (no setters, so the conflicting contract disappears), or don't relate them by inheritance (both implement `Shape`).

#### The contract rules

A subclass satisfies LSP when it follows these rules (from Design by Contract):

| Rule | Meaning | Violation example |
|---|---|---|
| **Preconditions cannot be strengthened** | A subclass can't demand more from callers than the parent did | Parent accepts any amount > 0; subclass throws for amounts over 1,000 |
| **Postconditions cannot be weakened** | A subclass must deliver at least what the parent promised | Parent's `save()` guarantees persistence; subclass silently does nothing |
| **Invariants must be preserved** | Properties always true for the parent must remain true | Square's width/height coupling breaks Rectangle's independence |
| **No new unexpected exceptions** | Don't throw exceptions callers of the parent wouldn't expect | `Penguin.fly()` throwing `UnsupportedOperationException` |
| **History constraint** | A subclass shouldn't allow state changes the parent forbids | An `ImmutablePoint` subclass that adds setters |

Signature rules also apply (enforced by most typed languages): parameter types can be the same or more general (contravariance), and return types the same or more specific (covariance).

#### How to spot LSP violations

- Overridden methods that throw `UnsupportedOperationException` or `NotImplementedException`.
- Overridden methods with empty bodies that silently do nothing where the parent did real work.
- Client code using `instanceof` / type checks to handle particular subclasses specially.
- Subclasses that ignore or reject inputs the parent accepted.
- Tests written for the parent class that fail when run against a subclass.

A real Java example: `Collections.unmodifiableList()` returns a `List` whose `add()` throws `UnsupportedOperationException`. The `List` interface documents these methods as "optional operations" to stay technically correct, but it's widely cited as an LSP compromise.

#### Benefits

- Polymorphism can be trusted: code written against a base type works with any subtype, which is what makes OCP possible.
- Fewer type checks and special cases.

---

### 2.4 I: Interface Segregation Principle (ISP)

#### Definition

> **Clients should not be forced to depend on methods they do not use.**

Prefer several small, focused ("role") interfaces over one large, general-purpose ("fat") interface.

#### The problem

```java
// BAD: a "fat" interface
interface Worker {
    void work();
    void eat();
    void rest();
}

class HumanEmployee implements Worker {
    public void work() { /* ... */ }
    public void eat()  { /* lunch break */ }
    public void rest() { /* sleep */ }
}

class Robot implements Worker {
    public void work() { /* ... */ }
    public void eat()  { throw new UnsupportedOperationException(); }   // meaningless
    public void rest() { /* do nothing? */ }
}
```

Problems:
- `Robot` is **forced** to implement methods that make no sense for it, ending up with dummy or exception-throwing implementations, which also breaks LSP.
- Any code that receives a `Worker` might call `eat()` on a robot.
- **Unnecessary coupling:** if `rest()`'s signature changes, `Robot` must be modified and recompiled even though it doesn't rest. A class that only needs `work()` still depends on the entire interface.

#### The fix: split by role

```java
interface Workable { void work(); }
interface Eatable  { void eat(); }
interface Restable { void rest(); }

class HumanEmployee implements Workable, Eatable, Restable {
    public void work() { /* ... */ }
    public void eat()  { /* ... */ }
    public void rest() { /* ... */ }
}

class Robot implements Workable {
    public void work() { /* ... */ }
}

// Each client depends only on the capability it needs
class ProductionLine {
    void run(List<Workable> workers) { workers.forEach(Workable::work); }
}

class Cafeteria {
    void serveLunch(List<Eatable> eaters) { eaters.forEach(Eatable::eat); }
}
```

#### A more realistic example: printers

```java
// BAD
interface MultiFunctionDevice {
    void print(Document d);
    void scan(Document d);
    void fax(Document d);
}
// A basic printer must implement scan() and fax() with errors.

// GOOD
interface Printer { void print(Document d); }
interface Scanner { void scan(Document d); }
interface Fax     { void fax(Document d); }

class BasicPrinter implements Printer { ... }
class OfficeMachine implements Printer, Scanner, Fax { ... }
```

#### Another common case: repositories

A reporting service that only reads data shouldn't depend on an interface exposing `delete()` and `update()`. Splitting into `ReadRepository` and `WriteRepository` (the idea behind CQRS) means the reporting service can't accidentally modify data, and its tests only need to fake read methods.

#### How to spot ISP violations

- Implementations with empty methods or `throw new UnsupportedOperationException()`.
- Interfaces with many methods where most clients use only a few.
- Mocks in tests that must stub many methods irrelevant to the test.
- Interface names like `IManager` or `IService` that cover everything.

#### Benefits

- Classes implement only what makes sense for them.
- Smaller interfaces are easier to implement, mock, and understand.
- Changes to one capability don't ripple into unrelated clients.

#### Pitfalls

- Going too far, with one interface per method everywhere, fragments the design. Split interfaces based on **how clients actually use them**, not hypothetically.

**SRP vs ISP:** SRP is about a class having one reason to change; ISP is about clients not depending on more than they need. They often lead to similar splits, viewed from different sides (implementer vs consumer).

---

### 2.5 D: Dependency Inversion Principle (DIP)

#### Definition

> 1. **High-level modules should not depend on low-level modules. Both should depend on abstractions.**
> 2. **Abstractions should not depend on details. Details should depend on abstractions.**

- **High-level module:** contains business rules and policy (e.g., "when the user toggles the switch, the device turns on/off"; "an order is placed, then the customer is notified").
- **Low-level module:** implementation details (a specific database, a specific SMS provider, a specific fan model).

#### The problem

```java
// BAD: the high-level Switch is welded to the low-level Fan
class Fan {
    void turnOn()  { System.out.println("Fan on"); }
    void turnOff() { System.out.println("Fan off"); }
}

class Switch {
    private Fan fan = new Fan();        // concrete dependency, created internally
    private boolean on = false;

    void toggle() {
        if (on) fan.turnOff(); else fan.turnOn();
        on = !on;
    }
}
```

Problems:
- The `Switch` can only ever control a `Fan`. Controlling a `Light` means modifying `Switch` (violates OCP) or duplicating it.
- `Switch` can't be tested without a real `Fan`.
- The dependency points from the important policy ("toggling") to a detail ("this particular fan").

#### The fix: both depend on an abstraction

```java
interface Switchable {
    void turnOn();
    void turnOff();
}

class Fan implements Switchable {
    public void turnOn()  { System.out.println("Fan on"); }
    public void turnOff() { System.out.println("Fan off"); }
}

class Light implements Switchable {
    public void turnOn()  { System.out.println("Light on"); }
    public void turnOff() { System.out.println("Light off"); }
}

class Switch {
    private final Switchable device;     // depends on the abstraction
    private boolean on = false;

    Switch(Switchable device) {          // the dependency is injected from outside
        this.device = device;
    }

    void toggle() {
        if (on) device.turnOff(); else device.turnOn();
        on = !on;
    }
}

Switch fanSwitch   = new Switch(new Fan());
Switch lightSwitch = new Switch(new Light());
```

#### Why it's called "inversion"

In a traditional layered design, dependencies follow the call direction: business logic → database code. The high-level module depends on the low-level one.

```
Before:   OrderService ──────────► MySqlOrderRepository

After:    OrderService ──────────► «interface» OrderRepository
                                            ▲
                                            │ implements
                                   MySqlOrderRepository
```

After applying DIP, the low-level module depends on an abstraction that is **owned by the high-level module** (the interface is defined in terms of what the business logic needs). The source-code dependency now points against the call direction: it has been inverted. Business logic no longer knows or cares whether data lives in MySQL, DynamoDB, or memory.

This is the core idea of **Clean Architecture, Hexagonal Architecture (Ports and Adapters), and Onion Architecture**: the domain sits at the center, and infrastructure plugs into it.

#### A realistic example: notifications

```java
// Abstraction defined by the business layer
interface NotificationSender {
    void send(String recipient, String message);
}

// High-level policy
class OrderService {
    private final OrderRepository orders;
    private final NotificationSender notifier;

    OrderService(OrderRepository orders, NotificationSender notifier) {
        this.orders = orders;
        this.notifier = notifier;
    }

    void placeOrder(Order order) {
        orders.save(order);
        notifier.send(order.getCustomerEmail(), "Order " + order.getId() + " confirmed");
    }
}

// Low-level details
class SesEmailSender implements NotificationSender { /* AWS SES */ }
class TwilioSmsSender implements NotificationSender { /* Twilio */ }

// Test double: no network needed
class FakeNotificationSender implements NotificationSender {
    List<String> sent = new ArrayList<>();
    public void send(String r, String m) { sent.add(r + ":" + m); }
}
```

#### DIP vs Dependency Injection vs Inversion of Control

These are related but different:

| Concept | What it is |
|---|---|
| **Dependency Inversion Principle** | A *design principle*: depend on abstractions; let details depend on abstractions owned by the policy |
| **Dependency Injection (DI)** | A *technique*: give an object its dependencies from outside (constructor, setter, or method parameter) instead of having it create them with `new` |
| **Inversion of Control (IoC)** | A broader *idea*: a framework or container controls object creation and the flow of the program, calling your code ("don't call us, we'll call you") |
| **DI container** | A *tool* that automates DI: Spring, Guice, Dagger, .NET's built-in container, NestJS |

```java
// Constructor injection (preferred): dependencies explicit, required, can be final
@Service
class OrderService {
    private final OrderRepository orders;
    OrderService(OrderRepository orders) { this.orders = orders; }   // Spring injects it
}
```

DI is the usual way to *achieve* DIP, but you can use DI with concrete classes (no inversion), and you can follow DIP without a container (wire objects manually in `main`).

#### Benefits

- **Swappable implementations:** change databases, providers, or devices without touching business logic.
- **Testability:** inject fakes or mocks.
- **Parallel development:** teams agree on the interface and work independently.
- **Business rules protected** from infrastructure churn.

#### Pitfalls

- **Interfaces with exactly one implementation forever** add indirection without benefit. Introduce abstractions at real boundaries (I/O, external services, things that vary or need faking in tests), not for every class.
- Abstractions should express what the high-level module needs (`NotificationSender.send(recipient, message)`), not mirror one vendor's API (`TwilioClient.createMessage(...)`); otherwise the detail leaks through.

---

### 2.6 SOLID Together

The principles reinforce each other:

- **SRP** produces small, focused classes.
- **ISP** gives those classes small, focused interfaces.
- **DIP** makes high-level code depend on those interfaces rather than on concrete classes.
- **LSP** guarantees any implementation can be substituted safely.
- Together they enable **OCP**: new behavior arrives as new implementations plugged into existing abstractions.

**Example:** a notification feature built with SOLID.
- `NotificationSender` is a small interface (ISP).
- `EmailSender`, `SmsSender`, `PushSender` each handle one channel (SRP) and all honor the contract "deliver the message or throw `DeliveryException`" (LSP).
- `OrderService` depends on `NotificationSender`, injected (DIP).
- Adding WhatsApp means adding `WhatsAppSender`, with no changes elsewhere (OCP).

**Balance:** SOLID is a means to maintainability, not a goal in itself. Signs of over-application: interfaces for every class, many layers of indirection for simple CRUD, and code where finding the actual behavior requires opening ten files. Apply the principles where code changes often, at boundaries with external systems, and where testing needs isolation. Keep simple things simple (YAGNI: "You Aren't Gonna Need It"; KISS: "Keep It Simple").

---

## Revision Summary

| Principle | Core idea | Classic violation | Fix |
|---|---|---|---|
| **SRP** | One reason to change | `User` class that saves, logs, and emails | Separate `UserRepository`, `Logger`, `EmailService`; an orchestrating service |
| **OCP** | Extend without modifying | `if/else` on payment type that grows with each method | `PaymentMethod` interface, one class per method |
| **LSP** | Subtypes must honor the parent's contract | `Penguin extends Bird` throwing on `fly()`; `Square extends Rectangle` | Model capabilities (`Flyable`); avoid inheritance that breaks invariants |
| **ISP** | No client forced to depend on unused methods | `Robot` implementing `Worker.eat()` | Split into `Workable`, `Eatable`, `Restable` |
| **DIP** | Depend on abstractions; details depend on abstractions | `Switch` creating `new Fan()` internally | `Switch` receives a `Switchable` via constructor injection |

**SOLID is about managing change:** small focused classes (SRP), extension through new code (OCP), trustworthy polymorphism (LSP), lean interfaces (ISP), and business logic independent of details (DIP). They are principles, not laws — apply them where change is likely.
