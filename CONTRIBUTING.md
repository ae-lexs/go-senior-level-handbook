# Contributing

<!-- 
  TEMPLATE: Copy this file verbatim into new Go projects.
  References the Go Senior-Level Handbook as the authoritative style guide.
  Remove this comment block after copying.
-->

> Code review cheat sheet: the invariants, decisions, and patterns we enforce.

---

## How to Use This Document

This is a condensed reference, not a learning resource. If concepts here are unfamiliar, read the [Go Senior-Level Handbook](https://github.com/ae-lexs/go-senior-level-handbook) first. Use this during code review to verify contributions meet our standards.

---

## Go Proverbs (Non-Negotiable)

These proverbs define our coding philosophy. Violations will be flagged in review:

- **Don't communicate by sharing memory; share memory by communicating.**
- **The bigger the interface, the weaker the abstraction.**
- **Make the zero value useful.**
- **interface{} says nothing.**
- **Clear is better than clever.**
- **A little copying is better than a little dependency.**
- **Gofmt's style is no one's favorite, yet gofmt is everyone's favorite.**

---

## Invariants by Theme

Invariants are rules that must never be violated. PRs breaking these will not be merged.

### Philosophy

| Invariant | Why It Matters |
|-----------|----------------|
| Clear is better than clever | Optimize for the reader, not the author. Clever code is a maintenance burden. |
| A little copying is better than a little dependency | Dependencies carry transitive risk, version conflicts, and cognitive load. |
| Make the zero value useful | Types should work without explicit initialization. |
| Constructors are optional; invariants are not | If a type *requires* a constructor, its zero value is probably invalid—document this. |

### Types & Interfaces

| Invariant | Why It Matters |
|-----------|----------------|
| The bigger the interface, the weaker the abstraction | Small interfaces are easy to implement, fake, and reason about. |
| Don't design with interfaces, discover them | Define interfaces at consumers when abstraction is needed, not upfront. |
| Accept interfaces, return structs | Decouples callers; callers define their own interfaces as needed. |

### Slices, Maps & Aliasing

| Invariant | Why It Matters |
|-----------|----------------|
| Maps are reference types; copying a map copies the header, not the data | Two variables pointing to the same map mutate shared state. |
| Slices alias underlying arrays; `append` may or may not reallocate | Mutations through one slice can affect another. |
| Never expose internal slices or maps without copying | Callers can mutate your internal state. Return copies. |

### Errors

| Invariant | Why It Matters |
|-----------|----------------|
| Handle an error or return it, never both | Logging and returning creates duplicate noise; pick one. |
| Errors at boundaries must be translated, not leaked | Internal details must not reach external consumers. |
| Only wrap with `%w` when callers should depend on the underlying error | `%w` preserves identity for `errors.Is`/`As`—use deliberately. |

### Context & Lifecycle

| Invariant | Why It Matters |
|-----------|----------------|
| Context is created at boundaries, propagated through core | HTTP handlers, `main()`—these create context. Core logic receives it. |
| The creator of a context owns its cancellation | Whoever calls `WithCancel` must call `cancel()`—even on success. |
| Cancellation flows downstream only | Children can be cancelled; parents cannot be cancelled by children. |
| Never store context in structs | Context is per-operation; storing it couples struct to one operation's lifetime. |

### Concurrency

| Invariant | Why It Matters |
|-----------|----------------|
| Every goroutine must have an owner responsible for its termination | No fire-and-forget. Every `go` implies a termination contract. |
| Share memory by communicating; don't communicate by sharing memory | Channels for coordination; mutexes for protection. |
| The sender owns the channel; receivers never close | Closing signals "no more values"—only the producer knows when that's true. |
| Goroutines must have bounded lifetimes | Every goroutine either completes or responds to cancellation. |

### Graceful Shutdown

| Invariant | Why It Matters |
|-----------|----------------|
| Shutdown order is the reverse of startup order | Dependencies must outlive dependents. |
| Every component must have a shutdown path | If it can start, it must be stoppable. |
| Shutdown must complete within bounded time | Open-ended shutdown = hanging. SIGKILL is the backstop. |

### Testing

| Invariant | Why It Matters |
|-----------|----------------|
| Fakes over mocks | Mocks verify calls; fakes verify contracts. Fakes survive refactoring. |
| Assert behavioral contracts, not call order | Test *what happened*, not *how*. |
| Concurrency tests assert eventual outcomes, not timing | Never `time.Sleep` to synchronize. |
| Time is a dependency; inject it like any other | Direct `time.Now()` calls are untestable. |

### Package Design

| Invariant | Why It Matters |
|-----------|----------------|
| Name packages by responsibility, not by type | `order`, not `models`. Purpose over form. |
| Dependencies point inward: boundary → core | Domain logic must not import HTTP, database drivers, etc. |
| `internal/` protects your right to change | Compiler-enforced privacy. Use it aggressively. |

---

## The Boundary vs Core Model

This framing determines how we structure code. Internalize it.

```
┌─────────────────────────────────────────────────────────────────┐
│                     BOUNDARY LAYER                              │
│  HTTP handlers, gRPC servers, CLI commands, DB adapters        │
│  • Creates context with timeouts                                │
│  • Translates errors (domain → HTTP status)                     │
│  • Handles serialization/deserialization                        │
│  • Implements interfaces defined in core                        │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                       CORE LAYER                                │
│  Domain logic, business rules, pure functions                   │
│  • Receives context, respects cancellation                      │
│  • Returns domain errors                                        │
│  • Defines interfaces for dependencies                          │
│  • No knowledge of HTTP, SQL, wire formats                      │
└─────────────────────────────────────────────────────────────────┘
```

**Dependency rule:** Boundary imports core. Core never imports boundary.

---

## Decision Matrices

Use these to make consistent choices across the codebase.

### When to Use What

| If You Need... | Use... | Not... |
|----------------|--------|--------|
| Coordination between goroutines | Channels | Shared memory + mutex |
| Protection of shared state | Mutex | Channel (overkill) |
| Cancellation propagation | `context.Context` | Custom done channels |
| Multiple implementations | Interface at consumer | Interface at producer |
| Optional parameters | Functional options | Config struct with zero-value ambiguity |
| Required parameters | Explicit constructor args | Functional options |

### Error Type Selection

| Situation | Error Type | Example |
|-----------|------------|---------|
| Expected condition, callers check identity | Sentinel | `var ErrNotFound = errors.New("not found")` |
| Callers need structured data | Typed | `type ValidationError struct { Field, Reason string }` |
| Implementation detail, no caller action | Opaque | `fmt.Errorf("internal: %w", err)` |

### Interface Size Guide

| Methods | Verdict | Examples |
|---------|---------|----------|
| 1 | Ideal | `io.Reader`, `fmt.Stringer`, `http.Handler` |
| 2-3 | Good if cohesive | `io.ReadWriter`, `sort.Interface` |
| 4+ | Needs justification | Split or accept coupling |

---

## Common Mistakes

Flag these in code review:

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

---

## Code Review Questions

When reviewing, verify the author can answer these:

| Topic | Question |
|-------|----------|
| Interfaces | Why is this interface defined here? Who consumes it? |
| Errors | Is this error handled or returned? Not both? |
| Context | Where is this context created? Is cancellation respected? |
| Goroutines | Who owns this goroutine? How does it stop? |
| Channels | Who closes this channel? Is it the sender? |
| Testing | Does this test verify behavior or implementation? |
| Dependencies | Does this import direction follow boundary → core? |

---

## Pull Request Checklist

Before requesting review:

```bash
# Required
go fmt ./...
go vet ./...
go test -race ./...
```

### Self-Review

- [ ] No fire-and-forget goroutines
- [ ] Context passed explicitly, not stored in structs
- [ ] Errors handled OR returned, never both
- [ ] Interfaces defined at consumers, not producers
- [ ] Dependencies point inward (boundary → core)
- [ ] Tests use fakes, not mocks (where applicable)
- [ ] No `time.Sleep` in tests
- [ ] Internal slices/maps not exposed directly

---

## Commit Messages

```
<type>: <description>

[optional body]
```

**Types:** `feat`, `fix`, `refactor`, `test`, `docs`, `chore`

**Unacceptable:**
- `fix stuff`
- `WIP`
- `addressing review comments`

Commits describe *what changed*, not *why you touched the code*.

---

## Project Structure

```
.
├── cmd/
│   └── myapp/
│       └── main.go           # Composition root
├── internal/
│   ├── order/                # CORE: domain logic
│   │   ├── order.go          # Domain types
│   │   ├── service.go        # Business rules
│   │   └── repository.go     # Interface definition
│   ├── postgres/             # BOUNDARY: implements interfaces
│   │   └── order_repo.go
│   └── httpapi/              # BOUNDARY: HTTP transport
│       └── order_handler.go
└── go.mod
```

---

## Further Reading

| Topic | Reference |
|-------|-----------|
| Full handbook | [Go Senior-Level Handbook](https://github.com/ae-lexs/go-senior-level-handbook) |
| Philosophy | [01_GO_DESIGN_PHILOSOPHY.md](https://github.com/ae-lexs/go-senior-level-handbook/blob/main/01_GO_DESIGN_PHILOSOPHY.md) |
| Types & interfaces | [02_TYPES_AND_COMPOSITION.md](https://github.com/ae-lexs/go-senior-level-handbook/blob/main/02_TYPES_AND_COMPOSITION.md) |
| Errors | [03_ERROR_PHILOSOPHY.md](https://github.com/ae-lexs/go-senior-level-handbook/blob/main/03_ERROR_PHILOSOPHY.md) |
| Context & lifecycle | [04_CONTEXT_AND_LIFECYCLE.md](https://github.com/ae-lexs/go-senior-level-handbook/blob/main/04_CONTEXT_AND_LIFECYCLE.md) |
| Concurrency | [05_CONCURRENCY_ARCHITECTURE.md](https://github.com/ae-lexs/go-senior-level-handbook/blob/main/05_CONCURRENCY_ARCHITECTURE.md) |
| Graceful shutdown | [06_GRACEFUL_SHUTDOWN.md](https://github.com/ae-lexs/go-senior-level-handbook/blob/main/06_GRACEFUL_SHUTDOWN.md) |
| Testing | [07_TESTING_PHILOSOPHY.md](https://github.com/ae-lexs/go-senior-level-handbook/blob/main/07_TESTING_PHILOSOPHY.md) |
| Package design | [08_PACKAGE_AND_PROJECT_DESIGN.md](https://github.com/ae-lexs/go-senior-level-handbook/blob/main/08_PACKAGE_AND_PROJECT_DESIGN.md) |
