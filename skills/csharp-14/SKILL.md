---
name: csharp-14
description: Write, review and modernize C# 14 (.NET 10, net10.0) code correctly. Covers extension members (`extension` blocks, extension properties, static extensions, operators), the `field` keyword, null-conditional assignment, implicit span conversions, `nameof` on unbound generics, lambda parameter modifiers without types, partial constructors and events, and user-defined compound assignment operators, plus the list of C# 15 features that must NOT be used. Use when code targets net10.0 or LangVersion 14, or when the build reports CS9258, CS9272, or errors on `extension`. Normally loaded through the `csharp` router skill; load it directly only when the project is known to be C# 14 / net10.0.
---

# C# 14

C# 14 ships with the .NET 10 SDK and Visual Studio 2026, and is the default language version for
`net10.0`. Do **not** use anything from the "Not available" section below in a C# 14 project.

## Not available in C# 14 (do not use)

| Feature | Introduced in | What to do instead |
| --- | --- | --- |
| `[with(...), ...]` collection expression arguments | C# 15 | construct the collection, then add |
| `union` types | C# 15 | marker interface, abstract base class, or a result type |
| `closed` classes | C# 15 | `abstract` base with a `_` arm or a throw in switches |
| Indexers inside `extension` blocks | C# 15 | an extension method such as `ElementAtOrDefault` |
| Labeled `break` / `continue` | C# 15 | extract a method, a flag, or `goto` |
| Memory-safety preview (`unsafe(expr)`, `safe`, requires-unsafe members) | C# 15 preview | classic `unsafe` blocks |

## 0. Before using a C# 14 feature

1. Confirm `TargetFramework` is `net10.0` and `LangVersion` is unset or `14`. Don't set
   `LangVersion` to `latest`.
2. Using C# 14 while targeting something older than net10.0 is unsupported. For example, on older
   targets `array.Reverse()` can bind to `MemoryExtensions.Reverse` (in place, returns `void`) and
   not to LINQ.
3. New contextual keywords: **`extension`** and **`field`** (inside accessors). Escape them with
   `@extension` and `@field` when an existing identifier collides.

## 1. Extension members

The new `extension` block declares extension properties, methods and static members (including
user-defined operators) for a receiver type. Classic `this` extension methods still work.

```csharp
public static class EnumerableExtensions
{
    extension<T>(IEnumerable<T> source)            // instance-style members, named receiver
    {
        public bool IsEmpty => !source.Any();
        public IEnumerable<T> Where(Func<T, bool> predicate) => Enumerable.Where(source, predicate);
    }

    extension<T>(IEnumerable<T>)                   // receiver type only: static members
    {
        public static IEnumerable<T> Identity => Enumerable.Empty<T>();
        public static IEnumerable<T> operator +(IEnumerable<T> a, IEnumerable<T> b) => a.Concat(b);
    }
}

var empty = numbers.IsEmpty;
var none = IEnumerable<int>.Identity;
```

- Instance-style members need a **named** receiver parameter. Static members don't.
- Every type parameter of a generic block must be used by the receiver.
- Extension indexers are **not** in C# 14. They arrive in C# 15.
- `extension` is a contextual keyword: a type named `extension` needs `@extension`, and
  `extension M() { }` as a return type no longer compiles.

## 2. `field` keyword

`field` refers to the compiler-synthesized backing field inside a property accessor, so one or
both accessors can have a body without a declared field.

```csharp
public string Message
{
    get;
    set => field = value ?? throw new ArgumentNullException(nameof(value));
}

public SqlConnection Connection => field ??= new SqlConnection(connectionString);   // lazy
public string Name { get; set => field = value?.Trim() ?? string.Empty; }           // normalize
```

- If the type has a member named `field`, a bare `field` in an accessor now binds to the backing
  field and produces warning CS9258. Use `this.field` or `@field`, or rename the member.
- A local or parameter named `field` inside an accessor is error CS9272. Rename it.
- Migrate to `field` only where the accessor has real logic. Plain auto-properties stay as they
  are.

## 3. Null-conditional assignment

