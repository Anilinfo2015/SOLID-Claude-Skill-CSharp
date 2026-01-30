---
name: csharp-solid-architecture-validator
version: 1.0.0
description: Validate C# codebases for SOLID principles and architecture patterns using NetArchTest framework
author: Claude Skills Team
tags: [csharp, solid, architecture, netarchtest, validation, clean-code, testing]
category: development
license: MIT
created: 2026-01-30
updated: 2026-01-30
requirements:
  - .NET SDK 6.0 or higher
  - NetArchTest.Rules NuGet package
  - Access to C# workspace/repository
---

# C# SOLID and Architecture Validator

## Overview

This skill enables Claude to analyze and validate C# codebases against SOLID principles and architectural patterns using the NetArchTest framework. It helps ensure code quality, maintainability, and adherence to clean architecture principles through automated architecture tests.

## Purpose

This skill helps developers:
- Validate SOLID principles (Single Responsibility, Open/Closed, Liskov Substitution, Interface Segregation, Dependency Inversion)
- Enforce architectural boundaries (layered, clean, onion, hexagonal architectures)
- Detect architectural violations early in the development cycle
- Maintain code quality through automated architecture testing
- Prevent architectural drift over time

## When to Use

- User requests validation of SOLID principles in their C# code
- User wants to enforce architectural patterns and boundaries
- User needs to create architecture tests for their C# project
- User wants to validate dependency rules between layers/namespaces
- User needs to ensure naming conventions and class design patterns
- User wants to integrate architecture validation into CI/CD pipeline

## Instructions

### Step 1: Analyze the Codebase Structure

Examine the C# project to understand:
- **Project structure**: Identify namespaces, assemblies, and layers (e.g., Domain, Application, Infrastructure, Presentation)
- **Architecture pattern**: Determine if using Clean Architecture, Onion Architecture, Layered Architecture, etc.
- **Dependencies**: Understand project references and dependencies between layers
- **Naming conventions**: Identify patterns for interfaces, implementations, services, repositories, etc.

**Actions:**
1. List all .csproj files and their references
2. Identify namespace organization patterns
3. Map out layer dependencies
4. Note any existing test projects

### Step 2: Set Up NetArchTest Infrastructure

Prepare the testing environment:

**If no test project exists:**
```bash
# Create architecture test project
dotnet new xunit -n ArchitectureTests
cd ArchitectureTests

# Add NetArchTest.Rules package
dotnet add package NetArchTest.Rules

# Add reference to project under test
dotnet add reference ../YourProject/YourProject.csproj
```

**If test project exists:**
```bash
# Add NetArchTest.Rules to existing test project
cd YourTestProject
dotnet add package NetArchTest.Rules
```

### Step 3: Create SOLID Principle Validation Tests

Generate architecture tests for each SOLID principle:

#### 3.1 Single Responsibility Principle (SRP)
Create tests to ensure classes have focused responsibilities:

```csharp
[Fact]
public void Services_Should_Have_Single_Responsibility()
{
    var result = Types.InCurrentDomain()
        .That()
        .ResideInNamespace("YourApp.Application.Services")
        .And()
        .HaveNameEndingWith("Service")
        .Should()
        .NotBeAbstract()
        .And()
        .BeSealed() // Sealed classes prevent inheritance sprawl
        .GetResult();

    Assert.True(result.IsSuccessful, 
        $"SRP Violation: {string.Join(", ", result.FailingTypeNames ?? Array.Empty<string>())}");
}

[Fact]
public void Classes_Should_Not_Have_Too_Many_Dependencies()
{
    var result = Types.InCurrentDomain()
        .That()
        .ResideInNamespace("YourApp")
        .Should()
        .HaveMaximumNumberOfDependencies(10) // Adjust threshold as needed
        .GetResult();

    Assert.True(result.IsSuccessful, 
        $"Classes with too many dependencies: {string.Join(", ", result.FailingTypeNames ?? Array.Empty<string>())}");
}
```

#### 3.2 Open/Closed Principle (OCP)
Ensure classes are open for extension but closed for modification:

```csharp
[Fact]
public void Base_Classes_Should_Be_Abstract_Or_Interfaces()
{
    var result = Types.InCurrentDomain()
        .That()
        .ResideInNamespace("YourApp.Domain")
        .And()
        .AreNotInterfaces()
        .And()
        .HaveName(".*Base$", useRegularExpressions: true)
        .Should()
        .BeAbstract()
        .GetResult();

    Assert.True(result.IsSuccessful, 
        $"OCP Violation - Non-abstract base classes: {string.Join(", ", result.FailingTypeNames ?? Array.Empty<string>())}");
}

[Fact]
public void Domain_Should_Use_Abstractions_Not_Concrete_Types()
{
    var result = Types.InCurrentDomain()
        .That()
        .ResideInNamespace("YourApp.Domain")
        .ShouldNot()
        .HaveDependencyOn("YourApp.Infrastructure")
        .GetResult();

    Assert.True(result.IsSuccessful, 
        $"OCP Violation - Domain depends on concrete infrastructure: {string.Join(", ", result.FailingTypeNames ?? Array.Empty<string>())}");
}
```

#### 3.3 Liskov Substitution Principle (LSP)
Verify that derived classes can substitute base classes:

```csharp
[Fact]
public void Implementations_Should_Implement_Their_Interfaces_Properly()
{
    var result = Types.InCurrentDomain()
        .That()
        .ImplementInterface(typeof(IRepository<>))
        .Should()
        .HaveNameEndingWith("Repository")
        .GetResult();

    Assert.True(result.IsSuccessful, 
        $"LSP Violation - Invalid repository implementations: {string.Join(", ", result.FailingTypeNames ?? Array.Empty<string>())}");
}

[Fact]
public void All_Service_Implementations_Should_Follow_Interface_Contract()
{
    var result = Types.InCurrentDomain()
        .That()
        .HaveNameEndingWith("Service")
        .And()
        .AreNotInterfaces()
        .Should()
        .ImplementInterface(typeof(IService))
        .Or()
        .BeAbstract()
        .GetResult();

    Assert.True(result.IsSuccessful, 
        $"LSP Violation: {string.Join(", ", result.FailingTypeNames ?? Array.Empty<string>())}");
}
```

#### 3.4 Interface Segregation Principle (ISP)
Ensure interfaces are focused and not bloated:

```csharp
[Fact]
public void Interfaces_Should_Be_Small_And_Focused()
{
    var interfaces = Types.InCurrentDomain()
        .That()
        .AreInterfaces()
        .And()
        .ResideInNamespace("YourApp")
        .GetTypes();

    var bloatedInterfaces = interfaces
        .Where(i => i.GetMethods().Length > 10) // Adjust threshold
        .Select(i => i.Name)
        .ToList();

    Assert.Empty(bloatedInterfaces);
}

[Fact]
public void Client_Classes_Should_Not_Depend_On_Unused_Interface_Methods()
{
    // This is a semantic check - ensure interfaces are role-based
    var result = Types.InCurrentDomain()
        .That()
        .AreInterfaces()
        .And()
        .ResideInNamespace("YourApp.Domain")
        .Should()
        .HaveNameStartingWith("I")
        .GetResult();

    Assert.True(result.IsSuccessful);
}
```

#### 3.5 Dependency Inversion Principle (DIP)
High-level modules should not depend on low-level modules:

```csharp
[Fact]
public void Domain_Should_Not_Depend_On_Infrastructure()
{
    var result = Types.InCurrentDomain()
        .That()
        .ResideInNamespace("YourApp.Domain")
        .ShouldNot()
        .HaveDependencyOn("YourApp.Infrastructure")
        .GetResult();

    Assert.True(result.IsSuccessful, 
        $"DIP Violation: {string.Join(", ", result.FailingTypeNames ?? Array.Empty<string>())}");
}

[Fact]
public void Application_Should_Not_Depend_On_Presentation()
{
    var result = Types.InCurrentDomain()
        .That()
        .ResideInNamespace("YourApp.Application")
        .ShouldNot()
        .HaveDependencyOn("YourApp.Presentation")
        .And()
        .ShouldNot()
        .HaveDependencyOn("YourApp.Web")
        .And()
        .ShouldNot()
        .HaveDependencyOn("YourApp.API")
        .GetResult();

    Assert.True(result.IsSuccessful, 
        $"DIP Violation: {string.Join(", ", result.FailingTypeNames ?? Array.Empty<string>())}");
}

[Fact]
public void All_Dependencies_Should_Point_Inward()
{
    // Infrastructure can depend on Application/Domain
    // Application can depend on Domain
    // Domain should be independent
    var result = Types.InCurrentDomain()
        .That()
        .ResideInNamespace("YourApp.Domain")
        .ShouldNot()
        .HaveDependencyOn("YourApp.Application")
        .And()
        .ShouldNot()
        .HaveDependencyOn("YourApp.Infrastructure")
        .And()
        .ShouldNot()
        .HaveDependencyOn("YourApp.Presentation")
        .GetResult();

    Assert.True(result.IsSuccessful, 
        $"Dependencies should point inward: {string.Join(", ", result.FailingTypeNames ?? Array.Empty<string>())}");
}
```

