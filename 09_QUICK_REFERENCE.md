# Quick Reference

> Interview-day cheat sheet: the invariants, decisions, and patterns that define senior-level Go.

---

## How to Use This Document

This is a condensed reference, not a learning resource. If concepts here are unfamiliar, return to the source document. Use this the day before an interview to refresh principles you've already internalized.

---

## Go Proverbs (Memorize These)

The Go Proverbs are compressed wisdom about what makes Go code effective. These come up in interviews directly or underlie correct answers:

- **Don't communicate by sharing memory; share memory by communicating.**
- **The bigger the interface, the weaker the abstraction.**
- **Make the zero value useful.**
- **interface{} says nothing.**
- **Clear is better than clever.**
- **A little copying is better than a little dependency.**
- **Gofmt's style is no one's favorite, yet gofmt is everyone's favorite.**

When discussing any Go design decision, these proverbs provide the vocabulary and framing interviewers expect.

---

## Invariants by Theme

Invariants are rules that must never be violated. They're the non-negotiables of senior-level Go.

### Philosophy

| Invariant | Source | Why It Matters |
|-----------|--------|----------------|
| Clear is better than clever | Design Philosophy | Optimize for the reader, not the author. Clever code is a maintenance burden. |
| A little copying is better than a little dependency | Design Philosophy | Dependencies carry transitive risk, version conflicts, and cognitive load. |
| Make the zero value useful | Design Philosophy | Correctness constraint: types should work without explicit initialization. |
| Constructors are optional; invariants are not | Design Philosophy | If a type *requires* a constructor, its zero value is probably invalid—document this. |

### Types & Interfaces

| Invariant | Source | Why It Matters |
|-----------|--------|----------------|
| The bigger the interface, the weaker the abstraction | Types & Composition | Small interfaces are easy to implement, mock, and reason about. |
| Don't design with interfaces, discover them | Types & Composition | Define interfaces at consumers when abstraction is needed, not upfront. |
| Accept interfaces, return structs | Types & Composition | Decouples callers; callers define their own interfaces as needed. |

### Slices, Maps & Aliasing

| Invariant | Why It Matters |
|-----------|----------------|
| Maps are reference types; copying a map copies the header, not the data | Two variables pointing to the same map mutate shared state. |
| Slices alias underlying arrays; `append` may or may not reallocate | Mutations through one slice can affect another; `append` beyond capacity creates a new array. |
| Never expose internal slices or maps without copying | Callers can mutate your internal state. Return copies or use accessor methods. |

### Errors

| Invariant | Source | Why It Matters |
|-----------|--------|----------------|
| Handle an error or return it, never both | Error Philosophy | Logging and returning creates duplicate noise; pick one. |
| Errors at boundaries must be translated, not leaked | Error Philosophy | Internal details (DB errors, wire failures) must not reach external consumers. |
| Only wrap with `%w` when callers should depend on the underlying error | Error Philosophy | `%w` preserves identity for `errors.Is`/`As`—use deliberately. |
| Errors are part of your API contract | Error Philosophy | Once exposed across a boundary, error shape and semantics are a compatibility commitment. |

### Context & Lifecycle

| Invariant | Source | Why It Matters |
|-----------|--------|----------------|
| Context is created at boundaries, propagated through core | Context & Lifecycle | HTTP handlers, `main()`—these create context. Core logic receives it. |
| The creator of a context owns its cancellation | Context & Lifecycle | Whoever calls `WithCancel` must call `cancel()`—even on success. |
| Cancellation flows downstream only | Context & Lifecycle | Children can be cancelled; parents cannot be cancelled by children. |
| Never store context in structs | Context & Lifecycle | Context is per-operation; storing it couples struct to one operation's lifetime. |
| Deadlines compose by taking the earliest | Context & Lifecycle | You cannot extend a parent's deadline by creating a child context. |
| Cancellation is not rollback | Context & Lifecycle | Cancellation means "stop waiting"—side effects already performed persist. |

### Concurrency

| Invariant | Source | Why It Matters |
|-----------|--------|----------------|
| Every goroutine must have an owner responsible for its termination | Concurrency Architecture | No fire-and-forget. Every `go` implies a termination contract. |
| Share memory by communicating; don't communicate by sharing memory | Concurrency Architecture | Channels for coordination; mutexes for protection. |
| The sender owns the channel; receivers never close | Concurrency Architecture | Closing signals "no more values"—only the producer knows when that's true. |
| Goroutines must have bounded lifetimes | Concurrency Architecture | Every goroutine either completes or responds to cancellation. |

