# C# SOLID and Architecture Validation - Claude Skill

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)
[![.NET](https://img.shields.io/badge/.NET-6.0%2B-512BD4)](https://dotnet.microsoft.com/)
[![NetArchTest](https://img.shields.io/badge/NetArchTest-1.3.2-blue)](https://github.com/BenMorris/NetArchTest)

A comprehensive Claude skill for validating C# codebases against SOLID principles and architectural patterns using the NetArchTest framework. This skill enables automated architecture testing, helping developers maintain clean code architecture and prevent architectural drift.

## Overview

This skill provides Claude with the ability to:
- ✅ Validate SOLID principles (SRP, OCP, LSP, ISP, DIP)
- 🏗️ Enforce architectural patterns (Clean, Onion, Hexagonal, Layered)
- 🔍 Detect layer dependency violations
- 📏 Verify naming conventions and design patterns
- 🧪 Generate comprehensive architecture tests
- 📊 Provide detailed violation reports with remediation guidance

## Quick Start

### Prerequisites

- .NET SDK 6.0 or higher
- NetArchTest.Rules NuGet package
- C# project with test framework (xUnit, NUnit, or MSTest)

### Installation

1. Add NetArchTest to your test project:
```bash
dotnet add package NetArchTest.Rules
```

2. Reference the skill in your Claude conversation:
```
Use the C# SOLID and Architecture Validator skill to check my codebase.
```

## Usage Examples

### Example 1: Validate Clean Architecture

**Request:**
```
Validate my C# project follows Clean Architecture principles and SOLID
```

**Claude will:**
1. Analyze project structure and identify layers
2. Generate architecture tests for layer dependencies
3. Create SOLID principle validation tests
4. Run tests and provide detailed violation report
5. Suggest specific fixes for each violation

### Example 2: Check SOLID Principles

**Request:**
```
Check if my services follow the Single Responsibility Principle
```

**Claude will:**
1. Analyze service classes for dependency counts
2. Check method counts per class
3. Validate namespace organization
4. Report violations with recommendations

### Example 3: Enforce Dependency Direction

**Request:**
```
Ensure dependencies only flow from outer to inner layers
```

**Claude will:**
1. Map dependency graph
2. Validate dependency direction
3. Identify circular dependencies
4. Generate tests to prevent future violations

## File Structure

```
SOLID-Claude-Skill-CSharp/
├── SKILL.md          # Main skill documentation with instructions
├── examples.md       # Comprehensive examples and patterns
├── reference.md      # Technical reference and API documentation
└── README.md         # This file
```

## Features

### SOLID Principle Validation

- **Single Responsibility Principle**: Detect classes with too many dependencies or responsibilities
- **Open/Closed Principle**: Ensure proper abstraction and extension points
- **Liskov Substitution Principle**: Validate inheritance hierarchies and contracts
- **Interface Segregation Principle**: Check interface sizes and cohesion
- **Dependency Inversion Principle**: Enforce dependency direction and abstraction usage

### Architecture Pattern Support

- **Clean Architecture**: Domain → Application → Infrastructure → Presentation
- **Onion Architecture**: Dependencies point inward to core
- **Hexagonal Architecture**: Ports and Adapters pattern
- **Layered Architecture**: Traditional N-tier validation
- **Microservices**: Service boundary and isolation checks

### Design Pattern Validation

- Repository Pattern
- CQRS Pattern
- Factory Pattern
- Strategy Pattern
- Builder Pattern
- And more...

## Documentation

### Core Documents

1. **[SKILL.md](SKILL.md)** - Complete skill instructions
   - When to use the skill
   - Step-by-step validation process
   - Test generation templates
   - Report formatting guidelines

2. **[examples.md](examples.md)** - Practical examples
   - Complete Clean Architecture example
   - SOLID principle examples for each principle
   - Architecture pattern examples
   - Advanced custom rules

3. **[reference.md](reference.md)** - Technical reference
   - NetArchTest API documentation
   - SOLID principles in detail
   - Architecture patterns explained
   - Troubleshooting guide
   - Advanced topics

## Sample Validation Report

```
Architecture Validation Report
==============================

Summary:
- Total Tests: 25
- Passed: 20 ✅
- Failed: 5 ❌

SOLID Principle Violations:
---------------------------
❌ Dependency Inversion Principle (DIP)
   - Domain.UserService depends on Infrastructure.SqlUserRepository
   - Recommendation: Use IUserRepository abstraction

❌ Single Responsibility Principle (SRP)
   - Application.OrderProcessor has 12 dependencies (max: 10)
   - Recommendation: Split into OrderValidator and OrderPersister

Architecture Violations:
-----------------------
❌ Layer Boundary Violation
   - Presentation.UserController references Infrastructure.Database
   - Recommendation: Use Application layer services

Priority Actions:
----------------
[CRITICAL] Fix DIP violation in Domain.UserService
[HIGH] Refactor OrderProcessor to reduce dependencies
[MEDIUM] Restructure Presentation layer dependencies
```

## Integration with CI/CD

### GitHub Actions

```yaml
name: Architecture Tests

on: [push, pull_request]

jobs:
  architecture-validation:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - name: Setup .NET
        uses: actions/setup-dotnet@v3
        with:
          dotnet-version: '8.0.x'
      - name: Run Architecture Tests
        run: dotnet test ArchitectureTests
```

### Azure DevOps

```yaml
- task: DotNetCoreCLI@2
  displayName: 'Run Architecture Tests'
  inputs:
    command: 'test'
    projects: '**/ArchitectureTests.csproj'
```

## Best Practices

1. **Start Simple**: Begin with basic layer dependency tests
2. **Iterate**: Add more specific rules over time
3. **Document**: Explain why certain rules exist
4. **Automate**: Run tests in CI/CD pipeline
5. **Review**: Regularly update rules as architecture evolves
6. **Educate**: Use tests to teach architectural principles

## Limitations

- Cannot validate semantic correctness (only structural)
- Limited support for runtime dependency injection patterns
- Requires consistent naming conventions for effective pattern detection
- May produce false positives with overly strict rules

## Contributing

Contributions are welcome! This skill can be extended with:
- Additional architecture patterns
- More SOLID validation strategies
- Custom rule templates
- Language-specific patterns

## License

This project is licensed under the MIT License - see the [LICENSE](LICENSE) file for details.

## Resources

### Tools
- [NetArchTest](https://github.com/BenMorris/NetArchTest) - .NET architecture testing framework
- [xUnit](https://xunit.net/) - Testing framework
- [NUnit](https://nunit.org/) - Alternative testing framework

### Books
- "Clean Architecture" by Robert C. Martin
- "Implementing Domain-Driven Design" by Vaughn Vernon
- "Patterns of Enterprise Application Architecture" by Martin Fowler

### Articles
- [SOLID Principles](https://en.wikipedia.org/wiki/SOLID)
- [Clean Architecture Guide](https://blog.cleancoder.com/uncle-bob/2012/08/13/the-clean-architecture.html)
- [Architecture Testing with NetArchTest](https://code-maze.com/csharp-architecture-tests-with-netarchtest-rules/)

## Support

For issues, questions, or suggestions:
- Review the [reference.md](reference.md) for detailed documentation
- Check [examples.md](examples.md) for usage patterns
- Consult the main [SKILL.md](SKILL.md) for instructions

## Version History

- **1.0.0** (2026-01-30)
  - Initial release
  - Full SOLID principle validation
  - Clean, Onion, Hexagonal, and Layered architecture support
  - Comprehensive documentation and examples

---

**Made with ❤️ for clean code architecture**
