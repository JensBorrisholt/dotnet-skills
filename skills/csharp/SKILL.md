---
name: csharp
description: Invoke this FIRST, before you write, review, refactor, modernize or explain ANY C# or .NET code, including small edits and questions about syntax. It determines which C# language version the project uses (from TargetFramework and LangVersion) and which version-specific skill to load (csharp-12, csharp-13, csharp-14 or csharp-15), so you never use syntax the project's compiler does not support.
---

# C# version router

This skill does one thing: find the language version of the code you are about to touch, then load
the matching skill. Do not write C# before you have done this.

## Step 1: find the version

Look at the project that **owns the file** you are working in: the nearest `.csproj` above it.
Check these in order, and stop at the first that answers:

1. `<LangVersion>` in the `.csproj`, then in `Directory.Build.props` or `Directory.Build.targets`
   in any parent folder.
   - A number (`12`, `13`, `14`, `15`) is the answer.
   - `latest`, `default`, `preview` or `latestMajor` do **not** give a number. Treat them as
     "unknown, go to step 3" and run `dotnet --version`: `latest` is the newest version the
     installed SDK supports, and `preview` is newer still. Report that to the user instead of
     guessing.
2. `<TargetFramework>` or `<TargetFrameworks>`, in the same files. Use this table:

| TargetFramework | Default C# | Skill to load |
| --- | --- | --- |
| `net11.0` | 15 | `csharp-15` |
| `net10.0` | 14 | `csharp-14` |
| `net9.0` | 13 | `csharp-13` |
| `net8.0` | 12 | `csharp-12` |
| `net7.0` | 11 | none, see "Older targets" |
| `net6.0` | 10 | none, see "Older targets" |
| `net5.0` | 9 | none, see "Older targets" |
| `netcoreapp3.x`, `netstandard2.1` | 8 | none, see "Older targets" |
| `netstandard2.0`, `net4x` (.NET Framework) | 7.3 | none, see "Older targets" |

3. Nothing found, or the version is unclear: follow "When in doubt".

## Step 2: load the skill

Load the skill from the table. It may carry a plugin prefix in its name, for example
`dotnet-skills:csharp-13`. That is the same skill.

Then follow it. Its "Not available" table lists the newer features you must not use.

## Special cases

- **Multi-targeting** (`net8.0;net10.0`): the code must compile on all targets, so use the
  **lowest** version and its skill.
- **Mixed solution**: the version belongs to the project, not the solution. Decide per project.
  If you create or edit files in several projects, check each one.
- **An explicit `LangVersion` above the framework's default** (for example 14 on `net9.0`) is
  allowed for some features but unsupported for others. Use the `LangVersion`, and warn the user
  if it is higher than the target framework's default.
- **Source generators, analyzers** (`netstandard2.0`): these usually run on an old compiler
  target. Use the lowest version.

## Older targets

For C# 11 and earlier there is no skill. Use conservative syntax:
- Nothing from C# 12 or later. In particular no primary constructors on classes, no collection
  expressions (`[1, 2]`), no `params` on spans.
- For C# 7.3 (`netstandard2.0`, .NET Framework) also avoid records, `init`, file-scoped
  namespaces, nullable reference annotations (unless the project enables them through
  `LangVersion`), switch expressions, `using` declarations and ranges `..`.

When unsure, compile: `dotnet build` is the final judge of what the project accepts.

## When in doubt

If you cannot determine the version, **use the oldest syntax that would plausibly work** and say
so in one line: "I couldn't find a C# version, so I assumed C# 12." Don't ask unless the choice
would change a design decision. To confirm quickly, you can add `#error version` to a file:
the compiler reports CS8304 with the language version, so remove it afterwards.
