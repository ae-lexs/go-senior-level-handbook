# Types and Composition

> Go models domain concepts through structs, methods, and interfaces—combined via composition, not inheritance. Types define capabilities ("what can this do?"), not taxonomies ("what is this a kind of?").

---

## Core Principle

**Discover interfaces; don't design them.**

Interfaces emerge from usage patterns. Define them at the consumer side when you need abstraction—not upfront in the producer package. Premature interface design creates complexity without benefit.

---

## Invariants

> Rules that must hold true. Violating these leads to bugs, leaks, or architectural debt.

- **The bigger the interface, the weaker the abstraction.** Favor small, cohesive interfaces. One or two methods is a good heuristic when they represent a single responsibility—more methods are acceptable when tightly cohesive (like `sort.Interface`). Large interfaces signal unclear responsibility.
- **Don't design with interfaces, discover them.** Wait until you have multiple implementations or need to decouple. The consumer defines the interface, not the producer. Exception: hard boundaries (plugins, external APIs, SDK contracts) may require upfront interface design.
- **Accept interfaces, return structs.** Functions should accept the narrowest interface that satisfies their needs and return concrete types that expose full functionality.

---

## The "Why" Behind This

Go's type system serves a specific goal: **composition over inheritance**. In inheritance-based languages, you model the world as a taxonomy—`Dog extends Animal extends LivingThing`. This creates tight coupling: change the base class, and every descendant might break. It also forces premature abstraction—you must decide the hierarchy before you understand your domain.

Go takes a different approach. Types are standalone; they don't inherit from anything. You compose behavior by embedding types and by satisfying interfaces implicitly. This means you can create a `Dog` that has an `Animal` inside it (composition), but `Dog` is not a subtype of `Animal`. The question shifts from "what is this a kind of?" to "what can this do?"

This shift has profound consequences for API design. In Go, **interfaces belong to consumers**, not producers. The package that *uses* a capability defines the interface; the package that *provides* it returns a concrete type. This is the opposite of Java-style design where interfaces live alongside their implementations. Go's implicit interface satisfaction makes this possible—a type implements an interface without declaring it, simply by having the right methods.

---

## Key Concepts

### Structs as Data + Behavior

Structs bundle data and methods. Methods are functions with a receiver—the type they operate on.

```go
type Order struct {
    ID        string
    Items     []LineItem
    CreatedAt time.Time
}

func (o *Order) Total() Money {
    var sum Money
    for _, item := range o.Items {
        sum = sum.Add(item.Subtotal())
    }
    return sum
}

func (o *Order) IsExpired(now time.Time) bool {
    return now.Sub(o.CreatedAt) > 24*time.Hour
}
```

Methods give types behavior. The receiver (`o *Order`) is explicit—no hidden `this` or `self`. This explicitness is intentional: you always know what you're operating on.

### Pointer vs Value Receivers

The receiver type determines whether methods can mutate the struct and how the struct is passed.

**Use pointer receivers when:**
- The method modifies the receiver
- The struct is large (avoids copying)
- Consistency: if any method needs a pointer receiver, use pointers for all methods

**Use value receivers when:**
- The method doesn't modify the receiver
- The struct is small and immutable (like `time.Time`)
- You want the type to behave like a primitive

**Idiomatic:**

```go
type Counter struct {
    count int
}

// Pointer receiver: modifies the struct
func (c *Counter) Increment() {
    c.count++
}

// Value receiver would be wrong here—it modifies a copy
func (c Counter) IncrementBroken() {
    c.count++ // Modifies a copy, original unchanged
}
```

**Critical rule:** Methods with pointer receivers can only be called on addressable values. This affects interface satisfaction:

```go
type Incrementer interface {
    Increment()
}

var c Counter
var i Incrementer

i = &c  // OK: *Counter has Increment()
i = c   // Compile error: Counter does not implement Incrementer
        // (Increment method has pointer receiver)
```

**Method sets and interface satisfaction:**

This behavior stems from Go's *method set* rules:
- The method set of `T` includes methods with value receiver `T`
- The method set of `*T` includes methods with *both* value receiver `T` and pointer receiver `*T`

In other words: `T`'s method set ⊂ `*T`'s method set. Interfaces are satisfied based on method sets, which is why `Counter` (value) doesn't satisfy `Incrementer` but `*Counter` (pointer) does.

