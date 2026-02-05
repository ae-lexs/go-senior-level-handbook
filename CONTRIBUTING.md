# Contributing

<!-- 
  TEMPLATE NOTE: This file is intended to be copied verbatim into new Go projects.
  It references the Go Senior-Level Handbook as the authoritative style guide.
  Customize the "Non-Goals" section if your project has different constraints.
  Remove this comment block after copying.
-->

> Guidelines for contributing to this project, following the principles of the [Go Senior-Level Handbook](https://github.com/ae-lexs/go-senior-level-handbook).

---

## Our Standards

This project follows the [Go Senior-Level Handbook](https://github.com/ae-lexs/go-senior-level-handbook) as our authoritative Go style guide. The handbook emphasizes three core concepts:

- **Invariants** — Rules that must never be violated
- **Lifecycle** — How things start, run, and stop
- **Ownership** — Who is responsible for what

Before contributing, familiarize yourself with the handbook's philosophy: *clarity over cleverness, explicit over implicit, composition over inheritance*.

---

## Who This Is For

These guidelines favor long-term maintainability over onboarding speed. New contributors are welcome, but we expect familiarity with:

- `context.Context` and cancellation propagation
- Error handling patterns (wrapping, sentinel vs typed errors)
- Goroutine ownership and lifecycle management
- Interface design (small, consumer-defined)

If these concepts are unfamiliar, the [Go Senior-Level Handbook](https://github.com/ae-lexs/go-senior-level-handbook) is an excellent starting point.

---

## Non-Goals

This project does **not** optimize for:

- **Maximum abstraction** — Indirection only when it solves a concrete problem
- **Framework-driven design** — Standard library and explicit wiring preferred
- **Micro-optimizations without evidence** — Profile first, optimize second
- **Consensus-driven style** — `gofmt` decides formatting; the handbook decides patterns

---

## Code Standards

### Formatting

**Non-negotiable.** Run before every commit:

```bash
gofmt -w .
# or
goimports -w .
```

### Naming

| Element | Convention | Example |
|---------|------------|---------|
| Packages | Lowercase, single-word, by responsibility | `order`, `auth`, `postgres` |
| Interfaces | `-er` suffix for single-method | `Reader`, `Handler`, `Validator` |
| Exported | MixedCaps | `ProcessOrder`, `ValidateInput` |
| Unexported | mixedCaps | `parseConfig`, `handleError` |
| Acronyms | All caps | `HTTPServer`, `UserID` |

**Avoid:** `utils`, `common`, `helpers`, `models`, `types` — these reveal nothing about responsibility.

**Avoid stuttering:** `order.Service`, not `order.OrderService`.

> 📖 See handbook: [Package and Project Design](https://github.com/ae-lexs/go-senior-level-handbook/blob/main/08_PACKAGE_AND_PROJECT_DESIGN.md)

---

## Invariants

These rules must never be violated. PRs that break these will not be merged.

### Interfaces

| Rule | Rationale |
|------|-----------|
| The bigger the interface, the weaker the abstraction | Small interfaces are easy to implement, fake, and reason about |
| Accept interfaces, return structs | Decouples callers; they define their own interfaces as needed |
| Define interfaces at the consumer, not the producer | The package that uses a capability defines what it needs |
| Don't design interfaces upfront—discover them | Wait for concrete need: multiple implementations, testing, decoupling |

> 📖 See handbook: [Types and Composition](https://github.com/ae-lexs/go-senior-level-handbook/blob/main/02_TYPES_AND_COMPOSITION.md), [Interface Patterns](https://github.com/ae-lexs/go-senior-level-handbook/blob/main/DD_INTERFACE_PATTERNS.md)

### Errors

| Rule | Rationale |
|------|-----------|
| Handle an error or return it—never both | Logging and returning causes duplicate handling |
| Wrap with context: `fmt.Errorf("...: %w", err)` | Error chains should tell a story |
| Translate errors at boundaries | Domain errors → HTTP codes; internals stay hidden |

```go
// ✓ Wrapping with context
if err != nil {
    return fmt.Errorf("processing order %s: %w", orderID, err)
}

// ✓ Boundary translation
if errors.Is(err, order.ErrNotFound) {
    http.Error(w, "order not found", http.StatusNotFound)
    return
}
```

> 📖 See handbook: [Error Philosophy](https://github.com/ae-lexs/go-senior-level-handbook/blob/main/03_ERROR_PHILOSOPHY.md)

### Context

| Rule | Rationale |
|------|-----------|
| Context is the first parameter, named `ctx` | Go convention; enables grep-ability |
| Never store context in structs | Context is request-scoped, not instance-scoped |
| Create at boundaries, propagate through core | Handlers create; domain logic receives |
| Respect cancellation | Check `ctx.Done()` in long-running operations |

```go
// ✓ Correct signature
func (s *Service) Process(ctx context.Context, id string) error

// ✗ Never do this
type Service struct {
    ctx context.Context // Wrong: storing context
}
```

> 📖 See handbook: [Context and Lifecycle](https://github.com/ae-lexs/go-senior-level-handbook/blob/main/04_CONTEXT_AND_LIFECYCLE.md)

### Concurrency

| Rule | Rationale |
|------|-----------|
| Every goroutine must have an owner | The starter ensures it can stop |
| Share memory by communicating | Channels for coordination; mutexes for protection |
| Sender owns the channel; receivers never close | Only producers know when there are no more values |
| Use `errgroup` for structured concurrency | Groups goroutines, propagates errors, enables cancellation |

```go
// ✓ Structured concurrency
g, ctx := errgroup.WithContext(ctx)

g.Go(func() error {
    return processA(ctx)
})

g.Go(func() error {
    return processB(ctx)
})

return g.Wait()
```

> 📖 See handbook: [Concurrency Architecture](https://github.com/ae-lexs/go-senior-level-handbook/blob/main/05_CONCURRENCY_ARCHITECTURE.md)

### Graceful Shutdown

| Rule | Rationale |
|------|-----------|
| Shutdown order is reverse of startup order | Dependencies must outlive dependents |
| Every component must have a shutdown path | If it can start, it must be stoppable |
| Shutdown must complete within bounded time | Open-ended shutdown = hanging; SIGKILL is the backstop |

> 📖 See handbook: [Graceful Shutdown](https://github.com/ae-lexs/go-senior-level-handbook/blob/main/06_GRACEFUL_SHUTDOWN.md)

### Testing

| Rule | Rationale |
|------|-----------|
| Fakes over mocks | Fakes verify contracts; mocks verify calls. Fakes survive refactoring |
| Assert behavioral contracts, not call order | Test *what happened*, not *how* |
| Never `time.Sleep` for synchronization | Use channels, polling with timeout, `goleak` |
| Time is a dependency; inject it | Direct `time.Now()` calls are untestable |

```go
// ✓ Table-driven test
func TestProcess(t *testing.T) {
    tests := []struct {
        name    string
        input   string
        wantErr bool
    }{
        {"valid input", "abc", false},
        {"empty input", "", true},
    }

    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            err := Process(context.Background(), tt.input)
            if (err != nil) != tt.wantErr {
                t.Errorf("got error %v, wantErr %v", err, tt.wantErr)
            }
        })
    }
}
```

> 📖 See handbook: [Testing Philosophy](https://github.com/ae-lexs/go-senior-level-handbook/blob/main/07_TESTING_PHILOSOPHY.md)

### Package Design

| Rule | Rationale |
|------|-----------|
| Name by responsibility, not by type | `order`, not `models` |
| Dependencies point inward: boundary → core | Domain logic must not import HTTP, database packages |
| Use `internal/` aggressively | Compiler-enforced privacy; protects your right to refactor |

```
internal/
├── order/           # CORE: domain logic, interfaces
│   ├── service.go
│   └── repository.go
├── postgres/        # BOUNDARY: implements order.Repository
│   └── order_repo.go
└── httpapi/         # BOUNDARY: HTTP handlers
    └── order_handler.go
```

> 📖 See handbook: [Package and Project Design](https://github.com/ae-lexs/go-senior-level-handbook/blob/main/08_PACKAGE_AND_PROJECT_DESIGN.md)

---

## Pull Request Process

### Before Opening

```bash
# Required checks
go fmt ./...
go vet ./...
go test -race ./...

# Recommended
golangci-lint run
```

### PR Description Template

```markdown
## What
[One sentence: what does this change?]

## Why
[Context: why is this needed?]

## How Tested
[Manual steps or test coverage]

## Trade-offs
[Any alternatives considered or accepted costs]
```

### Review Expectations

**Authors:**
- Respond to all comments
- Small, focused PRs get reviewed faster
- If you disagree, explain reasoning—be open to being wrong

**Reviewers:**
- Be specific: "This could leak goroutines because..." not "This looks wrong"
- Distinguish blocking issues from suggestions
- Review within 24 hours when possible

---

## Commit Messages

```
<type>: <description>

[optional body]

[optional footer]
```

**Types:** `feat`, `fix`, `refactor`, `test`, `docs`, `chore`

**Good:**
```
feat: add order cancellation endpoint

Implements POST /orders/{id}/cancel with proper
lifecycle management and event emission.

Closes #123
```

**Unacceptable:**
```
fix stuff
WIP
addressing review comments
```

Commits must describe *what changed*, not *why you touched the code*.

---

## Code Review Checklist

### Correctness
- [ ] Does the code do what it claims?
- [ ] Are error cases handled?
- [ ] Are edge cases considered?

### Clarity
- [ ] Could someone understand this in 6 months?
- [ ] Is there unnecessary cleverness?

### Lifecycle & Ownership
- [ ] Do all goroutines have termination paths?
- [ ] Is context propagated correctly?
- [ ] Are resources cleaned up?

### Invariants
- [ ] Interfaces small and consumer-defined?
- [ ] Errors handled or returned, not both?
- [ ] Dependencies point inward?

---

## Quick Reference

| Category | Invariant |
|----------|-----------|
| Philosophy | Clear is better than clever |
| Philosophy | A little copying is better than a little dependency |
| Interfaces | The bigger the interface, the weaker the abstraction |
| Interfaces | Accept interfaces, return structs |
| Errors | Handle or return—never both |
| Errors | Translate at boundaries, don't leak internals |
| Context | First parameter, named `ctx`; never store in structs |
| Concurrency | Every goroutine has an owner responsible for termination |
| Concurrency | Share memory by communicating |
| Shutdown | Reverse of startup order; bounded time |
| Testing | Fakes over mocks; behavioral contracts over call order |
| Packages | Name by responsibility; dependencies point inward |

---

## Further Reading

- [Go Senior-Level Handbook](https://github.com/ae-lexs/go-senior-level-handbook) — Our authoritative style guide
- [Effective Go](https://go.dev/doc/effective_go) — Official language patterns
- [Go Code Review Comments](https://github.com/golang/go/wiki/CodeReviewComments) — Common review feedback
- [Uber Go Style Guide](https://github.com/uber-go/guide) — Additional production patterns

---

*Thank you for contributing. Every improvement—however small—makes the codebase better for everyone.*
