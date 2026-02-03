# Contributing to Go Projects

> Contributing guidelines that mirror the handbook's philosophy: clarity over cleverness, invariants over opinions, and teaching over lecturing.

---

## Core Principle

**Every contribution should make the codebase clearer, not just correct.**

A pull request that fixes a bug but obscures the code is not a net positive. A feature that works but violates lifecycle invariants introduces future debt. We optimize for the reader, the maintainer, and the team—not for the author's cleverness.

---

## Who This Is For

These guidelines intentionally favor long-term maintainability over onboarding speed. New contributors are welcome, but this is not a beginner Go project. If `context.Context`, `errgroup`, or interface segregation are unfamiliar, start with the handbook's core documents before contributing.

We'd rather have fewer, high-quality contributions than many that require extensive revision. This isn't gatekeeping—it's respect for everyone's time.

---

## Non-Goals

This project explicitly does **not** optimize for:

- **Maximum abstraction.** We add indirection only when it solves a concrete problem, not for architectural purity.
- **Framework-driven design.** No Gin, Echo, or dependency injection containers. Standard library and explicit wiring.
- **Micro-optimizations without evidence.** Premature optimization is the root of all evil. Profile first, optimize second.
- **Consensus-driven style.** `gofmt` decides formatting. The handbook decides patterns. Bikeshedding is not welcome.

If these constraints feel limiting, this may not be the right project for you—and that's okay.

---

## Before You Contribute

### Understand the Philosophy

This project follows Go's design principles strictly:

- **Clear is better than clever.** If you find yourself proud of how tricky your solution is, rewrite it.
- **Explicit over implicit.** No magic. What you read is what executes.
- **Composition over inheritance.** Types define capabilities, not taxonomies.
- **A little copying is better than a little dependency.** Think twice before adding external packages.

Read the handbook's [Go Design Philosophy](01_GO_DESIGN_PHILOSOPHY.md) before your first contribution. The patterns here aren't arbitrary—they're carefully chosen trade-offs.

### Check Existing Patterns

Before implementing something new, search for similar patterns in the codebase. Consistency matters more than local optimality. If the codebase uses a particular error handling style or testing approach, follow it—even if you prefer another.

---

## Contribution Guidelines

### Code Style

**Formatting is non-negotiable.** Run `gofmt` or `goimports` before every commit. Unformatted code will not be reviewed.

**Naming follows Go conventions:**

| Element | Convention | Example |
|---------|------------|---------|
| Packages | Lowercase, single-word, by responsibility | `order`, `auth`, `postgres` |
| Interfaces | `-er` suffix for single-method | `Reader`, `Handler`, `Validator` |
| Exported functions | MixedCaps, verb phrases | `ProcessOrder`, `ValidateInput` |
| Unexported | mixedCaps | `parseConfig`, `handleError` |
| Acronyms | All caps | `HTTPServer`, `XMLParser`, `ID` |

**Avoid stuttering.** A type in package `order` should be `order.Service`, not `order.OrderService`. The package name provides context.

**Package names to avoid:** `utils`, `common`, `helpers`, `models`, `types`. These tell you nothing about responsibility.

### Interface Design

> This section operationalizes the handbook's invariants from [Types and Composition](02_TYPES_AND_COMPOSITION.md) and [Interface Patterns](DD_INTERFACE_PATTERNS.md).

Follow these interface invariants:

- **The bigger the interface, the weaker the abstraction.** One method is ideal; two is good; more than three requires strong justification.
- **Don't design interfaces upfront—discover them.** Wait until you have a concrete need (multiple implementations, testing, decoupling).
- **Accept interfaces, return structs.** Functions accept the narrowest interface needed; return concrete types.
- **Define interfaces at the consumer, not the producer.** The package that uses a capability defines the interface.

**Anti-pattern—producer-defined kitchen sink interface:**

```go
// DON'T: Large interface defined alongside implementation
type Repository interface {
    Get(ctx context.Context, id string) (*Entity, error)
    Save(ctx context.Context, e *Entity) error
    Delete(ctx context.Context, id string) error
    List(ctx context.Context, filter Filter) ([]*Entity, error)
    // ... 10 more methods
}
```

