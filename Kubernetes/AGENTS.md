# Repository Guidelines

This repository is an Obsidian Markdown knowledge base for a production-grade Kubernetes learning roadmap. It is not a runnable code project: there is no build system, test suite, or package manifest.

## Project Structure & Module Organization

- Notes live at the vault root (e.g., `00_简介.md`).
- `00_简介.md` is the roadmap "map"; it links to per-stage notes and is the single source of truth for the learning path.
- Stage notes use numbered filenames: `NN_Topic.md` (e.g., `01_核心架构与对象模型.md`, `03_调度_资源_QoS.md`).
- Every note starts with YAML frontmatter containing `tags` and `created` (ISO date), followed by exactly one `#` title.

## Coding Style & Naming Conventions

- Markdown, UTF-8, LF line endings.
- Headings: `#` once per note, `##` for sections, `###` for subsections.
- Note bodies are written in Chinese (zh-CN). Keep standard English technical terms inline where appropriate (e.g., `Service`, `etcd`).
- Use Obsidian wikilinks for cross-references, e.g. `[[03_调度_资源_QoS]]`. Link text must match the target note name exactly.
- Use `- [ ]` checklists and pipe-aligned tables; avoid horizontal rules inside notes.

## Content Quality Guidelines

- Follow the three-layer structure: 概念 → 生产实践 → 要点自测 (concept → production practice → self-check).
- Every wikilink must resolve to an existing note or a note intentionally created in the same change.
- Commands, versions, and behavior claims should be verifiable against the official Kubernetes docs.
- No linters or automated checks are configured; review changes manually before committing.

## Commit & Pull Request Guidelines

- No Git history exists yet. When versioning begins, use Conventional Commits: `docs: add 01_核心架构与对象模型`, `docs: update QoS notes`, `chore: add AGENTS.md`.
- One logical note change per commit; keep the summary short and specific.
- Before merging, confirm wikilinks resolve and that `00_简介.md`'s checklist and links match any new or renamed notes.

## Agent-Specific Instructions

- Preserve the existing roadmap structure and Chinese voice; do not translate note content.
- Do not renumber existing notes without updating the link graph in `00_简介.md`.
- When adding a stage, add both a checklist item and a wikilink in `00_简介.md`.
- Do not invent or remove Kubernetes commands or behavior without verifying against the official documentation.
- When a knowledge point is uncertain (versions, flags, defaults, edge-case behavior), look it up in the official Kubernetes documentation or the Kubernetes source repositories (e.g., `kubernetes/kubernetes`, `kubernetes/website`) instead of relying on memory.
