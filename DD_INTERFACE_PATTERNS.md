# Interface Patterns

> Interfaces in Go are small contracts discovered through usage, not large taxonomies designed upfront. Mastering interface design means knowing when to define them, where to place them, and how to compose them for maximum flexibility with minimum coupling.

---

## Core Principle

**Interfaces belong to consumers, not producers.**

The package that *uses* a capability defines the interface; the package that *provides* it returns a concrete type. This inversion—compared to Java-style design—enables Go's implicit satisfaction model. You can define new interfaces for existing types without modifying them, decoupling code that was never designed to work together.

**The exception: ecosystem contracts.** Producer-owned interfaces are justified when you're publishing a stable, long-lived contract with backward-compatibility guarantees. The standard library's `io.Reader`, `http.Handler`, and `database/sql/driver` interfaces are producer-owned because they define ecosystem-wide extension points. For *application-internal* boundaries, consumer-owned interfaces are almost always correct.

---

## Invariants

> Rules that must hold true. Violating these leads to bugs, leaks, or architectural debt.

- **The bigger the interface, the weaker the abstraction.** One method is ideal; two is good; more than three requires strong justification (tight cohesion like `sort.Interface`). Large interfaces signal unclear responsibility.
- **Don't design with interfaces, discover them.** Wait until you have a concrete need—multiple implementations, testing requirements, or decoupling at a boundary. Premature interface design creates complexity without benefit.
- **Accept interfaces, return structs.** Functions should accept the narrowest interface that satisfies their needs and return concrete types that expose full functionality.
- **Define interfaces at the point of use.** Interfaces declared near their consumers reduce coupling and make the dependency relationship explicit.

---

## The "Why" Behind This

Go's interface system is fundamentally different from languages like Java, C#, or Rust. In those languages, a type must explicitly declare that it implements an interface. In Go, implementation is *implicit*—if a type has the right methods, it satisfies the interface, with no `implements` keyword required.

This design choice has profound consequences. You can create an interface *after* the types that implement it exist. You can define an interface in one package that's satisfied by types in completely unrelated packages. The standard library's `io.Reader` is implemented by `os.File`, `bytes.Buffer`, `net.Conn`, `strings.Reader`, compression streams, HTTP response bodies, and countless third-party types—none of which import `io` to do so.

This is Go's secret weapon for composability. Because interfaces don't create import dependencies, they don't couple code. A function accepting `io.Reader` can work with any type that has a `Read` method, including types that haven't been written yet. This is *ad-hoc polymorphism* in its purest form.

The corollary is that interfaces should be small. A large interface is harder to satisfy, harder to fake in tests, and harder to reason about. The standard library demonstrates this consistently: `io.Reader` has one method, `io.Writer` has one method, `fmt.Stringer` has one method. These tiny interfaces are used everywhere because they're easy to implement and compose.

Rob Pike's advice—"Don't design with interfaces, discover them"—captures the philosophy. In other languages, you design the interface taxonomy first, then implement concrete types. In Go, you write concrete types first, then extract interfaces when you need abstraction. This approach yields smaller, more focused interfaces that represent *actual* usage patterns rather than imagined ones.

---

## Key Concepts

### The Standard Library as Interface Design Guide

Go's standard library is a masterclass in interface design. Study these patterns:

| Interface | Methods | Used By |
|-----------|---------|---------|
| `io.Reader` | 1 (`Read`) | Files, buffers, network, compression, HTTP bodies |
| `io.Writer` | 1 (`Write`) | Files, buffers, network, stdout, loggers |
| `io.Closer` | 1 (`Close`) | Files, connections, response bodies |
| `fmt.Stringer` | 1 (`String`) | Any type with string representation |
| `error` | 1 (`Error`) | All error types |
| `sort.Interface` | 3 (`Len`, `Less`, `Swap`) | Any sortable collection |
| `http.Handler` | 1 (`ServeHTTP`) | All HTTP handlers |

