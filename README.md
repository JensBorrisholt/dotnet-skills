# dotnet-skills

Claude Code skills for writing C# that matches the language version the project actually uses.

A project on `net9.0` compiles C# 13. If Claude writes C# 14 or 15 syntax there, the build breaks.
These skills prevent that: one **router** skill reads the project's version and loads the matching
**version skill**, which describes that version's features and lists the newer ones that must not
be used.

## Skills

| Skill | Purpose | Target framework |
| --- | --- | --- |
| `csharp` | Router. Finds the version, loads one of the skills below. | any |
| `csharp-12` | C# 12 features and the "do not use" list for C# 13 to 15. | `net8.0` |
| `csharp-13` | C# 13 features and the "do not use" list for C# 14 and 15. | `net9.0` |
| `csharp-14` | C# 14 features and the "do not use" list for C# 15. | `net10.0` |
| `csharp-15` | C# 15 features, including the preview memory-safety model. | `net11.0` |

For C# 11 and earlier there is no version skill. The router tells Claude to use conservative
syntax.

## How the version is chosen

The router looks at the project that owns the file being edited, in this order:

1. `LangVersion` in the `.csproj`, then `Directory.Build.props` and `Directory.Build.targets`.
2. `TargetFramework` or `TargetFrameworks` in the same places, mapped to its default C# version.
3. Otherwise it assumes the oldest plausible version and says so.

Multi-targeted projects use the lowest version. Nothing is added to the project: no
`CLAUDE.md`, no settings, no files.

## Install

### As a plugin (for a team)

```
/plugin marketplace add <owner>/dotnet-skills
/plugin install dotnet-skills@dotnet-skills
```

Replace `<owner>` with the GitHub account or organization that hosts this repository. Plugin skills
are namespaced, for example `dotnet-skills:csharp-13`.

### Personal install (just you)

Copy the folders in `skills/` into `~/.claude/skills/` (on Windows
`%USERPROFILE%\.claude\skills\`), so that each folder contains its own `SKILL.md`.

Use **one** of the two. Having both installed gives Claude duplicate skills to choose from.

## Layout

```
.claude-plugin/
  plugin.json
  marketplace.json
skills/
  csharp/SKILL.md        router
  csharp-12/SKILL.md
  csharp-13/SKILL.md
  csharp-14/SKILL.md
  csharp-15/SKILL.md
```

## Sources

The version skills are written from the Microsoft Learn pages "What's new in C# N" and "Compiler
breaking changes", linked at the bottom of each skill. Where a page gives no sample or detail, the
skill says so rather than inventing syntax. C# 15 and its memory-safety model were still
prerelease when this was written, so check Learn if a feature behaves differently.

## Adding a version

1. Copy the closest `skills/csharp-NN` folder.
2. Rewrite its features, its "Not available" table and its sources.
3. Add the target framework to the table in `skills/csharp/SKILL.md` and to this README.
