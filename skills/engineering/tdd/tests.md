# tests.md — C# xUnit testing guidelines

## What makes a good test

A good test:
- Tests one behavior, not one method
- Has a name that reads like a specification
- Would fail if the behavior broke, and pass if it works correctly
- Doesn't care about internal implementation — only observable outcomes
- Can be understood in isolation without reading the production code

A bad test:
- Is named after a method (`TestCheckout`, `TestProcess`)
- Asserts on internal state rather than public output
- Breaks when you rename a private field
- Requires 40 lines of setup to understand what's being tested

---

## Fact examples

Simple, single-case tests with no parameters:

```csharp
[Fact]
public void DirectorDealParser_WithBuyAnnouncement_ReturnsBuyTransactionType()
{
    var parser = new DirectorDealParser();

    var result = parser.Parse("<xml>...<TransactionType>Buy</TransactionType>...</xml>");

    Assert.Equal(TransactionType.Buy, result.TransactionType);
}

[Fact]
public void DirectorDealParser_WithMissingCompanyName_ThrowsParseException()
{
    var parser = new DirectorDealParser();

    Assert.Throws<ParseException>(() =>
        parser.Parse("<xml>...<CompanyName></CompanyName>...</xml>"));
}
```

---

## Theory examples

Parameterised tests for multiple input/output pairs:

```csharp
[Theory]
[InlineData("2024-01-15", 2024, 1, 15)]
[InlineData("15/01/2024", 2024, 1, 15)]
[InlineData("January 15, 2024", 2024, 1, 15)]
public void DateParser_WithVariousFormats_ParsesCorrectly(
    string input, int year, int month, int day)
{
    var parser = new AnnouncementDateParser();

    var result = parser.Parse(input);

    Assert.Equal(new DateTime(year, month, day), result);
}

[Theory]
[InlineData("")]
[InlineData(null)]
[InlineData("   ")]
public void AnnouncementValidator_WithBlankTitle_ReturnsFalse(string title)
{
    var validator = new AnnouncementValidator();

    var result = validator.IsValid(new Announcement { Title = title });

    Assert.False(result);
}
```

---

## MemberData for complex objects

When InlineData won't hold your test data (complex types, collections):

```csharp
public class DirectorDealParserTests
{
    public static IEnumerable<object[]> InvalidXmlInputs =>
        new List<object[]>
        {
            new object[] { "<xml></xml>", "Missing required fields" },
            new object[] { "not xml at all", "Invalid XML format" },
            new object[] { "<xml><Amount>abc</Amount></xml>", "Invalid amount" },
        };

    [Theory]
    [MemberData(nameof(InvalidXmlInputs))]
    public void Parse_WithInvalidInput_ThrowsWithMessage(string input, string expectedMessage)
    {
        var parser = new DirectorDealParser();

        var ex = Assert.Throws<ParseException>(() => parser.Parse(input));

        Assert.Contains(expectedMessage, ex.Message);
    }
}
```

---

## Async tests

Always return `Task`, never block with `.Result` or `.Wait()`:

```csharp
[Fact]
public async Task AnnouncementService_WithValidId_ReturnsParsedAnnouncement()
{
    var fakeRepository = Substitute.For<IAnnouncementRepository>();
    fakeRepository.GetByIdAsync("ANN-001")
        .Returns(Task.FromResult(new RawAnnouncement { Id = "ANN-001", Xml = "..." }));

    var service = new AnnouncementService(fakeRepository);

    var result = await service.GetAsync("ANN-001");

    Assert.Equal("ANN-001", result.Id);
    Assert.NotNull(result.ParsedData);
}
```

---

## Exception testing

Prefer `Assert.Throws` for synchronous, `Assert.ThrowsAsync` for async:

```csharp
// Synchronous
[Fact]
public void SetAmount_WithNegativeValue_ThrowsArgumentOutOfRangeException()
{
    var deal = new DirectorDeal();

    Assert.Throws<ArgumentOutOfRangeException>(() => deal.SetAmount(-100m));
}

// Async
[Fact]
public async Task FetchAnnouncement_WithInvalidId_ThrowsNotFoundException()
{
    var service = new AnnouncementService(/* deps */);

    await Assert.ThrowsAsync<AnnouncementNotFoundException>(
        () => service.GetAsync("DOES-NOT-EXIST"));
}
```

---

## Collection assertions

xUnit's built-in assertions for collections:

```csharp
Assert.Empty(result.Errors);
Assert.NotEmpty(result.Announcements);
Assert.Single(result.Directors);                          // exactly one item
Assert.Equal(3, result.Transactions.Count);
Assert.Contains(result.Tags, t => t == "Director Buy");
Assert.All(result.Prices, p => Assert.True(p > 0));
```

---

## Shouldly (optional, more readable assertions)

If the project uses Shouldly, prefer it over raw Assert:

```csharp
result.TransactionType.ShouldBe(TransactionType.Buy);
result.Errors.ShouldBeEmpty();
result.Amount.ShouldBeGreaterThan(0);
result.CompanyName.ShouldContain("Ltd");
```

Install: `dotnet add package Shouldly`

---

## Test organisation

One test class per production class. File lives in the same namespace:

```
MyProject/
  Services/
    AnnouncementService.cs        → namespace MyProject.Services
MyProject.Tests/
  Services/
    AnnouncementServiceTests.cs   → namespace MyProject.Tests.Services
```

Group related tests with nested classes when a class has many distinct behaviors:

```csharp
public class AnnouncementServiceTests
{
    public class GetAsync
    {
        [Fact]
        public async Task WithValidId_ReturnsParsedAnnouncement() { }

        [Fact]
        public async Task WithInvalidId_ThrowsNotFoundException() { }
    }

    public class SaveAsync
    {
        [Fact]
        public async Task WithValidAnnouncement_PersistsToDatabase() { }
    }
}
```
