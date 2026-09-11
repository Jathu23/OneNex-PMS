# OneNex — Code Quality Enforcement

> Status: Living Document
> Last updated: 2026-09-07
> Covers: EditorConfig, Directory.Build.props, Roslyn Analyzers, NetArchTest, pre-commit hooks

---

## What Gets Enforced Where

```
As you type (IDE red underline):
  → EditorConfig naming rules
  → Nullable reference violations
  → Fire-and-forget Task warnings

On build (Ctrl+Shift+B):
  → All above +
  → Code style violations (EnforceCodeStyleInBuild)
  → Unused variables
  → All Roslyn analyzer warnings → treated as errors

On commit (pre-commit hook):
  → All above +
  → Module boundary violations (NetArchTest)
  → Layer dependency violations (Domain → Infra etc.)
  → Naming convention rules (Handler sealed, Async suffix)

On CI (GitHub Actions):
  → Full suite — build + unit + integration + architecture tests
```

---

## 1. Directory.Build.props — Project-wide Settings

Solution root-ல் ஒரு file — all projects automatically inherit. Per-project settings வேண்டாம்.

```xml
<!-- Directory.Build.props (solution root) -->
<Project>
  <PropertyGroup>
    <!-- Warnings → Errors — build fails if any warning -->
    <TreatWarningsAsErrors>true</TreatWarningsAsErrors>

    <!-- EditorConfig style rules enforced at build time -->
    <EnforceCodeStyleInBuild>true</EnforceCodeStyleInBuild>

    <!-- Enable all .NET built-in analyzers -->
    <EnableNETAnalyzers>true</EnableNETAnalyzers>
    <AnalysisLevel>latest-All</AnalysisLevel>

    <!-- Nullable reference types — enforced everywhere -->
    <Nullable>enable</Nullable>
    <ImplicitUsings>enable</ImplicitUsings>

    <!-- Target framework — all projects same -->
    <TargetFramework>net10.0</TargetFramework>
    <LangVersion>latest</LangVersion>
  </PropertyGroup>

  <!-- No extra analyzer packages needed.
       EnableNETAnalyzers + AnalysisLevel activates Microsoft's built-in CA rules.
       EditorConfig controls severity of individual diagnostics. -->
</Project>
```

---

## 2. .editorconfig — Naming + Style Rules

Solution root-ல். IDE real-time red underline காட்டும் + build fails.

