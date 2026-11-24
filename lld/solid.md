# SOLID Principles and Design Patterns

> Learning-oriented notes on object-oriented design. Each topic explains **what** it is, **the problem it solves**, **how it works** (with code), **when to use it**, and **its tradeoffs**. Code examples are in Java, because its explicit interfaces and classes make the structure easy to see; the ideas apply equally to Python, TypeScript, C#, Go, and others.

---

## Table of Contents

1. [OOD Foundations](#1-ood-foundations)
2. [SOLID Principles](#2-solid-principles)
   - [S: Single Responsibility](#21-s-single-responsibility-principle-srp)
   - [O: Open/Closed](#22-o-openclosed-principle-ocp)
   - [L: Liskov Substitution](#23-l-liskov-substitution-principle-lsp)
   - [I: Interface Segregation](#24-i-interface-segregation-principle-isp)
   - [D: Dependency Inversion](#25-d-dependency-inversion-principle-dip)
3. [Design Patterns Overview](#3-design-patterns-overview)
4. [Creational Patterns](#4-creational-patterns): Singleton, Factory Method, Abstract Factory, Builder
5. [Structural Patterns](#5-structural-patterns): Adapter, Decorator, Facade, Proxy, Composite
6. [Behavioral Patterns](#6-behavioral-patterns): Strategy, Observer, Chain of Responsibility, Template Method, State
7. [Commonly Confused Patterns](#7-commonly-confused-patterns)
8. [How Patterns Relate to SOLID](#8-how-patterns-relate-to-solid)
9. [Revision Summary](#9-revision-summary)

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

## 3. Design Patterns Overview

### 3.1 What a Design Pattern Is

A design pattern is a **reusable, named solution to a commonly recurring design problem** in a particular context. It's not finished code you copy; it's a template for structuring classes and objects, which you adapt to your situation.

The catalog most people refer to comes from *Design Patterns: Elements of Reusable Object-Oriented Software* (1994) by Gamma, Helm, Johnson, and Vlissides, the **"Gang of Four" (GoF)**, which describes 23 patterns.

**Why patterns matter:**
- **Proven solutions:** they encode experience about what works and what goes wrong.
- **Shared vocabulary:** saying "wrap it in a decorator" or "use a strategy" communicates a whole design in a few words.
- **Recognizing them in frameworks** makes libraries easier to understand: Java I/O streams are decorators, Spring AOP uses proxies, servlet filters are a chain of responsibility.

**Caution:** patterns are tools for specific problems. Forcing them where they aren't needed ("pattern fever") adds complexity. Start from the problem, then recognize which pattern fits.

### 3.2 Categories

| Category | Concern | Patterns in these notes |
|---|---|---|
| **Creational** | *How objects are created*: decouple code from concrete classes and complex construction | Singleton, Factory Method, Abstract Factory, Builder |
| **Structural** | *How objects are composed* into larger structures while keeping them flexible | Adapter, Decorator, Facade, Proxy, Composite |
| **Behavioral** | *How objects interact* and divide responsibilities and algorithms | Strategy, Observer, Chain of Responsibility, Template Method, State |

### 3.3 How Each Pattern Is Described Below

- **Intent:** the purpose in one or two sentences.
- **Problem:** the situation that calls for it.
- **Solution and structure:** participants and how they relate.
- **Code example.**
- **Real-world uses** in common libraries and frameworks.
- **When to use / when not to.**
- **Pros and cons.**
- **Related patterns.**

---

## 4. Creational Patterns

### 4.1 Singleton

#### Intent

Ensure a class has **only one instance**, and provide a **global point of access** to it.

#### Problem

Some resources should exist exactly once in a process: a configuration registry loaded at startup, a connection pool, a cache, a hardware interface. Creating multiple instances would waste resources or cause inconsistent state (two caches holding different values for the same key).

#### Solution

1. Make the constructor **private** so no one else can call `new`.
2. Keep the single instance in a **static field**.
3. Provide a **static method** that returns it.

#### Implementations (Java)

**Eager initialization:** simple and thread-safe; the instance is created when the class loads, even if never used.
```java
public class AppConfig {
    private static final AppConfig INSTANCE = new AppConfig();
    private AppConfig() { /* load config */ }
    public static AppConfig getInstance() { return INSTANCE; }
}
```

**Lazy initialization (not thread-safe):** two threads can both see `null` and create two instances.
```java
public static AppConfig getInstance() {
    if (instance == null) instance = new AppConfig();   // race condition
    return instance;
}
```

**Double-checked locking:** lazy and thread-safe. The `volatile` keyword is essential: without it, another thread might see a reference to a partially constructed object because of instruction reordering.
```java
public class AppConfig {
    private static volatile AppConfig instance;
    private AppConfig() { }

    public static AppConfig getInstance() {
        if (instance == null) {                         // 1st check: avoid locking after init
            synchronized (AppConfig.class) {
                if (instance == null) {                 // 2nd check: only one thread creates it
                    instance = new AppConfig();
                }
            }
        }
        return instance;
    }
}
```

**Initialization-on-demand holder (Bill Pugh idiom):** lazy, thread-safe, no explicit locking. The JVM loads the inner class only when `getInstance()` is first called, and class loading is guaranteed thread-safe.
```java
public class AppConfig {
    private AppConfig() { }
    private static class Holder {
        static final AppConfig INSTANCE = new AppConfig();
    }
    public static AppConfig getInstance() { return Holder.INSTANCE; }
}
```

**Enum singleton:** recommended by Joshua Bloch (*Effective Java*). Thread-safe, and protected against duplicate instances created via serialization or reflection, which the other approaches need extra code to prevent.
```java
public enum AppConfig {
    INSTANCE;
    public String get(String key) { /* ... */ return null; }
}
// AppConfig.INSTANCE.get("db.url");
```

In Python, a module is effectively a singleton (imported once); in JavaScript/TypeScript, an exported object from an ES module behaves similarly.

#### Why Singleton is often considered problematic

Singleton is the most criticized GoF pattern; some call it an anti-pattern. Understanding why is as important as knowing how to write it.

1. **Global state in disguise.** Any code anywhere can call `getInstance()` and modify shared state. Behavior then depends on hidden interactions, making bugs hard to trace.
2. **Hidden dependencies.** A class that calls `Database.getInstance()` internally doesn't declare that dependency in its constructor. You can't tell what a class needs by looking at its API. This violates the **Dependency Inversion Principle**, because the class depends on a concrete class it reaches out to grab.
3. **Hard to test.** You can't easily substitute a fake; tests share the same instance, so state leaks between tests and test order starts to matter. (Workarounds like reset methods or reflection are hacks.)
4. **Tight coupling.** Code is bound to the concrete singleton class; switching implementations means editing every call site.
5. **Violates SRP.** The class manages both its real job and its own lifecycle/instance control.
6. **Concurrency issues.** A shared mutable instance requires careful synchronization; naive lazy initialization is a race condition.
7. **"One instance" assumptions break.** Requirements change (two databases, per-tenant configuration), and the single-instance assumption is baked in everywhere.
8. **Not truly single in all environments.** Multiple class loaders (application servers), multiple processes, or multiple service instances behind a load balancer each have their own "singleton." It's one per class loader per process, not one per system.

#### The better alternative: single instance via dependency injection

Keep "only one instance" as a **configuration decision**, not a property hard-coded into the class:

```java
// A normal class with a public constructor
class Cache { ... }

// The DI container (or main) creates exactly one and injects it everywhere
@Configuration
class AppConfiguration {
    @Bean       // Spring beans are singleton-scoped by default
    Cache cache() { return new Cache(); }
}

@Service
class ProductService {
    private final Cache cache;
    ProductService(Cache cache) { this.cache = cache; }    // explicit, replaceable in tests
}
```

You get one shared instance in production, explicit dependencies, and easy substitution in tests.

#### When Singleton is reasonable

- Stateless or effectively immutable shared objects (a configuration loaded once and never changed).
- Logging frameworks (`LoggerFactory.getLogger` is a common accepted case).
- When there's no DI framework and the object is truly unique to the process, with care taken for testability.

**Real-world uses:** `java.lang.Runtime.getRuntime()`, `Desktop.getDesktop()`, Spring beans (singleton scope, managed by the container rather than the class), logger instances.

**Related:** Abstract Factories, Builders, and Facades are often implemented as singletons. Monostate is an alternative (many instances sharing static state).

---

### 4.2 Factory Method (and Simple Factory)

#### Intent

Define an interface for creating an object, but **let subclasses (or a dedicated method) decide which concrete class to instantiate**. Client code asks for "a notification" without knowing whether it gets an email, SMS, or push notification.

#### Problem

```java
// Client code tied to concrete classes and construction details
Notification n;
if (type.equals("EMAIL")) n = new EmailNotification(smtpHost, port, credentials);
else if (type.equals("SMS")) n = new SmsNotification(twilioSid, twilioToken);
else if (type.equals("PUSH")) n = new PushNotification(firebaseKey);
```

- This creation logic gets duplicated wherever notifications are created.
- Every new type requires finding and editing all those places (violates OCP).
- Clients depend on every concrete class and its constructor arguments (violates DIP).

#### Variant 1: Simple Factory (a common idiom, not a GoF pattern)

Centralize creation in one place:

```java
interface Notification {
    void send(String to, String message);
}

class EmailNotification implements Notification { public void send(String to, String m) { /* ... */ } }
class SmsNotification   implements Notification { public void send(String to, String m) { /* ... */ } }
class PushNotification  implements Notification { public void send(String to, String m) { /* ... */ } }

class NotificationFactory {
    public static Notification create(String type) {
        return switch (type) {
            case "EMAIL" -> new EmailNotification();
            case "SMS"   -> new SmsNotification();
            case "PUSH"  -> new PushNotification();
            default -> throw new IllegalArgumentException("Unknown type: " + type);
        };
    }
}

// Client
Notification n = NotificationFactory.create(request.getChannel());
n.send(user.getContact(), "Your order shipped");
```

The `switch` still exists, but **in one place**. Clients depend only on the `Notification` interface. Adding a type means adding a class and one factory line.

A registry removes even that `switch`:
```java
class NotificationFactory {
    private final Map<String, Supplier<Notification>> registry = new HashMap<>();
    void register(String type, Supplier<Notification> creator) { registry.put(type, creator); }
    Notification create(String type) {
        Supplier<Notification> s = registry.get(type);
        if (s == null) throw new IllegalArgumentException("Unknown: " + type);
        return s.get();
    }
}
```
(With Spring, you can inject a `Map<String, Notification>` of all beans implementing the interface, keyed by bean name.)

#### Variant 2: Factory Method (GoF)

The creation step is a method that **subclasses override**. A base class contains logic that needs an object, but defers the decision of *which* object to subclasses.

```java
// Creator
abstract class Logistics {
    // The factory method
    protected abstract Transport createTransport();

    // Business logic that uses the product; same for all subclasses
    public void planDelivery(Order order) {
        Transport t = createTransport();
        t.deliver(order);
    }
}

// Concrete creators
class RoadLogistics extends Logistics {
    protected Transport createTransport() { return new Truck(); }
}

class SeaLogistics extends Logistics {
    protected Transport createTransport() { return new Ship(); }
}

// Product
interface Transport { void deliver(Order order); }
class Truck implements Transport { public void deliver(Order o) { /* by road */ } }
class Ship  implements Transport { public void deliver(Order o) { /* by sea */ } }
```

**Participants:**
- **Product** (`Transport`): the interface of objects created.
- **Concrete Products** (`Truck`, `Ship`).
- **Creator** (`Logistics`): declares the factory method and usually contains logic using the product.
- **Concrete Creators** (`RoadLogistics`, `SeaLogistics`): override the factory method.

Factory Method is essentially a **Template Method specialized for object creation**.

#### Static factory methods (a related idiom)

Named static methods used instead of constructors: `List.of(...)`, `Optional.of(x)`, `Integer.valueOf(5)`, `LocalDate.of(2026, 11, 24)`, `Executors.newFixedThreadPool(4)`. Advantages: descriptive names, can return cached instances (`Integer.valueOf` caches small values), can return any subtype (`List.of` returns hidden implementation classes).

#### Real-world uses

`Calendar.getInstance()`, `NumberFormat.getInstance(locale)`, `DriverManager.getConnection(url)` (returns the right JDBC driver's connection), Spring's `BeanFactory`, `javax.xml.parsers.DocumentBuilderFactory`, `Iterable.iterator()` (each collection returns its own iterator type).

#### When to use

- The exact type to create is determined at runtime (from config, user input, request data).
- Construction is complex or should be hidden from clients.
- You want a single place to add new types.
- A framework provides a base class and lets users plug in specific objects.

#### Pros and cons

| Pros | Cons |
|---|---|
| Clients depend on interfaces, not concrete classes (DIP) | More classes and indirection |
| New products added without changing clients (OCP) | Factory Method's subclassing can create parallel hierarchies (a creator per product) |
| Creation logic centralized (SRP) | Overkill if there's only one product type that will never vary |

**Related:** Abstract Factory is often built from several factory methods; Template Method (factory method is a special case); Prototype (create by cloning instead).

---

### 4.3 Abstract Factory

#### Intent

Provide an interface for creating **families of related objects** without specifying their concrete classes, guaranteeing the created objects are **compatible** with each other.

#### Problem

A UI toolkit must support two themes (or platforms): Light and Dark. Each theme has a `Button`, a `Checkbox`, and a `TextField`. A dark button next to a light checkbox looks broken, so components must always come from the same family. Scattering `if (theme == DARK) new DarkButton()` throughout the code is error-prone, and adding a third theme means editing everything.

#### Solution

- Define an interface for each product type (`Button`, `Checkbox`).
- Define an **abstract factory** interface with a creation method for each product (`createButton()`, `createCheckbox()`).
- Implement one **concrete factory per family** (`LightThemeFactory`, `DarkThemeFactory`), each returning products from its own family.
- The client receives one factory and uses it for everything, so mixing families is impossible.

```java
// Abstract products
interface Button   { void render(); }
interface Checkbox { void render(); }

// Concrete products: Light family
class LightButton   implements Button   { public void render() { System.out.println("Light button"); } }
class LightCheckbox implements Checkbox { public void render() { System.out.println("Light checkbox"); } }

// Concrete products: Dark family
class DarkButton    implements Button   { public void render() { System.out.println("Dark button"); } }
class DarkCheckbox  implements Checkbox { public void render() { System.out.println("Dark checkbox"); } }

// Abstract factory
interface UIFactory {
    Button createButton();
    Checkbox createCheckbox();
}

// Concrete factories
class LightThemeFactory implements UIFactory {
    public Button createButton()     { return new LightButton(); }
    public Checkbox createCheckbox() { return new LightCheckbox(); }
}

class DarkThemeFactory implements UIFactory {
    public Button createButton()     { return new DarkButton(); }
    public Checkbox createCheckbox() { return new DarkCheckbox(); }
}

// Client: never mentions a concrete class
class SettingsScreen {
    private final Button saveButton;
    private final Checkbox notifications;

    SettingsScreen(UIFactory factory) {
        this.saveButton = factory.createButton();
        this.notifications = factory.createCheckbox();
    }

    void render() { saveButton.render(); notifications.render(); }
}

// Wiring: the family is chosen once
UIFactory factory = userPrefersDark ? new DarkThemeFactory() : new LightThemeFactory();
new SettingsScreen(factory).render();
```

#### A backend example: cloud providers

```java
interface CloudFactory {
    BlobStorage createStorage();
    MessageQueue createQueue();
    SecretStore createSecrets();
}

class AwsFactory implements CloudFactory {
    public BlobStorage createStorage()  { return new S3Storage(); }
    public MessageQueue createQueue()   { return new SqsQueue(); }
    public SecretStore createSecrets()  { return new AwsSecretsManager(); }
}

class GcpFactory implements CloudFactory {
    public BlobStorage createStorage()  { return new GcsStorage(); }
    public MessageQueue createQueue()   { return new PubSubQueue(); }
    public SecretStore createSecrets()  { return new GcpSecretManager(); }
}
```
The application never mixes S3 with Pub/Sub accidentally. Other examples: database vendor families (connection, command, and parameter objects for SQL Server vs Postgres), test vs production environments (all fakes vs all real services), and regional payment and tax rule families.

#### Factory Method vs Abstract Factory

| | Factory Method | Abstract Factory |
|---|---|---|
| Creates | One product | A family of related products |
| Mechanism | Inheritance: a subclass overrides one creation method | Composition: the client is given a factory object with several creation methods |
| Focus | Defer *which class* to instantiate | Guarantee *products are compatible* |
| Typical size | One method | An interface with several methods |

Concrete factories in Abstract Factory are often implemented using factory methods, and since one factory per family is usually enough, they're frequently singletons or DI-managed beans.

#### Pros and cons

| Pros | Cons |
|---|---|
| Products from one family are guaranteed consistent | **Adding a new product type** (e.g., `Slider`) means changing the factory interface and *every* concrete factory |
| Client fully decoupled from concrete classes | Many interfaces and classes |
| Switching an entire family is a one-line change | Overkill when there's only one family |
| New families added without touching clients (OCP for families) | |

The asymmetry is worth remembering: Abstract Factory makes adding **families** easy and adding **product types** hard.

**Real-world uses:** `javax.xml.parsers.DocumentBuilderFactory`, `javax.xml.transform.TransformerFactory`, cross-platform UI toolkits, ADO.NET's `DbProviderFactory`.

**Related:** Factory Method (building block), Singleton (one factory per family), Builder (focuses on step-by-step construction of one complex object rather than families).

---

### 4.4 Builder

#### Intent

**Separate the construction of a complex object from its representation**, building it step by step, so the same construction process can produce readable, valid objects even with many optional parts.

#### Problem: the telescoping constructor

```java
// Which argument is which? What does `true, false, true` mean?
new Pizza("large", "thin", true, false, true, false, true, "extra spicy", 2);

// Overloads multiply for every combination of optional parameters
Pizza(String size)
Pizza(String size, String crust)
Pizza(String size, String crust, boolean cheese)
Pizza(String size, String crust, boolean cheese, boolean pepperoni) ...
```

The alternative, the JavaBeans approach (empty constructor plus setters), makes the object **mutable** and allows it to exist in an **inconsistent, half-built state** between setter calls, with no single place to validate.

#### Solution

A separate `Builder` object collects parameters through readable, chainable methods and creates the final object in one step with `build()`, where validation happens. The resulting object can be **immutable**.

```java
public final class HttpRequest {
    private final String method;
    private final String url;
    private final Map<String, String> headers;
    private final String body;
    private final int timeoutSeconds;

    private HttpRequest(Builder b) {           // only the builder can construct it
        this.method = b.method;
        this.url = b.url;
        this.headers = Map.copyOf(b.headers);  // defensive, immutable copy
        this.body = b.body;
        this.timeoutSeconds = b.timeoutSeconds;
    }

    public static Builder builder(String url) { return new Builder(url); }

    public static class Builder {
        // Required
        private final String url;
        // Optional, with defaults
        private String method = "GET";
        private final Map<String, String> headers = new HashMap<>();
        private String body;
        private int timeoutSeconds = 30;

        private Builder(String url) { this.url = url; }

        public Builder method(String method)          { this.method = method; return this; }
        public Builder header(String name, String v)  { headers.put(name, v); return this; }
        public Builder body(String body)              { this.body = body; return this; }
        public Builder timeout(int seconds)           { this.timeoutSeconds = seconds; return this; }

        public HttpRequest build() {
            // Validate the whole object once, before it exists
            if (url == null || url.isBlank()) throw new IllegalStateException("URL required");
            if (body != null && method.equals("GET")) throw new IllegalStateException("GET cannot have a body");
            if (timeoutSeconds <= 0) throw new IllegalStateException("Timeout must be positive");
            return new HttpRequest(this);
        }
    }
}

// Usage: readable, self-documenting, optional parts skipped
HttpRequest request = HttpRequest.builder("https://api.example.com/orders")
        .method("POST")
        .header("Content-Type", "application/json")
        .header("Authorization", "Bearer abc123")
        .body("{\"item\":\"pen\"}")
        .timeout(10)
        .build();
```

Each setter returns `this`, enabling the chained **fluent interface**.

#### The GoF version: Director

The original pattern adds a **Director** that encodes construction sequences, so the same steps can produce different representations:

```java
interface HouseBuilder {
    void buildFoundation();
    void buildWalls();
    void buildRoof();
    House getResult();
}

class WoodenHouseBuilder implements HouseBuilder { /* ... */ }
class StoneHouseBuilder  implements HouseBuilder { /* ... */ }

class Director {
    void constructSimpleHouse(HouseBuilder b) {
        b.buildFoundation(); b.buildWalls(); b.buildRoof();
    }
}
```

The Director knows the *order of steps*; each builder knows *how* to perform them. In everyday code, the fluent builder (without a director) is far more common.

#### Real-world uses

- Java: `StringBuilder`, `java.net.http.HttpRequest.newBuilder()`, `Stream.builder()`, `Locale.Builder`, `ProcessBuilder`.
- Lombok's `@Builder` generates builders automatically.
- AWS SDK v2 request objects: `PutObjectRequest.builder().bucket("b").key("k").build()`.
- Test data builders: `aUser().withEmail("x@y.com").build()` keep tests readable.
- Query builders (jOOQ, Criteria API, Knex), Retrofit, OkHttp, Protocol Buffers messages.
- Kotlin and Python often use named and default arguments instead, which removes much of the need for builders.

#### When to use

- Objects with many parameters, especially many optional ones (roughly four or more).
- Immutable objects that need validation across multiple fields.
- Step-by-step or conditional construction (`if (premium) builder.addFeature(...)`).
- Producing different representations from the same construction process (Director variant).

#### Pros and cons

| Pros | Cons |
|---|---|
| Readable, self-documenting construction | More code (a whole extra class), unless generated |
| Supports immutability and validation at build time | Overkill for simple objects with few fields |
| No inconsistent half-built objects visible to the rest of the code | Required parameters can be forgotten unless enforced (put them in the builder's constructor, or use a "step builder" with typed stages) |
| Optional parameters with sensible defaults | |

**Related:** Abstract Factory (creates families immediately; Builder builds one complex object step by step), Composite (builders are often used to construct composite trees), Fluent interfaces.

---

## 5. Structural Patterns

### 5.1 Adapter

#### Intent

**Convert the interface of a class into another interface clients expect**, so classes with incompatible interfaces can work together. Like a travel plug adapter: the device and the socket stay the same; the adapter translates between them.

#### Problem

Your application has a `PaymentGateway` interface used throughout the code:
```java
interface PaymentGateway {
    PaymentResult charge(String customerId, BigDecimal amountInRupees);
}
```
You need to integrate a third-party SDK (or a legacy system) whose API looks completely different: different method names, amounts in paise as `long`, a different result type, checked exceptions. You can't modify the SDK, and you don't want SDK-specific code spread across your business logic.

```java
// Third-party SDK: can't change it
class RazorpayClient {
    RazorpayResponse createPayment(Map<String, Object> params) throws RazorpayException { ... }
}
```

#### Solution

Write an **adapter** that implements the interface your code expects (the **Target**) and internally calls the incompatible class (the **Adaptee**), translating requests, data formats, and errors.

```java
class RazorpayAdapter implements PaymentGateway {
    private final RazorpayClient client;      // the adaptee

    RazorpayAdapter(RazorpayClient client) { this.client = client; }

    @Override
    public PaymentResult charge(String customerId, BigDecimal amountInRupees) {
        Map<String, Object> params = Map.of(
            "customer_id", customerId,
            "amount", amountInRupees.movePointRight(2).longValueExact(),   // rupees → paise
            "currency", "INR"
        );
        try {
            RazorpayResponse response = client.createPayment(params);
            return new PaymentResult(response.isCaptured(), response.getId());   // translate result
        } catch (RazorpayException e) {
            throw new PaymentFailedException("Razorpay charge failed", e);       // translate errors
        }
    }
}

// Business code is unaware of Razorpay
class CheckoutService {
    private final PaymentGateway gateway;
    CheckoutService(PaymentGateway gateway) { this.gateway = gateway; }
}

// Switching providers = writing a StripeAdapter; CheckoutService doesn't change
```

**Participants:** Target (interface the client expects), Client, Adaptee (existing incompatible class), Adapter (implements Target, wraps Adaptee).

#### Object adapter vs class adapter

- **Object adapter** (above): the adapter *contains* the adaptee (composition). Preferred; works in any language and can adapt subclasses of the adaptee too.
- **Class adapter:** the adapter *inherits* from the adaptee and implements the target. Requires multiple inheritance (C++), or works in Java only when the target is an interface. Less flexible.

#### Real-world uses

- `Arrays.asList()` adapts an array to the `List` interface.
- `InputStreamReader` adapts a byte stream (`InputStream`) to a character stream (`Reader`).
- `Collections.enumeration()` / `Collections.list()` adapt between `Enumeration` and collections.
- SLF4J bindings adapt various logging frameworks to one API.
- Spring MVC's `HandlerAdapter` lets the dispatcher call different kinds of controllers uniformly.
- **Hexagonal architecture:** "adapters" connect the domain's ports (interfaces) to databases, message queues, and external APIs.
- Anti-corruption layers in Domain-Driven Design, protecting your domain model from a legacy system's model.

#### When to use

- Integrating third-party libraries, external APIs, or legacy code whose interface doesn't match yours.
- Wrapping vendor SDKs so they can be swapped and so tests can use fakes.
- Reusing existing classes in a new system without modifying them.

#### Pros and cons

| Pros | Cons |
|---|---|
| Reuse existing code without modifying it (OCP) | An extra layer of indirection |
| Isolates conversion logic in one place (SRP) | Too many adapters may indicate interfaces that should be redesigned |
| Business logic decoupled from vendors (DIP) | Some mismatches can't be adapted cleanly (missing features) |

**Related:** Decorator (same interface, adds behavior; Adapter changes the interface), Facade (new simplified interface over many classes; Adapter makes one existing interface match another expected one), Proxy (same interface, controls access), Bridge (designed upfront to separate abstraction and implementation; Adapter is applied after the fact).

---

### 5.2 Decorator

#### Intent

**Attach additional responsibilities to an object dynamically** by wrapping it in objects that implement the same interface. Decorators provide a flexible alternative to subclassing for extending behavior.

#### Problem

A coffee shop sells `Espresso` and `HouseBlend`, and customers can add milk, mocha, whipped cream, and soy, in any combination and quantity (double mocha). With inheritance, you'd need `EspressoWithMilk`, `EspressoWithMilkAndMocha`, `HouseBlendWithSoyAndWhip`, ...: a **class explosion**, and adding a new topping multiplies the count again. Inheritance is also static: you can't add milk to an existing order at runtime.

#### Solution

- A **Component** interface (`Beverage`).
- **Concrete components** (`Espresso`) implement the basic behavior.
- A **Decorator** implements the same interface **and holds a reference to a component**. It forwards calls to the wrapped object, adding its own behavior before or after.
- Decorators can wrap other decorators, so behaviors **stack**.

```java
interface Beverage {
    String description();
    double cost();
}

class Espresso implements Beverage {
    public String description() { return "Espresso"; }
    public double cost() { return 120; }
}

// Base decorator: holds a wrapped Beverage and forwards by default
abstract class BeverageDecorator implements Beverage {
    protected final Beverage wrapped;
    BeverageDecorator(Beverage wrapped) { this.wrapped = wrapped; }
}

class Milk extends BeverageDecorator {
    Milk(Beverage b) { super(b); }
    public String description() { return wrapped.description() + ", Milk"; }
    public double cost() { return wrapped.cost() + 20; }
}

class Mocha extends BeverageDecorator {
    Mocha(Beverage b) { super(b); }
    public String description() { return wrapped.description() + ", Mocha"; }
    public double cost() { return wrapped.cost() + 30; }
}

// Compose at runtime, in any combination
Beverage order = new Mocha(new Mocha(new Milk(new Espresso())));
order.description();   // "Espresso, Milk, Mocha, Mocha"
order.cost();          // 120 + 20 + 30 + 30 = 200
```

Each call travels inward through the layers and the results build up on the way back out, like layers of an onion.

#### A backend example: adding cross-cutting behavior

```java
interface DataSource {
    void write(String data);
    String read();
}

class FileDataSource implements DataSource { /* plain file I/O */ }

class CompressionDecorator implements DataSource {
    private final DataSource inner;
    CompressionDecorator(DataSource inner) { this.inner = inner; }
    public void write(String data) { inner.write(compress(data)); }
    public String read()           { return decompress(inner.read()); }
}

class EncryptionDecorator implements DataSource {
    private final DataSource inner;
    EncryptionDecorator(DataSource inner) { this.inner = inner; }
    public void write(String data) { inner.write(encrypt(data)); }
    public String read()           { return decrypt(inner.read()); }
}

// Encrypt, then compress, then write to file; read reverses automatically
DataSource source = new CompressionDecorator(new EncryptionDecorator(new FileDataSource("data.bin")));
```

Other common decorators: **caching** (check a cache before calling the wrapped repository), **logging and metrics** (time each call), **retry** (re-invoke on transient failure), **rate limiting**, and **authorization checks**. Each concern lives in its own class (SRP) and can be added or removed without touching the core class (OCP).

```java
UserRepository repo = new MetricsUserRepository(
                        new CachingUserRepository(
                          new PostgresUserRepository(dataSource), cache), metrics);
```

**Order matters:** in the example above, metrics wrap the cache, so they measure cache hits too. Swapping the order measures only database calls.

#### Real-world uses

- **Java I/O** is the textbook example:
  ```java
  Reader r = new BufferedReader(new InputStreamReader(new FileInputStream("file.txt")));
  InputStream in = new GZIPInputStream(new BufferedInputStream(new FileInputStream("data.gz")));
  ```
  `BufferedInputStream` adds buffering, `GZIPInputStream` adds decompression, `DataInputStream` adds typed reads, all to any `InputStream`.
- `Collections.synchronizedList()`, `Collections.unmodifiableList()`: wrap a list with synchronization or read-only behavior.
- Python's `@decorator` syntax (function decorators: `@lru_cache`, `@retry`), TypeScript decorators: same idea applied to functions.
- HTTP middleware that wraps a handler (`http.Handler` wrappers in Go) is decorator-like.

#### When to use

- Adding behavior to individual objects at runtime without affecting others of the same class.
- Many combinations of optional features, where subclassing would explode.
- Cross-cutting concerns (logging, caching, retries, metrics) that you want to keep separate from core logic.
- Extending classes you can't subclass (final classes).

#### Pros and cons

| Pros | Cons |
|---|---|
| Combine behaviors flexibly at runtime | Many small classes; deep wrapping is hard to debug (stack traces through many layers) |
| Avoids subclass explosion | Order of wrapping matters and can cause subtle bugs |
| Each behavior in its own class (SRP) | Removing a specific decorator from the middle of a stack is awkward |
| Extend without modifying (OCP) | Code checking an object's concrete type (`instanceof Espresso`) breaks, because it's wrapped |

**Related:** Adapter (changes the interface; Decorator keeps it), Proxy (same structure; Proxy controls access, Decorator adds behavior; see Section 7), Composite (also uses recursive composition, but to represent part-whole trees; a decorator has exactly one child), Chain of Responsibility (handlers can stop the chain; decorators normally always forward), Strategy (changes an object's internals; Decorator changes its "skin").

---

### 5.3 Facade

#### Intent

**Provide a single, simplified interface to a complex subsystem**, making the subsystem easier to use for common tasks.

#### Problem

Placing an order involves many subsystems: inventory checks, pricing and discounts, payment, shipping, invoicing, and notifications. Each has its own API, ordering requirements, and error handling. If every controller, CLI command, and batch job coordinates them directly:
- Clients are coupled to every subsystem and their internal details.
- The same orchestration logic is duplicated everywhere.
- Changing one subsystem forces changes across all clients.

#### Solution

Create a **facade** class that offers a few high-level operations and delegates the work to the subsystem classes in the right order. Clients talk only to the facade.

```java
// Complex subsystem classes
class InventoryService   { boolean reserve(String sku, int qty) { ... } void release(String sku, int qty) { ... } }
class PricingService     { BigDecimal calculateTotal(Cart cart, String couponCode) { ... } }
class PaymentService     { String charge(String customerId, BigDecimal amount) { ... } }
class ShippingService    { String schedule(Address address, List<Item> items) { ... } }
class NotificationService{ void sendOrderConfirmation(String email, String orderId) { ... } }

// Facade
class OrderFacade {
    private final InventoryService inventory;
    private final PricingService pricing;
    private final PaymentService payment;
    private final ShippingService shipping;
    private final NotificationService notifications;

    OrderFacade(InventoryService i, PricingService pr, PaymentService pa,
                ShippingService s, NotificationService n) {
        this.inventory = i; this.pricing = pr; this.payment = pa;
        this.shipping = s; this.notifications = n;
    }

    public OrderConfirmation placeOrder(Customer customer, Cart cart, String coupon) {
        cart.items().forEach(item -> {
            if (!inventory.reserve(item.sku(), item.qty()))
                throw new OutOfStockException(item.sku());
        });
        BigDecimal total = pricing.calculateTotal(cart, coupon);
        String paymentId = payment.charge(customer.id(), total);
        String trackingId = shipping.schedule(customer.address(), cart.items());
        String orderId = UUID.randomUUID().toString();
        notifications.sendOrderConfirmation(customer.email(), orderId);
        return new OrderConfirmation(orderId, paymentId, trackingId, total);
    }
}

// Client: one call instead of orchestrating five services
OrderConfirmation c = orderFacade.placeOrder(customer, cart, "SAVE10");
```

(A production version would also handle failures, e.g. releasing inventory if payment fails.)

Key point: the facade **doesn't hide** the subsystem completely. Advanced clients can still use subsystem classes directly when they need fine-grained control; the facade just makes the common path easy.

#### Real-world uses

- `javax.faces.context.FacesContext`, `java.net.URL.openStream()` (hides connections, protocols, and streams).
- `JdbcTemplate` and `RestTemplate`/`RestClient` in Spring: simple calls hiding connection handling, exception translation, and resource cleanup.
- SLF4J as a facade over Logback, Log4j, and java.util.logging.
- Application service layers in layered architectures act as facades over domain objects and repositories.
- **API Gateways / Backend-for-Frontend (BFF)** in microservices: a single endpoint aggregating calls to many services is the facade idea at the architecture level.
- A home theater "watch movie" button that turns on the projector, dims the lights, starts the sound system, and plays the movie.

#### When to use

- A subsystem is complex and most clients need only a simple subset of its functionality.
- You want to layer a system: each layer exposes a facade to the layer above.
- You want to reduce coupling between clients and many subsystem classes.
- Wrapping a messy legacy system behind a clean entry point.

#### Pros and cons

| Pros | Cons |
|---|---|
| Simple interface for common tasks | Can become a **"god object"** that knows and does too much; split facades by use case |
| Clients decoupled from subsystem internals | Hides useful features if clients are forced through it exclusively |
| Changes inside the subsystem don't ripple to clients | One more layer to maintain |
| Centralizes orchestration logic | |

**Related:** Adapter (makes an existing interface match an expected one, usually wrapping a single object; Facade defines a new simpler interface over many objects), Mediator (also centralizes interaction, but subsystem objects communicate *through* the mediator; with a facade, subsystem objects don't know the facade exists), Abstract Factory (can be used with Facade to create subsystem objects), Singleton (a facade is often a single instance).

---

### 5.4 Proxy

#### Intent

**Provide a surrogate or placeholder for another object to control access to it.** The proxy implements the same interface as the real object, so clients can't tell the difference, but it can intercept calls to add access control, laziness, caching, remote communication, or logging.

#### Structure

- **Subject:** the common interface.
- **Real Subject:** the actual object doing the work.
- **Proxy:** implements Subject, holds (or knows how to obtain) a reference to the Real Subject, and decides when and how to forward calls.

#### Types of proxies

**1. Virtual proxy (lazy loading):** delays creation of an expensive object until it's actually needed.
```java
interface Image { void display(); }

class HighResImage implements Image {
    private final String path;
    HighResImage(String path) { this.path = path; loadFromDisk(); }   // slow: reads a large file
    public void display() { /* render */ }
    private void loadFromDisk() { System.out.println("Loading " + path); }
}

class LazyImageProxy implements Image {
    private final String path;
    private HighResImage realImage;           // not created yet

    LazyImageProxy(String path) { this.path = path; }

    public void display() {
        if (realImage == null) realImage = new HighResImage(path);   // created on first use
        realImage.display();
    }
}

// A gallery of 500 thumbnails creates 500 cheap proxies; only viewed images are loaded
```
**Hibernate/JPA lazy loading** works this way: `order.getCustomer()` returns a generated proxy, and the SQL query runs only when you first access a field. (This is also how the N+1 problem sneaks in.)

**2. Protection proxy (access control):** checks permissions before forwarding.
```java
interface DocumentService {
    Document read(String id);
    void delete(String id);
}

class SecuredDocumentService implements DocumentService {
    private final DocumentService real;
    private final CurrentUser user;

    SecuredDocumentService(DocumentService real, CurrentUser user) {
        this.real = real; this.user = user;
    }

    public Document read(String id) {
        return real.read(id);
    }

    public void delete(String id) {
        if (!user.hasRole("ADMIN")) throw new AccessDeniedException("Only admins can delete");
        real.delete(id);
    }
}
```

**3. Remote proxy:** represents an object in another process or machine; the client calls local methods, and the proxy handles serialization and network communication. Examples: Java RMI stubs, gRPC client stubs, Feign clients, AWS SDK clients, CORBA.

**4. Caching proxy:** returns cached results for repeated requests instead of calling the real (slow or expensive) object.
```java
class CachingWeatherProxy implements WeatherService {
    private final WeatherService real;
    private final Map<String, CachedValue> cache = new ConcurrentHashMap<>();

    public Forecast getForecast(String city) {
        CachedValue cv = cache.get(city);
        if (cv != null && !cv.isExpired()) return cv.forecast();
        Forecast f = real.getForecast(city);
        cache.put(city, new CachedValue(f, Instant.now().plusSeconds(600)));
        return f;
    }
}
```

**5. Logging / monitoring / smart reference proxy:** records calls, measures timing, counts references, manages locks or transactions around the real call.

#### Dynamic proxies

Writing a proxy class per interface is tedious. Java can **generate proxies at runtime**:
```java
UserService proxy = (UserService) Proxy.newProxyInstance(
    UserService.class.getClassLoader(),
    new Class<?>[]{ UserService.class },
    (p, method, args) -> {
        long start = System.nanoTime();
        try {
            return method.invoke(realUserService, args);
        } finally {
            System.out.printf("%s took %d µs%n", method.getName(), (System.nanoTime() - start) / 1000);
        }
    });
```

This is how **Spring AOP** implements `@Transactional`, `@Cacheable`, `@Async`, `@PreAuthorize`, and `@Retryable`: Spring wraps your bean in a proxy (JDK dynamic proxy for interfaces, CGLIB subclass proxy otherwise) that starts a transaction, checks a cache, or checks permissions before calling your method.

**Common gotcha:** calling a `@Transactional` method from another method **in the same class** (`this.saveOrder()`) bypasses the proxy, so no transaction starts. The call never goes through the wrapper.

#### Real-world uses

Hibernate lazy-loaded entities, Spring AOP proxies, Java RMI, gRPC stubs, Mockito mocks (generated proxies), JavaScript `Proxy` objects (used by Vue's reactivity system), and, at the infrastructure level, reverse proxies (Nginx), forward proxies, and API gateways.

#### When to use

- Expensive objects that should be created only when needed.
- Access control around an object.
- Hiding the fact that an object lives on another machine.
- Adding caching, logging, or transactions transparently.

#### Pros and cons

| Pros | Cons |
|---|---|
| Control access without changing the real object or its clients | Adds latency and indirection |
| Lazy initialization saves resources | Behavior can be surprising (self-invocation bypassing Spring proxies, lazy-loading exceptions outside a session) |
| Transparent to clients (same interface) | Remote proxies can hide the cost of network calls, tempting chatty designs |

**Related:** Decorator (same structure; the difference is intent: see Section 7), Adapter (different interface; Proxy keeps the same one), Facade (simplifies many objects; Proxy stands in for one).

---

### 5.5 Composite

#### Intent

**Compose objects into tree structures to represent part-whole hierarchies, and let clients treat individual objects and compositions uniformly.**

#### Problem

A file system has files and folders; folders contain files and other folders. To compute the total size, client code would need to check each item's type: "if it's a file, take its size; if it's a folder, loop over its children and recurse." That type-checking logic spreads everywhere: size, search, permissions, rendering.

Similar structures: UI components (panels containing buttons and other panels), organization charts (managers and employees), menus (menus containing items and submenus), product bundles (a bundle containing products and other bundles), expression trees (`(2 + 3) * 4`).

#### Solution

- **Component:** a common interface for both simple and complex elements (`FileSystemItem` with `getSize()`).
- **Leaf:** a basic element with no children (`File`); does the actual work.
- **Composite:** an element with children (`Folder`); implements operations by **delegating to its children** and combining the results. It stores children as Components, so it doesn't care whether each child is a leaf or another composite.

```java
interface FileSystemItem {
    String name();
    long getSize();
    void print(String indent);
}

// Leaf
class File implements FileSystemItem {
    private final String name;
    private final long size;
    File(String name, long size) { this.name = name; this.size = size; }

    public String name() { return name; }
    public long getSize() { return size; }
    public void print(String indent) { System.out.println(indent + "- " + name + " (" + size + " B)"); }
}

// Composite
class Folder implements FileSystemItem {
    private final String name;
    private final List<FileSystemItem> children = new ArrayList<>();
    Folder(String name) { this.name = name; }

    public void add(FileSystemItem item)    { children.add(item); }
    public void remove(FileSystemItem item) { children.remove(item); }

    public String name() { return name; }

    public long getSize() {
        return children.stream().mapToLong(FileSystemItem::getSize).sum();   // recursive
    }

    public void print(String indent) {
        System.out.println(indent + "+ " + name + "/");
        children.forEach(c -> c.print(indent + "  "));
    }
}

// Build a tree
Folder root = new Folder("root");
Folder docs = new Folder("docs");
docs.add(new File("resume.pdf", 120_000));
docs.add(new File("notes.md", 4_000));
root.add(docs);
root.add(new File("photo.jpg", 2_500_000));

root.getSize();   // 2,624,000: the client never checks types
root.print("");
```

Calling `getSize()` on the root recursively visits the whole tree. The client treats a single file and an entire directory exactly the same.

#### Design choice: where do `add`/`remove` live?

- **Transparency:** declare `add()`/`remove()` in the Component interface so all elements look identical. Leaves must then implement them meaninglessly (throwing exceptions), which conflicts with ISP and LSP.
- **Safety** (shown above): only Composite has `add()`/`remove()`. The type system prevents adding children to a leaf, but clients must know whether they hold a composite when building the tree.

Modern practice generally favors **safety**.

#### Real-world uses

- UI frameworks: Java AWT/Swing (`Container` holds `Component`s and is itself a `Component`), the HTML DOM, Android `ViewGroup`/`View`, React component trees.
- File systems, XML/JSON document trees.
- Organization hierarchies, bill of materials, nested menus and categories.
- Compilers: abstract syntax trees, where an expression node contains sub-expressions.
- Permission groups that contain users and other groups.
- Composite validators or rules: an `AndRule` containing other rules.

#### When to use

- Data naturally forms a tree (part-whole hierarchy).
- Clients should treat single objects and groups uniformly.
- Operations naturally recurse over the structure.

#### Pros and cons

| Pros | Cons |
|---|---|
| Simple client code: no type checks for leaves vs containers | The common interface may become too general (some operations don't make sense for leaves) |
| Easy to add new element types (OCP) | Hard to restrict which types of children a composite may hold (type checks at runtime) |
| Recursive operations are natural and concise | Very deep or large trees can cause performance issues with naive recursion |

**Related:** Decorator (similar recursive structure, but a decorator wraps exactly one object to add behavior; a composite holds many children to aggregate them), Iterator (traverse composites), Visitor (add new operations over a composite tree without changing its classes), Builder (construct complex trees), Chain of Responsibility (requests often travel up a composite tree from child to parent, e.g., event bubbling in UIs).

---

## 6. Behavioral Patterns

### 6.1 Strategy

#### Intent

**Define a family of algorithms, encapsulate each one in its own class, and make them interchangeable.** The client chooses (or is given) a strategy and can switch it at runtime without changing the code that uses it.

#### Problem

A navigation app calculates routes by car, walking, cycling, or public transit. A ride-hailing app computes fares differently for standard rides, surge periods, and promotions. Putting every variant in one class with conditionals:

```java
// BAD
class FareCalculator {
    double calculate(Ride ride, String pricingType) {
        if (pricingType.equals("STANDARD")) { ... }
        else if (pricingType.equals("SURGE")) { ... }
        else if (pricingType.equals("PROMO")) { ... }
        // new pricing → modify this class
    }
}
```
The class grows with every variant, each algorithm can't be tested alone, and adding one means modifying working code (violates OCP and SRP).

#### Solution

- **Strategy interface:** declares the algorithm's method.
- **Concrete strategies:** each implements one variant.
- **Context:** holds a reference to a strategy and delegates the work to it. It doesn't know which concrete strategy it has.

```java
interface PricingStrategy {
    double calculateFare(Ride ride);
}

class StandardPricing implements PricingStrategy {
    public double calculateFare(Ride r) { return 50 + r.distanceKm() * 12; }
}

class SurgePricing implements PricingStrategy {
    private final double multiplier;
    SurgePricing(double multiplier) { this.multiplier = multiplier; }
    public double calculateFare(Ride r) { return (50 + r.distanceKm() * 12) * multiplier; }
}

class PromoPricing implements PricingStrategy {
    private final PricingStrategy base;
    private final double discountPct;
    PromoPricing(PricingStrategy base, double discountPct) { this.base = base; this.discountPct = discountPct; }
    public double calculateFare(Ride r) { return base.calculateFare(r) * (1 - discountPct / 100); }
}

// Context
class FareService {
    private PricingStrategy strategy;
    FareService(PricingStrategy strategy) { this.strategy = strategy; }
    void setStrategy(PricingStrategy strategy) { this.strategy = strategy; }   // swap at runtime
    double fareFor(Ride ride) { return strategy.calculateFare(ride); }
}

// Choose based on runtime conditions
FareService service = new FareService(new StandardPricing());
if (demandIsHigh) service.setStrategy(new SurgePricing(1.8));
double fare = service.fareFor(ride);
```

Often a factory or a map selects the strategy:
```java
Map<String, PricingStrategy> strategies = Map.of(
    "STANDARD", new StandardPricing(),
    "SURGE", new SurgePricing(1.5));
PricingStrategy s = strategies.get(request.pricingType());
```

#### Strategies as lambdas

When the strategy interface has a single method, modern languages let you pass a function instead of writing a class:

```java
// Comparator is the classic strategy
users.sort(Comparator.comparing(User::lastName));
users.sort(Comparator.comparing(User::createdAt).reversed());

// Custom functional strategy
PricingStrategy nightPricing = ride -> 80 + ride.distanceKm() * 15;
```

#### Real-world uses

- `Comparator` passed to `Collections.sort()` / `List.sort()`.
- `java.util.concurrent.ThreadPoolExecutor`'s `RejectedExecutionHandler` (abort, discard, caller-runs policies).
- Spring Security's `PasswordEncoder` (BCrypt, Argon2, ...), Spring's `Resource` loading strategies.
- Compression algorithms (gzip, zstd), serialization formats (JSON, Protobuf), payment methods, shipping cost calculators, retry/backoff policies, load-balancing algorithms (round robin, least connections), cache eviction policies (LRU, LFU).

#### When to use

- Several variants of an algorithm exist and you want to switch between them, possibly at runtime.
- You want to eliminate large conditionals selecting behavior.
- Algorithms should be isolated for testing or hidden from clients (their internal data structures).
- Different clients need different behavior from otherwise identical classes.

#### Pros and cons

| Pros | Cons |
|---|---|
| Swap algorithms at runtime | Clients (or a factory) must know the strategies exist to choose one |
| Each algorithm isolated and testable (SRP) | More classes (mitigated with lambdas) |
| Add strategies without changing the context (OCP) | Overkill for two variants that will never grow |
| Replaces inheritance with composition | Context and strategies must agree on what data is passed |

**Related:** State (same structure, but states switch themselves based on internal conditions; see Section 7), Template Method (varies parts of an algorithm through inheritance; Strategy swaps the whole algorithm through composition), Decorator (changes behavior by wrapping), Factory (often used to pick a strategy), Dependency Injection (strategies are commonly injected).

---

### 6.2 Observer (Publish-Subscribe)

#### Intent

**Define a one-to-many dependency between objects so that when one object (the subject) changes state, all its dependents (observers) are notified automatically.**

#### Problem

When an order is placed, many things should happen: send a confirmation email, update inventory, award loyalty points, notify the analytics system, notify the warehouse. If `OrderService` calls each of these directly:

```java
// BAD: OrderService knows about every downstream concern
void placeOrder(Order o) {
    repository.save(o);
    emailService.sendConfirmation(o);
    inventoryService.decrement(o);
    loyaltyService.addPoints(o);
    analytics.track(o);
    // every new reaction means editing OrderService
}
```

`OrderService` is coupled to every consumer, violates OCP (new reactions require modification) and SRP (it now coordinates unrelated concerns).

The alternative of consumers **polling** the subject ("anything new?") wastes resources and adds delay.

#### Solution

- **Subject (publisher):** maintains a list of observers and provides methods to `subscribe`/`unsubscribe`. When its state changes, it calls `notify` on each observer.
- **Observer (subscriber) interface:** declares the update method.
- **Concrete observers:** implement their reaction.

The subject knows only the observer *interface*, not the concrete classes.

```java
interface OrderObserver {
    void onOrderPlaced(Order order);
}

class OrderService {
    private final List<OrderObserver> observers = new CopyOnWriteArrayList<>();

    void subscribe(OrderObserver o)   { observers.add(o); }
    void unsubscribe(OrderObserver o) { observers.remove(o); }

    void placeOrder(Order order) {
        repository.save(order);
        observers.forEach(o -> o.onOrderPlaced(order));     // notify all
    }
}

class EmailNotifier implements OrderObserver {
    public void onOrderPlaced(Order o) { /* send confirmation email */ }
}

class InventoryUpdater implements OrderObserver {
    public void onOrderPlaced(Order o) { /* decrement stock */ }
}

class LoyaltyPointsAwarder implements OrderObserver {
    public void onOrderPlaced(Order o) { /* add points */ }
}

// Wiring
orderService.subscribe(new EmailNotifier());
orderService.subscribe(new InventoryUpdater());
orderService.subscribe(new LoyaltyPointsAwarder());
// New reaction (e.g., fraud check)? Add an observer; OrderService unchanged.
```

#### Push vs pull

- **Push model:** the subject sends the relevant data with the notification (`onOrderPlaced(order)`). Simple, but the subject decides what data observers get.
- **Pull model:** the subject sends minimal information (or itself), and observers query what they need. More flexible, but observers become coupled to the subject's API.

#### Observer vs Pub/Sub with a message broker

| | Classic Observer | Pub/Sub (event bus / broker) |
|---|---|---|
| Coupling | Subject holds direct references to observers | Publishers and subscribers don't know each other; a broker/channel sits between them |
| Delivery | Usually synchronous, in-process | Often asynchronous, can be cross-process (Kafka, RabbitMQ, SNS/SQS, Redis Pub/Sub) |
| Failure isolation | A slow or failing observer affects the subject | Isolated; messages can be retried and persisted |

Spring's `ApplicationEventPublisher` with `@EventListener` is an in-process event bus; with `@TransactionalEventListener` the event fires only after the transaction commits. For cross-service events, the outbox pattern ensures events are published reliably.

#### Real-world uses

- UI event listeners: `button.addActionListener(...)`, DOM `addEventListener`, Android click listeners.
- `java.beans.PropertyChangeListener`; `java.util.Observer` (deprecated since Java 9 in favor of better alternatives).
- Reactive programming: RxJava, Project Reactor, RxJS (`Observable.subscribe`), Java's `Flow` API.
- Front-end state management and reactivity: Redux store subscriptions, Vue/MobX reactivity, Angular's `EventEmitter`.
- Webhooks, domain events, event-driven microservices, model-view-controller (views observe the model).

#### When to use

- A change in one object requires changes in others, and you don't know (or don't want to hard-code) how many or which.
- Objects should be able to subscribe and unsubscribe dynamically.
- Decoupling core logic from side effects (notifications, analytics, audit).

#### Pros and cons

| Pros | Cons |
|---|---|
| Loose coupling: subject knows only the observer interface | **Notification order** is usually unspecified; don't rely on it |
| Add new observers without changing the subject (OCP) | Hard to trace flow: "what happens when an order is placed?" requires finding all subscribers |
| Supports broadcast communication | **Memory leaks:** observers that are never unsubscribed stay referenced (the "lapsed listener" problem); use weak references or explicit unsubscription |
| Dynamic relationships at runtime | Synchronous observers can slow down the subject; an exception in one observer can stop others or the subject |
| | Cascading updates (observers triggering other notifications) can cause loops or unexpected chains |

**Related:** Mediator (centralizes communication between objects; observers communicate via notifications), Chain of Responsibility (passes a request until one handler handles it; Observer notifies everyone), Singleton (an event bus is often a single instance).

---

### 6.3 Chain of Responsibility

#### Intent

**Pass a request along a chain of handlers. Each handler decides either to process the request or to pass it to the next handler in the chain.** The sender doesn't know which handler will ultimately handle it.

#### Problem

Every incoming API request must go through several checks: authentication, authorization, rate limiting, input validation, and caching, before reaching the business logic. Putting all of these in one method creates a huge, tangled function. The checks also differ per endpoint and their order matters. Similar situations: customer-support ticket escalation (bot → agent → supervisor → manager), expense approvals by amount (manager up to ₹10,000, director up to ₹1,00,000, CFO above), logging levels.

#### Solution

- **Handler interface:** declares a method to handle a request and (usually) a way to set the next handler.
- **Base handler** (optional): stores the next handler and forwards by default.
- **Concrete handlers:** each contains one check or processing step. A handler can:
  - Handle the request and **stop** the chain (e.g., reject unauthenticated requests).
  - Do its work and **pass it on**.
  - Simply pass it on if it's not responsible.

```java
abstract class RequestHandler {
    private RequestHandler next;

    public RequestHandler setNext(RequestHandler next) {
        this.next = next;
        return next;                         // allows chaining setup calls
    }

    public void handle(Request request) {
        if (next != null) next.handle(request);
    }
}

class AuthenticationHandler extends RequestHandler {
    public void handle(Request req) {
        if (req.token() == null || !tokenService.isValid(req.token()))
            throw new UnauthorizedException();           // stop the chain
        super.handle(req);                               // pass along
    }
}

class RateLimitHandler extends RequestHandler {
    public void handle(Request req) {
        if (!rateLimiter.tryAcquire(req.userId()))
            throw new TooManyRequestsException();
        super.handle(req);
    }
}

class ValidationHandler extends RequestHandler {
    public void handle(Request req) {
        if (req.body() == null) throw new BadRequestException("Body required");
        super.handle(req);
    }
}

class BusinessLogicHandler extends RequestHandler {
    public void handle(Request req) {
        orderService.process(req);           // final handler; doesn't call next
    }
}

// Build the chain
RequestHandler chain = new AuthenticationHandler();
chain.setNext(new RateLimitHandler())
     .setNext(new ValidationHandler())
     .setNext(new BusinessLogicHandler());

chain.handle(incomingRequest);
```

**Approval chain variant** (exactly one handler processes the request):
```java
abstract class Approver {
    protected Approver next;
    void setNext(Approver next) { this.next = next; }
    abstract void approve(Expense e);
}

class Manager extends Approver {
    void approve(Expense e) {
        if (e.amount() <= 10_000) System.out.println("Manager approved");
        else if (next != null) next.approve(e);
    }
}

class Director extends Approver {
    void approve(Expense e) {
        if (e.amount() <= 100_000) System.out.println("Director approved");
        else if (next != null) next.approve(e);
    }
}

class Cfo extends Approver {
    void approve(Expense e) { System.out.println("CFO approved"); }
}
```

#### Two flavors

1. **Classic (GoF):** the request travels until **one** handler handles it, then stops (approvals, event bubbling in UIs, exception handling up a call stack).
2. **Pipeline/middleware:** **every** handler may do some work and pass the request along, and any handler may stop it early (servlet filters, Express middleware).

Always decide what happens if nobody handles the request: a default final handler, an error, or silently ignoring it.

#### Real-world uses

- **Servlet filters** (`FilterChain.doFilter()`), **Spring Security's filter chain**.
- **Middleware:** Express.js/Koa (`next()`), ASP.NET Core middleware, Django middleware, Go HTTP middleware.
- Spring MVC `HandlerInterceptor`s, Netty `ChannelPipeline`, OkHttp interceptors, Axios interceptors.
- Logging frameworks: loggers pass records up to parent loggers.
- DOM event bubbling: an event goes from the clicked element up through its ancestors until a handler stops propagation.
- Exception handling: a thrown exception travels up the call stack until a matching `catch`.
- Support escalation, approval workflows, fraud check pipelines.

#### When to use

- More than one object may handle a request, and the handler isn't known in advance.
- You want to process a request through a configurable sequence of steps.
- The set and order of handlers should be changeable at runtime or per configuration.
- You want to decouple the sender from the receivers.

#### Pros and cons

| Pros | Cons |
|---|---|
| Sender decoupled from receivers | **No guarantee a request is handled** unless a default handler exists |
| Each handler has one responsibility (SRP) | Harder to debug: tracing which handler did what across the chain |
| Add, remove, or reorder handlers without changing others (OCP) | Long chains can hurt performance |
| Flexible per-route or per-config chains | Order dependencies between handlers can be subtle (authentication must precede authorization) |

**Related:** Decorator (similar structure; decorators typically always forward and add behavior, while chain handlers can stop the request), Composite (requests often travel up a composite tree to parents), Command (requests in the chain can be command objects), Observer (notifies all subscribers rather than finding one handler).

---

### 6.4 Template Method

#### Intent

**Define the skeleton of an algorithm in a base-class method, deferring some steps to subclasses.** Subclasses redefine certain steps without changing the algorithm's overall structure.

#### Problem

A data-import feature reads CSV, JSON, and XML files. Each importer follows the same process: open the source, parse records, validate them, transform them, save them to the database, and log a summary. Only parsing differs between formats. Writing three separate importers duplicates the shared steps; fixing a bug in saving or validation means fixing it in three places, and the processes gradually drift apart.

#### Solution

- An **abstract class** defines a **template method** that calls the steps in a fixed order. It's typically `final` so subclasses can't change the sequence.
- Steps are of three kinds:
  - **Concrete steps:** implemented in the base class, shared by all (`save`, `logSummary`).
  - **Abstract steps:** must be implemented by subclasses (`parse`).
  - **Hooks:** optional steps with a default (often empty) implementation that subclasses *may* override (`beforeImport`, `shouldValidate`).

```java
abstract class DataImporter {

    // The template method: fixed algorithm, can't be overridden
    public final ImportReport importFile(Path file) {
        beforeImport(file);                                     // hook
        String raw = readFile(file);                            // concrete
        List<Record> records = parse(raw);                      // abstract: varies
        if (shouldValidate()) records = validate(records);      // hook decides
        List<Record> transformed = transform(records);          // overridable, default provided
        save(transformed);                                      // concrete
        return buildReport(transformed);                        // concrete
    }

    protected abstract List<Record> parse(String raw);

    protected void beforeImport(Path file) { }                  // hook: no-op by default
    protected boolean shouldValidate() { return true; }         // hook with default
    protected List<Record> transform(List<Record> r) { return r; }

    private String readFile(Path p) { /* shared */ return ""; }
    private List<Record> validate(List<Record> r) { /* shared */ return r; }
    private void save(List<Record> r) { /* shared database logic */ }
    private ImportReport buildReport(List<Record> r) { /* shared */ return null; }
}

class CsvImporter extends DataImporter {
    protected List<Record> parse(String raw) { /* CSV parsing */ return List.of(); }
}

class JsonImporter extends DataImporter {
    protected List<Record> parse(String raw) { /* JSON parsing */ return List.of(); }
    @Override protected boolean shouldValidate() { return false; }   // already schema-validated
}

DataImporter importer = new CsvImporter();
importer.importFile(Path.of("customers.csv"));
```

#### The Hollywood Principle

> **"Don't call us, we'll call you."**

The base class controls the flow and calls the subclass's methods at the right time; subclasses don't call the base class's algorithm. This inversion of control is also how frameworks work: the framework defines the lifecycle and calls your code at hook points (e.g., servlet `doGet`, JUnit `@BeforeEach`, Android `onCreate`).

#### Real-world uses

- `java.io.InputStream`: `read(byte[], int, int)` is implemented in terms of the abstract `read()`.
- `AbstractList`, `AbstractMap`: implement `get()` and `size()`, and you get iterators, `contains`, `equals`, etc.
- `HttpServlet.service()` dispatches to `doGet()`, `doPost()` that you override.
- Spring's `JdbcTemplate`, `RestTemplate`, `TransactionTemplate`: the "template" name comes from this idea, though they mostly use callbacks (lambdas) for the varying step, a composition-based variant.
- Test frameworks: setup → test → teardown lifecycle.
- Game loops: `initialize()`, `update()`, `render()`.

#### Template Method vs Strategy

Both let you vary part of an algorithm:

| | Template Method | Strategy |
|---|---|---|
| Mechanism | **Inheritance**: subclass overrides steps | **Composition**: object delegates to an injected strategy |
| What varies | **Some steps** of a fixed algorithm | The **whole algorithm** |
| When chosen | Compile time (which subclass you instantiate) | Runtime (can swap) |
| Coupling | Subclass tightly coupled to base class | Looser |

The callback variant (pass a lambda for the varying step, as `JdbcTemplate` does) is essentially Template Method implemented with Strategy, and is often preferred in modern code because it avoids inheritance.

#### When to use

- Several classes share the same algorithm structure but differ in specific steps.
- You want to eliminate duplicated code while enforcing a consistent process.
- You're building a framework that should control the flow while letting users customize specific points.

#### Pros and cons

| Pros | Cons |
|---|---|
| Removes duplication; shared logic in one place | Inheritance-based: subclasses are coupled to the base class (fragile base class problem) |
| Enforces a consistent algorithm structure | Fixed skeleton: hard to vary step order |
| Controlled extension points (only designated steps can change) | Can violate LSP if a subclass suppresses a step in a way that breaks expectations |
| | Hard to follow when the flow jumps between base class and subclasses; harder with many steps |

**Related:** Strategy (composition alternative), Factory Method (a template method step that creates an object), Hook methods in frameworks.

---

### 6.5 State

#### Intent

**Allow an object to alter its behavior when its internal state changes.** The object appears to change its class. Each state is represented by a separate class, and the object delegates state-dependent behavior to its current state object.

#### Problem

An order goes through states: `NEW → PAID → SHIPPED → DELIVERED`, and can be `CANCELLED` from some of them. What each action does depends on the current state: you can cancel a paid order (with a refund) but not a delivered one; you can ship a paid order but not a new one. Implemented with conditionals:

```java
// BAD: every method switches on state
class Order {
    private String status = "NEW";

    void pay() {
        switch (status) {
            case "NEW" -> status = "PAID";
            case "PAID", "SHIPPED", "DELIVERED" -> throw new IllegalStateException("Already paid");
            case "CANCELLED" -> throw new IllegalStateException("Order cancelled");
        }
    }

    void ship()   { switch (status) { /* ... */ } }
    void cancel() { switch (status) { /* ... */ } }
    void deliver(){ switch (status) { /* ... */ } }
}
```

- Every method has a switch over all states; adding a state (e.g., `RETURNED`) means editing every method.
- Rules for one state are scattered across methods, making them hard to see and verify.
- Easy to make invalid transitions by accident.

#### Solution

- **State interface:** declares methods for every state-dependent action.
- **Concrete states:** one class per state, implementing the behavior valid in that state, including **transitions** to the next state.
- **Context:** the object whose behavior changes (`Order`). It holds the current state and delegates actions to it.

```java
interface OrderState {
    default void pay(Order o)     { throw new IllegalStateException("Cannot pay in " + name()); }
    default void ship(Order o)    { throw new IllegalStateException("Cannot ship in " + name()); }
    default void deliver(Order o) { throw new IllegalStateException("Cannot deliver in " + name()); }
    default void cancel(Order o)  { throw new IllegalStateException("Cannot cancel in " + name()); }
    String name();
}

class NewState implements OrderState {
    public void pay(Order o)    { o.setState(new PaidState()); }
    public void cancel(Order o) { o.setState(new CancelledState()); }
    public String name() { return "NEW"; }
}

class PaidState implements OrderState {
    public void ship(Order o)   { o.setState(new ShippedState()); }
    public void cancel(Order o) { o.refund(); o.setState(new CancelledState()); }
    public String name() { return "PAID"; }
}

class ShippedState implements OrderState {
    public void deliver(Order o) { o.setState(new DeliveredState()); }
    public String name() { return "SHIPPED"; }
}

class DeliveredState implements OrderState {
    public String name() { return "DELIVERED"; }      // terminal: all actions invalid
}

class CancelledState implements OrderState {
    public String name() { return "CANCELLED"; }      // terminal
}

// Context
class Order {
    private OrderState state = new NewState();

    void setState(OrderState s) { this.state = s; }
    String status() { return state.name(); }
    void refund() { /* call payment service */ }

    // Public actions delegate to the current state
    void pay()     { state.pay(this); }
    void ship()    { state.ship(this); }
    void deliver() { state.deliver(this); }
    void cancel()  { state.cancel(this); }
}

Order order = new Order();
order.pay();        // NEW → PAID
order.ship();       // PAID → SHIPPED
order.cancel();     // throws: Cannot cancel in SHIPPED
```

Each state's rules are now in one place. Adding a `ReturnedState` means adding a class and updating only the states that can transition to it. Invalid actions fail clearly.

**Design choices:**
- **Who triggers transitions:** states (as above) or the context. State-driven transitions keep the rules together, but states then know about each other.
- **State object lifetime:** stateless state objects can be shared singletons (or enum constants) rather than created on each transition.
- **Persistence:** in a database, store the state *name* (`status = 'PAID'`) and map it back to the state object when loading.

**Enum-based state machine** (concise for simpler cases):
```java
enum TrafficLight {
    RED    { TrafficLight next() { return GREEN; } },
    GREEN  { TrafficLight next() { return YELLOW; } },
    YELLOW { TrafficLight next() { return RED; } };
    abstract TrafficLight next();
}
```

#### Real-world uses

- Order, payment, and shipment lifecycles; document workflows (draft → review → published); ticket systems (open → in progress → resolved → closed).
- TCP connections (LISTEN, SYN_SENT, ESTABLISHED, CLOSE_WAIT, ...), the GoF book's original example.
- Vending machines, ATMs, elevators, traffic lights: classic low-level design interview problems.
- Media players (playing, paused, stopped), game characters (idle, running, jumping, attacking).
- Libraries: Spring State Machine, XState (JavaScript), AWS Step Functions (state machines as a service).

#### When to use

- An object's behavior depends heavily on its state, and it must change behavior at runtime.
- Methods contain large conditionals on the same state field.
- The state transition rules are complex and need to be explicit and maintainable.

#### Pros and cons

| Pros | Cons |
|---|---|
| Each state's behavior is localized in its own class (SRP) | Many classes for many states |
| Adding states doesn't require editing a giant switch (OCP) | Overkill for objects with two or three simple states (an enum + switch may be clearer) |
| Transitions are explicit, and invalid transitions are easy to reject | States coupled to each other when they trigger transitions |
| Removes duplicated conditionals | Whole picture of the state machine is spread across classes (a diagram helps) |

**Related:** Strategy (same structure; see Section 7), Singleton/Flyweight (share stateless state objects), Template Method.

---

## 7. Commonly Confused Patterns

Several patterns have nearly identical class diagrams. What distinguishes them is **intent**: the problem each one solves.

### 7.1 Strategy vs State

Both have a context that delegates to an interchangeable object behind an interface.

| | Strategy | State |
|---|---|---|
| Purpose | Choose **how** to do something (which algorithm) | Change behavior based on **what condition** the object is in |
| Who switches | Usually the **client** chooses the strategy from outside | The object **switches itself** (state objects trigger transitions) |
| Awareness | Strategies are independent and don't know about each other | States often know about other states (transitions) |
| Typical change frequency | Set once, or swapped occasionally | Changes frequently as the object moves through its lifecycle |
| Example | Sorting with a different `Comparator`; pricing by car vs bike | Order moving NEW → PAID → SHIPPED |

A quick test: if the client says "use this algorithm," it's Strategy. If the object says "I'm now in a different phase, so I behave differently," it's State.

### 7.2 Decorator vs Proxy vs Adapter

All three wrap another object.

| | Adapter | Decorator | Proxy |
|---|---|---|---|
| Interface | **Changes** it (converts to what the client expects) | **Same** as the wrapped object | **Same** as the wrapped object |
| Purpose | Make incompatible things work together | **Add** behavior or responsibilities | **Control access** to the object |
| Who creates the wrapped object | Client usually passes it in | Client passes it in; decorators are stacked by the client | Proxy often creates or manages the real object itself (lazy loading, remote) |
| Stacking | Rare | Common (layers of behaviors) | Usually one layer |
| Example | `InputStreamReader` (bytes → chars) | `BufferedInputStream` adds buffering | Hibernate lazy entity, Spring `@Transactional` proxy |

The line between Decorator and Proxy is blurry in practice (a caching wrapper could be described either way). The intent question helps: "Am I enriching what the object does?" (Decorator) vs "Am I controlling whether, when, or how the object is reached?" (Proxy).

### 7.3 Facade vs Adapter

| | Facade | Adapter |
|---|---|---|
| Wraps | A whole subsystem (many classes) | Usually one class |
| Interface | Defines a **new, simpler** one | Converts to an **existing, expected** one |
| Goal | Simplify usage | Achieve compatibility |

### 7.4 Factory Method vs Abstract Factory vs Builder

| | Factory Method | Abstract Factory | Builder |
|---|---|---|---|
| Creates | One product (type decided by subclass or parameter) | A **family** of related products | One **complex** product |
| How | In one call | One call per product, from one factory | **Step by step**, then `build()` |
| Main concern | Which class to instantiate | Consistency across related products | Many parameters, optional parts, validation, readability |
| Example | `NotificationFactory.create("SMS")` | `DarkThemeFactory` making buttons and checkboxes | `HttpRequest.builder().header(...).build()` |

### 7.5 Template Method vs Strategy

Covered in Section 6.4: Template Method varies **steps** through **inheritance**; Strategy varies the **whole algorithm** through **composition**.

### 7.6 Chain of Responsibility vs Decorator vs Observer

| | Chain of Responsibility | Decorator | Observer |
|---|---|---|---|
| Flow | Request passes along handlers in sequence | Call passes through wrapper layers | Event broadcast to all subscribers |
| Can stop early | Yes, any handler can stop it | Normally no (always forwards) | No; every observer is notified |
| Purpose | Find a handler / process in stages | Add behavior | Notify interested parties |

### 7.7 Composite vs Decorator

Both use recursive composition. A **Composite** has *many* children and **aggregates** their results (a folder summing its files' sizes). A **Decorator** has *exactly one* wrapped object and **adds** behavior to it.

---

## 8. How Patterns Relate to SOLID

Design patterns are largely concrete applications of SOLID ideas:

| Pattern | Main SOLID principles supported | How |
|---|---|---|
| **Strategy** | OCP, SRP, DIP | New algorithms as new classes; each algorithm isolated; context depends on the strategy interface |
| **Factory Method / Simple Factory** | DIP, OCP, SRP | Clients depend on product interfaces; creation logic centralized |
| **Abstract Factory** | DIP, OCP | Client depends on abstract factory and products; new families added without changing clients |
| **Builder** | SRP | Construction logic separated from the object's behavior |
| **Adapter** | OCP, DIP, SRP | Integrate without modifying existing code; business code depends on its own interface, not a vendor's |
| **Decorator** | OCP, SRP | Add behavior without modifying the class; one concern per decorator |
| **Facade** | Reduces coupling (Law of Demeter); SRP for orchestration | Clients know one entry point instead of many subsystem classes |
| **Proxy** | OCP, SRP | Access control, caching, or laziness added without changing the real object |
| **Composite** | OCP, LSP | New component types added freely; leaves and composites substitutable through one interface |
| **Observer** | OCP, DIP | New reactions added without changing the subject; subject depends only on the observer interface |
| **Chain of Responsibility** | SRP, OCP | One concern per handler; handlers added or reordered without changing others |
| **Template Method** | OCP (for designated steps); DRY | Algorithm structure fixed; steps extended through subclasses |
| **State** | SRP, OCP | Each state's behavior in its own class; new states without editing a giant switch |
| **Singleton** | Often **violates** SRP and DIP | Manages its own lifecycle; clients reach for a concrete global instead of receiving an abstraction |

Recurring themes behind almost every pattern:
- **Program to an interface**, not an implementation.
- **Favor composition over inheritance.**
- **Encapsulate what varies:** identify the part of the code likely to change and put it behind an abstraction (the algorithm in Strategy, the creation in Factory, the state-specific behavior in State).

---

## 9. Revision Summary

### SOLID

| Principle | Core idea | Classic violation | Fix |
|---|---|---|---|
| **SRP** | One reason to change | `User` class that saves, logs, and emails | Separate `UserRepository`, `Logger`, `EmailService`; an orchestrating service |
| **OCP** | Extend without modifying | `if/else` on payment type that grows with each method | `PaymentMethod` interface, one class per method |
| **LSP** | Subtypes must honor the parent's contract | `Penguin extends Bird` throwing on `fly()`; `Square extends Rectangle` | Model capabilities (`Flyable`); avoid inheritance that breaks invariants |
| **ISP** | No client forced to depend on unused methods | `Robot` implementing `Worker.eat()` | Split into `Workable`, `Eatable`, `Restable` |
| **DIP** | Depend on abstractions; details depend on abstractions | `Switch` creating `new Fan()` internally | `Switch` receives a `Switchable` via constructor injection |

### Design patterns

| Pattern | Category | One-line purpose | Recognize it by |
|---|---|---|---|
| **Singleton** | Creational | Exactly one instance with global access | Private constructor + static `getInstance()`; prefer DI-managed single instances |
| **Factory Method** | Creational | Defer which class to instantiate | A method returning an interface type, chosen by subclass or parameter |
| **Abstract Factory** | Creational | Create families of compatible objects | A factory interface with several `createX()` methods, one implementation per family |
| **Builder** | Creational | Construct complex objects step by step | Fluent chained setters ending in `build()` |
| **Adapter** | Structural | Make an incompatible interface fit | Wrapper implementing *your* interface, calling a *different* API |
| **Decorator** | Structural | Add behavior dynamically by wrapping | Same interface, wraps one object, stackable layers |
| **Facade** | Structural | Simplify a complex subsystem | One class with high-level methods delegating to many services |
| **Proxy** | Structural | Control access to an object | Same interface; lazy loading, access checks, caching, remote calls |
| **Composite** | Structural | Treat trees of objects uniformly | Container holding children of its own interface type; recursive operations |
| **Strategy** | Behavioral | Interchangeable algorithms | Context holding an injected algorithm interface (often a lambda) |
| **Observer** | Behavioral | One-to-many change notification | `subscribe`/`notify`, listeners, event handlers |
| **Chain of Responsibility** | Behavioral | Pass a request through handlers | Handlers with a `next` reference; middleware, filters |
| **Template Method** | Behavioral | Fixed algorithm with customizable steps | `final` method in an abstract class calling abstract or hook methods |
| **State** | Behavioral | Behavior depends on internal state | One class per state; context delegates; states trigger transitions |

### Key takeaways

- **SOLID is about managing change:** small focused classes (SRP), extension through new code (OCP), trustworthy polymorphism (LSP), lean interfaces (ISP), and business logic independent of details (DIP).
- **Patterns are named solutions to recurring problems**; identify the problem first, then the pattern. Use them where they reduce complexity, not to show off.
- **Composition, interfaces, and encapsulating what varies** are the ideas underneath nearly every pattern.
- **Patterns with similar structures differ by intent:** Strategy vs State, Decorator vs Proxy vs Adapter, Facade vs Adapter.
- **Singleton** is easy to write but introduces global state and hidden dependencies; prefer a single instance managed by dependency injection.
- **Avoid over-engineering:** an interface with one implementation that will never vary, or a pattern applied to a two-line problem, adds cost without benefit.