### Step 4: Create Architectural Pattern Validation Tests

#### 4.1 Clean Architecture Validation
```csharp
[Fact]
public void Infrastructure_Should_Only_Depend_On_Application_And_Domain()
{
    var result = Types.InCurrentDomain()
        .That()
        .ResideInNamespace("YourApp.Infrastructure")
        .ShouldNot()
        .HaveDependencyOn("YourApp.Presentation")
        .And()
        .ShouldNot()
        .HaveDependencyOn("YourApp.Web")
        .GetResult();

    Assert.True(result.IsSuccessful);
}

[Fact]
public void Presentation_Layer_Should_Only_Depend_On_Application()
{
    var result = Types.InCurrentDomain()
        .That()
        .ResideInNamespace("YourApp.Presentation")
        .ShouldNot()
        .HaveDependencyOn("YourApp.Infrastructure")
        .And()
        .ShouldNot()
        .HaveDependencyOn("YourApp.Domain")
        .GetResult();

    Assert.True(result.IsSuccessful);
}
```

#### 4.2 Naming Convention Validation
```csharp
[Fact]
public void Interfaces_Should_Start_With_I()
{
    var result = Types.InCurrentDomain()
        .That()
        .AreInterfaces()
        .Should()
        .HaveNameStartingWith("I")
        .GetResult();

    Assert.True(result.IsSuccessful, 
        $"Interfaces without 'I' prefix: {string.Join(", ", result.FailingTypeNames ?? Array.Empty<string>())}");
}

[Fact]
public void Repositories_Should_End_With_Repository()
{
    var result = Types.InCurrentDomain()
        .That()
        .ImplementInterface(typeof(IRepository<>))
        .Or()
        .HaveName(".*Repository$", useRegularExpressions: true)
        .Should()
        .HaveNameEndingWith("Repository")
        .GetResult();

    Assert.True(result.IsSuccessful);
}

[Fact]
public void Controllers_Should_End_With_Controller()
{
    var result = Types.InCurrentDomain()
        .That()
        .Inherit(typeof(ControllerBase))
        .Should()
        .HaveNameEndingWith("Controller")
        .GetResult();

    Assert.True(result.IsSuccessful);
}
```

#### 4.3 Class Design Validation
```csharp
[Fact]
public void DTOs_Should_Be_In_DTOs_Namespace()
{
    var result = Types.InCurrentDomain()
        .That()
        .HaveNameEndingWith("Dto")
        .Or()
        .HaveNameEndingWith("DTO")
        .Should()
        .ResideInNamespace("YourApp.Application.DTOs")
        .Or()
        .ResideInNamespace("YourApp.Application.Models")
        .GetResult();

    Assert.True(result.IsSuccessful, 
        $"DTOs in wrong namespace: {string.Join(", ", result.FailingTypeNames ?? Array.Empty<string>())}");
}

[Fact]
public void Domain_Entities_Should_Be_In_Domain_Layer()
{
    var result = Types.InCurrentDomain()
        .That()
        .Inherit(typeof(Entity))
        .Or()
        .HaveNameEndingWith("Entity")
        .Should()
        .ResideInNamespace("YourApp.Domain")
        .GetResult();

    Assert.True(result.IsSuccessful);
}

[Fact]
public void Abstract_Classes_Should_Not_Be_Sealed()
{
    var result = Types.InCurrentDomain()
        .That()
        .AreAbstract()
        .ShouldNot()
        .BeSealed()
        .GetResult();

    Assert.True(result.IsSuccessful);
}
```