### Embedding for Composition

Embedding promotes fields and methods from an inner type to the outer type. It's syntactic sugar for delegation, not inheritance.

**Idiomatic:**

```go
type Logger struct {
    out io.Writer
}

func (l *Logger) Log(msg string) {
    fmt.Fprintln(l.out, msg)
}

// Server embeds Logger—gains Log() method
type Server struct {
    *Logger
    addr string
}

func main() {
    srv := &Server{
        Logger: &Logger{out: os.Stdout},
        addr:   ":8080",
    }
    srv.Log("starting server") // Promoted from Logger
}
```

**What embedding is not:**

```go
// This is NOT inheritance. Server is not a subtype of Logger.
// You cannot pass *Server where *Logger is expected.
func LogMany(l *Logger, msgs []string) { /* ... */ }

LogMany(srv, msgs)        // Compile error
LogMany(srv.Logger, msgs) // OK—explicit access to embedded field
```

Embedding is delegation with syntactic convenience. The embedded type's methods receive the embedded type as their receiver, not the outer type. This is fundamentally different from inheritance where overridden methods receive the subclass.

**Common embedding patterns:**

```go
// Embedding sync.Mutex for convenient locking
type SafeMap struct {
    sync.Mutex
    data map[string]int
}

func (m *SafeMap) Get(key string) int {
    m.Lock()
    defer m.Unlock()
    return m.data[key]
}

// Embedding for interface composition
type ReadWriteCloser struct {
    io.Reader
    io.Writer
    io.Closer
}
```

**Embedding is an API commitment:**

When you embed a type in an exported struct, its promoted methods become part of your public API. This is a design decision, not just a convenience. If `sync.Mutex` is embedded (not a named field), callers can call `Lock()` and `Unlock()` directly. Consider whether that's intentional:

```go
// Public API includes Lock/Unlock — intentional?
type SafeMap struct {
    sync.Mutex // Promoted: callers can call SafeMap.Lock()
    data map[string]int
}

// Lock/Unlock hidden — usually better for encapsulation
type SafeMap struct {
    mu   sync.Mutex // Not promoted: implementation detail
    data map[string]int
}
```

**Anti-pattern—building type hierarchies:**

```go
// Don't do this: fake inheritance
type BaseHandler struct { /* ... */ }
type UserHandler struct { BaseHandler }
type AdminHandler struct { BaseHandler }

// This builds a taxonomy. Go doesn't want taxonomies.
// Use composition with explicit fields instead.
```

### Interfaces: Implicit and Small

Go interfaces are satisfied implicitly. Any type with the right methods implements the interface—no `implements` keyword.

**Idiomatic interface design:**

```go
// One method. Clear responsibility. Easy to implement.
type Reader interface {
    Read(p []byte) (n int, err error)
}

type Writer interface {
    Write(p []byte) (n int, err error)
}

// Compose larger interfaces from smaller ones
type ReadWriter interface {
    Reader
    Writer
}
```

The standard library's `io.Reader` and `io.Writer` are the gold standard: one method each, used everywhere, implemented by files, network connections, buffers, compression streams, and more. This is the power of small interfaces.

**Anti-pattern—kitchen sink interface:**

```go
// Too many methods. Hard to implement. Hard to mock.
// Signals unclear responsibility.
type Repository interface {
    GetUser(id string) (*User, error)
    CreateUser(u *User) error
    UpdateUser(u *User) error
    DeleteUser(id string) error
    ListUsers(filter Filter) ([]*User, error)
    GetUserByEmail(email string) (*User, error)
    // ... 10 more methods
}
```

Break large interfaces into focused ones. Let consumers compose what they need.

### Accept Interfaces, Return Structs

This principle is central to Go API design.

**Accept interfaces:** Your function depends only on the behavior it needs. This decouples you from specific implementations and enables testing.

**Return structs:** Callers get the full concrete type with all its methods and fields. They can assign it to any interface they define.

**Idiomatic:**

