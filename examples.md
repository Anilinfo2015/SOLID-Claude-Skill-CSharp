# SOLID and Architecture Validation Examples

This document provides comprehensive examples of using NetArchTest to validate SOLID principles and architectural patterns in C# codebases.

## Table of Contents

1. [Complete Clean Architecture Example](#complete-clean-architecture-example)
2. [SOLID Principles Examples](#solid-principles-examples)
3. [Layered Architecture Examples](#layered-architecture-examples)
4. [Hexagonal Architecture Examples](#hexagonal-architecture-examples)
5. [Microservices Architecture Examples](#microservices-architecture-examples)
6. [Common Pattern Validations](#common-pattern-validations)
7. [Advanced Custom Rules](#advanced-custom-rules)

---

## Complete Clean Architecture Example

### Project Structure
```
MyApp/
├── MyApp.Domain/
│   ├── Entities/
│   ├── ValueObjects/
│   ├── Interfaces/
│   └── Exceptions/
├── MyApp.Application/
│   ├── Services/
│   ├── DTOs/
│   ├── Interfaces/
│   └── UseCases/
├── MyApp.Infrastructure/
│   ├── Persistence/
│   ├── Repositories/
│   ├── Services/
│   └── Configuration/
├── MyApp.Web/
│   ├── Controllers/
│   ├── ViewModels/
│   └── Middleware/
└── MyApp.Tests.Architecture/
    └── ArchitectureTests.cs
```

### Complete Test Suite

```csharp
using NetArchTest.Rules;
using System;
using System.Linq;
using System.Reflection;
using Xunit;

namespace MyApp.Tests.Architecture
{
    public class CleanArchitectureTests
    {
        private static readonly Assembly DomainAssembly = typeof(MyApp.Domain.Entity).Assembly;
        private static readonly Assembly ApplicationAssembly = typeof(MyApp.Application.IService).Assembly;
        private static readonly Assembly InfrastructureAssembly = typeof(MyApp.Infrastructure.Repository).Assembly;
        private static readonly Assembly WebAssembly = typeof(MyApp.Web.Program).Assembly;

        #region Layer Dependency Tests

        [Fact]
        public void Domain_Should_Not_Depend_On_Any_Other_Layer()
        {
            var result = Types.InAssembly(DomainAssembly)
                .ShouldNot().HaveDependencyOn("MyApp.Application")
                .And().ShouldNot().HaveDependencyOn("MyApp.Infrastructure")
                .And().ShouldNot().HaveDependencyOn("MyApp.Web")
                .GetResult();

            Assert.True(result.IsSuccessful, 
                $"Domain layer should not depend on other layers. Violations: {FormatFailures(result)}");
        }

        [Fact]
        public void Application_Should_Only_Depend_On_Domain()
        {
            var result = Types.InAssembly(ApplicationAssembly)
                .ShouldNot().HaveDependencyOn("MyApp.Infrastructure")
                .And().ShouldNot().HaveDependencyOn("MyApp.Web")
                .GetResult();

            Assert.True(result.IsSuccessful, 
                $"Application should only depend on Domain. Violations: {FormatFailures(result)}");
        }

        [Fact]
        public void Infrastructure_Should_Only_Depend_On_Application_And_Domain()
        {
            var result = Types.InAssembly(InfrastructureAssembly)
                .ShouldNot().HaveDependencyOn("MyApp.Web")
                .GetResult();

            Assert.True(result.IsSuccessful, 
                $"Infrastructure should not depend on Web layer. Violations: {FormatFailures(result)}");
        }

        [Fact]
        public void Web_Should_Only_Depend_On_Application()
        {
            var result = Types.InAssembly(WebAssembly)
                .ShouldNot().HaveDependencyOn("MyApp.Infrastructure")
                .And().ShouldNot().HaveDependencyOn("MyApp.Domain")
                .GetResult();

            Assert.True(result.IsSuccessful, 
                $"Web layer should only depend on Application. Violations: {FormatFailures(result)}");
        }

        #endregion

        #region SOLID Principles Tests

        [Fact]
        public void Domain_Entities_Should_Not_Have_Too_Many_Dependencies()
        {
            var entities = Types.InAssembly(DomainAssembly)
                .That().ResideInNamespace("MyApp.Domain.Entities")
                .GetTypes();

            var violations = entities
                .Where(e => e.GetConstructors()
                    .Any(c => c.GetParameters().Length > 5))
                .Select(e => e.Name)
                .ToList();

            Assert.Empty(violations);
        }

        [Fact]
        public void All_Interfaces_Should_Follow_ISP()
        {
            var interfaces = Types.InAssembly(DomainAssembly)
                .That().AreInterfaces()
                .GetTypes();

            var bloatedInterfaces = interfaces
                .Where(i => i.GetMethods().Length > 7)
                .Select(i => $"{i.Name} has {i.GetMethods().Length} methods")
                .ToList();

            Assert.Empty(bloatedInterfaces);
        }

        [Fact]
        public void Services_Should_Depend_On_Abstractions_Not_Implementations()
        {
            var result = Types.InAssembly(ApplicationAssembly)
                .That().HaveNameEndingWith("Service")
                .ShouldNot().HaveDependencyOn("MyApp.Infrastructure.Persistence")
                .GetResult();

            Assert.True(result.IsSuccessful, 
                $"Services should depend on abstractions. Violations: {FormatFailures(result)}");
        }

        #endregion

        #region Naming Convention Tests

        [Fact]
        public void All_Interfaces_Should_Start_With_I()
        {
            var result = Types.InAssemblies(new[] { DomainAssembly, ApplicationAssembly })
                .That().AreInterfaces()
                .Should().HaveNameStartingWith("I")
                .GetResult();

            Assert.True(result.IsSuccessful, 
                $"Interfaces must start with 'I'. Violations: {FormatFailures(result)}");
        }

        [Fact]
        public void All_Repositories_Should_End_With_Repository()
        {
            var result = Types.InAssembly(InfrastructureAssembly)
                .That().ResideInNamespace("MyApp.Infrastructure.Repositories")
                .And().AreNotInterfaces()
                .Should().HaveNameEndingWith("Repository")
                .GetResult();

            Assert.True(result.IsSuccessful, 
                $"Repository classes must end with 'Repository'. Violations: {FormatFailures(result)}");
        }

        [Fact]
        public void All_Controllers_Should_End_With_Controller()
        {
            var result = Types.InAssembly(WebAssembly)
                .That().ResideInNamespace("MyApp.Web.Controllers")
                .Should().HaveNameEndingWith("Controller")
                .GetResult();

            Assert.True(result.IsSuccessful, 
                $"Controller classes must end with 'Controller'. Violations: {FormatFailures(result)}");
        }

        #endregion

        #region Class Design Tests

        [Fact]
        public void Domain_Entities_Should_Not_Have_Public_Setters()
        {
            var entities = Types.InAssembly(DomainAssembly)
                .That().ResideInNamespace("MyApp.Domain.Entities")
                .GetTypes();

            var violations = entities
                .SelectMany(e => e.GetProperties())
                .Where(p => p.SetMethod?.IsPublic == true && !p.Name.Equals("Id"))
                .Select(p => $"{p.DeclaringType?.Name}.{p.Name}")
                .ToList();

            Assert.Empty(violations);
        }

        [Fact]
        public void Value_Objects_Should_Be_Immutable()
        {
            var valueObjects = Types.InAssembly(DomainAssembly)
                .That().ResideInNamespace("MyApp.Domain.ValueObjects")
                .GetTypes();

            var violations = valueObjects
                .SelectMany(vo => vo.GetProperties())
                .Where(p => p.SetMethod != null && p.SetMethod.IsPublic)
                .Select(p => $"{p.DeclaringType?.Name}.{p.Name}")
                .ToList();

            Assert.Empty(violations);
        }

        [Fact]
        public void DTOs_Should_Only_Exist_In_Application_Layer()
        {
            var result = Types.InAssemblies(new[] { DomainAssembly, InfrastructureAssembly, WebAssembly })
                .That().HaveNameEndingWith("Dto")
                .Or().HaveNameEndingWith("DTO")
                .Should().BeEmpty()
                .GetResult();

            Assert.True(result.IsSuccessful, 
                $"DTOs should only exist in Application layer. Violations: {FormatFailures(result)}");
        }

        #endregion

        private static string FormatFailures(TestResult result)
        {
            return result.FailingTypeNames != null && result.FailingTypeNames.Any()
                ? string.Join(", ", result.FailingTypeNames)
                : "None";
        }
    }
}
```

---

## SOLID Principles Examples

### Single Responsibility Principle (SRP)

#### Example 1: Validate Service Classes Have Single Purpose

```csharp
[Fact]
public void Each_Service_Should_Focus_On_Single_Concern()
{
    var services = Types.InCurrentDomain()
        .That().HaveNameEndingWith("Service")
        .And().ResideInNamespace("MyApp.Application.Services")
        .GetTypes();

    // Check constructor dependencies (indicator of too many responsibilities)
    var violations = services
        .Where(s => s.GetConstructors()
            .Any(c => c.GetParameters().Length > 8))
        .Select(s => s.Name)
        .ToList();

    Assert.Empty(violations);
}

[Fact]
public void Handlers_Should_Handle_Single_Command_Or_Query()
{
    // Each CQRS handler should handle exactly one command or query
    var handlers = Types.InCurrentDomain()
        .That().ResideInNamespace("MyApp.Application.Handlers")
        .GetTypes();

    var violations = handlers
        .Where(h => h.GetInterfaces().Count(i => 
            i.Name.Contains("IRequestHandler") || 
            i.Name.Contains("ICommandHandler") ||
            i.Name.Contains("IQueryHandler")) > 1)
        .Select(h => h.Name)
        .ToList();

    Assert.Empty(violations);
}
```

#### Example 2: Prevent God Objects

```csharp
[Fact]
public void Classes_Should_Not_Be_God_Objects()
{
    var classes = Types.InCurrentDomain()
        .That().AreClasses()
        .And().AreNotAbstract()
        .GetTypes();

    // God objects typically have many methods and dependencies
    var godObjects = classes
        .Where(c => c.GetMethods(BindingFlags.Public | BindingFlags.Instance)
            .Count(m => !m.IsSpecialName) > 20)
        .Select(c => $"{c.Name} has {c.GetMethods(BindingFlags.Public | BindingFlags.Instance).Count(m => !m.IsSpecialName)} public methods")
        .ToList();

    Assert.Empty(godObjects);
}
```

### Open/Closed Principle (OCP)

#### Example 1: Ensure Base Classes Are Abstract

```csharp
[Fact]
public void Base_Classes_Should_Be_Abstract()
{
    var result = Types.InCurrentDomain()
        .That().HaveNameEndingWith("Base")
        .Should().BeAbstract()
        .GetResult();

    Assert.True(result.IsSuccessful, 
        $"Base classes should be abstract: {FormatFailures(result)}");
}

[Fact]
public void Strategy_Pattern_Base_Should_Be_Abstract()
{
    var result = Types.InCurrentDomain()
        .That().ResideInNamespace("MyApp.Domain.Strategies")
        .And().DoNotHaveNameEndingWith("Context")
        .Should().BeAbstract()
        .Or().AreInterfaces()
        .GetResult();

    Assert.True(result.IsSuccessful);
}
```

#### Example 2: Validate Plugin Architecture

```csharp
[Fact]
public void Plugins_Should_Implement_Plugin_Interface()
{
    var result = Types.InCurrentDomain()
        .That().ResideInNamespace("MyApp.Plugins")
        .And().AreNotInterfaces()
        .And().AreNotAbstract()
        .Should().ImplementInterface(typeof(IPlugin))
        .GetResult();

    Assert.True(result.IsSuccessful, 
        $"All plugins must implement IPlugin: {FormatFailures(result)}");
}

[Fact]
public void Core_Should_Not_Know_About_Specific_Plugins()
{
    var result = Types.InCurrentDomain()
        .That().ResideInNamespace("MyApp.Core")
        .ShouldNot().HaveDependencyOn("MyApp.Plugins")
        .GetResult();

    Assert.True(result.IsSuccessful);
}
```

### Liskov Substitution Principle (LSP)

#### Example 1: Validate Proper Inheritance

```csharp
[Fact]
public void All_Entity_Subclasses_Should_Be_Valid_Entities()
{
    var result = Types.InCurrentDomain()
        .That().Inherit(typeof(Entity))
        .Should().ResideInNamespace("MyApp.Domain.Entities")
        .GetResult();

    Assert.True(result.IsSuccessful, 
        $"All entity subclasses should be in Entities namespace: {FormatFailures(result)}");
}

[Fact]
public void Repository_Implementations_Should_Follow_Interface_Contract()
{
    var result = Types.InCurrentDomain()
        .That().ImplementInterface(typeof(IRepository<>))
        .And().AreNotInterfaces()
        .Should().HaveNameEndingWith("Repository")
        .And().ResideInNamespace("MyApp.Infrastructure.Repositories")
        .GetResult();

    Assert.True(result.IsSuccessful);
}
```

#### Example 2: Prevent Contract Violations

```csharp
[Fact]
public void Exception_Classes_Should_Be_Serializable()
{
    var exceptions = Types.InCurrentDomain()
        .That().Inherit(typeof(Exception))
        .GetTypes();

    var nonSerializable = exceptions
        .Where(e => !e.IsSerializable)
        .Select(e => e.Name)
        .ToList();

    Assert.Empty(nonSerializable);
}
```

### Interface Segregation Principle (ISP)

#### Example 1: Prevent Bloated Interfaces

```csharp
[Fact]
public void Interfaces_Should_Not_Have_Too_Many_Members()
{
    var interfaces = Types.InCurrentDomain()
        .That().AreInterfaces()
        .And().DoNotHaveName("IDisposable")
        .GetTypes();

    var bloated = interfaces
        .Where(i => i.GetMembers().Length > 10)
        .Select(i => $"{i.Name} has {i.GetMembers().Length} members")
        .ToList();

    Assert.Empty(bloated);
}

[Fact]
public void Role_Interfaces_Should_Be_Cohesive()
{
    // Example: IReadRepository and IWriteRepository instead of IRepository
    var result = Types.InCurrentDomain()
        .That().AreInterfaces()
        .And().ResideInNamespace("MyApp.Domain.Interfaces")
        .Should().HaveNameMatching(@"^I[A-Z][a-zA-Z]*(Reader|Writer|Handler|Provider|Factory)$", useRegularExpressions: true)
        .Or().BeEmpty()
        .GetResult();

    // This ensures interfaces have role-based names indicating focused responsibility
    Assert.True(result.IsSuccessful || 
        result.FailingTypeNames?.All(n => n.StartsWith("I") && !n.Contains("Manager")) == true);
}
```

#### Example 2: Validate Client-Specific Interfaces

```csharp
[Fact]
public void Client_Should_Not_Depend_On_Unused_Interface_Members()
{
    // Ensure separate read/write interfaces
    var readInterface = typeof(IReadRepository<>);
    var writeInterface = typeof(IWriteRepository<>);

    var readOnlyClients = Types.InCurrentDomain()
        .That().HaveDependencyOn(readInterface.Namespace)
        .And().DoNotHaveDependencyOn(writeInterface.Namespace)
        .GetTypes();

    // Verify read-only clients exist (good ISP practice)
    Assert.NotEmpty(readOnlyClients);
}
```

### Dependency Inversion Principle (DIP)

#### Example 1: High-Level Modules Don't Depend on Low-Level

```csharp
[Fact]
public void Business_Logic_Should_Not_Depend_On_Data_Access()
{
    var result = Types.InCurrentDomain()
        .That().ResideInNamespace("MyApp.Domain")
        .Or().ResideInNamespace("MyApp.Application")
        .ShouldNot().HaveDependencyOn("System.Data.SqlClient")
        .And().ShouldNot().HaveDependencyOn("Microsoft.EntityFrameworkCore")
        .And().ShouldNot().HaveDependencyOn("Dapper")
        .GetResult();

    Assert.True(result.IsSuccessful, 
        $"Business logic should not depend on data access implementations: {FormatFailures(result)}");
}

[Fact]
public void Application_Should_Depend_On_Abstractions()
{
    var result = Types.InCurrentDomain()
        .That().ResideInNamespace("MyApp.Application")
        .ShouldNot().HaveDependencyOn("MyApp.Infrastructure.Persistence")
        .And().ShouldNot().HaveDependencyOn("MyApp.Infrastructure.ExternalServices")
        .GetResult();

    Assert.True(result.IsSuccessful);
}
```

#### Example 2: Validate Dependency Direction

```csharp
[Fact]
public void Dependencies_Should_Point_Inward_In_Onion_Architecture()
{
    // Core (Domain) - no dependencies
    var coreResult = Types.InCurrentDomain()
        .That().ResideInNamespace("MyApp.Domain")
        .ShouldNot().HaveDependencyOnAny(
            "MyApp.Application",
            "MyApp.Infrastructure",
            "MyApp.Web")
        .GetResult();

    // Application - depends only on Domain
    var appResult = Types.InCurrentDomain()
        .That().ResideInNamespace("MyApp.Application")
        .ShouldNot().HaveDependencyOnAny(
            "MyApp.Infrastructure",
            "MyApp.Web")
        .GetResult();

    // Infrastructure - can depend on Domain and Application
    var infraResult = Types.InCurrentDomain()
        .That().ResideInNamespace("MyApp.Infrastructure")
        .ShouldNot().HaveDependencyOn("MyApp.Web")
        .GetResult();

    Assert.True(coreResult.IsSuccessful && appResult.IsSuccessful && infraResult.IsSuccessful,
        $"Dependency direction violated. Core: {FormatFailures(coreResult)}, " +
        $"App: {FormatFailures(appResult)}, Infra: {FormatFailures(infraResult)}");
}
```

---

## Layered Architecture Examples

### Three-Tier Architecture

```csharp
public class LayeredArchitectureTests
{
    [Fact]
    public void Presentation_Should_Not_Reference_Data_Layer()
    {
        var result = Types.InCurrentDomain()
            .That().ResideInNamespace("MyApp.Presentation")
            .ShouldNot().HaveDependencyOn("MyApp.Data")
            .GetResult();

        Assert.True(result.IsSuccessful);
    }

    [Fact]
    public void Business_Layer_Should_Be_Independent_Of_Presentation()
    {
        var result = Types.InCurrentDomain()
            .That().ResideInNamespace("MyApp.Business")
            .ShouldNot().HaveDependencyOn("MyApp.Presentation")
            .GetResult();

        Assert.True(result.IsSuccessful);
    }

    [Fact]
    public void Data_Layer_Should_Be_Lowest_Layer()
    {
        var result = Types.InCurrentDomain()
            .That().ResideInNamespace("MyApp.Data")
            .ShouldNot().HaveDependencyOn("MyApp.Business")
            .And().ShouldNot().HaveDependencyOn("MyApp.Presentation")
            .GetResult();

        Assert.True(result.IsSuccessful);
    }
}
```

---

## Hexagonal Architecture Examples

### Ports and Adapters Pattern

```csharp
public class HexagonalArchitectureTests
{
    [Fact]
    public void Core_Should_Only_Define_Ports_Not_Adapters()
    {
        var result = Types.InCurrentDomain()
            .That().ResideInNamespace("MyApp.Core.Ports")
            .Should().BeInterfaces()
            .GetResult();

        Assert.True(result.IsSuccessful, 
            "Ports should be interfaces only");
    }

    [Fact]
    public void Adapters_Should_Implement_Ports()
    {
        var result = Types.InCurrentDomain()
            .That().ResideInNamespace("MyApp.Adapters")
            .And().AreNotInterfaces()
            .Should().ImplementInterface(typeof(IPort))
            .Or().HaveNameEndingWith("Adapter")
            .GetResult();

        Assert.True(result.IsSuccessful);
    }

    [Fact]
    public void Core_Should_Not_Depend_On_Adapters()
    {
        var result = Types.InCurrentDomain()
            .That().ResideInNamespace("MyApp.Core")
            .ShouldNot().HaveDependencyOn("MyApp.Adapters")
            .GetResult();

        Assert.True(result.IsSuccessful);
    }

    [Fact]
    public void Primary_And_Secondary_Adapters_Should_Be_Separated()
    {
        var primaryResult = Types.InCurrentDomain()
            .That().ResideInNamespace("MyApp.Adapters.Primary")
            .ShouldNot().HaveDependencyOn("MyApp.Adapters.Secondary")
            .GetResult();

        var secondaryResult = Types.InCurrentDomain()
            .That().ResideInNamespace("MyApp.Adapters.Secondary")
            .ShouldNot().HaveDependencyOn("MyApp.Adapters.Primary")
            .GetResult();

        Assert.True(primaryResult.IsSuccessful && secondaryResult.IsSuccessful);
    }
}
```

---

## Microservices Architecture Examples

### Service Boundaries

```csharp
public class MicroservicesArchitectureTests
{
    [Fact]
    public void Services_Should_Not_Share_Database_Models()
    {
        var orderServiceResult = Types.InCurrentDomain()
            .That().ResideInNamespace("OrderService")
            .ShouldNot().HaveDependencyOn("UserService.Data")
            .GetResult();

        var userServiceResult = Types.InCurrentDomain()
            .That().ResideInNamespace("UserService")
            .ShouldNot().HaveDependencyOn("OrderService.Data")
            .GetResult();

        Assert.True(orderServiceResult.IsSuccessful && userServiceResult.IsSuccessful);
    }

    [Fact]
    public void Each_Service_Should_Have_Its_Own_Domain_Models()
    {
        var services = new[] { "OrderService", "UserService", "PaymentService" };

        foreach (var service in services)
        {
            var result = Types.InCurrentDomain()
                .That().ResideInNamespace($"{service}.Domain")
                .Should().NotBeEmpty()
                .GetResult();

            Assert.True(result.IsSuccessful, 
                $"{service} should have its own domain models");
        }
    }

    [Fact]
    public void Services_Should_Communicate_Through_Contracts()
    {
        var result = Types.InCurrentDomain()
            .That().ResideInNamespace("OrderService")
            .And().HaveDependencyOn("UserService")
            .Should().HaveDependencyOn("UserService.Contracts")
            .GetResult();

        Assert.True(result.IsSuccessful || 
            !Types.InCurrentDomain()
                .That().ResideInNamespace("OrderService")
                .And().HaveDependencyOn("UserService")
                .GetTypes().Any());
    }
}
```

---

## Common Pattern Validations

### Repository Pattern

```csharp
[Fact]
public void All_Repositories_Should_Implement_Base_Interface()
{
    var result = Types.InCurrentDomain()
        .That().HaveNameEndingWith("Repository")
        .And().AreNotInterfaces()
        .And().AreNotAbstract()
        .Should().ImplementInterface(typeof(IRepository<>))
        .GetResult();

    Assert.True(result.IsSuccessful, 
        $"Repositories must implement IRepository<T>: {FormatFailures(result)}");
}

[Fact]
public void Repository_Interfaces_Should_Be_In_Domain()
{
    var result = Types.InCurrentDomain()
        .That().AreInterfaces()
        .And().HaveNameEndingWith("Repository")
        .Should().ResideInNamespace("MyApp.Domain.Interfaces")
        .GetResult();

    Assert.True(result.IsSuccessful);
}

[Fact]
public void Repository_Implementations_Should_Be_In_Infrastructure()
{
    var result = Types.InCurrentDomain()
        .That().ImplementInterface(typeof(IRepository<>))
        .And().AreNotInterfaces()
        .Should().ResideInNamespace("MyApp.Infrastructure.Repositories")
        .GetResult();

    Assert.True(result.IsSuccessful);
}
```

### CQRS Pattern

```csharp
[Fact]
public void Commands_Should_Be_In_Commands_Namespace()
{
    var result = Types.InCurrentDomain()
        .That().ImplementInterface(typeof(ICommand))
        .Or().HaveNameEndingWith("Command")
        .Should().ResideInNamespace("MyApp.Application.Commands")
        .GetResult();

    Assert.True(result.IsSuccessful);
}

[Fact]
public void Queries_Should_Be_In_Queries_Namespace()
{
    var result = Types.InCurrentDomain()
        .That().ImplementInterface(typeof(IQuery<>))
        .Or().HaveNameEndingWith("Query")
        .Should().ResideInNamespace("MyApp.Application.Queries")
        .GetResult();

    Assert.True(result.IsSuccessful);
}

[Fact]
public void Command_Handlers_Should_Not_Return_Data()
{
    var handlers = Types.InCurrentDomain()
        .That().ImplementInterface(typeof(ICommandHandler<>))
        .GetTypes();

    var violations = handlers
        .Where(h => h.GetInterfaces()
            .Any(i => i.IsGenericType && 
                     i.GetGenericTypeDefinition() == typeof(ICommandHandler<>) &&
                     i.GetGenericArguments()[0] != typeof(void)))
        .Select(h => h.Name)
        .ToList();

    Assert.Empty(violations);
}
```

### Factory Pattern

```csharp
[Fact]
public void Factories_Should_End_With_Factory()
{
    var result = Types.InCurrentDomain()
        .That().ImplementInterface(typeof(IFactory<>))
        .Should().HaveNameEndingWith("Factory")
        .GetResult();

    Assert.True(result.IsSuccessful);
}

[Fact]
public void Factories_Should_Be_In_Factories_Namespace()
{
    var result = Types.InCurrentDomain()
        .That().HaveNameEndingWith("Factory")
        .And().AreNotInterfaces()
        .Should().ResideInNamespace("MyApp.Domain.Factories")
        .Or().ResideInNamespace("MyApp.Application.Factories")
        .GetResult();

    Assert.True(result.IsSuccessful);
}
```

---

## Advanced Custom Rules

### Custom Predicate Examples

```csharp
public class AdvancedCustomRulesTests
{
    [Fact]
    public void Entities_Should_Have_Private_Parameterless_Constructor_For_ORM()
    {
        var entities = Types.InCurrentDomain()
            .That().Inherit(typeof(Entity))
            .GetTypes();

        var violations = entities
            .Where(e => !e.GetConstructors(BindingFlags.NonPublic | BindingFlags.Instance)
                .Any(c => c.GetParameters().Length == 0))
            .Select(e => e.Name)
            .ToList();

        Assert.Empty(violations);
    }

    [Fact]
    public void Async_Methods_Should_End_With_Async()
    {
        var types = Types.InCurrentDomain()
            .That().ResideInNamespaceStartingWith("MyApp")
            .GetTypes();

        var violations = types
            .SelectMany(t => t.GetMethods(BindingFlags.Public | BindingFlags.Instance))
            .Where(m => m.ReturnType.Name.Contains("Task") && 
                       !m.Name.EndsWith("Async") &&
                       !m.IsSpecialName)
            .Select(m => $"{m.DeclaringType?.Name}.{m.Name}")
            .ToList();

        Assert.Empty(violations);
    }

    [Fact]
    public void Events_Should_Be_Immutable()
    {
        var events = Types.InCurrentDomain()
            .That().ImplementInterface(typeof(IDomainEvent))
            .Or().ResideInNamespace("MyApp.Domain.Events")
            .GetTypes();

        var violations = events
            .SelectMany(e => e.GetProperties())
            .Where(p => p.SetMethod != null && p.SetMethod.IsPublic)
            .Select(p => $"{p.DeclaringType?.Name}.{p.Name}")
            .ToList();

        Assert.Empty(violations);
    }

    [Fact]
    public void Validators_Should_Implement_IValidator()
    {
        var result = Types.InCurrentDomain()
            .That().HaveNameEndingWith("Validator")
            .And().AreNotInterfaces()
            .Should().ImplementInterface(typeof(IValidator<>))
            .GetResult();

        Assert.True(result.IsSuccessful, 
            $"Validators must implement IValidator<T>: {FormatFailures(result)}");
    }

    [Fact]
    public void Configuration_Classes_Should_Be_Internal()
    {
        var result = Types.InCurrentDomain()
            .That().ResideInNamespace("MyApp.Infrastructure.Configuration")
            .And().HaveNameEndingWith("Configuration")
            .Should().NotBePublic()
            .GetResult();

        Assert.True(result.IsSuccessful);
    }

    [Fact]
    public void Extension_Methods_Should_Be_In_Extensions_Namespace()
    {
        var types = Types.InCurrentDomain()
            .That().AreClasses()
            .And().AreSealed()
            .And().AreNotNested()
            .GetTypes();

        var extensionClasses = types
            .Where(t => t.GetMethods(BindingFlags.Static | BindingFlags.Public)
                .Any(m => m.IsDefined(typeof(System.Runtime.CompilerServices.ExtensionAttribute), false)))
            .ToList();

        var violations = extensionClasses
            .Where(t => !t.Namespace?.Contains(".Extensions") == true)
            .Select(t => t.Name)
            .ToList();

        Assert.Empty(violations);
    }

    private static string FormatFailures(TestResult result)
    {
        return result.FailingTypeNames != null && result.FailingTypeNames.Any()
            ? string.Join(", ", result.FailingTypeNames)
            : "None";
    }
}
```

### Complex Architectural Rules

```csharp
[Fact]
public void Validate_Complete_Clean_Architecture_Rules()
{
    var assemblies = new[]
    {
        typeof(MyApp.Domain.Entity).Assembly,
        typeof(MyApp.Application.IService).Assembly,
        typeof(MyApp.Infrastructure.Repository).Assembly,
        typeof(MyApp.Web.Program).Assembly
    };

    // Domain rules
    var domainRules = Types.InAssembly(assemblies[0])
        .ShouldNot().HaveDependencyOnAny("MyApp.Application", "MyApp.Infrastructure", "MyApp.Web")
        .And().ShouldNot().HaveDependencyOn("Microsoft.EntityFrameworkCore")
        .And().ShouldNot().HaveDependencyOn("System.Data")
        .GetResult();

    // Application rules
    var appRules = Types.InAssembly(assemblies[1])
        .ShouldNot().HaveDependencyOnAny("MyApp.Infrastructure", "MyApp.Web")
        .And().ShouldNot().HaveDependencyOn("Microsoft.AspNetCore")
        .GetResult();

    // Infrastructure rules
    var infraRules = Types.InAssembly(assemblies[2])
        .ShouldNot().HaveDependencyOn("MyApp.Web")
        .And().ShouldNot().HaveDependencyOn("Microsoft.AspNetCore.Mvc")
        .GetResult();

    Assert.True(domainRules.IsSuccessful, $"Domain violations: {FormatFailures(domainRules)}");
    Assert.True(appRules.IsSuccessful, $"Application violations: {FormatFailures(appRules)}");
    Assert.True(infraRules.IsSuccessful, $"Infrastructure violations: {FormatFailures(infraRules)}");
}
```

---

## Helper Methods

```csharp
public static class ArchitectureTestHelpers
{
    public static string FormatFailures(TestResult result)
    {
        if (result.FailingTypeNames == null || !result.FailingTypeNames.Any())
            return "None";

        return string.Join(Environment.NewLine, 
            result.FailingTypeNames.Select((name, index) => $"{index + 1}. {name}"));
    }

    public static bool HasMaxDependencies(Type type, int maxDependencies)
    {
        var constructors = type.GetConstructors();
        if (!constructors.Any())
            return true;

        return constructors.All(c => c.GetParameters().Length <= maxDependencies);
    }

    public static bool IsImmutable(Type type)
    {
        var properties = type.GetProperties();
        return properties.All(p => p.SetMethod == null || !p.SetMethod.IsPublic);
    }

    public static IEnumerable<string> GetPublicApiSurface(Assembly assembly)
    {
        return Types.InAssembly(assembly)
            .That().ArePublic()
            .GetTypes()
            .Select(t => t.FullName ?? t.Name);
    }
}
```

---

## Integration with CI/CD

### GitHub Actions Workflow

```yaml
name: Architecture Tests

on:
  push:
    branches: [ main, develop ]
  pull_request:
    branches: [ main, develop ]

jobs:
  architecture-validation:
    runs-on: ubuntu-latest
    
    steps:
    - uses: actions/checkout@v3
    
    - name: Setup .NET
      uses: actions/setup-dotnet@v3
      with:
        dotnet-version: '8.0.x'
    
    - name: Restore dependencies
      run: dotnet restore
    
    - name: Build
      run: dotnet build --no-restore --configuration Release
    
    - name: Run Architecture Tests
      run: dotnet test MyApp.Tests.Architecture --no-build --configuration Release --logger "trx;LogFileName=architecture-test-results.trx"
    
    - name: Publish Test Results
      uses: EnricoMi/publish-unit-test-result-action@v2
      if: always()
      with:
        files: '**/architecture-test-results.trx'
```

---

## Additional Resources

- [NetArchTest GitHub Repository](https://github.com/BenMorris/NetArchTest)
- [Clean Architecture by Robert C. Martin](https://blog.cleancoder.com/uncle-bob/2012/08/13/the-clean-architecture.html)
- [SOLID Principles](https://en.wikipedia.org/wiki/SOLID)
- [Architecture Testing Best Practices](https://www.archunit.org/)