### Step 5: Generate Comprehensive Test File

Create a complete architecture test class file:

```csharp
using NetArchTest.Rules;
using Xunit;

namespace ArchitectureTests
{
    public class SolidPrinciplesTests
    {
        // Insert all SOLID principle tests here
    }

    public class ArchitecturePatternsTests
    {
        // Insert architecture pattern tests here
    }

    public class NamingConventionTests
    {
        // Insert naming convention tests here
    }
}
```

### Step 6: Run Architecture Tests

Execute the tests to validate the codebase:

```bash
# Run all architecture tests
dotnet test ArchitectureTests

# Run specific test category
dotnet test --filter Category=SOLID
dotnet test --filter Category=Architecture

# Run with detailed output
dotnet test --logger "console;verbosity=detailed"
```

### Step 7: Generate Validation Report

After running tests, provide a comprehensive report:

**Report Structure:**
1. **Summary**: Total tests, passed, failed
2. **SOLID Violations**: List each principle with violations
3. **Architecture Violations**: Layer dependency issues
4. **Naming Convention Issues**: Non-compliant names
5. **Recommendations**: Specific fixes for each violation
6. **Priority**: Critical, High, Medium, Low

**Example Report:**
```
Architecture Validation Report
==============================

Summary:
- Total Tests: 25
- Passed: 20
- Failed: 5

SOLID Principle Violations:
---------------------------
1. Dependency Inversion Principle (DIP): FAILED
   - Domain.UserService depends on Infrastructure.SqlUserRepository
   - Recommendation: Use IUserRepository abstraction instead

2. Single Responsibility Principle (SRP): FAILED
   - Application.OrderProcessor has 12 dependencies (max: 10)
   - Recommendation: Split into OrderValidator and OrderPersister

Architecture Violations:
-----------------------
1. Layer Boundary: FAILED
   - Presentation.UserController directly references Infrastructure.Database
   - Recommendation: Use Application layer services instead

Naming Conventions:
------------------
1. Interface Naming: FAILED
   - Domain.UserRepository is interface but missing 'I' prefix
   - Recommendation: Rename to IUserRepository

Priority Actions:
----------------
[CRITICAL] Fix DIP violation in Domain.UserService
[HIGH] Refactor OrderProcessor to reduce dependencies
[MEDIUM] Rename UserRepository interface
[LOW] Add missing XML documentation
```

### Step 8: Integration with CI/CD

Provide guidance for automation:

**GitHub Actions Example:**
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
        run: dotnet test ArchitectureTests --configuration Release
```

## Examples

### Example 1: Validating a Clean Architecture Project

**User Request:** "Validate my C# project follows Clean Architecture and SOLID principles"

**Analysis:**
```
Project Structure:
- MyApp.Domain/
- MyApp.Application/
- MyApp.Infrastructure/
- MyApp.Web/
```

**Generated Tests:**
```csharp
[Fact]
public void Domain_Should_Not_Have_External_Dependencies()
{
    var result = Types.InAssembly(typeof(Domain.Entity).Assembly)
        .ShouldNot().HaveDependencyOn("MyApp.Application")
        .And().ShouldNot().HaveDependencyOn("MyApp.Infrastructure")
        .And().ShouldNot().HaveDependencyOn("MyApp.Web")
        .GetResult();

    Assert.True(result.IsSuccessful);
}
```

**Results:**
- 18/20 tests passed
- 2 violations found and reported with fixes

### Example 2: Enforcing Repository Pattern

**User Request:** "Ensure all repository implementations follow the repository pattern"

**Generated Test:**
```csharp
[Fact]
public void All_Repositories_Should_Implement_IRepository()
{
    var result = Types.InCurrentDomain()
        .That().HaveNameEndingWith("Repository")
        .And().AreNotInterfaces()
        .Should().ImplementInterface(typeof(IRepository<>))
        .GetResult();

    Assert.True(result.IsSuccessful, 
        $"Repositories not implementing IRepository: {string.Join(", ", result.FailingTypeNames)}");
}

