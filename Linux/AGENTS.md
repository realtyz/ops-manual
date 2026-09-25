# Repository Guidelines

This directory is an Obsidian vault of Chinese-language Linux/SRE notes. It contains Markdown only: no source code, build system, or automated tests. `写作规约.md` is authoritative for structure, prose, and review; where this file and that spec disagree, the spec wins.

## Project Structure & Module Organization

- `00_引言.md` — vault learning map.
- `写作规约.md` — layer definitions, article skeletons, and the final self-check list.
- `NN_主题/` — 12 stage directories (for example `04_内存管理与OOM/`), each with `00_导读与知识地图.md` plus articles named `NN_层_关键词.md`, where 层 is `原理`, `体系`, `实操`, `判例`, or `治理`.
- Sibling vaults `Kubernetes/` and `AI-Infra/` are referenced, never duplicated.
- Scratch scripts, raw command output, and draft reports are never committed.

## Build, Test, and Development Commands

There is no build step; authoring happens in Obsidian or any Markdown editor. Useful checks:

```powershell
rg --files -g "*.md"      # inventory Markdown files
rg -n "\[\[" -g "*.md" .  # wikilinks; navigation pages only
git diff --check          # whitespace errors
```

After adding, renaming, or reordering articles, sync `00_引言.md`, the root `README.md` counts and tables, and cross-vault references.

## Coding Style & Naming Conventions

- File names use `NN_层_关键词.md`; narration belongs in section headings.
- Use one H1 per file and Obsidian frontmatter (`tags`, `created`) in content pages.
- Write Chinese prose in an academic-report register: complete sentences with explicit connectors (因此, 由于, 若, 反之), not noun-phrase telegraphs.
- Avoid wikilinks outside navigation pages, `→` in place of 若……则……, level codes such as `L1`/`M1`, and metaphors in headings or table cells.
- Units are MiB, GiB, and seconds. Read defaults and version differences on the target machine (kernel 6.x baseline), never from memory.

## Testing Guidelines

There is no test framework. Verification is the manual self-check in `写作规约.md` §10 plus each article's `要点自测` and 「第一反应不要做什么」 note. For every field cited, state what it is, its unit, the criterion, what to read alongside it, and the next step. Mark destructive operations experiment-machine-only and give a rollback path.

## Commit & Pull Request Guidelines

Git history follows Conventional Commits with a Chinese subject, for example `docs: 重写阶段 4-6 为原理+体系+实操+判例+治理结构` and `fix: 07 治理篇交叉指向改为判例篇的真实小标题`. Keep one logical change per commit and explain structural or reference-sync work in the body.

Pull requests should state scope, list affected chapter paths, link related issues, and include before/after excerpts or screenshots for prose and table changes. Confirm README counts and cross-chapter terminology before review.
