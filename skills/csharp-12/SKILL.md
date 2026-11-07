---
name: csharp-12
description: Write, review and modernize C# 12 (.NET 8, net8.0) code correctly. Covers primary constructors, collection expressions, inline arrays, lambda default parameters, `ref readonly` parameters, aliasing any type with `using`, `[Experimental]` and preview interceptors, plus the list of newer-language features (C# 13, 14, 15) that must NOT be used. Normally loaded through the `csharp` router skill; load it directly only when the project is known to be C# 12 / net8.0.
---

# C# 12

C# 12 ships with the .NET 8 SDK and is the default language version for `net8.0`. Do **not** use
anything from the "Not available" section below in a C# 12 project: it will not compile.

## Not available in C# 12 (do not use)

| Feature | Introduced in | What to do instead |
| --- | --- | --- |
| `params ReadOnlySpan<T>` / `params` collections | C# 13 | `params T[]` |
| `System.Threading.Lock` and its `lock` semantics | C# 13 (.NET 9 runtime) | `lock (object)` |
| `\e` escape | C# 13 | `\u001b` (not `\x1b`: following hex digits join the escape) |
| `^` index in object initializers | C# 13 | index from the front |
| `allows ref struct`, `ref struct` implementing interfaces | C# 13 | no equivalent |
| Partial properties and indexers | C# 13 | normal properties |
| `OverloadResolutionPriority` | C# 13 | no equivalent |
| `field` keyword | C# 14 (preview in 13) | declare a backing field yourself |
| `extension(...)` blocks, extension properties | C# 14 | classic `this` extension methods |
| Null-conditional assignment `a?.b = x` | C# 14 | `if (a is not null) a.b = x;` |
| Implicit `T[]` / `Span<T>` / `ReadOnlySpan<T>` conversions (new ones) | C# 14 | `.AsSpan()` |
| Lambda parameter modifiers without types | C# 14 | write the parameter types |
| `nameof(List<>)` | C# 14 | `nameof(List<int>)` |
| Partial constructors and events | C# 14 | no equivalent |
| `[with(...), ...]` collection arguments, `union`, `closed`, extension indexers, labeled `break`/`continue` | C# 15 | classic constructors, base classes, flags or `goto` |

Features from C# 11 and earlier (raw string literals, `required` members, list patterns, file-scoped
namespaces, generic math and so on) are all available.

## 0. Before using a C# 12 feature

1. Confirm `TargetFramework` is `net8.0` and `LangVersion` is unset or `12`. Don't set
   `LangVersion` to `latest` yourself.
   - An existing `latest` means the newest C# the installed SDK supports. Another machine's SDK
     may give a newer version. Stay with C# 12 syntax anyway.
   - An existing `preview` enables in-progress features of the installed compiler. Use preview
     features only when the user asks for them or the project already uses them.
2. Interceptors need an opt-in property (section 8) and are preview.

## 1. Primary constructors

Allowed on any `class` and `struct`, not only records.

```csharp
public class Service(ILogger<Service> logger, IOptions<Settings> options)
{
    public void Run() => logger.LogInformation("Value {Value}", options.Value.Name);
}
```

- The parameters are in scope for the whole type body.
- Every other constructor you declare must call the primary constructor with `this(...)`.
- A primary constructor on a class suppresses the implicit parameterless constructor.
- **Properties are generated only for `record class` and `record struct`.** On an ordinary class
  or struct the parameters are plain captured values, not members. If you need a public
  property, declare it and initialize it from the parameter.
- Captured parameters are **mutable**. Assigning to one changes state for the whole type.
- A parameter that is both captured and passed to a base constructor gets stored twice. Avoid
  using it in the body after passing it to the base class.
- In a struct, the implicit parameterless constructor zeroes every field, including the primary
  constructor parameters.

## 2. Collection expressions

```csharp
int[] a = [1, 2, 3];
List<string> b = ["one", "two", "three"];
int[] joined = [.. row0, .. row1];     // spread
ReadOnlySpan<int> span = [1, 2, 3];
```

- Work for arrays, `Span<T>`, `ReadOnlySpan<T>`, types with a collection initializer, and
  `[CollectionBuilder]` types. They can be used as arguments.
- `[]` needs a target type, so `var x = [1, 2];` does not compile in C# 12.
- A target type without `[CollectionBuilder]` needs an accessible parameterless constructor and an
  accessible single-argument `Add` when the expression has elements.
- For an `IEnumerable` implementation without a typed `GetEnumerator()`, each element must convert
  to `object`. A lambda element then needs an explicit type, `[(int _) => { }]` (CS8917).
- There is **no** `with(...)` element in C# 12. To set a capacity or comparer, use a constructor.

## 3. Inline arrays

```csharp
[System.Runtime.CompilerServices.InlineArray(10)]
public struct Buffer { private int _element0; }
```

Behaves like a fixed-size array: indexing and `foreach` work. Most code consumes them through
`Span<T>` APIs and rarely declares its own.

## 4. Optional parameters in lambdas

```csharp
var increment = (int value, int step = 1) => value + step;
```

Same rules as default values on methods and local functions.

## 5. `ref readonly` parameters

An explicit read-only reference parameter modifier. Replacing `ref` with `ref readonly` in an
existing API is not a breaking change for callers. Prefer it over `in` when callers should pass
the argument by reference visibly (`ref` at the call site).

## 6. Alias any type

A `using` alias can name tuples, arrays, pointers and other unsafe types.

```csharp
using Point = (int X, int Y);
using Matrix = int[,];
```

## 7. `[Experimental]`

`System.Diagnostics.CodeAnalysis.ExperimentalAttribute` on a type, method or assembly makes the
compiler warn at use sites. Marking an assembly makes all of its types experimental.

## 8. Interceptors (preview)

Preview in C# 12. They let a source generator replace a call at compile time. They require
`InterceptorsPreviewNamespaces` in the project file and may change. Don't use them in
application code.

```xml
<InterceptorsPreviewNamespaces>$(InterceptorsPreviewNamespaces);MyLibrary.Generated</InterceptorsPreviewNamespaces>
```

## 9. Compiler breaking changes when moving to the .NET 8 SDK

- A `dynamic` argument without `ref` passed to a `ref dynamic` parameter is now an error (CS1620).
- Collection-expression conversion now requires a constructor and `Add` (see section 2), which
  changes overload resolution: `Print(['a', 'b'])` binds to `Print(char[])` rather than
  `Print(string)`.
- With LangVersion 12, a `ref` argument can bind to an `in` parameter, so some overload
  resolutions change or become ambiguous. Cast an argument, or call the extension method
  explicitly (`E.M(new C(), ref i)`).
- `await using` prefers a public pattern-based `DisposeAsync()` over an explicit
  `IAsyncDisposable.DisposeAsync()` implementation.

## Agent checklist

- [ ] Project is `net8.0` or `LangVersion` 12. No `LangVersion` of `latest`.
- [ ] Nothing from the "Not available" table appears in the code.
- [ ] `var x = [..]` is not used: collection expressions need a target type.
- [ ] Primary constructor parameters are not mistaken for properties on non-record types.

## Sources

- What's new in C# 12: https://learn.microsoft.com/dotnet/csharp/whats-new/csharp-12
- Compiler breaking changes in .NET 8: https://learn.microsoft.com/dotnet/csharp/whats-new/breaking-changes/compiler%20breaking%20changes%20-%20dotnet%208