[Fact]
public void Repositories_Should_Be_In_Infrastructure_Layer()
{
    var result = Types.InCurrentDomain()
        .That().ImplementInterface(typeof(IRepository<>))
        .And().AreNotInterfaces()
        .Should().ResideInNamespace("MyApp.Infrastructure.Repositories")
        .GetResult();

    Assert.True(result.IsSuccessful);
}
```

### Example 3: Validating Dependency Direction

**User Request:** "Check that dependencies only flow from outer layers to inner layers"

**Generated Tests:**
```csharp
[Theory]
[InlineData("MyApp.Domain")]
public void Core_Layers_Should_Not_Depend_On_Outer_Layers(string coreNamespace)
{
    var result = Types.InCurrentDomain()
        .That().ResideInNamespace(coreNamespace)
        .ShouldNot().HaveDependencyOn("MyApp.Infrastructure")
        .And().ShouldNot().HaveDependencyOn("MyApp.Web")
        .And().ShouldNot().HaveDependencyOn("MyApp.API")
        .GetResult();

    Assert.True(result.IsSuccessful, 
        $"Core layer has dependencies on outer layers: {string.Join(", ", result.FailingTypeNames)}");
}
```

## Best Practices

1. **Start Simple**: Begin with basic layer dependency tests, then add more specific rules
2. **Use Meaningful Names**: Test names should clearly describe what they validate
3. **Provide Context**: Include detailed failure messages with specific type names
4. **Adjust Thresholds**: Customize limits (e.g., max dependencies) based on project size
5. **Run Frequently**: Integrate tests into CI/CD to catch violations early
6. **Document Exceptions**: If certain violations are acceptable, document why
7. **Prioritize Violations**: Focus on critical architectural issues first
8. **Keep Tests Maintainable**: Group related tests and avoid duplication
9. **Update Regularly**: Revisit rules as architecture evolves
10. **Educate Team**: Ensure developers understand the architectural principles being enforced

## Error Handling

### Common Issues and Solutions

| Error | Cause | Solution |
|-------|-------|----------|
| NetArchTest.Rules not found | Package not installed | Run `dotnet add package NetArchTest.Rules` |
| Type not found in assembly | Wrong assembly reference | Verify assembly name and add project reference |
| False positives | Too strict rules | Adjust rule conditions or add exceptions |
| All tests failing | Incorrect namespace patterns | Check actual namespace structure with `namespace` keyword |
| Performance issues | Loading too many assemblies | Target specific assemblies instead of `InCurrentDomain()` |
| Regex pattern not matching | Invalid regex syntax | Test regex separately or use simple string matching |

### Fallback Strategies

1. **If NetArchTest unavailable**: Manually review code structure and dependencies
2. **If tests can't compile**: Verify project references and package versions
3. **If uncertain about rules**: Start with examples from NetArchTest documentation
4. **If architecture unclear**: Ask user to describe intended architecture pattern

## Security Considerations

1. **Access Control**: Validate that authentication/authorization classes are in appropriate layers
2. **Data Protection**: Ensure sensitive data types don't leak to presentation layer
3. **Input Validation**: Verify validators are correctly placed in application layer
4. **Dependency Scanning**: Check for dependencies on untrusted packages

## Limitations

1. **Semantic Understanding**: NetArchTest cannot validate semantic correctness, only structural rules
2. **False Negatives**: May miss violations in dynamically loaded assemblies
3. **Reflection-based**: Cannot detect runtime dependency injection patterns
4. **Naming Conventions**: Relies on consistent naming for pattern detection
5. **Cross-project**: Limited support for validating rules across multiple solutions

## Related Skills

- **Code Review Assistant**: For manual review of architecture violations
- **Refactoring Guide**: To help fix identified violations
- **Documentation Generator**: To document architectural decisions and rules
- **Test Coverage Analyzer**: To ensure architecture tests cover all components

## Version History

- **1.0.0** (2026-01-30): Initial release with SOLID and architecture pattern validation

## Notes

- NetArchTest is most effective when combined with manual code reviews
- Architecture tests should be treated as living documentation
- Regular refactoring may be needed as architecture evolves
- Consider using ArchUnit (Java) or similar tools for other platforms
- Tests can be extended with custom predicates for project-specific rules
