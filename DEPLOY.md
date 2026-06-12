# 部署作业书 · 静态站点 → GitHub Pages

> 这是一份**通用、可复制**的部署说明。任何「纯静态前端工程」（HTML/CSS/JS，无构建步骤）
> 把本文件连同 `.github/workflows/deploy.yml` 一起带过去，就能照着完成 GitHub Pages 部署。
> 内容来自真实踩坑总结，**给人看也给 AI agent 读取执行**。

---

## 0. 适用前提

- 纯静态站点：仓库根目录有 `index.html`，打开即用，**无需 npm build / 打包**。
- 部署目标：GitHub Pages，最终访问地址为 `https://<用户名>.github.io/<仓库名>/`。
- 部署方式：**GitHub Actions**（本文唯一推荐方式，已验证可用）。

---

## 1. 放置部署文件（复制即可）

### 1.1 `.github/workflows/deploy.yml`

```yaml
name: Deploy static site to GitHub Pages

on:
  push:
    branches: [main]
  workflow_dispatch:

permissions:
  contents: read
  pages: write
  id-token: write

concurrency:
  group: pages
  cancel-in-progress: true

jobs:
  deploy:
    runs-on: ubuntu-latest
    environment:
      name: github-pages
      url: ${{ steps.deployment.outputs.page_url }}
    steps:
      - uses: actions/checkout@v4
      - uses: actions/configure-pages@v5      # ⚠️ 不要加 enablement: true，见“坑 #1”
      - uses: actions/upload-pages-artifact@v3
        with:
          path: '.'                            # 整个仓库根目录作为站点；如有子目录改这里
      - id: deployment
        uses: actions/deploy-pages@v4
```

### 1.2 根目录加一个空文件 `.nojekyll`

防止 GitHub Pages 的 Jekyll 引擎处理你的文件（尤其是带下划线开头的文件/目录会被它忽略）。

---

## 2. 推送代码

```bash
git init -b main          # 已是 git 仓库则跳过
git add -A
git commit -m "init: deploy to pages"
git remote add origin https://github.com/<用户名>/<仓库名>.git   # 已有 origin 则跳过
git push -u origin main
```

---

## 3. ⭐ 仓库设置（**最关键的一步，90% 的失败都在这里**）

在浏览器打开仓库 → **Settings → Pages → Build and deployment**：

> **Source 必须选 `GitHub Actions`**

❌ 不要选 "Deploy from a branch"。
如果选了 "Deploy from a branch"，却又跑着上面的 Actions workflow，两者**不匹配**，站点会一直发布不出来（表现为持续 404）。

设置正确后，第 2 步的 push 会自动触发部署；之后每次 push 到 `main` 都会自动重新部署。

---

## 4. 验证是否成功

1. 仓库 **Actions 标签页** → 最新一次 "Deploy static site to GitHub Pages" 运行应为**绿色 ✅**。
2. 浏览器打开 `https://<用户名>.github.io/<仓库名>/`，应看到你的 `index.html`。
3. **首次部署有延迟**：构建 + CDN 生效通常需 1–3 分钟，期间出现 404 属正常，多刷新几次。

---

## 5. 踩坑清单（务必逐条核对）

| # | 坑 | 现象 | 解决 |
|---|----|------|------|
| 1 | `configure-pages` 加了 `enablement: true` | workflow 在该步报错失败 | **删掉这两行**。`enablement` 需要额外管理权限，是个无谓的失败点；Source 设成 "GitHub Actions" 后根本不需要它 |
| 2 | Source 设成 "Deploy from a branch" | 站点持续 404，Actions 跑了也没用 | 改成 **"GitHub Actions"**（见第 3 节） |
| 3 | 缺 `.nojekyll` | 下划线开头的文件/目录 404 | 根目录加空的 `.nojekyll` |
| 4 | 把首次构建延迟当成失败 | 刚 push 完访问 404 | 等 1–3 分钟，看 Actions 变绿再访问 |
| 5 | 子页面用了绝对路径 `/foo.html` | 子页面 404 | 项目站点有 `/<仓库名>/` 前缀，**一律用相对路径** `foo.html` |
| 6 | 用 API/token 想自动开 Pages | `POST /pages` 返回 403 | 普通仓库 token 没有 Pages 管理权限。**别走 API，用第 3 节手动设一次 Source 即可**（一次性，之后全自动） |

---

## 6. 给 AI agent 的执行检查清单

当被要求"部署某静态工程到 GitHub Pages"时，按此执行并**逐项自检**：

- [ ] 确认是纯静态站点（根目录有 `index.html`，无需 build）。
- [ ] 写入 `.github/workflows/deploy.yml`（用第 1.1 节原样内容，**不加 enablement**）。
- [ ] 根目录写入空 `.nojekyll`。
- [ ] `git add/commit/push` 到 `main`（`git push` 通常可用；`gh` CLI、`curl` 不一定可用）。
- [ ] **明确告知用户**：必须去 Settings → Pages → Source 手动选 "GitHub Actions"（这一步 agent 无法代劳，是成功前提）。
- [ ] 让用户在**自己的浏览器**确认最终 URL 可访问——**不要仅凭工具返回的 HTTP 结果就断言"已上线"**；以用户浏览器所见为准。
- [ ] 如失败：让用户打开 Actions 里失败 run 的**红色步骤报错文本**，据此定位（具体报错比泛泛的运行时状态更可靠）。

---

## 7. 最终地址速记

```
项目站点：https://<用户名>.github.io/<仓库名>/
用户主页站点（仓库名恰为 <用户名>.github.io 时）：https://<用户名>.github.io/
```
