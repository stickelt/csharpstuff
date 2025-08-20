# C# Project Documentation

## Overview
Brief description of your C# project.

## Getting Started

### Prerequisites
- .NET SDK (version X.X or higher)
- Visual Studio or VS Code
- Any other dependencies

### Installation
```bash
# Clone the repository
git clone <repository-url>

# Navigate to project directory
cd <project-name>

# Restore dependencies
dotnet restore

# Build the project
dotnet build
```

### Running the Application
```bash
dotnet run
```

## Project Structure
```
├── src/
│   ├── Controllers/
│   ├── Models/
│   ├── Services/
│   └── Program.cs
├── tests/
└── README.md
```

## Features
- Feature 1
- Feature 2
- Feature 3

## Usage
Provide examples of how to use your application or library.

```csharp
// Example: GetProposalDetails method with Line of Business resolution
public ProposalDTO GetProposalDetails(string paNumber)
{
    var dto = /* existing load */;
    var raw = dto.Line_of_Business; // PLD text (e.g., "SAC" or "Cyber, Ships and Advanced Technologies")
    var resolved = _lobResolver.Resolve(raw);

    if (resolved is { } lob)
    {
        dto.LineOfBusiness   = lob.Name;  // exact picklist text (with '&' / casing)
        dto.LineOfBusinessID = lob.Id;    // e.g., 2005
    }
    else
    {
        dto.LineOfBusinessID = null;      // let UI validate if unmatched
        dto.LineOfBusiness   = raw;
    }

    return dto;
}

// LobResolver class for resolving Line of Business text to standardized values
public sealed class LobResolver
{
    private readonly Dictionary<string,(int Id,string Name)> _exact;
    private readonly Dictionary<string,(int Id,string Name)> _relaxed;
    private readonly Dictionary<string,string> _aliases;

    public LobResolver(LineOfBusinessDataLoader lobLoader)
    {
        var pick = lobLoader.GetPickListValues(); // Id/Text
        _exact = pick.ToDictionary(
            p => p.Text.Trim().ToLowerInvariant(),
            p => (p.Id, p.Text));

        _relaxed = _exact.ToDictionary(kvp => Relax(kvp.Key), kvp => kvp.Value);

        _aliases = new(StringComparer.OrdinalIgnoreCase)
        {
            ["sac"]  = "sikorsky",
            ["iwfs"] = "integrated warfare systems and sensors",
            ["tls"]  = "training and logistics solutions",
            ["c6isr"]= "c6isr",
            ["cyber ships and advanced technologies"]  = "cyber, ships & advanced technologies",
            ["cyber, ships and advanced technologies"] = "cyber, ships & advanced technologies",
            ["cyber ships & advanced technologies"]    = "cyber, ships & advanced technologies"
        };
    }

    public (int Id,string Name)? Resolve(string raw)
    {
        if (string.IsNullOrWhiteSpace(raw)) return null;
        var key = raw.Trim().ToLowerInvariant();
        var canonical = _aliases.TryGetValue(key, out var mapped) ? mapped : key;

        if (_exact.TryGetValue(canonical, out var hit)) return hit;
        return _relaxed.TryGetValue(Relax(canonical), out hit) ? hit : null;
    }

    private static string Relax(string s) =>
        System.Text.RegularExpressions.Regex.Replace(s.Replace("&","and"), @"\W+", "")
                                            .ToLowerInvariant();
}

// Alternative Simplified Design: Static Mapper Approach
// This approach centralizes alias logic in a static helper for cleaner architecture

public static class LineOfBusinessMapper
{
    private static readonly Dictionary<string, string> _aliases = new(StringComparer.OrdinalIgnoreCase)
    {
        { "sac", "Sikorsky" },
        { "infss", "Integrated Warfare Systems and Sensors" },
        { "tlss", "Training and Logistics Solutions" },
        { "cdism", "CDISM" },
        { "cyber ships and advanced technologies", "Cyber, Ships & Advanced Technologies" }
    };

    public static string Resolve(string raw)
    {
        if (string.IsNullOrWhiteSpace(raw)) return raw;
        return _aliases.TryGetValue(raw.Trim(), out var resolved) ? resolved : raw;
    }
}

// Simplified Controller using the static mapper
public JsonResult GetProposalDetails(string paNumber)
{
    ProposalDTO result = _pldDTODataLoader.GetProposalDetails(paNumber);

    if (result != null)
    {
        result.Line_of_Business = LineOfBusinessMapper.Resolve(result.Line_of_Business);
    }

    return Json(result, JsonRequestBehavior.AllowGet);
}
```