### Graceful Shutdown

| Invariant | Source | Why It Matters |
|-----------|--------|----------------|
| Shutdown order is the reverse of startup order | Graceful Shutdown | Dependencies must outlive dependents. |
| Every component must have a shutdown path | Graceful Shutdown | If it can start, it must be stoppable. |
| Shutdown must complete within bounded time | Graceful Shutdown | Open-ended shutdown = hanging. SIGKILL is the backstop. |

### Testing

| Invariant | Source | Why It Matters |
|-----------|--------|----------------|
| Fakes over mocks | Testing Philosophy | Mocks verify calls; fakes verify contracts. Fakes survive refactoring. |
| Assert behavioral contracts, not call order | Testing Philosophy | Test *what happened*, not *how*. |
| Concurrency tests assert eventual outcomes, not timing | Testing Philosophy | Never `time.Sleep` to synchronize. Use channels, timeouts, `goleak`. |
| Time is a dependency; inject it like any other | Testing Philosophy | Direct `time.Now()` calls are untestable. |

### Package Design

| Invariant | Source | Why It Matters |
|-----------|--------|----------------|
| Name packages by responsibility, not by type | Package Design | `order`, not `models`. Purpose over form. |
| Dependencies point inward: boundary → core | Package Design | Domain logic must not import HTTP, database drivers, etc. |
| `internal/` protects your right to change | Package Design | Compiler-enforced privacy. Use it aggressively. |

---

## The Boundary vs Core Model

This framing appears throughout the handbook. Internalize it.

```
┌─────────────────────────────────────────────────────────────┐
│                     BOUNDARY LAYER                          │
│  HTTP handlers, gRPC servers, CLI commands, DB adapters     │
│  • Creates context with timeouts                            │
│  • Translates errors (domain → HTTP status)                 │
│  • Handles serialization/deserialization                    │
│  • Logs with request context                                │
│  • Implements interfaces defined in core                    │
└─────────────────────────────────────────────────────────────┘
                           │
                           │ dependencies point inward
                           ▼
┌─────────────────────────────────────────────────────────────┐
│                       CORE LAYER                            │
│  Domain logic, business rules, service interfaces           │
│  • Receives context, respects cancellation                  │
│  • Returns domain errors (ErrNotFound, etc.)                │
│  • Defines interfaces for dependencies                      │
│  • No logging (caller decides)                              │
│  • Protocol-agnostic, testable in isolation                 │
└─────────────────────────────────────────────────────────────┘
```

**Key rule:** Core never imports boundary. The `order` package must not import `postgres` or `http`.

---

## Decision Matrices

### Channels vs Mutexes

| Use Channels When... | Use Mutexes When... |
|----------------------|---------------------|
| Passing ownership of data | Protecting internal state |
| Distributing units of work | Guarding a cache or map |
| Communicating async results | Simple counter increments |
| Coordinating multiple goroutines | Critical sections are short |

**Rule of thumb:** If you're protecting shared state, a mutex is usually simpler and safer than a channel pretending to be a mutex.

### Error Types

| Type | Use When | API Commitment |
|------|----------|----------------|
| **Sentinel** (`var ErrNotFound = errors.New(...)`) | Callers need to match specific conditions | Identity is frozen—removing is breaking |
| **Typed** (struct implementing `error`) | Callers need structured data (`errors.As`) | Type and fields are frozen |
| **Opaque** (`fmt.Errorf` without `%w`) | Callers only need `err != nil` | None—implementation can change freely |

### `%w` vs `%v` in Error Wrapping

| Use `%w` When... | Use `%v` When... |
|------------------|------------------|
| Callers should inspect underlying error | You want to hide implementation details |
| Error identity is part of your API | Underlying type might change |
| Supporting `errors.Is` / `errors.As` | Breaking the error chain intentionally |

### Context Constructors

| Function | Use When |
|----------|----------|
| `context.Background()` | Root context in `main()`, tests, initialization |
| `context.TODO()` | Placeholder—context plumbing incomplete |
| `context.WithCancel(parent)` | Manual cancellation control needed |
| `context.WithTimeout(parent, d)` | Operation should timeout after duration |
| `context.WithDeadline(parent, t)` | Operation should timeout at specific time |
| `context.WithValue(parent, k, v)` | Request-scoped data (request ID, auth)—see rules below |