Notice the pattern: most interfaces have one method. `sort.Interface` has three, but they're tightly cohesive—you can't sort without all three. This is the heuristic: interfaces should be as small as possible while remaining semantically complete.

**Naming convention:** Single-method interfaces are named by the method plus `-er` suffix: `Reader`, `Writer`, `Closer`, `Stringer`, `Handler`. This convention is so strong that deviating from it signals design problems.

### Small Interfaces Enable Composition

Small interfaces compose into larger ones through embedding:

```go
// Small, focused interfaces
type Reader interface {
    Read(p []byte) (n int, err error)
}

type Writer interface {
    Write(p []byte) (n int, err error)
}

type Closer interface {
    Close() error
}

// Composed interfaces
type ReadWriter interface {
    Reader
    Writer
}

type ReadCloser interface {
    Reader
    Closer
}

type WriteCloser interface {
    Writer
    Closer
}

type ReadWriteCloser interface {
    Reader
    Writer
    Closer
}
```

A function that needs both reading and writing accepts `ReadWriter`. A function that needs only reading accepts `Reader`. The caller passes the same concrete type to both—no adaptation required. This is interface composition in action.

**Key insight:** Composed interfaces don't need new implementations. Any type that implements `Reader`, `Writer`, and `Closer` automatically implements `ReadWriteCloser`. The composition happens at the type-checking level, not the implementation level.

### Consumer-Defined Interfaces

The consumer defines the interface; the producer returns a concrete type. This is Go's answer to the Interface Segregation Principle.

**Anti-pattern—producer-defined interface:**

```go
// storage/storage.go — WRONG: producer defines large interface
package storage

type UserRepository interface {
    Get(ctx context.Context, id string) (*User, error)
    Save(ctx context.Context, user *User) error
    Delete(ctx context.Context, id string) error
    List(ctx context.Context, filter Filter) ([]*User, error)
    GetByEmail(ctx context.Context, email string) (*User, error)
    UpdateLastLogin(ctx context.Context, id string, t time.Time) error
    // ... 10 more methods
}

type PostgresUserRepository struct { db *sql.DB }

func NewPostgresUserRepository(db *sql.DB) UserRepository {
    return &PostgresUserRepository{db: db}
}
```

This design forces every consumer to depend on all methods, even if they use only one. Testing requires implementing or mocking six methods when your test exercises one.

**Idiomatic—consumer-defined interfaces:**

```go
// storage/postgres.go — Producer returns concrete type
package storage

type PostgresUserRepository struct { db *sql.DB }

func NewPostgresUserRepository(db *sql.DB) *PostgresUserRepository {
    return &PostgresUserRepository{db: db}
}

func (r *PostgresUserRepository) Get(ctx context.Context, id string) (*User, error) { /* ... */ }
func (r *PostgresUserRepository) Save(ctx context.Context, user *User) error { /* ... */ }
func (r *PostgresUserRepository) Delete(ctx context.Context, id string) error { /* ... */ }
// ... all methods implemented
```

```go
// service/user.go — Consumer defines minimal interface
package service

// UserGetter is what this service needs—nothing more
type UserGetter interface {
    Get(ctx context.Context, id string) (*storage.User, error)
}

type UserService struct {
    users UserGetter
}

func NewUserService(users UserGetter) *UserService {
    return &UserService{users: users}
}

func (s *UserService) GetUserProfile(ctx context.Context, id string) (*Profile, error) {
    user, err := s.users.Get(ctx, id)
    if err != nil {
        return nil, fmt.Errorf("getting user: %w", err)
    }
    return buildProfile(user), nil
}
```

The service defines exactly what it needs: `UserGetter` with one method. The `PostgresUserRepository` satisfies this interface implicitly. Testing is trivial—implement a one-method fake.

**Why this matters for testing:**