## Architecture Design Notes

### Line of Business Resolution Strategy

The goal is to handle Line of Business alias resolution server-side in C# rather than client-side in Angular. This provides:

- **Centralized Logic**: All alias mappings in one place
- **Clean Data**: UI receives standardized values
- **Single Responsibility**: One method handles both retrieval and resolution

### Design Approaches

#### 1. Full Resolver Pattern (LobResolver)
- Uses dependency injection with `LineOfBusinessDataLoader`
- Supports exact matching, relaxed matching, and aliases
- More robust for complex scenarios with picklist validation
- Best for enterprise applications with strict data governance

#### 2. Static Mapper Pattern (LineOfBusinessMapper)
- Simple static dictionary for common aliases
- Lightweight and fast
- Easier to maintain for straightforward mappings
- Best for applications with known, stable alias sets

### Key Benefits
- **Server-side Resolution**: No client-side alias logic needed
- **Unified Data Flow**: `GetProposalDetails()` returns clean, resolved data
- **Maintainability**: Aliases managed in C# code, not scattered across UI
- **Performance**: Resolution happens once on server vs. multiple times on client

### When to Use Each Approach
- Use **LobResolver** when you need picklist validation and complex matching rules
- Use **LineOfBusinessMapper** when you have simple, known aliases to resolve
- Use **LineOfBusinessHelper** when you need to combine aliases with official picklist validation

## Recommended Approach: LineOfBusinessHelper with Picklist Integration

This approach combines the best of both worlds - alias mapping with official picklist validation:

```csharp
// LineOfBusinessHelper.cs - Combines aliases with picklist validation
public static class LineOfBusinessHelper
{
    private static readonly Dictionary<string, string> _aliases = new(StringComparer.OrdinalIgnoreCase)
    {
        { "sac", "Sikorsky" },
        { "infss", "Integrated Warfare Systems and Sensors" },
        { "tlss", "Training and Logistics Solutions" },
        { "cdism", "CDISM" },
        { "cyber ships and advanced technologies", "Cyber, Ships & Advanced Technologies" }
    };

    public static string Resolve(string raw, IEnumerable<PickListDTO> pickList)
    {
        if (string.IsNullOrWhiteSpace(raw))
            return raw;

        // First, try alias map
        if (_aliases.TryGetValue(raw.Trim(), out var mapped))
            raw = mapped;

        // Then, resolve against the picklist
        var match = pickList.FirstOrDefault(p =>
            string.Equals(p.Text, raw, StringComparison.OrdinalIgnoreCase));

        return match?.Text ?? raw;
    }
}

// Updated PldDTODataLoader.GetProposalDetails implementation
public ProposalDTO GetProposalDetails(string paNumber)
{
    ProposalDTO result = ctx.Proposals
        .AsNoTracking()
        .Where(p => p.PA_Number == paNumber)
        .Select(p => new ProposalDTO
        {
            PA_Number = p.PA_Number,
            Line_of_Business = p.Line_of_Business,
            Project_Start_Date = p.Project_Start_Date,
            Project_End_Date = p.Project_End_Date
        })
        .FirstOrDefault();

    if (result != null)
    {
        var pickList = _lineOfBusinessDataLoader.GetPickListValues();
        result.Line_of_Business = LineOfBusinessHelper.Resolve(result.Line_of_Business, pickList);
    }

    return result;
}
```

### Why This Approach Works Best

✅ **Uses Official Picklist**: Validates against the actual picklist data
✅ **Supports Legacy Codes**: Maps "sac", "cdism", etc. to official names
✅ **Server-side Clean**: Angular receives standardized values
✅ **Maintainable**: Single source of truth for aliases
✅ **Flexible**: Easy to add new aliases or modify existing ones

### Component Responsibilities

| Component | Role |
|-----------|------|
| `_lineOfBusinessDataLoader.GetPickListValues()` | Source of valid business types |
| `_aliases` dictionary | Maps legacy codes like "sac" to official names |
| `LineOfBusinessHelper.Resolve()` | Combines alias and picklist resolution |
| `GetProposalDetails()` | Calls Resolve() before returning DTO |

## Unit Testing Examples

