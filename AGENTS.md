# AGENTS.md

This file provides guidance to Codex (Codex.ai/code) when working with code in this repository.

## Repository Overview

This is an **Obsidian Vault** (personal knowledge management), not a traditional software project. It contains no build system, tests, or package managers. Content is primarily in Chinese.

### Directory Structure

| Path | Purpose |
|------|---------|
| `Templates/` | Obsidian note templates using Templater plugin syntax |
| `Codex study/` | Learning notes for Codex source code (12 chapters, Chinese) |
| `benchmark/` | Technical paper research notes (e.g., Gaussian Splatting) |
| `.obsidian/` | Obsidian app configuration and community plugins |

## Templates

The vault uses the **Templater** community plugin with folder-based auto-templates. When creating a new file in these folders, Obsidian automatically applies the corresponding template:

- `Daily/` → `Templates/Daily Note.md`
- `Project/` → `Templates/Project Note.md`
- `Meeting/` → `Templates/Meeting Note.md`

Templates use Templater syntax (`<% tp.date.now(...) %>`, `<% tp.file.title %>`). Do not confuse this with Jinja2 or EJS.

Templater settings (`trigger_on_file_creation: true`) mean new files in mapped folders auto-expand template variables.

## Active Community Plugins

- **templater-obsidian** — Template engine with folder auto-mapping
- **obsidian-excalidraw-plugin** — Hand-drawn diagrams
- **obsidian-linter** — Markdown formatting/linting
- **obsidian-hover-editor** — Hover pop-out editors
- **multi-column-markdown** — Multi-column layouts
- **recent-files-obsidian** — Recent files panel

## Codex Study Notes

`Codex study/Learn-Codex-Source/` contains 12 chapters (`s01-agent-loop.md` through `s12-cli.md`) documenting Codex's architecture. These follow a consistent format:

- Each file starts with a `# Sxx: Title` heading
- Key insights are highlighted in `> **核心洞察**` callouts
- Code examples are in Python and Rust
- `index.md` uses VitePress-style frontmatter (`layout: home`, `hero:`, `features:`)

When editing or adding to this series, maintain the existing Chinese terminology and section structure.

## Git Workflow

The vault is tracked in Git. Obsidian's `alwaysUpdateLinks: true` setting means internal `[[wikilinks]]` are auto-updated when files are renamed or moved. After bulk file operations in this repo, verify that wikilinks remain intact.