```go
// service/user_test.go
type fakeUserGetter struct {
    user *storage.User
    err  error
}

func (f *fakeUserGetter) Get(ctx context.Context, id string) (*storage.User, error) {
    return f.user, f.err
}

func TestUserService_GetUserProfile(t *testing.T) {
    fake := &fakeUserGetter{
        user: &storage.User{ID: "123", Name: "Alice"},
    }
    svc := NewUserService(fake)
    
    profile, err := svc.GetUserProfile(ctx, "123")
    
    if err != nil {
        t.Fatalf("unexpected error: %v", err)
    }
    if profile.Name != "Alice" {
        t.Errorf("name = %q, want %q", profile.Name, "Alice")
    }
}
```

One-method interface means one-method fake. No mock libraries needed.

### Compile-Time Interface Satisfaction Checks

Go's implicit interface satisfaction is powerful but can hide mistakes. Use compile-time checks to verify your types implement expected interfaces:

```go
// Ensure *Server implements http.Handler at compile time
var _ http.Handler = (*Server)(nil)

// Ensure PostgresUserRepository implements storage.UserRepository
var _ storage.UserRepository = (*PostgresUserRepository)(nil)

// Using a zero value (works for non-pointer receivers)
var _ fmt.Stringer = MyType{}
```

The `var _ Interface = (*ConcreteType)(nil)` pattern creates a nil pointer of your type and assigns it to a variable of the interface type. If the type doesn't implement the interface, compilation fails. The blank identifier `_` means we don't actually use the variable—it exists only for the type check.

**Placement and caution:** Place these declarations near your type definition. They serve as documentation ("this type is intended to implement X") and as a safety net. Match the receiver form you intend—use `(*T)(nil)` for pointer receivers, `T{}` for value receivers. Avoid asserting against "kitchen-sink" interfaces; if you find yourself asserting a type against a 10-method interface, that interface is probably too large.

### The Empty Interface

The empty interface `interface{}` (or `any` since Go 1.18) accepts any value. The Go proverb "interface{} says nothing" is a warning: accepting `any` means you sacrifice compile-time type safety.

**Boundary rule:** `any` is acceptable at serialization, reflection, and plugin boundaries—but it should collapse back into typed domain structures immediately. JSON unmarshaling into `map[string]any` is fine at the boundary; passing that map through your domain logic is not.

```go
// Acceptable: boundary code that immediately types the result
func DecodeConfig(r io.Reader) (*Config, error) {
    var raw map[string]any  // any at boundary
    if err := json.NewDecoder(r).Decode(&raw); err != nil {
        return nil, err
    }
    return parseConfig(raw)  // collapses into typed Config
}
```

**`any` signals design problems when used for:** optional parameters (use functional options), heterogeneous collections in domain code (clarify the domain model), or avoiding proper interface design.

### Interface Pollution

Interface pollution is creating interfaces prematurely or unnecessarily. Signs of interface pollution:

1. **Interfaces created "just for mocking" or "future extensibility."** If the only reason for an interface is to enable mocking in tests, consider whether a fake with a concrete type would be clearer. Wait for actual extensibility needs.

2. **Interfaces defined in producer packages.** This often leads to large, monolithic interfaces that consumers don't need entirely.

3. **Interfaces that mirror implementations exactly.** If the interface changes every time the implementation changes, they're coupled anyway—the interface adds indirection without value.

**Nuance: single-implementation interfaces at boundaries.** An interface with one implementation today isn't *inherently* bad. At architectural boundaries (ports in hexagonal architecture), interfaces define the contract between core logic and infrastructure. A `MessageStore` interface in your domain package is valid even if `DynamoMessageStore` is the only implementation—the interface protects your core from infrastructure details and enables future adapters.

**Anti-pattern—interface for one implementation:**

