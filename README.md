# Trysil AI Skills

LLM skills for **[Trysil - Delphi ORM](https://github.com/davidlastrucci/Trysil)**: ORM, JSON and HTTP, for every major coding assistant.

These are instruction documents that teach an AI coding assistant how to use Trysil's public API correctly - entity mapping, CRUD and queries, JSON serialization, and REST hosting. Drop the right files into your project and your agent writes real Trysil code instead of guessing.

## Skills

| Skill | Covers |
|---|---|
| **trysil-orm** | Connections for all 7 FireDAC drivers, `TTContext`, entity mapping attributes, CRUD, `TTFilterBuilder<T>`, `TTSession<T>` (unit of work), lazy loading, change tracking & soft delete, JOIN queries, `RawSelect`. |
| **trysil-json** | `TTJSonContext`, `TTJSonSerializerConfig`, entity/list serialize & deserialize, `DatasetToJSon`, `MetadataToJSon`, `[TJSonIgnore]`. |
| **trysil-http** | Attribute-routed controllers, `TTHttpServer` bootstrap, per-request context (DI), CORS, Basic/Bearer/Digest/JWT authentication, JSON-driven server-side filtering, multi-tenant hosting, logging. |

`trysil-json` and `trysil-http` build on `trysil-orm` and reference it rather than repeating its content. For a REST API you typically want all three.

## Installing

### Via Trysil Expert (recommended)

In the Delphi IDE, use **Trysil Expert → Install AI assistant skills…**, pick your coding assistant, and the skills are written into the right place in your active project. Trysil Expert downloads them from this repository, so you always get the current version.

> Skills track the latest Trysil API. Stay on the latest Trysil release to use them.

### Manually

Each coding assistant has its own convention for where project instructions live. This repo ships a ready-made folder per tool; copy its contents into the **root of your project**:

| Assistant | Copy from | Lands in your project as |
|---|---|---|
| Claude Code | `claude-code/` | `.claude/skills/<skill>/SKILL.md` |
| Cursor | `cursor/` | `.cursor/rules/<skill>.mdc` |
| GitHub Copilot | `copilot/` | `.github/copilot-instructions.md` (the three skills concatenated) |
| Windsurf | `windsurf/` | `.windsurf/rules/<skill>.md` |
| Any other tool | `generic/` | `llm/<skill>.md` + `llms.txt` |

## Repository layout

```
claude-code/          ← canonical skills (edit these by hand)
  .claude/skills/trysil-orm/SKILL.md
  .claude/skills/trysil-json/SKILL.md
  .claude/skills/trysil-http/SKILL.md
cursor/               ┐
copilot/              │  generated from claude-code/
windsurf/             │  (do not edit by hand)
generic/              ┘
```

The Claude Code folder is the single source of truth - its `SKILL.md` format is the richest, so the other tool folders are derived from it. Edit `claude-code/`, never the generated folders.

## Contributing

Edit the canonical skills under `claude-code/.claude/skills/` and open a pull request. The `cursor/`, `copilot/`, `windsurf/` and `generic/` folders are generated artifacts - leave them to the maintainer, who regenerates them from `claude-code/` before each release.

## Scope

These documents describe the public, consumer-facing API of Trysil - how an application *uses* the framework. They impose no particular Delphi coding style. The source of truth is always the [Trysil source code](https://github.com/davidlastrucci/Trysil).

## License

Distributed under the same license as Trysil.
