# Repository Guidelines

本仓库是 Obsidian 中的 Linux 运维学习笔记库，不是代码工程：没有构建系统、依赖管理与自动化测试。贡献即「写作」，请优先保证笔记准确、可复现、可交叉引用。

## Project Structure & Module Organization

- `00_简介.md`：学习路线图，定义阶段编号、周期、产出目标与各阶段 checklist。
- `99_Linux笔记模板.md`：新笔记的唯一模板，包含 frontmatter、章节骨架与自检清单。
- `NN_主题.md`：正式笔记，`NN` 与路线图中的阶段编号一致（如 `01_启动流程与内核.md`）。
- 图片、附件等资源与本笔记同目录存放，文件名避免中文空格与特殊字符。

## Build, Test, and Development Commands

- 预览：用 Obsidian 打开本目录，确认 Mermaid 图、callout、wikilink 正常渲染。
- 列出全部笔记：`rg --files -g '*.md'`。
- 检查 frontmatter：`rg -n '^(tags|created):' *.md`，每篇须同时含有 `tags` 与 `created`。
- 检查标题层级：`rg -n '^#' *.md`，每篇只允许一个一级标题。
- 交叉引用自查：`rg -n '\[\[' *.md`，逐个确认链接指向存在的笔记。

## Coding Style & Naming Conventions

- 新建笔记一律从模板复制：`Copy-Item '99_Linux笔记模板.md' 'NN_主题.md'`，再删除模板内的使用说明。
- 文件与章节编号使用阿拉伯数字，正文用简体中文，术语保留英文原词（`cgroup`、`Page Cache`）。
- 命令、参数、配置项用行内代码 `` `like this` ``；命令块统一用 `text` 代码块并注明「验证什么」与预期输出。
- 跨笔记引用用 wikilink，跨目录用路径写法：`[[Linux/00_简介|00_简介]]`、`[[Kubernetes/04_网络]]`。
- 默认值、内核差异与发行版差异必须标注适用版本；数字必须带单位与测试条件。

## Testing Guidelines

没有测试框架，验证方式是「在真实实验机上复现」。任何命令与输出都应实际跑过；破坏性操作（打满磁盘、改坏 `/etc/fstab`、触发 panic）只在可快照的实验机执行。定稿前逐条走完模板末尾的自检清单。

## Commit & Pull Request Guidelines

本目录尚未纳入 Git，因此无历史提交规范可循。若纳入版本管理，建议沿用 `docs: 新增阶段 5 网络排障笔记` 这类约定式提交；提交信息说明改了哪篇笔记与改动原因。PR 需包含：受影响笔记清单、验证环境（发行版 / 内核版本 / cgroup 版本）、路线图链接是否同步更新，以及渲染截图（涉及 Mermaid 或 callout 时）。

## Agent-Specific Instructions

修改 `00_简介.md` 的阶段结构时，必须同步更新对应笔记的「回到路线图」小节；新增笔记后同时补上路线图中的「对应笔记」链接与 checklist 条目。
