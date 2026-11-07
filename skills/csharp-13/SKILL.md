---
name: csharp-13
description: Write, review and modernize C# 13 (.NET 9, net9.0) code correctly. Covers `params` collections, the `System.Threading.Lock` type, `\e`, `^` in object initializers, `ref`/`unsafe` in iterators and async, `allows ref struct`, ref structs implementing interfaces, partial properties and indexers, overload resolution priority, method group natural type, and the preview `field` keyword, plus the list of newer-language features (C# 14, 15) that must NOT be used. Normally loaded through the `csharp` router skill; load it directly only when the project is known to be C# 13 / net9.0.
---

# C# 13

C# 13 ships with the .NET 9 SDK and is the default language version for `net9.0`. Do **not** use
anything from the "Not available" section below in a C# 13 project: it will not compile.

## Not available in C# 13 (do not use)

| Feature | Introduced in | What to do instead |
| --- | --- | --- |
| `field` keyword (**preview** in 13, needs `LangVersion=preview`) | C# 14 | declare a backing field yourself |
| `extension(...)` blocks, extension properties and static extensions | C# 14 | classic `this` extension methods |
| Null-conditional assignment `a?.b = x` | C# 14 | `if (a is not null) a.b = x;` |
| New implicit `T[]` / `Span<T>` / `ReadOnlySpan<T>` conversions | C# 14 | `.AsSpan()` |
| Lambda parameter modifiers without types, `(text, out result) => ...` | C# 14 | write the parameter types |
| `nameof(List<>)` | C# 14 | `nameof(List<int>)` |
| Partial constructors and partial events | C# 14 | no equivalent |
| User-defined compound assignment operators | C# 14 | rely on `+` and the compiler's `x = x + y` |
| `[with(...), ...]` collection arguments | C# 15 | a constructor call, then add |
| `union`, `closed`, extension indexers, labeled `break`/`continue` | C# 15 | classic hierarchies, flags or `goto` |

Everything in C# 12 and earlier (primary constructors, collection expressions without `with(...)`,
inline arrays, `ref readonly` parameters, alias any type, raw strings, list patterns and so on) is
available.

## 0. Before using a C# 13 feature

1. Confirm `TargetFramework` is `net9.0` and `LangVersion` is unset or `13`. Don't set
   `LangVersion` to `latest` yourself.
   - An existing `latest` means the newest C# the installed SDK supports. Another machine's SDK
     may give a newer version. Stay with C# 13 syntax anyway.
   - An existing `preview` enables in-progress features of the installed compiler. For C# 13 that
     is `field` (section 10). Use preview features only when the user asks for them or the
     project already uses them.
2. `System.Threading.Lock` is a .NET 9 runtime type. Code that uses it cannot run on net8.0.
3. Span-based `params` and `allows ref struct` with BCL types also need the matching .NET 9
   libraries.

## 1. `params` collections

`params` is no longer limited to arrays. It works with `Span<T>`, `ReadOnlySpan<T>`, types that
implement `IEnumerable<T>` and have an `Add` method, and the interfaces `IEnumerable<T>`,
`IReadOnlyCollection<T>`, `IReadOnlyList<T>`, `ICollection<T>` and `IList<T>`.

```csharp
public static void Concat<T>(params ReadOnlySpan<T> items) { foreach (var item in items) { } }

Concat(1, 2, 3);        // no array allocation
```

Prefer `params ReadOnlySpan<T>` on new hot-path APIs, because the compiler can stack-allocate the
storage. For an interface type the compiler synthesizes the storage.

## 2. `System.Threading.Lock`

```csharp
private readonly Lock _gate = new();

public void Update()
{
    lock (_gate)          // compiles to Lock.EnterScope(), not Monitor
    {
        // critical section
    }
}
```

- Declare the field as `Lock`, not `object`. That is the only change needed.
- If the `Lock` is converted to another type (an `object` variable, for example), the compiler
  falls back to the old `Monitor`-based code.
- `Lock.EnterScope()` returns a `ref struct` whose `Dispose()` exits the scope, so
  `using (_gate.EnterScope())` also works.

