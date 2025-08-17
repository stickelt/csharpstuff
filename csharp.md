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