```ini
# .editorconfig (solution root)
root = true

[*]
end_of_line   = lf
charset       = utf-8
trim_trailing_whitespace = true
insert_final_newline     = true

[*.cs]
indent_style = space
indent_size  = 4

# ─── Naming Conventions ────────────────────────────────────────────────

# Interfaces must start with I — IBookingRepository not BookingRepository
dotnet_naming_rule.interfaces_must_start_with_i.symbols  = interface_symbols
dotnet_naming_rule.interfaces_must_start_with_i.style    = prefix_i_style
dotnet_naming_rule.interfaces_must_start_with_i.severity = error

dotnet_naming_symbols.interface_symbols.applicable_kinds = interface

dotnet_naming_style.prefix_i_style.required_prefix = I
dotnet_naming_style.prefix_i_style.capitalization   = pascal_case

# Private fields must start with _ — _connectionString not connectionString
dotnet_naming_rule.private_fields_underscore.symbols  = private_field_symbols
dotnet_naming_rule.private_fields_underscore.style    = underscore_camel_style
dotnet_naming_rule.private_fields_underscore.severity = error

dotnet_naming_symbols.private_field_symbols.applicable_kinds           = field
dotnet_naming_symbols.private_field_symbols.applicable_accessibilities = private

dotnet_naming_style.underscore_camel_style.required_prefix = _
dotnet_naming_style.underscore_camel_style.capitalization   = camel_case

# Async methods must end with Async — GetByIdAsync not GetById
dotnet_naming_rule.async_suffix.symbols  = async_method_symbols
dotnet_naming_rule.async_suffix.style    = end_with_async_style
dotnet_naming_rule.async_suffix.severity = error

dotnet_naming_symbols.async_method_symbols.applicable_kinds      = method
dotnet_naming_symbols.async_method_symbols.required_modifiers    = async

dotnet_naming_style.end_with_async_style.required_suffix = Async
dotnet_naming_style.end_with_async_style.capitalization  = pascal_case

# Constants — UPPER_CASE not allowed, PascalCase only
dotnet_naming_rule.constants_pascal.symbols  = constant_symbols
dotnet_naming_rule.constants_pascal.style    = pascal_case_style
dotnet_naming_rule.constants_pascal.severity = error

dotnet_naming_symbols.constant_symbols.applicable_kinds            = field
dotnet_naming_symbols.constant_symbols.required_modifiers          = const
dotnet_naming_symbols.constant_symbols.applicable_accessibilities  = *

dotnet_naming_style.pascal_case_style.capitalization = pascal_case

# ─── Code Style ────────────────────────────────────────────────────────

# Braces — always required
csharp_prefer_braces = true:error

# var usage
csharp_style_var_for_built_in_types    = false:error    # int x not var x
csharp_style_var_when_type_is_apparent = true:suggestion
csharp_style_var_elsewhere             = true:suggestion

# Expression body
csharp_style_expression_bodied_methods      = when_on_single_line:suggestion
csharp_style_expression_bodied_properties   = true:suggestion
csharp_style_expression_bodied_constructors = false:error   # never expression body ctor

# Prefer primary constructors (C# 12+)
csharp_style_prefer_primary_constructors = true:suggestion

# Null checks
dotnet_style_prefer_is_null_check_over_reference_equality_method = true:error
csharp_style_prefer_null_check_over_type_check                   = true:error

# Pattern matching
csharp_style_prefer_pattern_matching    = true:suggestion
csharp_style_prefer_switch_expression  = true:suggestion

# Using directives — outside namespace
csharp_using_directive_placement = outside_namespace:error

# this. qualifier — never use
dotnet_style_qualification_for_field    = false:error
dotnet_style_qualification_for_property = false:error
dotnet_style_qualification_for_method   = false:error

# Object initializers
dotnet_style_object_initializer         = true:suggestion
dotnet_style_collection_initializer     = true:suggestion

# ─── Diagnostic Severity Overrides ────────────────────────────────────

# Nullable reference errors
dotnet_diagnostic.CS8600.severity = error   # null assigned to non-nullable
dotnet_diagnostic.CS8601.severity = error   # null reference assignment
dotnet_diagnostic.CS8602.severity = error   # dereference of possibly null
dotnet_diagnostic.CS8603.severity = error   # null return from non-nullable method
dotnet_diagnostic.CS8604.severity = error   # null argument for non-nullable param
dotnet_diagnostic.CS8625.severity = error   # null literal for non-nullable

# Fire-and-forget Task — must await
dotnet_diagnostic.CS4014.severity = error

# Unused variables + parameters
dotnet_diagnostic.CS0219.severity = error   # variable assigned but never used
dotnet_diagnostic.IDE0060.severity = warning # unused parameter

# Obsolete usage
dotnet_diagnostic.CS0618.severity = error

# Async void — never (except event handlers)
dotnet_diagnostic.MA0004.severity = error

# ─── JSON / YAML files ─────────────────────────────────────────────────
[*.json]
indent_size = 2

[*.yml]
indent_size = 2
```

---

## 3. Architecture Tests — NetArchTest

### Project Setup

```xml
<!-- tests/Architecture.Tests/Architecture.Tests.csproj -->
<Project Sdk="Microsoft.NET.Sdk">
  <ItemGroup>
    <PackageReference Include="NetArchTest.Rules"    Version="1.3.*" />
    <PackageReference Include="xunit"                Version="2.*" />
    <PackageReference Include="FluentAssertions"     Version="6.*" />

    <!-- Reference ALL projects to inspect assemblies -->
    <ProjectReference Include="..\..\src\Stays.Domain\Stays.Domain.csproj" />
    <ProjectReference Include="..\..\src\Stays.Application\Stays.Application.csproj" />
    <ProjectReference Include="..\..\src\Stays.Infrastructure\Stays.Infrastructure.csproj" />
    <ProjectReference Include="..\..\src\Dining.Domain\Dining.Domain.csproj" />
    <ProjectReference Include="..\..\src\Shared.Kernel\Shared.Kernel.csproj" />
  </ItemGroup>
</Project>
```

### Layer Dependency Rules