**`WithValue` rules** (violation is a code smell):
- Keys must be unexported types (not strings)
- Values must be request-scoped, immutable, and cheap to copy
- Never use for optional parameters, configuration, or dependencies

### Sync Primitives

| Primitive | Use When |
|-----------|----------|
| `sync.WaitGroup` | Wait for N goroutines to complete |
| `sync.Once` | One-time initialization |
| `sync.Mutex` | Protect shared state (read-write) |
| `sync.RWMutex` | Protect shared state (many readers, few writers) |
| `sync/atomic` | Simple counters, flags—rarely needed in business logic |
| `errgroup.Group` | Wait + error handling + context cancellation |

---

## Key Code Patterns

### The For-Select Loop

Standard pattern for long-lived goroutines:

```go
func (w *Worker) run(ctx context.Context) {
    for {
        select {
        case <-ctx.Done():
            return // Always include cancellation path
        case job := <-w.jobs:
            w.process(job)
        }
    }
}
```

### Graceful HTTP Server Shutdown

```go
func runServer(ctx context.Context, addr string, handler http.Handler) error {
    server := &http.Server{Addr: addr, Handler: handler}
    
    errCh := make(chan error, 1)
    go func() {
        if err := server.ListenAndServe(); err != http.ErrServerClosed {
            errCh <- err
        }
        close(errCh)
    }()
    
    select {
    case err := <-errCh:
        return err
    case <-ctx.Done():
        shutdownCtx, cancel := context.WithTimeout(context.Background(), 15*time.Second)
        defer cancel()
        return server.Shutdown(shutdownCtx)
    }
}
```

### Signal Handling with Context

```go
func main() {
    ctx, stop := signal.NotifyContext(context.Background(),
        syscall.SIGTERM, syscall.SIGINT)
    defer stop()
    
    if err := run(ctx); err != nil && !errors.Is(err, context.Canceled) {
        log.Fatal(err)
    }
}
```

### Interface Defined in Core, Implemented at Boundary

```go
// internal/order/repository.go (CORE)
type Repository interface {
    Get(ctx context.Context, id string) (*Order, error)
    Save(ctx context.Context, order *Order) error
}

// internal/postgres/order_repo.go (BOUNDARY)
type OrderRepository struct { db *sql.DB }

var _ order.Repository = (*OrderRepository)(nil) // Compile-time check

func (r *OrderRepository) Get(ctx context.Context, id string) (*order.Order, error) {
    // PostgreSQL-specific implementation
}
```

### Table-Driven Test

```go
func TestParseAmount(t *testing.T) {
    tests := []struct {
        name    string
        input   string
        want    int64
        wantErr bool
    }{
        {name: "valid", input: "$100", want: 10000},
        {name: "invalid", input: "bad", wantErr: true},
    }
    
    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            got, err := ParseAmount(tt.input)
            if (err != nil) != tt.wantErr {
                t.Errorf("error = %v, wantErr = %v", err, tt.wantErr)
            }
            if got != tt.want {
                t.Errorf("got %v, want %v", got, tt.want)
            }
        })
    }
}
```

### Compile-Time Interface Check

```go
var _ http.Handler = (*Server)(nil)
var _ io.ReadCloser = (*MyReader)(nil)
```

### Fake Over Mock

```go
// Fake: implements interface with real (simplified) behavior
type FakeUserStore struct {
    users map[string]*User
    mu    sync.RWMutex
}

func (f *FakeUserStore) Get(ctx context.Context, id string) (*User, error) {
    f.mu.RLock()
    defer f.mu.RUnlock()
    user, ok := f.users[id]
    if !ok {
        return nil, ErrNotFound
    }
    return user, nil
}

// Test verifies outcome, not calls
func TestService(t *testing.T) {
    store := NewFakeUserStore()
    service := NewUserService(store)
    
    created, _ := service.CreateUser(ctx, "test@example.com")
    retrieved, _ := service.GetUser(ctx, created.ID)
    
    if retrieved.Email != "test@example.com" {
        t.Error("user not persisted correctly")
    }
}
```

---

## Interview Signals

When asked these questions, demonstrate these concepts:

| When Asked... | Demonstrate... |
|---------------|----------------|
| "Summarize Go's philosophy" | Clear is better than clever. Optimize for readers and teams at scale, not author expressiveness. Composition over inheritance, explicit over implicit. |
| "Pointer vs value receiver?" | Pointer if mutating or large struct. Value if small and immutable. Be consistent—if one method needs pointer, use pointer for all. Method sets: `T`'s methods ⊂ `*T`'s methods. |
| "Why accept interfaces, return structs?" | Accepting interfaces decouples and enables testing. Returning structs gives callers full access; they define their own interfaces. |
| "How do you handle errors in Go?" | Errors are values, not exceptions. Check explicitly, wrap with context at boundaries, translate at API edges. Handle or return, never both. |
| "Sentinel vs typed vs opaque errors?" | Sentinels for identity checks (`errors.Is`). Typed for structured data (`errors.As`). Opaque by default for implementation freedom. |
| "When do you panic?" | Almost never. Programmer errors only—impossible states, violated invariants. Never for user input, network errors, expected failures. |
| "What is context.Context for?" | Cancellation, deadlines, request-scoped values. Standard way to propagate lifecycle signals. Not for dependency injection. |
| "Why not store context in structs?" | Context is per-operation. Storing it couples struct lifetime to one operation's lifetime. Pass explicitly to each method. |
| "How do you prevent goroutine leaks?" | Every goroutine has an owner. Always provide cancellation via context. Use `goleak` in tests. Buffer channels appropriately. |
| "Channels vs mutexes?" | Channels for communication between goroutines. Mutexes for protecting shared state. Don't use channels where mutex is simpler. |
| "Who closes a channel?" | Sender (creator) closes; receivers never close. Closing signals "no more values"—only producer knows when that's true. |
| "How do you handle graceful shutdown?" | Signal handling with `signal.NotifyContext`, context cancellation to all components, reverse-order shutdown, timeout budgets. |
| "Why does shutdown order matter?" | Dependencies must outlive dependents. Closing DB before HTTP server causes in-flight failures. Reverse startup order. |
| "Mocks vs fakes?" | Mocks verify implementation (calls made). Fakes verify contracts (behavior occurred). Fakes survive refactoring. |
| "How do you test concurrent code?" | Race detector (`-race`), `goleak` for leak detection, channels for synchronization—never `time.Sleep`. Assert eventual outcomes. |
| "How do you structure Go projects?" | Start simple, grow as needed. `internal/` for private code. `cmd/` for binaries. Name packages by responsibility. Dependencies point inward. |
| "What's `internal/` for?" | Compiler-enforced privacy. Protects your right to refactor without breaking external consumers. |
| "How do slices work internally?" | Slice header (pointer, len, cap) references an underlying array. Multiple slices can alias the same array. Append beyond capacity allocates a new array. |
| "When does Go allocate on the heap?" | Escape analysis decides—not `new` vs value literal. If a value's lifetime exceeds its stack frame (returned, stored in interface, etc.), it escapes to heap. |
| "How does defer work?" | Arguments evaluated immediately, execution deferred until function returns. LIFO order. In loops, each iteration defers—be careful with closures. |

---

## Defer Semantics (Classic Interview Check)

- **Arguments are evaluated immediately; execution is deferred.** `defer f(x)` captures `x`'s current value, not its value at function exit.
- **Defers execute in LIFO order.** Last deferred, first executed.
- **Defer in loops requires care.** Each iteration defers; consider wrapping in a function or deferring a closure that captures correctly.
- **Cost is usually negligible.** Clarity beats micro-optimization. Use defer for cleanup.

```go
// Classic trap: i is captured by reference
for i := 0; i < 3; i++ {
    defer fmt.Println(i) // Prints 2, 2, 2 (in Go < 1.22) or 2, 1, 0 (in Go 1.22+)
}

// Safe: capture by value
for i := 0; i < 3; i++ {
    i := i // Shadow with local copy (pre-1.22 idiom)
    defer fmt.Println(i) // Prints 2, 1, 0
}
```

---

## Memory & Escape Analysis (Staff-Level Awareness)

Interviewers probe this to test runtime understanding. You don't need to optimize for it, but you should know it exists.

| Concept | What to Know |
|---------|--------------|
| Escape analysis | Compiler decides stack vs heap allocation—not `new` vs value literal. |
| Heap allocation | An implementation detail, not a design goal. Don't contort code to avoid allocations. |
| Pointers don't guarantee heap | A pointer to a local variable may stay on stack if it doesn't escape. |
| Premature optimization | Avoiding allocations at the cost of clarity is almost always wrong. |