**Idiomatic—consumer-defined focused interface:**

```go
// DO: Small interface defined where it's needed
type EntityGetter interface {
    Get(ctx context.Context, id string) (*Entity, error)
}
```

### Error Handling

> This section operationalizes the handbook's invariants from [Error Philosophy](03_ERROR_PHILOSOPHY.md). Errors are part of your API—treat them as carefully as your types.

**Rule: Handle an error or return it—never both.**
Logging and returning means the error gets handled twice, often inconsistently.

**Rule: Wrap errors with context using `fmt.Errorf("...: %w", err)`.**
The chain should tell a story from boundary to root cause.

**Rule: At boundaries, translate errors—don't leak internals.**
Domain errors become HTTP status codes; implementation details stay hidden from users.

**Rule: Choose error types deliberately.**
Sentinel errors for expected conditions; typed errors for actionable information; opaque errors for everything else.

```go
// Wrapping with context
if err != nil {
    return fmt.Errorf("processing order %s: %w", orderID, err)
}

// Boundary translation
func (h *Handler) ServeHTTP(w http.ResponseWriter, r *http.Request) {
    order, err := h.service.Get(r.Context(), id)
    if errors.Is(err, order.ErrNotFound) {
        http.Error(w, "order not found", http.StatusNotFound)
        return
    }
    if err != nil {
        log.Printf("internal error: %v", err) // Log internally
        http.Error(w, "internal error", http.StatusInternalServerError) // Generic to user
        return
    }
    // ...
}
```

### Context Usage

> This section operationalizes the handbook's invariants from [Context and Lifecycle](04_CONTEXT_AND_LIFECYCLE.md). Context is the backbone of lifecycle management.

**Rule: Context is the first parameter, named `ctx`.**
Always. No exceptions. This is not a style preference—it's Go convention.

**Rule: Never store context in structs.**
Context is request-scoped. Storing it conflates instance lifetime with request lifetime.

**Rule: Create context at boundaries, propagate through core.**
HTTP handlers create; domain logic receives. The boundary owns the lifecycle.

**Rule: Respect cancellation.**
Check `ctx.Done()` in long-running operations. Ignoring cancellation wastes resources and breaks shutdown.

```go
// Correct: context as first parameter
func (s *Service) ProcessOrder(ctx context.Context, orderID string) error {
    // Check for cancellation in loops or long operations
    select {
    case <-ctx.Done():
        return ctx.Err()
    default:
    }
    // ... processing
}
```

### Concurrency

> This section operationalizes the handbook's invariants from [Concurrency Architecture](05_CONCURRENCY_ARCHITECTURE.md) and [Graceful Shutdown](06_GRACEFUL_SHUTDOWN.md). Ownership is everything.

**Rule: Every goroutine must have an owner responsible for its termination.**
No fire-and-forget goroutines in production code. The goroutine that starts a goroutine must ensure it can stop.

**Rule: Share memory by communicating; don't communicate by sharing memory.**
Channels for coordination; mutexes for protecting shared state. Don't use channels where a mutex is simpler.

**Rule: The sender owns the channel; receivers never close.**
Closing signals "no more values"—only the producer knows when that's true. Closing from the receiver side causes panics.

**Rule: Use `errgroup` for structured concurrency.**
It ensures all goroutines complete before the parent returns. First error cancels context, signaling others to stop.

```go
// Structured concurrency with errgroup
g, ctx := errgroup.WithContext(ctx)

g.Go(func() error {
    return processA(ctx)
})

g.Go(func() error {
    return processB(ctx)
})

if err := g.Wait(); err != nil {
    return fmt.Errorf("processing failed: %w", err)
}
```

### Testing

> This section operationalizes the handbook's invariants from [Testing Philosophy](07_TESTING_PHILOSOPHY.md). Tests verify contracts, not implementations.

**Rule: Fakes over mocks.**
Mocks verify calls were made; fakes verify contracts work. Fakes survive refactoring; mocks break when internals change.

**Rule: Assert behavioral contracts, not call order.**
Test *what happened*, not *how*. "Order was saved" not "Save was called with these arguments."

**Rule: Table-driven tests for multiple cases.**
Clear, extensible, DRY. Each row is a scenario; the test body is the invariant.

