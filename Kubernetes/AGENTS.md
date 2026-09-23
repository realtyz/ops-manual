# Repository Guidelines

This repository is an Obsidian Markdown knowledge base for a production-grade Kubernetes learning roadmap. It is not a runnable code project: there is no build system, test suite, or package manifest.

## Project Structure & Module Organization

- `00_简介.md` is the roadmap "map"; it links to per-stage notes and is the single source of truth for the learning path.
- Stage notes use numbered filenames: `NN_Topic.md` (e.g., `01_核心架构与对象模型.md`, `03_调度_资源_QoS.md`).
- **Stage 0 is a directory chapter: `00_容器技术/`.** It is the opening chapter and covers the container technology Kubernetes depends on. It follows the layered structure of the sibling `Linux/` knowledge base:
  - `00_导读与知识地图.md` — the navigation note (knowledge map, term index, symptom-locator table, note index, references);
  - `NN_层_关键词.md` — body notes, where the prefix is the layer: `原理_` / `体系_` / `实操_` / `判例_` / `治理_`;
  - current composition: 原理 01–04 · 体系 05 · 实操 06 · 判例 07 · 治理 08 = 9 files.
- `00_前置与实验环境.md` was **removed** when stage 0 became the container chapter. Its container-runtime content is superseded by `00_容器技术/`; the local-cluster tooling (kind/minikube/k3d), kubectl basics and kubeconfig material was **not** carried over. The file itself is still recoverable from git history (`git show <commit>:Kubernetes/00_前置与实验环境.md`).
- Every note starts with YAML frontmatter containing `tags` and `created` (ISO date), followed by exactly one `#` title.

## Coding Style & Naming Conventions

- Markdown, UTF-8, LF line endings.
- Headings: `#` once per note, `##` for sections, `###` for subsections.
- Note bodies are written in Chinese (zh-CN). Keep standard English technical terms inline where appropriate (e.g., `Service`, `etcd`).
- **Two writing conventions currently coexist in this directory:**
  - **`00_容器技术/`** follows `Linux/写作规约.md`, which is the authoritative style guide for these notes. Navigation note: Obsidian wikilinks are allowed and expected. Body notes: **no wikilinks** — reference other notes in plain prose (use 《篇名》 for notes inside the same chapter). **No callouts** (`> [!type]`); blockquotes are only for quoted reasoning that the style guide permits. Every body note ends with `上一篇：X ｜ 下一篇：Y`, and every note carries `要点自测` plus a 「第一反应不要做什么」 boundary line.
  - **Stages 1–9** keep the older convention: `> [!type]` callouts and wikilinks are used throughout.
- Wikilink link text must match the target note name exactly. Use **full vault paths** for navigation notes, because `00_导读与知识地图` exists in every `Linux/` chapter (e.g. `[[Kubernetes/00_容器技术/00_导读与知识地图|00 容器技术]]`).
- Use `- [ ]` checklists and pipe-aligned tables in roadmap/outline notes. Do not use horizontal rules inside body notes.

## Content Quality Guidelines

- Stages 1–9 follow the three-layer structure: 概念 → 生产实践 → 要点自测.
- `00_容器技术/` follows the five-layer structure in `Linux/写作规约.md`: 原理（为什么存在、代价落在哪里）· 体系（一条时间主线 + 横切线）· 实操（用哪把工具、怎么读）· 判例（现象 → 误判 → 判据 → 出处）· 治理（制度、产物模板、行业做法）. Layers must not duplicate each other's job.
- Every wikilink must resolve to an existing note or a note intentionally created in the same change.
- Commands, versions, and behavior claims should be verifiable against the official documentation. In stage 0 the primary sources are the OCI Image/Distribution/Runtime specs, the Kubernetes documentation, the upstream CRI API, and the official containerd/CRI-O docs.
- **State version gates explicitly.** Container runtimes and Kubernetes both move configuration keys and defaults across major/minor versions; a claim without a version is not acceptable in stage 0.
- No linters or automated checks are configured; review changes manually before committing.

## Commit & Pull Request Guidelines

- Use Conventional Commits: `docs: add 00_容器技术 篇章`, `docs: update 01_核心架构与对象模型`, `chore: add AGENTS.md`.
- One logical note change per commit; keep the summary short and specific.
- Before merging, confirm wikilinks resolve and that `00_简介.md`'s checklist, roadmap rows and links match any new, renamed or removed notes.

## Agent-Specific Instructions

- **Read `Linux/写作规约.md` and `Kubernetes/00_容器技术/00_导读与知识地图.md` before touching stage 0.** The style guide governs body-note skeletons, the forbidden constructs, and the pre-commit self-check list.
- **After changing the composition of `00_容器技术/`, sync four places:** ① the note list in this file; ② `00_简介.md` (roadmap row 0, the 阶段 0 section, and the note-organisation list); ③ the chapter's `00_导读与知识地图.md` (note index and the adjacent `上一篇 / 下一篇` navigation lines); ④ the vault-root `README.md` counts.
- Do not renumber existing notes (stages 1–9) without updating the whole link graph in `00_简介.md`.
- **Do not write temporary research, evidence dumps or fact sheets into this directory.** Notes are the only artifacts that belong here; use a scratch location outside the vault and delete it when done.
- A full refactor of stages 1–9 onto the layered structure is planned. Do not start it as a side effect of an unrelated change.
- When a knowledge point is uncertain (versions, flags, defaults, edge-case behavior), look it up in the official documentation or the upstream source repositories (e.g. `kubernetes/kubernetes`, `kubernetes/website`, `containerd/containerd`, `opencontainers/*`) instead of relying on memory. Mark anything unconfirmed as 「未确认」 rather than guessing.