**When asked:** "I'm aware escape analysis determines allocation, but I don't micro-optimize for it unless profiling shows a hot path."

---

## Common Mistakes to Avoid

| Mistake | Correct Approach |
|---------|------------------|
| Fire-and-forget goroutines | Every goroutine has an owner who ensures it stops |
| `time.Sleep` in tests | Use channels, timeouts, synchronization primitives |
| Storing context in structs | Pass context to each method |
| Logging and returning errors | Handle OR return, never both |
| Large interfaces upfront | Discover small interfaces at consumers |
| `pkg/` for everything | Use `internal/`; anything outside is implicitly public |
| Packages named `utils`, `models` | Name by responsibility: `order`, `auth`, `postgres` |
| Core importing boundary | Dependencies point inward only |
| Closing channels from receiver | Sender owns the channel lifecycle |
| Mock-heavy tests | Fakes verify contracts; mocks verify implementation |
| Returning internal slices/maps | Return copies to prevent caller mutation |
| String keys in `context.WithValue` | Use unexported struct types as keys |
| Ignoring defer argument evaluation | Arguments captured at defer statement, not at execution |

---

## If They Push Deeper (Staff-Level Topics)

These topics signal deep Go knowledge. You don't need to implement them, but awareness shows maturity.

| Topic | One-Sentence Answer |
|-------|---------------------|
| **Escape analysis** | Compiler decides stack vs heap based on whether a value's reference escapes its scope. |
| **Scheduler (GMP model)** | M:N threading—M OS threads run N goroutines via P processors; work-stealing balances load. |
| **GC tradeoffs** | Go's concurrent GC optimizes for latency over throughput; sub-millisecond pauses at cost of some CPU. |
| **Method sets** | Value receivers: method set of `T` includes value methods. Pointer receivers: `*T` includes both. This affects interface satisfaction. |
| **Interface internals** | Interface value is (type, value) pair; nil interface ≠ interface holding nil pointer. |
| **Channel internals** | Channels are queues with synchronization; unbuffered channels synchronize sender and receiver. |

**How to use:** If asked, give the one-sentence answer. If they probe further and you don't know, say "I'd need to dive into the runtime source to answer precisely, but in practice I rely on profiling and the race detector."

---

## Checklist: Before the Interview

- [ ] Can you explain each Go proverb and give an example?
- [ ] Can you draw the boundary/core model and explain dependency direction?
- [ ] Can you write a for-select loop from memory?
- [ ] Can you explain when to use channels vs mutexes?
- [ ] Can you describe graceful shutdown sequence and why order matters?
- [ ] Can you explain context propagation rules and the deadline composition rule?
- [ ] Can you articulate why fakes are preferred over mocks?
- [ ] Can you describe what makes a goroutine "owned"?
- [ ] Can you explain error handling at boundaries vs core?
- [ ] Can you sketch a project layout with correct `internal/` usage?
- [ ] Can you explain slice aliasing and when `append` reallocates?
- [ ] Can you describe what escape analysis is (even if you don't optimize for it)?
- [ ] Can you explain defer argument evaluation timing?

---

## Further Reading

If any concept above is unclear, return to the source:

| Topic | Document |
|-------|----------|
| Philosophy, proverbs, simplicity | [01_GO_DESIGN_PHILOSOPHY.md](01_GO_DESIGN_PHILOSOPHY.md) |
| Types, interfaces, composition | [02_TYPES_AND_COMPOSITION.md](02_TYPES_AND_COMPOSITION.md) |
| Errors, wrapping, boundaries | [03_ERROR_PHILOSOPHY.md](03_ERROR_PHILOSOPHY.md) |
| Context, cancellation, lifecycle | [04_CONTEXT_AND_LIFECYCLE.md](04_CONTEXT_AND_LIFECYCLE.md) |
| Goroutines, channels, ownership | [05_CONCURRENCY_ARCHITECTURE.md](05_CONCURRENCY_ARCHITECTURE.md) |
| Shutdown, signals, ordering | [06_GRACEFUL_SHUTDOWN.md](06_GRACEFUL_SHUTDOWN.md) |
| Testing, fakes, race detector | [07_TESTING_PHILOSOPHY.md](07_TESTING_PHILOSOPHY.md) |
| Packages, `internal/`, structure | [08_PACKAGE_AND_PROJECT_DESIGN.md](08_PACKAGE_AND_PROJECT_DESIGN.md) |