**Rule: Never use `time.Sleep` for synchronization.**
Use channels, condition variables, or polling with timeouts. Use `goleak` to detect goroutine leaks.

**Rule: Time is a dependency; inject it.**
Functions that need `time.Now()` should accept a clock interface. Direct calls are untestable.

```go
func TestOrderService_Process(t *testing.T) {
    tests := []struct {
        name    string
        orderID string
        setup   func(*FakeRepository)
        wantErr bool
    }{
        {
            name:    "valid order processes successfully",
            orderID: "order-123",
            setup: func(r *FakeRepository) {
                r.orders["order-123"] = &Order{ID: "order-123", Status: "pending"}
            },
            wantErr: false,
        },
        {
            name:    "missing order returns error",
            orderID: "nonexistent",
            setup:   func(r *FakeRepository) {},
            wantErr: true,
        },
    }

    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            repo := NewFakeRepository()
            tt.setup(repo)
            svc := NewService(repo)

            err := svc.Process(context.Background(), tt.orderID)

            if (err != nil) != tt.wantErr {
                t.Errorf("Process() error = %v, wantErr %v", err, tt.wantErr)
            }
        })
    }
}
```

### Dependencies

> This section operationalizes the handbook's invariants from [Package and Project Design](08_PACKAGE_AND_PROJECT_DESIGN.md) and [Dependency Injection](DD_DEPENDENCY_INJECTION.md). Architecture is visible in import statements.

**Rule: Core packages know nothing about boundaries.**
Domain logic must not import HTTP, gRPC, or database packages. If it does, the abstraction is leaking.

**Rule: Boundary packages implement interfaces defined in core.**
The dependency arrow points from infrastructure toward business logic, never the reverse.

**Rule: Wire dependencies at the composition root.**
Only `main()` or an `internal/app` package should know all concrete types. Domain packages see interfaces only.

```
internal/
├── order/           # CORE: defines Repository interface
│   ├── order.go
│   ├── service.go   # Uses Repository interface
│   └── repository.go # Defines interface
├── postgres/        # BOUNDARY: implements order.Repository
│   └── order_repo.go
└── httpapi/         # BOUNDARY: calls order.Service
    └── order_handler.go
```

---

## Pull Request Process

### Before Opening a PR

1. **Run all checks locally:**
   ```bash
   go fmt ./...
   go vet ./...
   go test -race ./...
   golangci-lint run  # if configured
   ```

2. **Ensure tests pass and cover new code.** Untested code is broken code waiting to happen.

3. **Check for goroutine leaks** in concurrent code using `goleak`:
   ```go
   func TestMain(m *testing.M) {
       goleak.VerifyTestMain(m)
   }
   ```

4. **Review your own diff** before requesting review. Would you approve this if someone else submitted it?

### PR Description

Every PR description should answer:

- **What does this change?** (One sentence)
- **Why is this change needed?** (Context for reviewers)
- **How was this tested?** (Manual steps or test coverage)
- **Are there any trade-offs or alternatives considered?**

### Review Expectations

**For authors:**

- Respond to all comments, even if just to acknowledge
- Don't take feedback personally—we're reviewing code, not you
- If you disagree, explain your reasoning; be open to being wrong
- Small, focused PRs get reviewed faster than large ones

**For reviewers:**

- Be specific: "This could leak goroutines because..." not "This looks wrong"
- Distinguish blocking issues from suggestions
- Approve when concerns are addressed; don't gatekeep
- Review within 24 hours when possible

---

## Commit Messages

Follow conventional commit format:

```
<type>: <description>

[optional body]

[optional footer]
```

**Types:**

| Type | Use For |
|------|---------|
| `feat` | New features |
| `fix` | Bug fixes |
| `refactor` | Code changes that neither fix bugs nor add features |
| `test` | Adding or updating tests |
| `docs` | Documentation changes |
| `chore` | Maintenance tasks (deps, CI, etc.) |

**Good commit messages:**

```
feat: add order cancellation endpoint

Implements POST /orders/{id}/cancel with proper lifecycle
management and event emission.

Closes #123
```

```
fix: prevent goroutine leak in worker pool shutdown

Workers were not respecting context cancellation during
drain phase. Added ctx.Done() check in the work loop.
```