```csharp
// tests/Architecture.Tests/LayerDependencyTests.cs
public sealed class LayerDependencyTests
{
    private static readonly Assembly StaysDomain
        = typeof(Stays.Domain.Entities.Booking).Assembly;

    private static readonly Assembly StaysApplication
        = typeof(Stays.Application.DependencyInjection).Assembly;

    private static readonly Assembly StaysInfrastructure
        = typeof(Stays.Infrastructure.StaysInfrastructureMarker).Assembly;

    [Fact]
    public void Domain_ShouldNotDependOn_Application()
    {
        Types.InAssembly(StaysDomain)
            .Should().NotHaveDependencyOn("Stays.Application")
            .GetResult().IsSuccessful.Should().BeTrue();
    }

    [Fact]
    public void Domain_ShouldNotDependOn_Infrastructure()
    {
        Types.InAssembly(StaysDomain)
            .Should().NotHaveDependencyOn("Stays.Infrastructure")
            .GetResult().IsSuccessful.Should().BeTrue();
    }

    [Fact]
    public void Application_ShouldNotDependOn_Infrastructure()
    {
        Types.InAssembly(StaysApplication)
            .Should().NotHaveDependencyOn("Stays.Infrastructure")
            .GetResult().IsSuccessful.Should().BeTrue();
    }

    [Fact]
    public void Application_ShouldNotDependOn_WebAPI()
    {
        Types.InAssembly(StaysApplication)
            .Should().NotHaveDependencyOn("Stays.WebAPI")
            .GetResult().IsSuccessful.Should().BeTrue();
    }
}
```

### Module Isolation Rules

```csharp
// tests/Architecture.Tests/ModuleIsolationTests.cs
public sealed class ModuleIsolationTests
{
    private static readonly Assembly StaysDomain
        = typeof(Stays.Domain.Entities.Booking).Assembly;

    private static readonly Assembly DiningDomain
        = typeof(Dining.Domain.Entities.Reservation).Assembly;

    [Fact]
    public void StaysModule_ShouldNotDependOn_DiningModule()
    {
        Types.InAssembly(StaysDomain)
            .Should().NotHaveDependencyOn("Dining")
            .GetResult().IsSuccessful.Should().BeTrue();
    }

    [Fact]
    public void DiningModule_ShouldNotDependOn_StaysModule()
    {
        Types.InAssembly(DiningDomain)
            .Should().NotHaveDependencyOn("Stays")
            .GetResult().IsSuccessful.Should().BeTrue();
    }
}
```

### Naming Convention Rules

```csharp
// tests/Architecture.Tests/NamingConventionTests.cs
public sealed class NamingConventionTests
{
    private static readonly Assembly StaysApplication
        = typeof(Stays.Application.DependencyInjection).Assembly;

    private static readonly Assembly StaysDomain
        = typeof(Stays.Domain.Entities.Booking).Assembly;

    // All handlers must be sealed
    [Fact]
    public void CommandHandlers_ShouldBeSealed()
    {
        Types.InAssembly(StaysApplication)
            .That().ImplementInterface(typeof(IRequestHandler<,>))
            .And().HaveNameEndingWith("Handler")
            .Should().BeSealed()
            .GetResult().IsSuccessful.Should().BeTrue();
    }

    // All handlers must end with Handler
    [Fact]
    public void CommandHandlers_ShouldEndWithHandler()
    {
        Types.InAssembly(StaysApplication)
            .That().ImplementInterface(typeof(IRequestHandler<,>))
            .Should().HaveNameEndingWith("Handler")
            .GetResult().IsSuccessful.Should().BeTrue();
    }

    // All domain entities must live in Domain layer
    [Fact]
    public void Entities_ShouldOnlyBeIn_DomainLayer()
    {
        Types.InAssembly(StaysApplication)
            .That().Inherit(typeof(Entity<>))
            .Should().NotExist()   // entities must not be in Application
            .GetResult().IsSuccessful.Should().BeTrue();
    }

    // Validators must end with Validator
    [Fact]
    public void Validators_ShouldEndWithValidator()
    {
        Types.InAssembly(StaysApplication)
            .That().Inherit(typeof(AbstractValidator<>))
            .Should().HaveNameEndingWith("Validator")
            .GetResult().IsSuccessful.Should().BeTrue();
    }

    // Read repos must end with ReadRepository
    [Fact]
    public void ReadRepositories_ShouldEndWithReadRepository()
    {
        Types.InAssembly(typeof(Stays.Infrastructure.StaysInfrastructureMarker).Assembly)
            .That().ImplementInterface(typeof(IReadRepository))
            .Should().HaveNameEndingWith("ReadRepository")
            .GetResult().IsSuccessful.Should().BeTrue();
    }

    // Write repos must end with WriteRepository
    [Fact]
    public void WriteRepositories_ShouldEndWithWriteRepository()
    {
        Types.InAssembly(typeof(Stays.Infrastructure.StaysInfrastructureMarker).Assembly)
            .That().ImplementInterface(typeof(IWriteRepository))
            .Should().HaveNameEndingWith("WriteRepository")
            .GetResult().IsSuccessful.Should().BeTrue();
    }
}
```