```go
// Accept io.Reader—works with files, buffers, network, anything
func ParseConfig(r io.Reader) (*Config, error) {
    data, err := io.ReadAll(r)
    if err != nil {
        return nil, fmt.Errorf("reading config: %w", err)
    }
    var cfg Config
    if err := json.Unmarshal(data, &cfg); err != nil {
        return nil, fmt.Errorf("parsing config: %w", err)
    }
    return &cfg, nil // Return concrete *Config
}

// Callers can use any io.Reader
cfg, err := ParseConfig(os.Stdin)
cfg, err := ParseConfig(bytes.NewReader(data))
cfg, err := ParseConfig(resp.Body)
```

**Anti-pattern—returning interface:**

```go
// DON'T: Returns interface, hiding concrete capabilities
func NewStore() Store {
    return &PostgresStore{...}
}

// Caller can't access PostgresStore-specific methods
// Caller can't add the type to interfaces they define
```

Returning interfaces is occasionally appropriate at **framework boundaries** where you genuinely need to hide implementations (plugin systems, factory patterns with multiple backends). But this is rare—default to returning concrete types.

**Why does the consumer define interfaces?**

In Go, interfaces are defined where they're *consumed*, not where they're *implemented*. The producer returns a concrete struct. The consumer defines the interface if it needs abstraction.

```go
// producer/store.go — returns concrete type
package producer

type PostgresStore struct { db *sql.DB }

func NewPostgresStore(db *sql.DB) *PostgresStore {
    return &PostgresStore{db: db}
}

func (s *PostgresStore) GetUser(id string) (*User, error) { /* ... */ }
func (s *PostgresStore) SaveUser(u *User) error { /* ... */ }
```

```go
// consumer/service.go — defines the interface it needs
package consumer

// Only the methods this package actually uses
type UserGetter interface {
    GetUser(id string) (*User, error)
}

type UserService struct {
    store UserGetter // Depends on minimal interface
}

func NewUserService(store UserGetter) *UserService {
    return &UserService{store: store}
}
```

This pattern enables testing without mocks everywhere and follows the Interface Segregation Principle naturally.

### Compile-Time Interface Checks

Verify that your type implements an interface at compile time:

```go
// Compile error if *Server doesn't implement http.Handler
var _ http.Handler = (*Server)(nil)

// Or with a zero value for struct types
var _ http.Handler = Server{}
```

Place these declarations near your type definition. They're documentation and safety in one line.

---

## Trade-Off Matrix

| If You Need... | Choose... | Accept... |
|----------------|-----------|-----------|
| Shared behavior across types | Small interfaces | No shared state, no default implementations |
| Code reuse | Embedding or explicit delegation | No method overriding, receiver is always the inner type |
| Polymorphism | Interfaces | Limited expressiveness without generics (use type parameters deliberately) |
| Testability | Accept interfaces in constructors | Consumer defines the interface |
| Extend a type's methods | Embedding | Embedded type is visible, not true subtyping |

---

## Interview Signals

| When Asked... | Demonstrate... |
|---------------|----------------|
| "Pointer vs value receiver?" | Pointer if mutating or large struct; value if small and immutable. Consistency matters—if one method needs pointer, use pointer for all. Know that `T`'s method set ⊂ `*T`'s method set. |
| "Why accept interfaces, return structs?" | Accepting interfaces decouples and enables testing. Returning structs gives callers full access; they define their own interfaces as needed. |
| "What's the difference between embedding and inheritance?" | Embedding is delegation with promoted methods. The receiver is always the inner type. No subtyping—you can't pass the outer type where the inner is expected. Embedding is also an API commitment for exported types. |
| "How big should interfaces be?" | Small and cohesive. One or two methods is a good heuristic for single-responsibility interfaces. `io.Reader` has one method and is used everywhere. More methods are fine when tightly cohesive (like `sort.Interface`). |
| "Where should interfaces be defined?" | At the consumer, not the producer. The package that uses a capability defines the interface. Exception: hard boundaries like plugins or external APIs. |

---

## Bridge to Next

Types model your domain; errors are part of your API. In Go, errors are values—not exceptions. They're returned, not thrown. How you design error types, wrap them, and handle them at boundaries is as important as your data types. The next document explores **error philosophy**: the difference between sentinel errors, typed errors, and opaque errors—and introduces the **boundary vs core** framing that runs through the rest of this handbook.

→ Continue to [Error Philosophy](03_ERROR_PHILOSOPHY.md)