**Unacceptable commit messages:**

```
fix stuff
```

```
WIP
```

```
addressing review comments
```

```
refactor: code cleanup
```

Commit messages must describe *what changed*, not *why you touched the code*. "Addressing review comments" tells the reader nothing; squash those commits or rewrite them to describe the actual change.

---

## Code Review Checklist

Use this when reviewing or self-reviewing:

### Correctness

- [ ] Does the code do what it claims?
- [ ] Are error cases handled?
- [ ] Are edge cases considered?

### Clarity

- [ ] Could a new team member understand this in 6 months?
- [ ] Are names descriptive and consistent?
- [ ] Is there unnecessary cleverness?

### Lifecycle & Ownership

- [ ] Do all goroutines have owners and termination paths?
- [ ] Is context propagated correctly?
- [ ] Are resources properly cleaned up?

### Invariants

- [ ] Are the handbook's invariants respected?
- [ ] Dependencies point inward (boundary → core)?
- [ ] Interfaces are small and consumer-defined?
- [ ] Errors are handled or returned, not both?

### Testing

- [ ] Are new code paths tested?
- [ ] Do tests verify behavior, not implementation?
- [ ] Are there no flaky time-dependent assertions?

---

## Issue Reporting

When opening an issue:

### Bug Reports

Include:

1. **Go version** (`go version`)
2. **Steps to reproduce** (minimal, complete)
3. **Expected behavior**
4. **Actual behavior**
5. **Relevant logs or error messages**

### Feature Requests

Include:

1. **Problem statement** (what pain point does this address?)
2. **Proposed solution** (how you'd like it solved)
3. **Alternatives considered** (what else could work?)
4. **Trade-offs** (what are the costs?)

---

## Documentation Standards

Documentation follows the same clarity principles as code:

- **Document the why, not the what.** Code shows what; comments explain why it's not obvious.
- **Keep docs close to code.** Package-level `doc.go` for package purpose; inline comments for non-obvious decisions.
- **Update docs with code.** Stale documentation is worse than no documentation.
- **Examples compile.** Use `Example` functions in `_test.go` files for executable documentation.

### Comment Style

```go
// ProcessOrder handles the complete order lifecycle including validation,
// payment processing, and fulfillment scheduling. It's idempotent—calling
// it multiple times with the same order ID has no additional effect after
// the first successful call.
//
// The context controls the overall timeout. Individual operations have
// their own sub-timeouts derived from the remaining budget.
func (s *Service) ProcessOrder(ctx context.Context, orderID string) error {
    // ...
}
```

---

## Getting Help

- **Questions about patterns:** Open a discussion, not an issue
- **Unclear existing code:** Open an issue asking for clarification—unclear code is a bug
- **Stuck on implementation:** Draft PR with `[WIP]` prefix; ask specific questions in the description

---

## Recognition

Contributors who consistently demonstrate:

- Deep understanding of the handbook's principles
- High-quality, well-tested code
- Thoughtful, constructive code reviews
- Patience in helping others learn

will be recognized in the project's contributors list and may be invited to become maintainers.

---

## The Invariants (Quick Reference)

These must never be violated. If your contribution breaks any of these, it will not be merged:

| Category | Invariant |
|----------|-----------|
| Philosophy | Clear is better than clever |
| Philosophy | A little copying is better than a little dependency |
| Interfaces | The bigger the interface, the weaker the abstraction |
| Interfaces | Accept interfaces, return structs |
| Errors | Handle an error or return it—never both |
| Errors | Errors at boundaries must be translated, not leaked |
| Context | Never store context in structs |
| Context | Context is the first parameter, named `ctx` |
| Concurrency | Every goroutine must have an owner responsible for termination |
| Concurrency | Share memory by communicating; don't communicate by sharing |
| Shutdown | Shutdown order is reverse of startup order |
| Shutdown | Every component must have a shutdown path |
| Testing | Fakes over mocks |
| Testing | Assert behavioral contracts, not call order |
| Packages | Name by responsibility, not by type |
| Packages | Dependencies point inward: boundary → core |

---

*Thank you for contributing. Every improvement—however small—makes the codebase better for everyone.*
