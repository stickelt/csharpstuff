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
