# ThatOpen Claude Skill Package

<p align="center">
  <img src="docs/social-preview.png" alt="18 Deterministic Skills for ThatOpen BIM Development" width="100%">
</p>

> Production-ready Claude Code skills for building web-based BIM applications with [ThatOpen Engine](https://github.com/ThatOpen/engine_components) 3.3.x

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)
[![Skills: 18](https://img.shields.io/badge/Skills-18-blue.svg)](INDEX.md)
[![ThatOpen: 3.3.x](https://img.shields.io/badge/ThatOpen-3.3.x-green.svg)](https://github.com/ThatOpen/engine_components)

## What is this?

A collection of **18 deterministic skills** that give Claude Code deep knowledge of the ThatOpen BIM engine ecosystem. Each skill contains verified API patterns, working code examples, and documented anti-patterns — all sourced from the official ThatOpen repositories.

## Installation

Add these skills to your Claude Code project:

```bash
# Clone into your project's .claude/skills/ directory
git clone https://github.com/OpenAEC-Foundation/ThatOpen-Claude-Skill-Package.git .claude/skills/thatopen
```

Or add as a git submodule:

```bash
git submodule add https://github.com/OpenAEC-Foundation/ThatOpen-Claude-Skill-Package.git .claude/skills/thatopen
```

## Skills Overview

### Core (3 skills) — Foundation knowledge

| Skill | What it covers |
|-------|---------------|
| `thatopen-core-architecture` | Component system, lifecycle interfaces, World system, Event system |
| `thatopen-core-web-ifc` | web-ifc WASM engine, IFC parsing, geometry extraction |
| `thatopen-core-fragments` | Fragment binary format, FragmentsManager, Web Workers |

### Syntax (4 skills) — API patterns

| Skill | What it covers |
|-------|---------------|
| `thatopen-syntax-components` | `components.get()` singleton pattern, custom components |
| `thatopen-syntax-ifc-loading` | IfcLoader, WASM configuration, IfcImporter |
| `thatopen-syntax-properties` | Classifier, property queries, spatial structure |
| `thatopen-syntax-ui` | `@thatopen/ui` web components, CSS theming |

### Implementation (7 skills) — Step-by-step workflows

| Skill | What it covers |
|-------|---------------|
| `thatopen-impl-viewer` | Complete viewer setup with PostproductionRenderer |
| `thatopen-impl-navigation` | Camera modes (Orbit, FirstPerson, Plan), projection |
| `thatopen-impl-highlighting` | Highlighter, Hoverer, Outliner, Mesher, FastModelPicker |
| `thatopen-impl-measurements` | Length, Area, Volume, Angle measurements with snapping |
| `thatopen-impl-clipping-plans` | Clipper, ClipStyler, floor plans, cross-sections |
| `thatopen-impl-bcf` | BCF v2.1/v3.0 issue tracking, viewpoints |
| `thatopen-impl-federation` | Multi-model loading, coordinate alignment, visibility |

### Error Handling (2 skills) — Troubleshooting

| Skill | What it covers |
|-------|---------------|
| `thatopen-errors-loading` | WASM failures, IFC parse errors, version mismatches |
| `thatopen-errors-performance` | Memory leaks, disposal patterns, large model strategies |

### Agents (2 skills) — Guided workflows

| Skill | What it covers |
|-------|---------------|
| `thatopen-agents-viewer-builder` | Scaffold a complete BIM viewer from scratch |
| `thatopen-agents-model-analyzer` | Analyze IFC model contents and generate reports |

## Target Versions

| Package | Version |
|---------|---------|
| `@thatopen/components` | 3.3.x |
| `@thatopen/components-front` | 3.3.x |
| `@thatopen/fragments` | 3.3.x |
| `web-ifc` | 0.0.77+ |
| `@thatopen/ui` | 3.3.x |
| `three` | >= 0.175 |

## Skill Structure

Each skill follows a consistent structure:

```
skills/source/thatopen-{category}/thatopen-{category}-{topic}/
  SKILL.md              # Main skill file (<500 lines)
  references/
    methods.md          # Complete API signatures
    examples.md         # Working code examples
    anti-patterns.md    # What NOT to do (with fixes)
```

## Quality Guarantees

- **Source-verified**: All API patterns traced to official ThatOpen repositories
- **Deterministic language**: ALWAYS/NEVER rules, not vague suggestions
- **Version-pinned**: Targets specific package versions with migration notes
- **Error-aware**: Every skill includes failure modes and recovery patterns
- **Copy-pasteable examples**: Complete, runnable code with imports

## Project Structure

```
ThatOpen-Claude-Skill-Package/
  CLAUDE.md         # Project configuration
  INDEX.md          # Complete skill catalog with dependency graph
  ROADMAP.md        # Development status
  REQUIREMENTS.md   # Quality requirements
  DECISIONS.md      # Architectural decisions
  LESSONS.md        # Lessons learned
  SOURCES.md        # Verified source URLs
  skills/source/    # All 18 skills (72 files)
  docs/research/    # Deep research (vooronderzoek)
  docs/masterplan/  # Execution plan
```

## Contributing

This skill package is maintained by the [OpenAEC Foundation](https://github.com/OpenAEC-Foundation). Contributions welcome — please ensure all API references are verified against official ThatOpen sources.

## License

MIT License - see [LICENSE](LICENSE) for details.

---

Built with the [Skill Package Workflow Template](https://github.com/OpenAEC-Foundation/Skill-Package-Workflow-Template) methodology.
