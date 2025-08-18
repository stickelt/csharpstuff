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