```go
// DON'T: Interface exists only for the single implementation
type UserService interface {
    CreateUser(ctx context.Context, req CreateUserRequest) (*User, error)
    GetUser(ctx context.Context, id string) (*User, error)
    // ...
}

type userServiceImpl struct { /* ... */ }

func NewUserService() UserService {
    return &userServiceImpl{}
}
```

This adds indirection without benefit. The interface matches the implementation exactly. If the implementation changes, the interface changes. They're coupled anyway.

**Idiomatic—concrete type with discovered interfaces:**

```go
// DO: Return concrete type
type UserService struct { /* ... */ }

func NewUserService(repo *UserRepository) *UserService {
    return &UserService{repo: repo}
}

// Consumers define interfaces when they need them
// (e.g., for testing or dependency injection)
```

Rob Pike's guidance applies: discover interfaces from usage patterns, don't design them speculatively.

### Interfaces for Dependency Injection

Interfaces enable dependency injection without frameworks. The pattern:

1. Core packages define interfaces for their dependencies
2. Boundary packages implement those interfaces
3. The composition root wires concrete types to interfaces

```go
// order/service.go (CORE) — defines what it needs
package order

type PaymentProcessor interface {
    Charge(ctx context.Context, amount Money, method PaymentMethod) error
}

type InventoryChecker interface {
    Reserve(ctx context.Context, items []LineItem) error
    Release(ctx context.Context, items []LineItem) error
}

type OrderService struct {
    payments  PaymentProcessor
    inventory InventoryChecker
}

func NewOrderService(p PaymentProcessor, i InventoryChecker) *OrderService {
    return &OrderService{payments: p, inventory: i}
}
```

```go
// stripe/processor.go (BOUNDARY) — implements PaymentProcessor
package stripe

type Processor struct { client *stripe.Client }

func (p *Processor) Charge(ctx context.Context, amount Money, method PaymentMethod) error {
    // Stripe-specific implementation
}
```

```go
// main.go (COMPOSITION ROOT) — wires everything
func main() {
    stripeProcessor := stripe.NewProcessor(apiKey)
    inventoryService := warehouse.NewInventoryService(db)
    
    orderService := order.NewOrderService(stripeProcessor, inventoryService)
    // ...
}
```

The `order` package knows nothing about Stripe or the warehouse system. It depends only on behaviors it needs. Swapping payment processors requires changing only the composition root.

### One Type, Multiple Consumer Interfaces

This is Go's killer interface trick: the same concrete type can satisfy different interfaces in different packages, with each consumer seeing only what it needs.

```go
// storage/dynamo.go — Producer: one concrete type with many capabilities
package storage

type DynamoMessageStore struct { client *dynamodb.Client }

func (s *DynamoMessageStore) Save(ctx context.Context, msg *Message) error { /* ... */ }
func (s *DynamoMessageStore) Get(ctx context.Context, id string) (*Message, error) { /* ... */ }
func (s *DynamoMessageStore) Delete(ctx context.Context, id string) error { /* ... */ }
func (s *DynamoMessageStore) List(ctx context.Context, roomID string) ([]*Message, error) { /* ... */ }
```

```go
// ingest/handler.go — Consumer A: needs only Save
package ingest

type MessageSaver interface {
    Save(ctx context.Context, msg *Message) error
}

type Handler struct { store MessageSaver }
```

```go
// query/service.go — Consumer B: needs Get and List
package query

type MessageReader interface {
    Get(ctx context.Context, id string) (*Message, error)
    List(ctx context.Context, roomID string) ([]*Message, error)
}

type Service struct { store MessageReader }
```

```go
// main.go — Same instance satisfies both interfaces
func main() {
    store := storage.NewDynamoMessageStore(dynamoClient)
    
    ingestHandler := ingest.NewHandler(store)  // sees MessageSaver
    queryService := query.NewService(store)    // sees MessageReader
    // ...
}
```

The `DynamoMessageStore` is wired once but presents different "faces" to different consumers. Each consumer depends only on the methods it calls. This is interface segregation emerging naturally from Go's implicit satisfaction.

