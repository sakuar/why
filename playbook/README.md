# 创意小站工厂 · 操作手册（Playbook）

> 一个**可复用的需求与流程手册**：把"批量生产纯前端单文件创意小站，并部署到 GitHub Pages"
> 这件事的**需求模板（话术）、构建规格、部署流程、避坑经验**全部固化在这里。
>
> **后续任何任务（人或 AI agent）只要读完本项目，就知道完整怎么做。**

---

## 这是什么

我们做的"产品"是一批**小众创意小站**：每个都是一个**自包含的单文件 HTML**
（HTML/CSS/JS 全内联、无任何外部依赖、断网可用），双击即开，再统一部署到 GitHub Pages。

本手册沉淀了做这件事的全部 know-how，**避免后续任务重复踩坑**。

## 怎么用（阅读顺序）

| 顺序 | 文件 | 作用 |
|------|------|------|
| ① 必读 | [`AGENT-GUIDE.md`](AGENT-GUIDE.md) | **给执行者（尤其 AI agent）的总指南**：完整流程 + 本环境的关键坑 + 验证纪律 |
| ② | [`BUILD-SPEC.md`](BUILD-SPEC.md) | **单站构建规格 + 可直接复用的生成 prompt 模板（核心话术）** |
| ③ | [`IDEAS.md`](IDEAS.md) | 创意池：已完成清单 + 100 个待做点子（分类） |
| ④ | [`DEPLOY.md`](DEPLOY.md) | GitHub Pages 部署作业书（已验证可用） |
| 模板 | [`templates/deploy.yml`](templates/deploy.yml) | 可直接复制的部署 workflow |

## 一句话流程

```
选创意(IDEAS) → 按规格与话术生成单文件HTML(BUILD-SPEC) → 可靠落盘 →
更新导航首页 index.html → 部署到 Pages(DEPLOY) → 用 git/远程验证真实上线(AGENT-GUIDE)
```

## 黄金法则（血泪教训，详见 AGENT-GUIDE）

1. **落盘靠主循环**：批量生成时，让生成 agent **返回 HTML 内容**，由主控制流用 `Write` 写盘——
   不要依赖子 agent 自己写文件（在隔离环境里可能不落到真实目录）。
2. **真相靠 git/远程**：本类环境的工具输出可能被污染/伪造，**不要凭工具回显就宣布成功**；
   以 `git` 的文件统计、`git push` 到远程、以及**用户浏览器/GitHub 网页所见**为准。
3. **部署 Source 必须选 `GitHub Actions`**，且 `configure-pages` **不要加 `enablement: true`**。