```csharp
customer?.Order = GetCurrentOrder();      // GetCurrentOrder() runs only if customer is not null
settings?[key] += 1;
```

- Works on `?.` and `?[]` on the left of `=` and of compound assignments (`+=`, `-=`, ...).
- `++` and `--` are **not** allowed.
- The right-hand side is not evaluated when the left side is null.

## 4. Implicit span conversions

There are new implicit conversions between `T[]`, `Span<T>` and `ReadOnlySpan<T>`. Span types can
be extension receivers and take part in generic inference.

Consequence: overload resolution can change or become ambiguous. When the compiler complains, or
a call silently changes meaning, make the choice explicit with `.AsSpan()` or `.AsEnumerable()`.
For a covariant array an `IEnumerable<T>` overload replaced by a `Span<T>` overload can throw
`ArrayTypeMismatchException` at run time.

## 5. `nameof` with unbound generics

`nameof(List<>)` is `"List"`. Before C# 14 only closed types such as `List<int>` were accepted.

## 6. Lambda parameters with modifiers, no types

```csharp
delegate bool TryParse<T>(string text, out T result);
TryParse<int> parse = (text, out result) => int.TryParse(text, out result);
```

`scoped`, `ref`, `in`, `out` and `ref readonly` can be used without parameter types. **`params`
still requires an explicitly typed parameter list.** In a lambda parameter list `scoped` is always
a modifier, so a type named `scoped` needs `@scoped`.

## 7. Partial constructors and events

- Instance constructors and events can be `partial`, with exactly one defining and one
  implementing declaration.
- Only the implementing constructor declaration can have a `this(...)` or `base(...)` initializer.
- Only one partial type declaration can use primary-constructor syntax.
- The implementing declaration of a partial event must have `add` and `remove` accessors. The
  defining declaration is a field-like event.
- Partial properties and events on an interface are now implicitly `public` and `virtual`.

## 8. User-defined compound assignment operators

A type can define `+=`, `-=` and so on directly, which lets it update in place instead of
allocating a new value. The Learn page doesn't give a sample, so check the spec before relying
on exact syntax:
https://learn.microsoft.com/dotnet/csharp/language-reference/proposals/csharp-14.0/user-defined-compound-assignment

## 9. Compiler breaking changes when moving to the .NET 10 SDK

- `field` conflicts: CS9258 (warning) and CS9272 (error), see section 2.
- Span overload resolution changes, see section 4. Examples: `Assert.Equal([2], x)` with a
  `long[]` becomes ambiguous, and `MemoryMarshal.Cast<double, ulong>(x)` with a `double[]` can
  stop compiling. Pass `x.AsSpan()`.
- `array.Reverse()` binding when targeting older than net10.0 (section 0). Call
  `Enumerable.Reverse(x)`.
- The compiler now warns on `is not null or 42` and `is not int or string`, because `not` binds
  tighter than `or`. Write `is not (null or 42)` for the intended meaning.
- `partial F()` as a return type no longer compiles. Use `@partial`.
- A record with pointer-typed instance members is rejected even if it defines `Equals` (CS8908).
- A lambda or local function with a `params` collection parameter needs
  `ParamCollectionAttribute` in `.netmodule` builds. This does not apply to DLL or EXE builds.
- Moving `UnscopedRefAttribute` code from old ref-safety rules can give CS8170 or warning CS9269.

## Agent checklist

- [ ] Project is `net10.0` or `LangVersion` 14. No `LangVersion` of `latest`.
- [ ] Nothing from the "Not available" table appears in the code.
- [ ] `extension` blocks use a named receiver for instance-style members, and contain no indexers.
- [ ] `field` is used only where an accessor has real logic, and no member is named `field`.
- [ ] After moving to net10.0, overload-resolution changes from the span conversions were checked.

## Sources

- What's new in C# 14: https://learn.microsoft.com/dotnet/csharp/whats-new/csharp-14
- Compiler breaking changes in .NET 10: https://learn.microsoft.com/dotnet/csharp/whats-new/breaking-changes/compiler%20breaking%20changes%20-%20dotnet%2010
- Extension members: https://learn.microsoft.com/dotnet/csharp/language-reference/keywords/extension