### Ports at Architectural Boundaries

At boundaries between core domain logic and infrastructure, interfaces define *ports*—contracts that protect your domain from infrastructure details. These are valid even with a single implementation today.

```go
// domain/message/store.go — Port defined in core
package message

// Store is the port for message persistence.
// Implementations must return ErrNotFound for missing messages.
type Store interface {
    Save(ctx context.Context, msg *Message) error
    Get(ctx context.Context, id string) (*Message, error)
}

// ErrNotFound is returned when a message doesn't exist.
var ErrNotFound = errors.New("message not found")
```

```go
// infra/dynamo/message_store.go — Adapter implements the port
package dynamo

type MessageStore struct { client *dynamodb.Client }

// Compile-time check: ensure we implement the port
var _ message.Store = (*MessageStore)(nil)

func (s *MessageStore) Get(ctx context.Context, id string) (*message.Message, error) {
    // ... DynamoDB-specific code ...
    if itemNotFound {
        return nil, message.ErrNotFound  // Translate to domain error
    }
    // ...
}
```

**Key principles for ports:**

- **Define ports in core packages** (domain logic), not infrastructure packages
- **Method signatures use domain types**, not infrastructure types (`*message.Message`, not `*dynamodb.AttributeValue`)
- **Error taxonomy is part of the contract**—document expected errors like `ErrNotFound`
- **Adapters translate** infrastructure errors to domain errors at the boundary

### Returning Interfaces (Rare but Valid)

The "accept interfaces, return structs" principle has exceptions. Returning an interface is appropriate when you want to:

1. **Hide unexported methods or fields** that callers shouldn't access
2. **Seal behavior** behind a narrow contract at a hard boundary
3. **Support multiple implementations** chosen at runtime (factory pattern)

```go
// Returning interface to hide implementation details
func NewRateLimiter(cfg Config) Limiter {
    if cfg.Distributed {
        return newRedisLimiter(cfg)  // unexported type
    }
    return newLocalLimiter(cfg)  // unexported type
}

type Limiter interface {
    Allow(ctx context.Context, key string) (bool, error)
}
```

This is rare. Default to returning concrete types; return interfaces only with clear justification.

---

## Interface Design Checklist

When designing interfaces, ask:

1. **What's the purpose?** "For mocking" or "future extensibility" are weak justifications. "Decoupling core from infrastructure" or "enabling multiple implementations" are strong.

2. **Who owns this interface?** Consumer-owned for application code. Producer-owned only for ecosystem contracts with stability guarantees.

3. **How many methods?** One is ideal. More than three requires strong justification (tight cohesion).

4. **Are all methods cohesive?** They should represent a single responsibility. Mixed concerns suggest splitting.

5. **Can I write a fake in under 20 lines?** If not, the interface is probably too large.

6. **Does the standard library already have this interface?** Use `io.Reader`, `io.Writer`, `fmt.Stringer`, `error` before inventing your own.

7. **Does it leak infrastructure types?** Port interfaces should use domain types, not `*sql.DB`, `*dynamodb.Client`, etc.

---

## Trade-Off Matrix

| If You Need... | Choose... | Accept... |
|----------------|-----------|-----------|
| Maximum flexibility | One-method interface | More interfaces to manage |
| Test isolation | Consumer-defined interface | Interface near each consumer |
| Standard library compatibility | `io.Reader`, `io.Writer`, etc. | Their specific method signatures |
| Type safety with abstraction | Small, explicit interfaces | Can't add methods without breaking implementations |
| Extension without modification | Implicit satisfaction | No compiler help for "did I implement this?" |
| Backward compatibility | Return concrete types | Consumers must define their own interfaces |

---

## Interview Signals

