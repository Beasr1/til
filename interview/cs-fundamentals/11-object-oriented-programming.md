# 11. Object-oriented programming

## The problem

In a large program, the hardest bugs come from state that anything can change. An
`Order`'s status is set from twelve places, one of them skips a check, and the order
ships without being paid for. Object-oriented programming's core answer is to put data
together with the only code allowed to change it, and to let the rest of the program
depend on *what an object can do* rather than *what it is*. The interview staples (the
four pillars, composition vs inheritance, SOLID, a few design patterns) are all
elaborations of those two ideas.

This chapter matters more than usual for a polyglot: the four languages here disagree
about what OOP even requires. C++ has classes and multiple inheritance. Go has no
inheritance at all. Rust has traits and no inheritance. JavaScript has prototypes with
class syntax on top. Seeing the same ideas built four ways is the fastest way to tell
which parts of OOP are essential and which are one language's habit.

Assumes nothing beyond [file 01](01-what-happens-when-a-program-runs.md). Examples are in
Go first, with Rust, C++ and JavaScript in collapsible blocks. Every example was compiled
and run.

## 11.1 Two definitions of OOP, and they disagree

The definition interviews use is the **four pillars**: encapsulation, abstraction,
inheritance, polymorphism. The person who coined the term had something else in mind.
Alan Kay, in a 2003 email ([text](http://userpage.fu-berlin.de/~ram/pub/pub_jf47ht81Ht/doc_kay_oop_en)):
"OOP to me means only messaging, local retention and protection and hiding of
state-process, and extreme late-binding of all things." No mention of classes or
inheritance.

The two lists overlap on hiding state (encapsulation) and late binding (polymorphism).
Inheritance is on one list and not the other, and that's exactly the feature the newer
languages dropped. The Go FAQ answers "Is Go an object-oriented language?" with "Yes and
no. Although Go has types and methods and allows an object-oriented style of programming,
there is no type hierarchy" ([Go FAQ](https://go.dev/doc/faq#is-go-an-object-oriented-language)).
The Rust book: "If a language must have inheritance to be object oriented, then Rust is
not such a language" ([ch18-01](https://doc.rust-lang.org/book/ch18-01-what-is-oo.html)).

So give interviewers the four pillars, because that's the question being asked. But know
that "OOP requires inheritance" is a position, not a fact.

## 11.2 The four pillars, each with its problem

| Pillar | The problem it solves | In one line |
|---|---|---|
| **Encapsulation** | Anyone can put an object into an invalid state | Hide the data; expose only operations that keep it valid |
| **Abstraction** | Callers depend on details that will change | Expose *what* something does, hide *how* |
| **Inheritance** | Several types share most of their code | Define a new type as an extension of an existing one |
| **Polymorphism** | Code must handle many types without a growing `switch` on type | One interface, many implementations; the right one chosen at run time (or compile time) |

Encapsulation and abstraction are easy to confuse. Encapsulation is about *protecting
state*: you can't set `balance` directly, only call `Deposit`. Abstraction is about
*hiding complexity behind an interface*: you call `Store.Save(order)` without knowing
whether it's Postgres or a file. One is a wall, the other a simplified front door. They
usually come together.

### How each language does them

| | C++ | Go | Rust | JavaScript |
|---|---|---|---|---|
| **Encapsulation** | `public` / `protected` / `private` per class member | Capitalised names are exported from the *package*; lower-case ones are package-private. No per-type privacy | Private by default; `pub` per item, at *module* level | `#private` fields and methods (ES2022), enforced by the language; before that, closures or convention (`_name`) |
| **Abstraction** | Abstract classes (pure virtual functions) | Interfaces, **satisfied implicitly** | Traits | Duck typing; no language-level interfaces |
| **Inheritance** | Classes, including **multiple** inheritance and `virtual` inheritance | **None.** Struct embedding promotes methods but creates no subtype | **None** for data; traits can have default methods and can require other traits | Single inheritance through the **prototype chain**; `class … extends` is built on it (§11.2's note on "sugar") |
| **Polymorphism** | `virtual` functions (dynamic); templates (static); function overloading | Interfaces (dynamic); generics (static, since 1.18) | Trait objects `dyn Trait` (dynamic); generics (static, monomorphised) | Always dynamic: method lookup along the prototype chain at call time |

Sources for the less obvious cells: C++ allows a class to be "derived from one or more
base classes" ([cppreference](https://en.cppreference.com/w/cpp/language/derived_class));
Go's spec says "An identifier is exported if … the first character of the identifier's
name is a Unicode uppercase letter" ([spec](https://go.dev/ref/spec#Exported_identifiers));
Rust's book says "by default everything else is private"; MDN says JavaScript "Classes are
in fact 'special functions'" and "A class can only have a single superclass"
([Classes](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Classes),
[extends](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Classes/extends)).

Is a JavaScript class "just syntactic sugar" over prototypes? MDN disagrees with itself.
Its prototype-chain guide says "Classes are syntax sugar over constructor functions" and
"classes do not bring a new inheritance pattern"
([guide](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Inheritance_and_the_prototype_chain)),
while its Classes reference says "Classes in JS are built on prototypes but also have some
syntax and semantics that are unique to classes". Both are true at different levels. The
*inheritance model* is unchanged, which is what the guide means. But some behaviour has no
pre-class equivalent: `#private` fields, and constructors that refuse to run as plain
functions. Calling `A()` for `class A {}` throws `Class constructor A cannot be invoked
without 'new'` (Node 24), where an old-style constructor function would happily run. So
"sugar for the prototype chain, plus a few real additions" is the accurate answer.

## 11.3 Polymorphism, four ways

One task in each language: a list of shapes of different types, and a total area that
doesn't care which types they are.

```go
// Abstraction: callers depend on this, not on any concrete shape.
type Shape interface {
	Area() float64
}

type Rect struct{ W, H float64 }
type Circle struct{ R float64 }

// No "implements" keyword: having the method is enough.
func (r Rect) Area() float64   { return r.W * r.H }
func (c Circle) Area() float64 { return math.Pi * c.R * c.R }

func total(shapes []Shape) float64 {
	sum := 0.0
	for _, s := range shapes {
		sum += s.Area() // which Area runs is decided at run time
	}
	return sum
}

// total([]Shape{Rect{3, 4}, Circle{1}}) → 15.14
```

Go's distinctive choice is **implicit satisfaction**: `Rect` never mentions `Shape`. In the
FAQ's words, "in Go a type automatically satisfies any interface that specifies a subset
of its methods" ([FAQ](https://go.dev/doc/faq#inheritance)). The consumer defines the
interface it needs, and existing types fit it without being changed, which is why Go
interfaces tend to be small and declared where they're used.

<details><summary>Rust</summary>

```rust
use std::f64::consts::PI;

trait Shape {
    fn area(&self) -> f64;
}

struct Rect { w: f64, h: f64 }
struct Circle { r: f64 }

impl Shape for Rect { fn area(&self) -> f64 { self.w * self.h } }   // explicit impl
impl Shape for Circle { fn area(&self) -> f64 { PI * self.r * self.r } }

// Dynamic dispatch: one compiled function, a vtable lookup per call.
fn total_dyn(shapes: &[Box<dyn Shape>]) -> f64 {
    shapes.iter().map(|s| s.area()).sum()
}

// Static dispatch: the compiler stamps out a copy per concrete type.
fn area_of<S: Shape>(s: &S) -> f64 {
    s.area()
}

fn main() {
    let shapes: Vec<Box<dyn Shape>> =
        vec![Box::new(Rect { w: 3.0, h: 4.0 }), Box::new(Circle { r: 1.0 })];
    println!("{:.2}", total_dyn(&shapes));                  // 15.14
    println!("{:.2}", area_of(&Rect { w: 2.0, h: 2.0 }));  // 4.00
}
```

Rust makes you choose the dispatch explicitly. The book: monomorphised generics do
"*static dispatch*", while "When we use trait objects, Rust must use dynamic dispatch …
This lookup incurs a runtime cost that doesn't occur with static dispatch"
([ch18-02](https://doc.rust-lang.org/book/ch18-02-trait-objects.html)).

</details>

<details><summary>C++</summary>

```cpp
#include <cstdio>
#include <memory>
#include <numbers>
#include <vector>

class Shape {
public:
    virtual ~Shape() = default;           // needed when deleting through a base pointer
    virtual double area() const = 0;      // pure virtual: Shape is abstract
};

class Rect : public Shape {
    double w_, h_;                        // private by default in a class
public:
    Rect(double w, double h) : w_(w), h_(h) {}
    double area() const override { return w_ * h_; }
};

class Circle : public Shape {
    double r_;
public:
    explicit Circle(double r) : r_(r) {}
    double area() const override { return std::numbers::pi * r_ * r_; }
};

int main() {
    std::vector<std::unique_ptr<Shape>> shapes;
    shapes.push_back(std::make_unique<Rect>(3, 4));
    shapes.push_back(std::make_unique<Circle>(1));
    double sum = 0;
    for (const auto& s : shapes) sum += s->area();  // virtual call
    std::printf("%.2f\n", sum);                      // 15.14
}
```

Here the relationship is declared (`: public Shape`) and the method must be marked
`virtual` to be dispatched dynamically. A non-virtual method called through a base
pointer runs the *base* version, a classic C++ surprise.

</details>

<details><summary>JavaScript</summary>

```js
class Shape {
  area() { throw new Error("not implemented"); }
}

class Rect extends Shape {
  #w; #h;                                   // private fields (ES2022)
  constructor(w, h) { super(); this.#w = w; this.#h = h; }
  area() { return this.#w * this.#h; }
}

class Circle extends Shape {
  #r;
  constructor(r) { super(); this.#r = r; }
  area() { return Math.PI * this.#r ** 2; }
}

const shapes = [new Rect(3, 4), new Circle(1), { area: () => 1 }]; // the last is duck-typed
console.log(shapes.reduce((sum, s) => sum + s.area(), 0).toFixed(2)); // 16.14
console.log(typeof Rect);                                             // "function"
console.log(Object.getPrototypeOf(Rect.prototype) === Shape.prototype); // true
```

The plain object `{ area: () => 1 }` works too: JavaScript only checks that the method
exists when it's called. And `typeof Rect` is `"function"`, with `extends` linking
prototypes, which is the prototype machinery showing through.

</details>

### Static and dynamic dispatch

| | Dynamic dispatch | Static dispatch |
|---|---|---|
| Decided | At run time, from the object's actual type | At compile time, from the declared type |
| Mechanism | Usually a **vtable**: a per-type table of function pointers the object points to | The compiler generates or picks the exact function |
| Cost | An indirect call; usually can't be inlined | Free at run time; larger binaries (one copy per type) |
| C++ | `virtual` | Templates, overloading |
| Go | Interface method calls | Generics (partly; Go's implementation shares code across some types) |
| Rust | `dyn Trait` | Generics `<T: Trait>` |
| JavaScript | Every method call (engines optimise heavily) | Not available |

The vtable is the standard implementation, not a language rule: cppreference's page on
virtual functions never mentions it, and the term comes from compiler ABIs, such as the
Itanium C++ ABI's "A virtual table (vtable) is a table of information used to dispatch
virtual functions" ([§2.5.1](https://itanium-cxx-abi.github.io/cxx-abi/abi.html)).

**Overloading vs overriding**, a common one-liner question:

| | Overloading | Overriding |
|---|---|---|
| What | Same name, different parameter types, in one scope | A subtype replaces a method of its supertype |
| Chosen | At compile time (static polymorphism) | At run time (dynamic polymorphism) |
| C++ | Yes | Yes (`virtual` … `override`) |
| Go | **No**: one name, one function per scope | No inheritance, so no overriding; interfaces fill the role |
| Rust | **No**; traits and generics instead | Trait implementations; default trait methods can be replaced |
| JavaScript | **No**: a second definition replaces the first | Yes, through the prototype chain |

## 11.4 Inheritance, and why the newer languages left it out

Inheritance does two jobs at once: it **reuses code** (the subclass gets the parent's
methods) and it **creates a subtype** (a `Dog` can be used wherever an `Animal` is
expected). Bundling them is the source of its problems:

- **The fragile base class.** A subclass depends on the parent's *implementation*, not just
  its interface. Change how a parent method works internally, perhaps by calling another
  overridable method from it, and subclasses you've never seen break.
- **Wrong hierarchies are expensive.** Taxonomies change. A `Bird` base class with `fly()`
  meets a penguin. Fixing the hierarchy means touching every subclass.
- **Multiple inheritance gets ambiguous.** If `C` inherits from `A` and `B`, which both
  inherit from `Base`, does `C` hold one `Base` or two? That's the **diamond problem**.
  C++ answers it with `virtual` inheritance: "For each distinct base class that is
  specified virtual, the most derived object contains only one base class subobject of
  that type" ([cppreference](https://en.cppreference.com/w/cpp/language/derived_class)).
  Java and JavaScript avoid it by allowing one superclass.

### The Liskov substitution principle

The rule that tells you whether a subtype relationship is legitimate. Barbara Liskov
introduced it in her OOPSLA '87 keynote, "Data Abstraction and Hierarchy" (published
1988), and Liskov and Wing formalised it ("A behavioral notion of subtyping", *TOPLAS*,
1994): objects of a subtype must be usable anywhere the supertype is expected, *without
the caller noticing*. Same methods isn't enough; same promised behaviour is required.

The classic violation: `Square extends Rectangle`. Mathematically a square is a
rectangle. But a `Rectangle` promises that `setWidth(5)` leaves the height alone, and a
`Square` can't keep that promise. Code written against `Rectangle` (set width 5, height
4, expect area 20) breaks when handed a `Square`. The is-a relationship holds in geometry
and fails in behaviour.

### Go: embedding is reuse without subtyping

Go keeps the reuse half of inheritance and drops the subtyping half:

```go
type Logger struct{ prefix string }

func (l Logger) Log(msg string) { fmt.Println(l.prefix + msg) }

type OrderService struct {
	Logger        // embedded: Log is promoted to OrderService
	orders []int // lower case: unexported outside this package
}

s := OrderService{Logger: Logger{prefix: "[orders] "}}
s.Log("started") // prints "[orders] started"; really s.Logger.Log("started")
```

Effective Go: "When we embed a type, the methods of that type become methods of the outer
type, but when they are invoked the receiver of the method is the inner type, not the
outer one" ([embedding](https://go.dev/doc/effective_go#embedding)). And an
`OrderService` is **not** a `Logger`:

```go
func useLogger(l Logger) {}
useLogger(OrderService{})
// cannot use OrderService{} (value of struct type OrderService) as Logger value in argument to useLogger
```

If you want "anything that can log", you define an interface with a `Log` method, and
`OrderService` satisfies it through the promoted method. Reuse and substitutability are
separate decisions, which is the point.

> **Teacher's aside.** "Composition vs inheritance" is usually presented as a style
> preference. Go and Rust make it concrete: they show that the two jobs inheritance
> bundles, *sharing code* and *being substitutable*, can be provided separately, by
> embedding or delegation for the first and interfaces or traits for the second. Once you
> see them as separate, most inheritance problems are visibly the result of getting one
> job when you only wanted the other. Wanted to reuse `Rectangle`'s code and accidentally
> promised `Square` behaves like one: that's the Liskov violation, in one sentence.

## 11.5 Composition over inheritance

The *Design Patterns* book (Gamma, Helm, Johnson and Vlissides, 1994, the "Gang of Four")
put it as a principle in its first chapter: favour object composition over class
inheritance. **Composition** means building an object out of other objects it holds and
delegates to, rather than out of a parent class.

| | Inheritance ("is-a") | Composition ("has-a") |
|---|---|---|
| Relationship | Fixed at compile time | Can be swapped at run time (inject a different part) |
| Coupling | To the parent's implementation | To the part's interface only |
| Testing | Hard to replace inherited behaviour | Replace a part with a fake |
| Good for | Genuine, stable is-a hierarchies (UI widget trees are the classic case) | Most business code |

## 11.6 SOLID

Five design principles, grouped and named by Robert C. Martin. Their history is messier
than most prep material says, which is worth knowing because interviewers sometimes ask
where they came from:

| Principle | Statement | Origin |
|---|---|---|
| **S** Single responsibility | A module should have one reason to change | Martin; not in his 2000 paper "Design Principles and Design Patterns" |
| **O** Open–closed | Open for extension, closed for modification | Bertrand Meyer, *Object-Oriented Software Construction* (1988); in Martin's 2000 paper |
| **L** Liskov substitution | Subtypes must be substitutable for their base types | Liskov, 1987; Liskov and Wing, 1994 |
| **I** Interface segregation | Clients shouldn't depend on methods they don't use | Martin, 2000 paper |
| **D** Dependency inversion | Depend on abstractions, not concretions | Martin, 2000 paper |

The acronym came later. Martin's own account: "Michael wrote me an e-mail, and said that
if you rearranged the principles they spell SOLID" ([InfoQ interview](https://www.infoq.com/podcasts/uncle-bob-solid-ddd)),
meaning Michael Feathers.

In Go terms, which is how you'll actually use them:

| Principle | Violation | Go-shaped fix |
|---|---|---|
| **S** | An `OrderHandler` that parses HTTP, validates, writes SQL and sends email | Separate the HTTP layer, the domain logic and the storage; each changes for its own reason |
| **O** | A `switch paymentMethod` that grows a case for every new provider | A `PaymentProvider` interface; a new provider is a new type, no edits to existing code |
| **L** | An implementation of `Store.Save` that silently drops orders over a size limit | Every implementation honours the interface's documented contract, or the interface is wrong |
| **I** | A 20-method `Repository` interface that every fake must implement in full | Small interfaces at the point of use: `type OrderSaver interface { Save(Order) error }`. Go's `io.Reader` has one method |
| **D** | Business logic constructs `postgres.NewClient()` itself | The logic accepts an interface; `main` wires in the concrete implementation |

Interface segregation and dependency inversion come almost for free in Go *because*
interfaces are satisfied implicitly: the consumer can declare exactly the one-method
interface it needs, and the concrete type never has to know.

## 11.7 Four patterns that come up

Patterns are named solutions to recurring design problems. Four are asked about far more
than the rest.

### Singleton: exactly one instance

**Problem:** some things should exist once per process: configuration, a connection pool.
**Danger:** a singleton is a global variable with good manners. It hides dependencies and
makes tests share state. Prefer creating the thing once in `main` and passing it down;
reach for a singleton only when that's impractical.

The interview follow-up is always "make it thread-safe", and each language has an idiom
that does the locking for you. Each example below was run with three threads calling it at
once and printed "loading config (runs once)" a single time:

```go
// Go 1.21+: sync.OnceValue runs the function once, even under concurrent calls.
var loadConfig = sync.OnceValue(func() *Config {
	fmt.Println("loading config (runs once)")
	return &Config{DSN: "postgres://..."}
})

cfg := loadConfig() // every call returns the same *Config
```

`sync.Once` ("an object that will perform exactly one action",
[pkg.go.dev](https://pkg.go.dev/sync#Once)) is the older form.

<details><summary>Rust</summary>

```rust
use std::sync::OnceLock; // stable since Rust 1.70

struct Config { dsn: String }

fn config() -> &'static Config {
    static CONFIG: OnceLock<Config> = OnceLock::new();
    CONFIG.get_or_init(|| {
        println!("loading config (runs once)");
        Config { dsn: "postgres://...".into() }
    })
}
// std::sync::LazyLock (stable since 1.80) does the same with less ceremony.
```

</details>

<details><summary>C++</summary>

```cpp
struct Config { std::string dsn; };

Config& config() {
    static Config instance = [] {        // C++11 "magic statics": initialised exactly once
        std::puts("loading config (runs once)");
        return Config{"postgres://..."};
    }();
    return instance;
}
```

cppreference: "If multiple threads attempt to initialize the same static local variable
concurrently, the initialization occurs exactly once"
([storage duration](https://en.cppreference.com/w/cpp/language/storage_duration)). This
replaced the double-checked locking pattern people used to hand-write, and often get
wrong.

</details>

<details><summary>JavaScript</summary>

```js
// config.mjs
console.log("loading config (runs once)");
export const config = { dsn: "postgres://..." };

// main.mjs
import { config as a } from "./config.mjs";
const { config: b } = await import("./config.mjs");
console.log(a === b); // true: the module ran once, and every importer shares its exports
```

A module is evaluated once and cached, so its exports are natural singletons (checked on
Node 24). No locking is needed: one thread.

</details>

### Factory: construction behind an interface

**Problem:** callers shouldn't need to know which concrete type to build, or how. In Go a
factory is just a constructor function returning an interface:

```go
type Notifier interface{ Notify(msg string) string }
type email struct{}
type sms struct{}

func (email) Notify(m string) string { return "email: " + m }
func (sms) Notify(m string) string   { return "sms: " + m }

func NewNotifier(kind string) (Notifier, error) {
	switch kind {
	case "email":
		return email{}, nil
	case "sms":
		return sms{}, nil
	}
	return nil, fmt.Errorf("unknown notifier %q", kind)
}
// n, _ := NewNotifier("sms"); n.Notify("order shipped") → "sms: order shipped"
```

The concrete types are unexported. Callers can only get one through the factory and only
see the interface.

### Strategy: swap the algorithm

**Problem:** the same operation needs different algorithms in different situations (a
festive discount, a regular price). Classic OOP defines a `PricingStrategy` interface with
one implementation per algorithm. In Go, Rust, C++ and JavaScript alike, a one-method
strategy is often just a function value:

```go
type PricingStrategy func(amount float64) float64

func checkout(amount float64, price PricingStrategy) float64 { return price(amount) }

festive := func(a float64) float64 { return a * 0.9 }
checkout(1000, festive)                                // 900
checkout(1000, func(a float64) float64 { return a })   // 1000
```

The pattern is "pass the varying behaviour in"; whether it's an interface, a trait object,
a `std::function` or a closure is a language detail.

### Observer: tell everyone who cares

**Problem:** when something happens (an order is placed), several unrelated parts must
react (send a receipt, reserve stock), and the code that places the order shouldn't know
about any of them.

```go
type OrderPlaced struct{ ID int }
type Bus struct{ subs []func(OrderPlaced) }

func (b *Bus) Subscribe(f func(OrderPlaced)) { b.subs = append(b.subs, f) }
func (b *Bus) Publish(e OrderPlaced) {
	for _, f := range b.subs {
		f(e)
	}
}

var bus Bus
bus.Subscribe(func(e OrderPlaced) { fmt.Println("email receipt for", e.ID) })
bus.Subscribe(func(e OrderPlaced) { fmt.Println("reserve stock for", e.ID) })
bus.Publish(OrderPlaced{ID: 42})
```

JavaScript has this built in (`EventTarget`, Node's `EventEmitter`). Across services, the
same pattern is a message broker: Kafka topics are observer at the scale of a system, with
the extra problems of delivery and ordering the
[kafka course](../../kafka/README.md) covers.

> ⚠️ **This `Bus` isn't safe for concurrent use.** `Subscribe` appends to a slice that
> `Publish` reads; called from different goroutines that's a data race
> ([file 04 §4.1](04-synchronisation-and-deadlock.md#41-a-race-watched-happening)). A real
> one needs a mutex, or subscriptions fixed at start-up.

| Pattern | Category | One-line purpose |
|---|---|---|
| Singleton | Creational | Exactly one instance, globally reachable |
| Factory | Creational | Construct without naming the concrete type |
| Strategy | Behavioural | Swap an algorithm at run time |
| Observer | Behavioural | Notify many dependants of an event, decoupled |

## Asked in interviews

- "What are the four pillars of OOP? Explain each with an example." → §11.2
- "Encapsulation vs abstraction?" → §11.2
- "Abstract class vs interface?" → §11.2 table (C++ abstract classes vs Go interfaces and Rust traits) and §11.3
- "Is Go object-oriented? How do you do inheritance in Go?" → §11.1, §11.4
- "Overloading vs overriding? Compile-time vs runtime polymorphism?" → §11.3
- "What's the diamond problem?" → §11.4
- "Composition vs inheritance?" → §11.4 aside, §11.5
- "Explain SOLID." → §11.6
- "Implement a thread-safe singleton." → §11.7

## Check yourself

1. Explain the difference between encapsulation and abstraction using a `BankAccount` type.
   Which one does a private `balance` field provide, and which one does an `AccountStore`
   interface provide?
2. In Go, `OrderService` embeds `Logger`. Can you pass an `OrderService` to a function
   taking `Logger`? Can you pass it to a function taking an interface with a `Log(string)`
   method? Explain the difference.
3. Explain why `Square extends Rectangle` violates the Liskov substitution principle, by
   writing (in words) a test that passes for `Rectangle` and fails for `Square`. How would
   you model the two instead?
4. A C++ class has a non-virtual `describe()` method that a subclass redefines. Code calls
   `describe()` through a base-class pointer to a subclass object. Which version runs, and
   why? What would happen in Go, Rust and JavaScript in the equivalent situation?
5. Your Go service's business logic calls `postgres.NewClient()` directly, and its tests
   need a running database. Which SOLID principle is involved, and what's the minimal
   change that lets the tests use an in-memory fake?
6. An interviewer asks for a thread-safe singleton in Go and you write a package-level
   variable with a nil check and a mutex. What's wrong with checking `if instance == nil`
   *before* taking the lock, and what does `sync.OnceValue` give you instead?

(Answers in [13-exercises.md](13-exercises.md).)
