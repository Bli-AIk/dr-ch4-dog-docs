# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## 项目概览

《从零做一个 Deltarune 同人游戏》Kristal 教程的文档仓库（mdbook 发布）。**文章写作流程与风格规范见 [agents.md](agents.md)——开始写作前必读。**

## 结构

- `src/`：mdbook 书源。`src/SUMMARY.md` 是目录，`src/文章/` 下的 markdown 是正文（序章 / 第一章-Dog-Battle / 番外 三个子目录），`src/课程总览.md` 是书首页。
- `00-序章/`、`01-第一章-Dog-Battle/`、`05-番外/`：大纲（规划文档，不进书）。
- `book.toml`：mdbook 配置（标题 / 语言 / 仓库链接）。
- `.github/workflows/pages.yml`：push 到 main 自动构建并部署 GitHub Pages。

## 常用命令

- 本地构建：`mdbook build`（产物在 `book/`，已 gitignore）
- 本地预览：`mdbook serve`（http://localhost:3000）
- 发布：推送 main 即可，CI 自动部署到 https://bli-aik.github.io/dr-ch4-dog-docs/

## 关键约束（踩过的坑）

- **SUMMARY.md 链接路径不能含空格**：mdbook 的 SUMMARY 解析器不支持空格路径，必须用 `%20` 编码（例如 `文章/序章/第%2000%20集%20-%20欢迎来到%20Kristal.md`）。文件名本身保留空格。
- 书只包含 `src/` 下的内容；仓库根目录的大纲、agents.md、CLAUDE.md 不会进书。
- 本仓库是 dr-ch4-dog 模组仓库的 docs 子模块；模组代码与引擎源码在父仓库那边，写文章需要核实代码事实时去那里看。
