# tdd

Test-driven development with red-green-refactor loop for C# projects.

Use when the user wants to build features or fix bugs using TDD, mentions
"red-green-refactor", wants integration tests, or asks for test-first development.

The test runner and framework should have been provided to you by
`/setup-matt-pocock-skills`. Default: `dotnet test`, xUnit.

---

## Core principle

Tests should verify behavior through public interfaces, not implementation details.
Code can change entirely; tests shouldn't.

**Good tests** are integration-style: they exercise real code paths through public
APIs. They describe what the system does, not how it does it. A good test reads
like a specification — `UserCanCheckOutWithValidCart` tells you exactly what
capability exists. These tests survive refactors because they don't care about
internal structure.

**Bad tests** are coupled to implementation. They mock internal collaborators,
test private methods, or verify through external means (like querying a database
directly instead of using the interface).

See `tests.md` for C# examples and `mocking.md` for mocking guidelines.

---

## The loop

Run tests with the command configured in `/setup-matt-pocock-skills`.
Default: `dotnet test` (or `dotnet test path/to/Project.Tests.csproj`).

```
RED   → Write a failing test. Confirm it fails with the right error,
        not a compile error or wrong exception.
GREEN → Write the minimum code to make it pass. Don't over-engineer.
REFACTOR → Clean up. Re-run tests. They must still pass.
```

Repeat one vertical slice at a time.

---

## DO NOT horizontal slice

DO NOT write all tests first, then all implementation. This is horizontal slicing.

**Why it fails:**
- You end up testing the shape of things (data structures, method signatures)
  rather than user-facing behavior
- Tests become insensitive to real changes — they pass when behavior breaks, fail
  when behavior is fine
- You outrun your headlights, committing to test structure before understanding
  the implementation

**Correct approach — vertical slices via tracer bullets:**

```
WRONG (horizontal):
  RED:   test1, test2, test3, test4, test5
  GREEN: impl1, impl2, impl3, impl4, impl5

RIGHT (vertical):
  RED→GREEN: test1→impl1
  RED→GREEN: test2→impl2
  RED→GREEN: test3→impl3
```

One test → one implementation → repeat. Each test responds to what you learned
from the previous cycle.

---

## Test project structure

If no test project exists yet, create one before writing tests:

```bash
dotnet new xunit -n MyProject.Tests
dotnet add MyProject.Tests/MyProject.Tests.csproj reference MyProject/MyProject.csproj
dotnet sln add MyProject.Tests/MyProject.Tests.csproj
```

Install mocking library if needed:

```bash
dotnet add MyProject.Tests package NSubstitute
# or
dotnet add MyProject.Tests package Moq
```

---

## Naming conventions

Test class: `<ClassUnderTest>Tests`
Test method: `<Method>_<Scenario>_<ExpectedResult>`

```csharp
public class OrderServiceTests
{
    [Fact]
    public void Checkout_WithValidCart_ReturnsOrderConfirmation() { }

    [Fact]
    public void Checkout_WithEmptyCart_ThrowsInvalidOperationException() { }

    [Theory]
    [InlineData(0)]
    [InlineData(-1)]
    public void SetQuantity_WithNonPositiveValue_ThrowsArgumentException(int qty) { }
}
```

---

## xUnit attributes

| Attribute | Use |
|---|---|
| `[Fact]` | Single test case, no parameters |
| `[Theory]` + `[InlineData]` | Parameterised test with literal values |
| `[Theory]` + `[MemberData]` | Parameterised test with complex objects |
| `[Theory]` + `[ClassData]` | Parameterised test with a data class |

---

## Arrange / Act / Assert

Always structure test bodies with a blank line between each phase. No comments
needed — the blank lines are the signal.

```csharp
[Fact]
public void ProcessPayment_WithSufficientFunds_DeductsBalance()
{
    var account = new BankAccount(initialBalance: 100m);
    var service = new PaymentService(account);

    service.ProcessPayment(amount: 30m);

    Assert.Equal(70m, account.Balance);
}
```

---

## Async tests

xUnit supports async natively — always use `Task` return type, never `.Result` or
`.Wait()`:

```csharp
[Fact]
public async Task FetchAnnouncement_WithValidId_ReturnsParsedAnnouncement()
{
    var service = new AnnouncementService(/* deps */);

    var result = await service.FetchAsync("WSCR-001");

    Assert.Equal("Director Dealing", result.AnnouncementType);
}
```

---

## Running tests

```bash
# Run all tests
dotnet test

# Run a specific project
dotnet test src/MyProject.Tests/MyProject.Tests.csproj

# Run tests matching a filter
dotnet test --filter "FullyQualifiedName~OrderServiceTests"

# Run with verbose output
dotnet test --logger "console;verbosity=detailed"

# Run and watch for changes
dotnet watch test
```

When a test fails, read the full output — xUnit gives expected vs actual values
and a stack trace. Fix the minimal amount of code to make it green before
proceeding to the next slice.

---

## Guardrails

- Never mock types you don't own (e.g. `HttpClient`, `DbContext`) — wrap them
  in an interface and mock the interface. See `mocking.md`.
- Never test private methods directly — if a private method needs a test, it
  probably belongs in its own class.
- Never use `Thread.Sleep` in tests — use async/await and `Task.Delay` only
  when genuinely testing timing behavior.
- Never access a database or external service in a unit test — use interfaces
  and fakes. Integration tests that hit a real DB are fine but must be clearly
  marked and run separately.
- If a test requires more than ~15 lines of Arrange, the production code is
  probably doing too much. Consider splitting the class.