## 3. `\e` escape

`\e` is the ESCAPE character (U+001B). Use it instead of `\u001b` or `\x1b`. The latter is
error-prone because following hex digits join the escape.

## 4. `^` in object initializers

```csharp
var timer = new TimerRemaining { buffer = { [^1] = 0, [^2] = 1 } };
```

Works for single-dimension collections. Before C# 13 you had to index from the front.

## 5. `ref`, `ref struct` and `unsafe` in iterators and async

- `async` methods can declare `ref` locals and `ref struct` locals, but they cannot be accessed
  across an `await`.
- Iterators can declare `ref` locals and `ref struct` locals, but not across a `yield return`.
- Iterators can contain `unsafe` contexts, but every `yield return` and `yield break` must be in
  safe code.

## 6. `allows ref struct` and ref structs implementing interfaces

```csharp
public class Holder<T> where T : allows ref struct
{
    public void Use(scoped T value) { }
}
```

- `where T : allows ref struct` lets a `ref struct` be a type argument. Ref safety rules then
  apply to the parameter.
- A `ref struct` can implement interfaces but cannot be **converted** to one, because that would
  box it.
- Explicit interface implementations on a `ref struct` are only reachable through a type parameter
  that `allows ref struct`.
- A `ref struct` must implement every interface member, including members that have a default
  implementation.

## 7. Partial properties and indexers

```csharp
public partial class Person
{
    public partial string Name { get; set; }          // declaring
}

public partial class Person
{
    private string _name = "";
    public partial string Name { get => _name; set => _name = value; }   // implementing
}
```

- Exactly one declaring and one implementing declaration, with matching signatures.
- An auto-property can't be the implementing declaration. It is aimed at source generators.

## 8. `OverloadResolutionPriority`

`[OverloadResolutionPriority(n)]` lets a library prefer one overload without breaking existing
callers. A higher priority wins. Use sparingly: it makes overload choice harder to follow.

## 9. Method group natural type

Overload resolution prunes inapplicable candidates (wrong generic arity, unsatisfied constraints)
at each scope and moves outward only when none are left. This makes `var f = Method;` work in more
cases.

## 10. `field` keyword (preview only)

`field` exists in C# 13 only behind `LangVersion=preview`. **Don't use it in a net9.0 project.** It
becomes stable in C# 14. If the type has a member named `field`, a future upgrade changes its
meaning, so prefer to rename it, or use `@field` or `this.field` now.

## 11. Compiler breaking changes when moving to the .NET 9 SDK

- `InlineArray` on a `record struct` is an error (CS9259).
- An iterator becomes a safe context: a local function inside an iterator that uses pointers needs
  its own `unsafe`.
- Collection expression overload resolution changed: an empty `[]` passed to `M(ReadOnlySpan<int>)`
  and `M(Span<object>)` is now an error, and an exact element-type match wins, so `M([1])` can
  resolve to `Span<int>` instead of `ReadOnlySpan<byte>`.
- Indexers without a proper `DefaultMemberAttribute` are an error (CS0656).
- Method group natural type: ambiguous extension candidates now give an error instead of silently
  inferring a delegate type. Use an explicit delegate type instead of `var`.
- `dotnet_style_require_accessibility_modifiers` now applies to interface members, which can
  produce new analyzer diagnostics.

## Agent checklist

- [ ] Project is `net9.0` or `LangVersion` 13. No `LangVersion` of `latest`, no `preview`.
- [ ] Nothing from the "Not available" table appears in the code, in particular no `field`.
- [ ] New lock fields are typed `Lock`, not `object`.
- [ ] `params` on new APIs is `ReadOnlySpan<T>` unless callers need a mutable array.

## Sources

- What's new in C# 13: https://learn.microsoft.com/dotnet/csharp/whats-new/csharp-13
- Compiler breaking changes in .NET 9: https://learn.microsoft.com/dotnet/csharp/whats-new/breaking-changes/compiler%20breaking%20changes%20-%20dotnet%209