Here's a comprehensive unit test example for the `WorkspaceControllerLogic.NextTrackingNumber` method:

```csharp
using System;
using System.Collections.Generic;
using System.Linq;
using Microsoft.VisualStudio.TestTools.UnitTesting;
using Moq;

// adjust namespaces to your real ones:
using GenBOE.ActionLogic.Web.ControllerLogic;
using GenBOE.DTO; // for WorkspaceDTO

namespace GenBOE.Tests.ActionLogic.Web.ControllerLogic
{
    [TestClass]
    public class WorkspaceControllerLogic_NextTrackingNumber_Tests
    {
        private WorkspaceControllerLogic CreateSut()
        {
            // If the ctor needs many deps, create Moq stubs for each.
            // Because NextTrackingNumber uses NONE of them, any dummy values are fine.
            // Example with a single-arg ctor:
            // var dep = new Mock<ISomeDependency>(MockBehavior.Strict);
            // return new WorkspaceControllerLogic(dep.Object);

            // If there's a parameterless ctor, use it:
            return new WorkspaceControllerLogic();
        }

        [TestMethod]
        public void NextTrackingNumber_WhenNoExistingMatches_ReturnsBasePaNumber()
        {
            // Arrange
            WorkspaceControllerLogic sut = CreateSut();
            IEnumerable<WorkspaceDTO> workspaces = new List<WorkspaceDTO>
            {
                new WorkspaceDTO { Shortname = "SOMETHING-ELSE" },
                new WorkspaceDTO { Shortname = "OTHER" }
            };
            string paNumber = "PA";

            // Act
            Dictionary<string, object> result = sut.NextTrackingNumber(workspaces, paNumber);

            // Assert
            Assert.IsNotNull(result);
            Assert.IsTrue(result.ContainsKey("ShortName"));
            Assert.AreEqual("PA", result["ShortName"]);
            Assert.AreEqual("PA", result["TrackingNumber"]);
            Assert.AreEqual(0, (int)result["Id"]);
            Assert.AreEqual(string.Empty, (string)result["WorkspaceName"]);
        }

        [TestMethod]
        public void NextTrackingNumber_WhenExistingSuffixes_ReturnsIncrementedSuffix()
        {
            // Arrange
            WorkspaceControllerLogic sut = CreateSut();
            IEnumerable<WorkspaceDTO> workspaces = new List<WorkspaceDTO>
            {
                new WorkspaceDTO { Shortname = "PA" },          // exact match -> relevant
                new WorkspaceDTO { Shortname = "PA-001" },
                new WorkspaceDTO { Shortname = "PA-007" },      // max = 7
                new WorkspaceDTO { Shortname = "PA-003" },
                new WorkspaceDTO { Shortname = "IRRELEVANT" }
            };
            string paNumber = "PA";

            // Act
            Dictionary<string, object> result = sut.NextTrackingNumber(workspaces, paNumber);

            // Assert
            Assert.IsNotNull(result);
            Assert.AreEqual("PA-008", result["ShortName"]); // padded increment of max
            Assert.AreEqual("PA", result["TrackingNumber"]);
        }

        [TestMethod]
        [ExpectedException(typeof(ArgumentNullException))]
        public void NextTrackingNumber_WhenWorkspacesIsNull_Throws()
        {
            // Arrange
            WorkspaceControllerLogic sut = CreateSut();
            IEnumerable<WorkspaceDTO> workspaces = null;
            string paNumber = "PA";

            // Act
            sut.NextTrackingNumber(workspaces, paNumber);

            // Assert handled by ExpectedException
        }

        [TestMethod]
        [ExpectedException(typeof(ArgumentException))]
        public void NextTrackingNumber_WhenPaNumberBlank_Throws()
        {
            // Arrange
            WorkspaceControllerLogic sut = CreateSut();
            IEnumerable<WorkspaceDTO> workspaces = Enumerable.Empty<WorkspaceDTO>();
            string paNumber = "  ";

            // Act
            sut.NextTrackingNumber(workspaces, paNumber);

            // Assert handled by ExpectedException
        }
    }
}
```

## API Documentation
If applicable, document your API endpoints here.

## Testing
```bash
# Run all tests
dotnet test

# Run tests with coverage
dotnet test --collect:"XPlat Code Coverage"
```

## Contributing
Guidelines for contributing to the project.

## License
Specify the license for your project.

## Contact
Your contact information or project maintainer details.