| When Asked... | Demonstrate... |
|---------------|----------------|
| "How big should interfaces be?" | One method is ideal. The standard library's most-used interfaces (`io.Reader`, `io.Writer`, `error`) have one method each. More than three methods requires strong justification—tight cohesion like `sort.Interface`. |
| "Where should interfaces be defined?" | At the consumer for application code. Producer-owned interfaces are justified only for stable ecosystem contracts with backward-compatibility guarantees (`io.Reader`, `http.Handler`, `database/sql/driver`). |
| "Why implicit interface satisfaction?" | Decoupling without import dependencies. Types can satisfy interfaces from packages they don't import. You can create interfaces for existing types without modifying them. The same concrete type can satisfy different interfaces in different packages. |
| "How do you test code with dependencies?" | Define small interfaces at the consumer. Implement simple fakes—not mocks with verification. One-method interfaces mean one-method fakes. |
| "What's interface pollution?" | Creating interfaces before they're needed—typically for "future extensibility" or "just for mocking." Exception: interfaces at architectural boundaries (ports) are valid even with one implementation if they protect core logic from infrastructure. |
| "Accept interfaces, return structs—why?" | Accepting interfaces decouples and enables testing. Returning structs gives callers full access; they define their own interfaces. Exception: return interfaces when hiding implementation details or supporting runtime-selected implementations. |
| "Single implementation interface—bad?" | Not inherently. Bad when created "just for mocking" with no architectural purpose. Valid at ports where the interface defines a boundary contract, even if there's only one adapter today. |

---

## Common Standard Library Interfaces

Reference for frequently-used interfaces you should know:

```go
// io package
type Reader interface { Read(p []byte) (n int, err error) }
type Writer interface { Write(p []byte) (n int, err error) }
type Closer interface { Close() error }
type Seeker interface { Seek(offset int64, whence int) (int64, error) }
type ReaderFrom interface { ReadFrom(r Reader) (n int64, err error) }
type WriterTo interface { WriteTo(w Writer) (n int64, err error) }

// fmt package
type Stringer interface { String() string }
type GoStringer interface { GoString() string }

// sort package
type Interface interface {
    Len() int
    Less(i, j int) bool
    Swap(i, j int)
}

// encoding package
type BinaryMarshaler interface { MarshalBinary() (data []byte, err error) }
type BinaryUnmarshaler interface { UnmarshalBinary(data []byte) error }
type TextMarshaler interface { MarshalText() (text []byte, err error) }
type TextUnmarshaler interface { UnmarshalText(text []byte) error }

// encoding/json package
type Marshaler interface { MarshalJSON() ([]byte, error) }
type Unmarshaler interface { UnmarshalJSON([]byte) error }

// context package (note: Value is an escape hatch, not a pattern to emulate)
type Context interface {
    Deadline() (deadline time.Time, ok bool)
    Done() <-chan struct{}
    Err() error
    Value(key any) any  // intentionally untyped; use sparingly
}

// net/http package
type Handler interface { ServeHTTP(ResponseWriter, *Request) }
type ResponseWriter interface {
    Header() Header
    Write([]byte) (int, error)
    WriteHeader(statusCode int)
}

// database/sql package
type Scanner interface { Scan(dest ...any) error }
```

When your types implement these interfaces, they plug into the entire Go ecosystem. A type that implements `io.Reader` can be passed to `json.NewDecoder`, `bufio.NewScanner`, `io.Copy`, and thousands of other functions.

---

## Bridge to Related Documents

This deep dive extends the interface concepts from [Types and Composition](02_TYPES_AND_COMPOSITION.md). The principles here—consumer-defined interfaces, small interfaces, implicit satisfaction—underpin the testing philosophy in [Testing Philosophy](07_TESTING_PHILOSOPHY.md), where fakes over mocks becomes natural when interfaces are small.

For dependency injection patterns that leverage these interface principles, see [DD_DEPENDENCY_INJECTION.md](DD_DEPENDENCY_INJECTION.md) (after completing [Package and Project Design](08_PACKAGE_AND_PROJECT_DESIGN.md)).
