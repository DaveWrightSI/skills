# mocking.md — C# mocking guidelines

## The rule

**Only mock types you own.**

If a type comes from a NuGet package, the .NET runtime, or an external system,
wrap it in an interface you own — then mock the interface.

```csharp
// WRONG — mocking HttpClient directly is painful and fragile
var mockHttp = new Mock<HttpClient>();

// RIGHT — wrap it in an interface you control
public interface IAnnouncementHttpClient
{
    Task<string> GetAnnouncementXmlAsync(string url);
}

// Mock your interface instead
var fakeClient = Substitute.For<IAnnouncementHttpClient>();
```

---

## Preferred library: NSubstitute

NSubstitute has cleaner syntax than Moq for most cases. Use it by default unless
the project already uses Moq.

```bash
dotnet add MyProject.Tests package NSubstitute
```

---

## NSubstitute basics

### Setting up a fake

```csharp
var fakeRepo = Substitute.For<IAnnouncementRepository>();
```

### Returning a value

```csharp
fakeRepo.GetByIdAsync("ANN-001")
    .Returns(Task.FromResult(new Announcement { Id = "ANN-001" }));

// Or with a lambda for dynamic returns
fakeRepo.GetByIdAsync(Arg.Any<string>())
    .Returns(call => Task.FromResult(new Announcement { Id = call.Arg<string>() }));
```

### Throwing from a fake

```csharp
fakeRepo.GetByIdAsync("MISSING")
    .Throws(new AnnouncementNotFoundException("MISSING"));
```

### Verifying a call was made

```csharp
await fakeRepo.Received(1).SaveAsync(Arg.Is<Announcement>(a => a.Id == "ANN-001"));

// Verify it was NOT called
await fakeRepo.DidNotReceive().DeleteAsync(Arg.Any<string>());
```

### Argument matchers

```csharp
Arg.Any<string>()                          // any string
Arg.Is<string>(s => s.StartsWith("ANN"))   // matching a predicate
Arg.Is("exact-value")                      // exact match
```

---

## Moq basics (when the project already uses it)

```bash
dotnet add MyProject.Tests package Moq
```

```csharp
var mockRepo = new Mock<IAnnouncementRepository>();

// Setup return
mockRepo.Setup(r => r.GetByIdAsync("ANN-001"))
    .ReturnsAsync(new Announcement { Id = "ANN-001" });

// Setup throw
mockRepo.Setup(r => r.GetByIdAsync("MISSING"))
    .ThrowsAsync(new AnnouncementNotFoundException("MISSING"));

// Verify
mockRepo.Verify(r => r.SaveAsync(It.Is<Announcement>(a => a.Id == "ANN-001")), Times.Once);

// Use the mock object
var service = new AnnouncementService(mockRepo.Object);
```

---

## What NOT to mock

| Type | Why not | What to do instead |
|---|---|---|
| `HttpClient` | Complex to mock, tests become brittle | Use `IAnnouncementHttpClient` wrapper |
| `DbContext` / MySQL connections | Hides real SQL bugs | Use in-memory DB or real test DB |
| Simple value objects | Pointless overhead | Use the real object |
| Static classes (`DateTime.Now`) | Can't be mocked directly | Inject `IDateTimeProvider` |
| `ILogger<T>` | Usually unnecessary | Pass `NullLogger<T>.Instance` |

---

## Fakes vs mocks

For complex dependencies, a hand-written **fake** is often better than a mock:

```csharp
// Fake implementation — simpler and more readable than a mock setup
public class FakeAnnouncementRepository : IAnnouncementRepository
{
    private readonly Dictionary<string, Announcement> _store = new();

    public Task<Announcement> GetByIdAsync(string id)
    {
        _store.TryGetValue(id, out var announcement);
        return Task.FromResult(announcement
            ?? throw new AnnouncementNotFoundException(id));
    }

    public Task SaveAsync(Announcement announcement)
    {
        _store[announcement.Id] = announcement;
        return Task.CompletedTask;
    }
}
```

Use fakes when:
- The same fake is reused across many tests
- The mock setup would be longer than the fake itself
- The fake needs to maintain state across calls

---

## IDateTimeProvider pattern

Never use `DateTime.Now` or `DateTime.UtcNow` directly in production code — it
makes tests non-deterministic. Inject a provider instead:

```csharp
// Production code
public interface IDateTimeProvider
{
    DateTime UtcNow { get; }
}

public class SystemDateTimeProvider : IDateTimeProvider
{
    public DateTime UtcNow => DateTime.UtcNow;
}

// In tests
public class FakeDateTimeProvider : IDateTimeProvider
{
    public DateTime UtcNow { get; set; } = new DateTime(2024, 6, 1, 12, 0, 0);
}

// Usage in test
var fakeClock = new FakeDateTimeProvider { UtcNow = new DateTime(2024, 1, 15) };
var service = new AnnouncementService(fakeRepo, fakeClock);
```

---

## MySQL / database testing

Unit tests: mock `IAnnouncementRepository` — never touch the database.

Integration tests: use a real test database with a known seed state. Mark with a
trait so they can be run separately:

```csharp
[Trait("Category", "Integration")]
[Fact]
public async Task SaveAsync_WithValidAnnouncement_CanBeRetrieved()
{
    // Uses a real MySQL connection to a test database
}
```

Run unit tests only:
```bash
dotnet test --filter "Category!=Integration"
```

Run integration tests only:
```bash
dotnet test --filter "Category=Integration"
```
