# C# SOLID and Architecture Validation - Technical Reference

## Table of Contents

1. [Introduction](#introduction)
2. [NetArchTest Framework Overview](#netarchtest-framework-overview)
3. [SOLID Principles in Detail](#solid-principles-in-detail)
4. [Architecture Patterns](#architecture-patterns)
5. [NetArchTest API Reference](#netarchtest-api-reference)
6. [Testing Strategies](#testing-strategies)
7. [Best Practices](#best-practices)
8. [Common Patterns](#common-patterns)
9. [Troubleshooting](#troubleshooting)
10. [Advanced Topics](#advanced-topics)

---

## Introduction

This reference guide provides comprehensive technical documentation for validating SOLID principles and architectural patterns in C# codebases using the NetArchTest framework. It serves as a complete resource for understanding architectural testing concepts, implementation patterns, and best practices.

### What is Architecture Testing?

Architecture testing is the practice of writing automated tests to validate that your codebase adheres to intended architectural patterns and design principles. Unlike functional tests that verify behavior, architecture tests verify structure and dependencies.

### Benefits

1. **Early Detection**: Catch architectural violations during development, not in code review
2. **Automated Enforcement**: Continuously validate architecture without manual checks
3. **Documentation**: Tests serve as executable documentation of architectural decisions
4. **Regression Prevention**: Prevent architectural erosion over time
5. **Team Alignment**: Ensure all developers follow the same architectural guidelines

---

## NetArchTest Framework Overview

### What is NetArchTest?

NetArchTest is a fluent API for .NET that enforces architectural rules and conventions through unit tests. It uses reflection to analyze assemblies and validate that code structure matches defined rules.

### Installation

```bash
# Using .NET CLI
dotnet add package NetArchTest.Rules

# Using Package Manager Console
Install-Package NetArchTest.Rules

# Using PackageReference
<PackageReference Include="NetArchTest.Rules" Version="1.3.2" />
```

### Core Concepts

#### 1. Type Selection
NetArchTest allows you to select types using various criteria:

```csharp
// Select types from current domain
Types.InCurrentDomain()

// Select types from specific assembly
Types.InAssembly(assembly)

// Select types from multiple assemblies
Types.InAssemblies(new[] { assembly1, assembly2 })

// Select types from namespace
Types.InNamespace("MyApp.Domain")
```

#### 2. Predicates (That/And/Or)
Filter selected types using predicates:

```csharp
Types.InCurrentDomain()
    .That()                              // Start predicate chain
    .ResideInNamespace("MyApp.Domain")   // Filter by namespace
    .And()                               // Combine predicates
    .HaveNameEndingWith("Service")       // Filter by name pattern
    .Or()                                // Alternative condition
    .ImplementInterface(typeof(IService)) // Filter by interface
```

#### 3. Conditions (Should/ShouldNot)
Define what the selected types should or should not do:

```csharp
.Should()
    .BeSealed()                    // Types should be sealed
    .BePublic()                    // Types should be public
    .HaveNameStartingWith("I")     // Names should start with I
    .ImplementInterface(typeof(T))  // Should implement interface

.ShouldNot()
    .HaveDependencyOn("Namespace")  // Should not depend on namespace
    .BeAbstract()                   // Should not be abstract
    .ResideInNamespace("BadNamespace") // Should not be in namespace
```

#### 4. Execution
Execute the rule and get results:

```csharp
var result = Types.InCurrentDomain()
    .That().ResideInNamespace("MyApp.Domain")
    .ShouldNot().HaveDependencyOn("MyApp.Infrastructure")
    .GetResult();

// Check if successful
Assert.True(result.IsSuccessful);

// Get failing types
var failures = result.FailingTypeNames;
```

---

## SOLID Principles in Detail

### Single Responsibility Principle (SRP)

**Definition**: A class should have only one reason to change.

#### Why It Matters
- Improves maintainability
- Reduces coupling
- Easier to test
- Clearer code purpose

#### Validation Strategies

**1. Constructor Dependency Count**
```csharp
[Fact]
public void Classes_Should_Not_Have_Too_Many_Dependencies()
{
    var classes = Types.InCurrentDomain()
        .That().AreClasses()
        .And().AreNotAbstract()
        .GetTypes();

    var violations = classes
        .Where(c => c.GetConstructors()
            .Any(ctor => ctor.GetParameters().Length > 7))
        .Select(c => c.Name)
        .ToList();

    Assert.Empty(violations);
}
```

**2. Method Count**
```csharp
[Fact]
public void Classes_Should_Not_Have_Too_Many_Public_Methods()
{
    var classes = Types.InCurrentDomain()
        .That().AreClasses()
        .GetTypes();

    var violations = classes
        .Where(c => c.GetMethods(BindingFlags.Public | BindingFlags.Instance)
            .Count(m => !m.IsSpecialName) > 15)
        .Select(c => $"{c.Name} has {c.GetMethods(BindingFlags.Public | BindingFlags.Instance).Count(m => !m.IsSpecialName)} methods")
        .ToList();

    Assert.Empty(violations);
}
```

**3. Namespace Organization**
```csharp
[Fact]
public void Services_Should_Be_Focused_By_Namespace()
{
    // User-related services should be in UserServices namespace
    var result = Types.InCurrentDomain()
        .That().HaveNameMatching(@"User.*Service$", useRegularExpressions: true)
        .Should().ResideInNamespace("MyApp.Application.UserServices")
        .GetResult();

    Assert.True(result.IsSuccessful);
}
```

#### Anti-Patterns to Detect
- **God Classes**: Classes with too many responsibilities
- **Manager/Util Classes**: Vague names indicating unclear responsibility
- **High Coupling**: Too many dependencies

### Open/Closed Principle (OCP)

**Definition**: Software entities should be open for extension but closed for modification.

#### Why It Matters
- Enables plugin architectures
- Reduces risk of breaking existing code
- Promotes reusability
- Facilitates new feature addition

#### Validation Strategies

**1. Base Classes Should Be Abstract**
```csharp
[Fact]
public void Base_Classes_Must_Be_Abstract()
{
    var result = Types.InCurrentDomain()
        .That().HaveNameEndingWith("Base")
        .Or().HaveNameMatching(@".*BaseClass$", useRegularExpressions: true)
        .Should().BeAbstract()
        .GetResult();

    Assert.True(result.IsSuccessful);
}
```

**2. Strategy Pattern Validation**
```csharp
[Fact]
public void Strategies_Should_Implement_Strategy_Interface()
{
    var result = Types.InCurrentDomain()
        .That().ResideInNamespace("MyApp.Strategies")
        .And().AreClasses()
        .And().AreNotAbstract()
        .Should().ImplementInterface(typeof(IStrategy))
        .GetResult();

    Assert.True(result.IsSuccessful);
}
```

**3. Core Should Not Depend on Implementations**
```csharp
[Fact]
public void Core_Should_Not_Know_About_Specific_Implementations()
{
    var result = Types.InCurrentDomain()
        .That().ResideInNamespace("MyApp.Core")
        .ShouldNot().HaveDependencyOn("MyApp.Implementations")
        .GetResult();

    Assert.True(result.IsSuccessful);
}
```

### Liskov Substitution Principle (LSP)

**Definition**: Objects of a superclass should be replaceable with objects of a subclass without breaking the application.

#### Why It Matters
- Ensures proper inheritance hierarchies
- Maintains contract integrity
- Enables polymorphism
- Prevents unexpected behavior

#### Validation Strategies

**1. Subtype Naming Consistency**
```csharp
[Fact]
public void Repository_Implementations_Should_Follow_Naming_Convention()
{
    var result = Types.InCurrentDomain()
        .That().ImplementInterface(typeof(IRepository<>))
        .And().AreNotInterfaces()
        .Should().HaveNameEndingWith("Repository")
        .GetResult();

    Assert.True(result.IsSuccessful);
}
```

**2. Inheritance Depth**
```csharp
[Fact]
public void Inheritance_Hierarchy_Should_Not_Be_Too_Deep()
{
    var types = Types.InCurrentDomain()
        .That().AreClasses()
        .GetTypes();

    var violations = types
        .Where(t => GetInheritanceDepth(t) > 4)
        .Select(t => $"{t.Name} has depth {GetInheritanceDepth(t)}")
        .ToList();

    Assert.Empty(violations);
}

private int GetInheritanceDepth(Type type)
{
    int depth = 0;
    var current = type.BaseType;
    while (current != null && current != typeof(object))
    {
        depth++;
        current = current.BaseType;
    }
    return depth;
}
```

**3. Exception Hierarchy**
```csharp
[Fact]
public void Custom_Exceptions_Should_Inherit_From_Base_Exception()
{
    var result = Types.InCurrentDomain()
        .That().HaveNameEndingWith("Exception")
        .And().DoNotHaveName("Exception")
        .Should().Inherit(typeof(Exception))
        .GetResult();

    Assert.True(result.IsSuccessful);
}
```

### Interface Segregation Principle (ISP)

**Definition**: Clients should not be forced to depend on interfaces they don't use.

#### Why It Matters
- Reduces coupling
- Improves flexibility
- Easier to implement
- Better testability

#### Validation Strategies

**1. Interface Size**
```csharp
[Fact]
public void Interfaces_Should_Be_Small_And_Focused()
{
    var interfaces = Types.InCurrentDomain()
        .That().AreInterfaces()
        .GetTypes();

    var bloatedInterfaces = interfaces
        .Where(i => i.GetMembers().Length > 7)
        .Select(i => $"{i.Name} has {i.GetMembers().Length} members")
        .ToList();

    Assert.Empty(bloatedInterfaces);
}
```

**2. Role-Based Interfaces**
```csharp
[Fact]
public void Read_And_Write_Operations_Should_Be_Separated()
{
    // IReadRepository should exist
    var readInterface = Types.InCurrentDomain()
        .That().AreInterfaces()
        .And().HaveName("IReadRepository")
        .GetTypes();

    // IWriteRepository should exist
    var writeInterface = Types.InCurrentDomain()
        .That().AreInterfaces()
        .And().HaveName("IWriteRepository")
        .GetTypes();

    Assert.NotEmpty(readInterface);
    Assert.NotEmpty(writeInterface);
}
```

**3. Client-Specific Interfaces**
```csharp
[Fact]
public void Interfaces_Should_Be_Cohesive()
{
    // Check that interfaces have focused, role-based names
    var result = Types.InCurrentDomain()
        .That().AreInterfaces()
        .And().ResideInNamespace("MyApp.Domain.Interfaces")
        .Should().HaveNameMatching(@"^I[A-Z].*?(Reader|Writer|Handler|Provider|Factory|Validator)$", useRegularExpressions: true)
        .Or().BeEmpty()
        .GetResult();

    Assert.True(result.IsSuccessful);
}
```

### Dependency Inversion Principle (DIP)

**Definition**: High-level modules should not depend on low-level modules. Both should depend on abstractions.

#### Why It Matters
- Decouples layers
- Enables testing
- Facilitates change
- Improves maintainability

#### Validation Strategies

**1. Layer Dependencies**
```csharp
[Fact]
public void High_Level_Should_Not_Depend_On_Low_Level()
{
    var result = Types.InCurrentDomain()
        .That().ResideInNamespace("MyApp.Domain")
        .Or().ResideInNamespace("MyApp.Application")
        .ShouldNot().HaveDependencyOn("MyApp.Infrastructure")
        .And().ShouldNot().HaveDependencyOn("MyApp.Persistence")
        .GetResult();

    Assert.True(result.IsSuccessful);
}
```

**2. Concrete Dependencies**
```csharp
[Fact]
public void Business_Logic_Should_Not_Depend_On_Concrete_Implementations()
{
    var result = Types.InCurrentDomain()
        .That().ResideInNamespace("MyApp.Business")
        .ShouldNot().HaveDependencyOn("System.Data.SqlClient")
        .And().ShouldNot().HaveDependencyOn("Microsoft.EntityFrameworkCore")
        .And().ShouldNot().HaveDependencyOn("Npgsql")
        .GetResult();

    Assert.True(result.IsSuccessful);
}
```

**3. Dependency Direction**
```csharp
[Fact]
public void Dependencies_Should_Point_To_Abstractions()
{
    // Infrastructure can depend on Application interfaces
    var result = Types.InCurrentDomain()
        .That().ResideInNamespace("MyApp.Infrastructure")
        .And().HaveDependencyOn("MyApp.Application")
        .Should().HaveDependencyOn("MyApp.Application.Interfaces")
        .Or().BeEmpty()
        .GetResult();

    Assert.True(result.IsSuccessful);
}
```

---

## Architecture Patterns

### Clean Architecture

**Layers** (from inner to outer):
1. **Domain/Entities**: Core business logic, no dependencies
2. **Application/Use Cases**: Application business rules, depends only on Domain
3. **Infrastructure**: External concerns (database, APIs), depends on Application
4. **Presentation**: UI/API layer, depends on Application

**Validation Template**:
```csharp
public class CleanArchitectureTests
{
    [Fact]
    public void Domain_Has_No_Dependencies()
    {
        var result = Types.InAssembly(domainAssembly)
            .ShouldNot().HaveDependencyOnAny(
                "Application", "Infrastructure", "Presentation")
            .GetResult();
        Assert.True(result.IsSuccessful);
    }

    [Fact]
    public void Application_Depends_Only_On_Domain()
    {
        var result = Types.InAssembly(applicationAssembly)
            .ShouldNot().HaveDependencyOnAny(
                "Infrastructure", "Presentation")
            .GetResult();
        Assert.True(result.IsSuccessful);
    }

    [Fact]
    public void Infrastructure_Does_Not_Depend_On_Presentation()
    {
        var result = Types.InAssembly(infrastructureAssembly)
            .ShouldNot().HaveDependencyOn("Presentation")
            .GetResult();
        Assert.True(result.IsSuccessful);
    }
}
```

### Onion Architecture

Similar to Clean Architecture but emphasizes domain-centric design with dependencies pointing inward.

**Validation**:
```csharp
[Fact]
public void All_Dependencies_Point_Inward()
{
    // Core (innermost) - no dependencies
    var coreResult = Types.InCurrentDomain()
        .That().ResideInNamespace("MyApp.Core")
        .ShouldNot().HaveDependencyOnAny(
            "MyApp.DomainServices",
            "MyApp.ApplicationServices",
            "MyApp.Infrastructure",
            "MyApp.UI")
        .GetResult();

    // Domain Services - depend only on Core
    var domainResult = Types.InCurrentDomain()
        .That().ResideInNamespace("MyApp.DomainServices")
        .ShouldNot().HaveDependencyOnAny(
            "MyApp.ApplicationServices",
            "MyApp.Infrastructure",
            "MyApp.UI")
        .GetResult();

    Assert.True(coreResult.IsSuccessful && domainResult.IsSuccessful);
}
```

### Hexagonal Architecture (Ports and Adapters)

**Core concepts**:
- **Ports**: Interfaces defining how to interact with the application
- **Adapters**: Implementations of ports for specific technologies
- **Core**: Business logic, depends only on ports

**Validation**:
```csharp
[Fact]
public void Ports_Should_Be_Interfaces_Only()
{
    var result = Types.InCurrentDomain()
        .That().ResideInNamespace("MyApp.Ports")
        .Should().BeInterfaces()
        .GetResult();
    
    Assert.True(result.IsSuccessful);
}

[Fact]
public void Core_Should_Not_Know_About_Adapters()
{
    var result = Types.InCurrentDomain()
        .That().ResideInNamespace("MyApp.Core")
        .ShouldNot().HaveDependencyOn("MyApp.Adapters")
        .GetResult();
    
    Assert.True(result.IsSuccessful);
}

[Fact]
public void Adapters_Must_Implement_Ports()
{
    var result = Types.InCurrentDomain()
        .That().ResideInNamespace("MyApp.Adapters")
        .And().AreClasses()
        .And().AreNotAbstract()
        .Should().ImplementInterface(typeof(IPort))
        .GetResult();
    
    Assert.True(result.IsSuccessful);
}
```

### Layered Architecture

Traditional three-tier or n-tier architecture.

**Validation**:
```csharp
[Fact]
public void Layers_Should_Only_Reference_Lower_Layers()
{
    // Presentation → Business → Data
    var presentationResult = Types.InCurrentDomain()
        .That().ResideInNamespace("MyApp.Presentation")
        .ShouldNot().HaveDependencyOn("MyApp.Data")
        .GetResult();

    var businessResult = Types.InCurrentDomain()
        .That().ResideInNamespace("MyApp.Business")
        .ShouldNot().HaveDependencyOn("MyApp.Presentation")
        .GetResult();

    var dataResult = Types.InCurrentDomain()
        .That().ResideInNamespace("MyApp.Data")
        .ShouldNot().HaveDependencyOnAny("MyApp.Business", "MyApp.Presentation")
        .GetResult();

    Assert.True(presentationResult.IsSuccessful && 
                businessResult.IsSuccessful && 
                dataResult.IsSuccessful);
}
```

---

## NetArchTest API Reference

### Type Selection Methods

| Method | Description | Example |
|--------|-------------|---------|
| `InCurrentDomain()` | Select types from current AppDomain | `Types.InCurrentDomain()` |
| `InAssembly(assembly)` | Select types from specific assembly | `Types.InAssembly(typeof(MyClass).Assembly)` |
| `InAssemblies(assemblies)` | Select types from multiple assemblies | `Types.InAssemblies(new[] { asm1, asm2 })` |
| `InNamespace(namespace)` | Select types from namespace | `Types.InNamespace("MyApp.Domain")` |

### Predicate Methods

#### Location Predicates
| Method | Description |
|--------|-------------|
| `ResideInNamespace(namespace)` | Types in specific namespace |
| `ResideInNamespaceStartingWith(prefix)` | Types in namespaces starting with prefix |
| `ResideInNamespaceEndingWith(suffix)` | Types in namespaces ending with suffix |
| `ResideInNamespaceMatching(pattern)` | Types matching regex pattern |

#### Name Predicates
| Method | Description |
|--------|-------------|
| `HaveName(name)` | Types with exact name |
| `HaveNameStartingWith(prefix)` | Types with name starting with prefix |
| `HaveNameEndingWith(suffix)` | Types with name ending with suffix |
| `HaveNameMatching(pattern, useRegex)` | Types matching pattern |

#### Type Predicates
| Method | Description |
|--------|-------------|
| `AreClasses()` | Only classes |
| `AreInterfaces()` | Only interfaces |
| `AreAbstract()` | Only abstract types |
| `AreSealed()` | Only sealed types |
| `ArePublic()` | Only public types |
| `AreNotPublic()` | Only non-public types |
| `AreNested()` | Only nested types |
| `AreNotNested()` | Only non-nested types |

#### Relationship Predicates
| Method | Description |
|--------|-------------|
| `Inherit(type)` | Types inheriting from type |
| `ImplementInterface(interface)` | Types implementing interface |
| `HaveDependencyOn(namespace)` | Types depending on namespace |
| `HaveDependencyOnAny(namespaces)` | Types depending on any namespace |
| `HaveDependencyOnAll(namespaces)` | Types depending on all namespaces |

#### Attribute Predicates
| Method | Description |
|--------|-------------|
| `HaveCustomAttribute(attribute)` | Types with specific attribute |
| `DoNotHaveCustomAttribute(attribute)` | Types without specific attribute |

### Condition Methods

| Method | Description |
|--------|-------------|
| `Should()` | Assert positive condition |
| `ShouldNot()` | Assert negative condition |
| `And()` | Combine with AND logic |
| `Or()` | Combine with OR logic |

### Execution Methods

| Method | Description | Returns |
|--------|-------------|---------|
| `GetResult()` | Execute rule and get result | `TestResult` |
| `GetTypes()` | Get matching types | `IEnumerable<Type>` |

### TestResult Properties

| Property | Type | Description |
|----------|------|-------------|
| `IsSuccessful` | `bool` | Whether test passed |
| `FailingTypeNames` | `IEnumerable<string>` | Names of failing types |
| `FailingTypes` | `IEnumerable<Type>` | Failing type objects |

---

## Testing Strategies

### 1. Progressive Implementation

Start with high-level rules, then add specific rules:

```csharp
// Phase 1: Basic layer dependencies
[Fact]
public void Domain_Independent() { /* ... */ }

// Phase 2: Naming conventions
[Fact]
public void Interfaces_Named_Correctly() { /* ... */ }

// Phase 3: Specific patterns
[Fact]
public void Repositories_Follow_Pattern() { /* ... */ }

// Phase 4: SOLID principles
[Fact]
public void Services_Follow_SRP() { /* ... */ }
```

### 2. Test Organization

Organize tests by concern:

```csharp
public class LayerDependencyTests { /* ... */ }
public class NamingConventionTests { /* ... */ }
public class SolidPrincipleTests { /* ... */ }
public class DesignPatternTests { /* ... */ }
```

### 3. Parameterized Tests

Use theory tests for similar rules:

```csharp
[Theory]
[InlineData("MyApp.Domain")]
[InlineData("MyApp.Application")]
public void Core_Layers_Independent(string coreNamespace)
{
    var result = Types.InCurrentDomain()
        .That().ResideInNamespace(coreNamespace)
        .ShouldNot().HaveDependencyOnAny(
            "MyApp.Infrastructure",
            "MyApp.Web")
        .GetResult();
    
    Assert.True(result.IsSuccessful);
}
```

### 4. Custom Assertions

Create helper methods for common assertions:

```csharp
public static class ArchitectureAssertions
{
    public static void AssertNoDependency(
        string sourceNamespace, 
        params string[] targetNamespaces)
    {
        var result = Types.InCurrentDomain()
            .That().ResideInNamespace(sourceNamespace)
            .ShouldNot().HaveDependencyOnAny(targetNamespaces)
            .GetResult();
        
        Assert.True(result.IsSuccessful,
            $"{sourceNamespace} should not depend on {string.Join(", ", targetNamespaces)}. " +
            $"Violations: {string.Join(", ", result.FailingTypeNames ?? Array.Empty<string>())}");
    }
}
```

---

## Best Practices

### 1. Start Simple
Begin with basic architectural rules and gradually add more specific ones.

### 2. Meaningful Failure Messages
Always include context in failure messages:

```csharp
Assert.True(result.IsSuccessful, 
    $"Domain should not depend on infrastructure. Violating types: {FormatFailures(result)}");
```

### 3. Use Theory Tests
Reduce duplication with parameterized tests:

```csharp
[Theory]
[InlineData("Repository")]
[InlineData("Service")]
[InlineData("Controller")]
public void Classes_Should_End_With_Suffix(string suffix)
{
    var result = Types.InCurrentDomain()
        .That().HaveNameEndingWith(suffix)
        .Should().ResideInNamespaceEndingWith($"{suffix}s")
        .GetResult();
    
    Assert.True(result.IsSuccessful);
}
```

### 4. Test at Multiple Levels
- Assembly level (coarse-grained)
- Namespace level (medium-grained)
- Type level (fine-grained)

### 5. Document Exceptions
If rules have exceptions, document them:

```csharp
[Fact]
public void Domain_Should_Not_Depend_On_Infrastructure()
{
    var result = Types.InCurrentDomain()
        .That().ResideInNamespace("MyApp.Domain")
        .And().DoNotHaveName("LegacyEntity") // Exception: documented technical debt
        .ShouldNot().HaveDependencyOn("MyApp.Infrastructure")
        .GetResult();
    
    Assert.True(result.IsSuccessful);
}
```

### 6. CI/CD Integration
Run architecture tests as part of your build pipeline.

### 7. Regular Review
Periodically review and update architecture tests as the codebase evolves.

---

## Common Patterns

### Repository Pattern Validation

```csharp
[Fact]
public void Repository_Pattern_Compliance()
{
    // Interfaces in domain
    var interfacesResult = Types.InCurrentDomain()
        .That().AreInterfaces()
        .And().HaveNameEndingWith("Repository")
        .Should().ResideInNamespace("MyApp.Domain.Interfaces")
        .GetResult();

    // Implementations in infrastructure
    var implResult = Types.InCurrentDomain()
        .That().ImplementInterface(typeof(IRepository<>))
        .And().AreNotInterfaces()
        .Should().ResideInNamespace("MyApp.Infrastructure.Repositories")
        .GetResult();

    Assert.True(interfacesResult.IsSuccessful && implResult.IsSuccessful);
}
```

### CQRS Pattern Validation

```csharp
[Fact]
public void CQRS_Pattern_Compliance()
{
    // Commands
    var commandsResult = Types.InCurrentDomain()
        .That().HaveNameEndingWith("Command")
        .Should().ResideInNamespace("MyApp.Application.Commands")
        .And().ImplementInterface(typeof(ICommand))
        .GetResult();

    // Queries
    var queriesResult = Types.InCurrentDomain()
        .That().HaveNameEndingWith("Query")
        .Should().ResideInNamespace("MyApp.Application.Queries")
        .And().ImplementInterface(typeof(IQuery<>))
        .GetResult();

    Assert.True(commandsResult.IsSuccessful && queriesResult.IsSuccessful);
}
```

### Factory Pattern Validation

```csharp
[Fact]
public void Factory_Pattern_Compliance()
{
    var result = Types.InCurrentDomain()
        .That().HaveNameEndingWith("Factory")
        .Should().ImplementInterface(typeof(IFactory<>))
        .Or().BeAbstract()
        .GetResult();

    Assert.True(result.IsSuccessful);
}
```

---

## Troubleshooting

### Common Issues

#### Issue 1: All Tests Failing

**Symptom**: Every architecture test fails
**Cause**: Incorrect namespace patterns or assembly loading
**Solution**:
```csharp
// Debug: Print actual namespaces
var types = Types.InCurrentDomain().GetTypes();
foreach (var type in types)
{
    Console.WriteLine($"{type.FullName} in {type.Assembly.GetName().Name}");
}
```

#### Issue 2: False Positives

**Symptom**: Tests pass but violations exist
**Cause**: Too loose rule definitions
**Solution**: Make rules more specific
```csharp
// Too loose
.That().HaveNameEndingWith("Service")

// More specific
.That().ResideInNamespace("MyApp.Application.Services")
.And().HaveNameEndingWith("Service")
.And().AreNotInterfaces()
```

#### Issue 3: Performance Issues

**Symptom**: Tests take too long
**Cause**: Loading too many assemblies
**Solution**: Target specific assemblies
```csharp
// Slow
Types.InCurrentDomain()

// Faster
Types.InAssembly(typeof(MyType).Assembly)
```

#### Issue 4: Regex Not Matching

**Symptom**: Pattern-based rules don't work
**Cause**: Incorrect regex syntax
**Solution**: Test regex separately
```csharp
var pattern = @"^I[A-Z][a-zA-Z]*Repository$";
var testName = "IUserRepository";
Assert.True(Regex.IsMatch(testName, pattern));
```

---

## Advanced Topics

### Custom Predicates

Create custom predicates for complex rules:

```csharp
public static class CustomPredicates
{
    public static ConditionList IsImmutable(this Predicates predicates)
    {
        var immutableTypes = Types.InCurrentDomain()
            .GetTypes()
            .Where(t => t.GetProperties()
                .All(p => p.SetMethod == null || !p.SetMethod.IsPublic))
            .Select(t => t.FullName)
            .ToList();

        return predicates.MeetCustomRule(
            new CustomRule(
                "IsImmutable",
                t => immutableTypes.Contains(t.FullName)));
    }
}

// Usage
[Fact]
public void Value_Objects_Should_Be_Immutable()
{
    var result = Types.InCurrentDomain()
        .That().ResideInNamespace("MyApp.Domain.ValueObjects")
        .Should().IsImmutable()
        .GetResult();

    Assert.True(result.IsSuccessful);
}
```

### Assembly Analysis

Analyze assembly dependencies programmatically:

```csharp
public static class AssemblyAnalyzer
{
    public static Dictionary<string, List<string>> GetDependencyGraph(Assembly assembly)
    {
        var graph = new Dictionary<string, List<string>>();
        var types = assembly.GetTypes();

        foreach (var type in types)
        {
            var dependencies = type.GetFields(BindingFlags.Instance | BindingFlags.NonPublic)
                .Select(f => f.FieldType.Namespace)
                .Concat(type.GetProperties().Select(p => p.PropertyType.Namespace))
                .Where(ns => ns != null && !ns.StartsWith("System"))
                .Distinct()
                .ToList();

            graph[type.FullName] = dependencies;
        }

        return graph;
    }
}
```

### Metrics Collection

Collect architectural metrics:

```csharp
public class ArchitectureMetrics
{
    public int TotalTypes { get; set; }
    public int PublicTypes { get; set; }
    public int Interfaces { get; set; }
    public int AbstractClasses { get; set; }
    public double AbstractionLevel => (double)(Interfaces + AbstractClasses) / TotalTypes;
    public double Instability { get; set; } // Ce / (Ce + Ca)

    public static ArchitectureMetrics Calculate(Assembly assembly)
    {
        var types = assembly.GetTypes();
        return new ArchitectureMetrics
        {
            TotalTypes = types.Length,
            PublicTypes = types.Count(t => t.IsPublic),
            Interfaces = types.Count(t => t.IsInterface),
            AbstractClasses = types.Count(t => t.IsAbstract && !t.IsInterface)
        };
    }
}
```

---

## Additional Resources

### Official Documentation
- [NetArchTest GitHub](https://github.com/BenMorris/NetArchTest)
- [NetArchTest NuGet](https://www.nuget.org/packages/NetArchTest.Rules/)

### Books
- "Clean Architecture" by Robert C. Martin
- "Patterns of Enterprise Application Architecture" by Martin Fowler
- "Domain-Driven Design" by Eric Evans

### Online Resources
- [SOLID Principles](https://en.wikipedia.org/wiki/SOLID)
- [Clean Code Blog](https://blog.cleancoder.com/)
- [Martin Fowler's Blog](https://martinfowler.com/)

### Similar Tools
- **ArchUnit** (Java): https://www.archunit.org/
- **Structure101** (Multi-language): https://structure101.com/
- **NDepend** (.NET): https://www.ndepend.com/

---

## Conclusion

Architecture testing with NetArchTest provides a powerful way to enforce SOLID principles and architectural patterns in C# codebases. By following the practices and patterns outlined in this reference, you can:

- Maintain clean architecture over time
- Catch violations early in development
- Document architectural decisions as executable tests
- Ensure team alignment on architectural standards
- Prevent technical debt accumulation

Remember that architecture tests are most effective when:
1. Integrated into CI/CD pipelines
2. Reviewed and updated regularly
3. Combined with code reviews and pair programming
4. Treated as living documentation
5. Used to educate new team members

Start simple, iterate often, and continuously improve your architectural validation strategy.