---

## 4. Pre-commit Hook — Git Level Enforcement

Architecture tests commit பண்ணும் முன்னாடி run ஆகும். Fail ஆனா commit block.

```bash
# .git/hooks/pre-commit
#!/bin/sh

echo "── OneNex Pre-commit Checks ──"

# Build — fail fast
echo "→ Building solution..."
dotnet build --no-incremental -q
if [ $? -ne 0 ]; then
  echo "❌ Build failed. Commit blocked."
  exit 1
fi

# Architecture tests
echo "→ Running architecture tests..."
dotnet test tests/Architecture.Tests --no-build -q
if [ $? -ne 0 ]; then
  echo "❌ Architecture tests failed. Commit blocked."
  echo "   Check module boundaries and naming conventions."
  exit 1
fi

echo "✅ All checks passed. Proceeding with commit."
exit 0
```

Make executable:
```bash
chmod +x .git/hooks/pre-commit
```

Team-wide share பண்ண (`.git/hooks` = not tracked by git):

```bash
# .husky/pre-commit  (if using Husky for .NET)
# or
# scripts/setup-hooks.sh — team members run once after clone
#!/bin/sh
cp scripts/pre-commit .git/hooks/pre-commit
chmod +x .git/hooks/pre-commit
echo "✅ Git hooks installed."
```

---

## 5. Test Watch — Dev Time

Development பண்ணும் போது background-ல் architecture tests watch mode-ல் run:

```bash
# Terminal window — always running
dotnet test tests/Architecture.Tests --watch
```

File save ஆகும்போது automatic run. Boundary violate பண்ணா உடனே தெரியும்.

---

## Enforcement Summary

| Rule | Tool | When caught | Blocks? |
|---|---|---|---|
| Interface starts with I | EditorConfig | Real-time IDE | Build error |
| Private field `_` prefix | EditorConfig | Real-time IDE | Build error |
| Async suffix `Async` | EditorConfig | Real-time IDE | Build error |
| Null reference safety | CS82xx | Real-time IDE | Build error |
| Fire-and-forget Task | CS4014 | Real-time IDE | Build error |
| Unused variables | CS0219 | Build | Build error |
| Code style violations | EnforceCodeStyleInBuild | Build | Build error |
| Domain → Infra dependency | NetArchTest | Pre-commit / CI | Commit blocked |
| Module cross-dependency | NetArchTest | Pre-commit / CI | Commit blocked |
| Handler not sealed | NetArchTest | Pre-commit / CI | Commit blocked |
| Wrong naming suffix | NetArchTest | Pre-commit / CI | Commit blocked |

---

## File Locations

```
solution root/
├── Directory.Build.props   ← global project settings (all projects inherit)
├── .editorconfig           ← naming + style rules (IDE + build)
│
├── src/
│   └── ... (all source projects)
│
├── tests/
│   └── Architecture.Tests/
│       ├── LayerDependencyTests.cs
│       ├── ModuleIsolationTests.cs
│       └── NamingConventionTests.cs
│
└── scripts/
    ├── pre-commit              ← git hook template
    └── setup-hooks.sh          ← run once after clone
```

---

## Rules

```
1. Directory.Build.props  → solution root. Never per-project duplicate.
2. TreatWarningsAsErrors  → always on. Zero warning tolerance.
3. Nullable               → always enabled. No ? suppression without reason.
4. EditorConfig           → naming + style. IDE enforces real-time.
5. NetArchTest            → architecture boundaries. Pre-commit + CI.
6. Pre-commit hook        → build + architecture tests. Commit blocked on fail.
7. dotnet test --watch    → dev time architecture test watch (background terminal).
8. New naming rule        → add to both .editorconfig AND NetArchTest (belt + suspenders).
```
