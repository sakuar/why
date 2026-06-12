# 部署作业书 · 静态站点 → GitHub Pages（含完整故障诊断）

> 通用、可复制的部署 + 排错手册。**给人看也给 AI agent 读。**
> 部署目标：`https://<用户名>.github.io/<仓库名>/`
> 出现 404 / 发布不出来时，**别瞎试**，照第 5 节的决策树一层层定位。

---

## 0. 适用前提
- 纯静态站点：仓库根目录有 `index.html`，打开即用，**无需 npm build**。
- 部署方式：**GitHub Actions**（本文主线）。

## 1. 部署文件（复制即可）

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
      - uses: actions/configure-pages@v5      # 不要加 enablement: true
      - uses: actions/upload-pages-artifact@v3
        with:
          path: '.'
      - id: deployment
        uses: actions/deploy-pages@v4
```

### 1.2 根目录空文件 `.nojekyll`
防止 Jekyll 忽略下划线开头的文件/目录。

## 2. 推送代码
```bash
git add -A && git commit -m "deploy" && git push origin main
```

## 3. ⭐ 仓库设置（最关键）
**Settings → Pages → Build and deployment → Source 选 `GitHub Actions`。**
⚠️ Source 必须和部署方式一致，**两种模式不可混用**（见坑 B）。

## 4. 正常验证
1. Actions 标签页 → 最新 "Deploy static site" run 绿色 ✅。
2. 浏览器开 `https://<用户名>.github.io/<仓库名>/`。
3. 首次部署 + CDN 生效需 1–3 分钟，期间 404 属正常。

---

# 🔍 故障诊断（404 / 发布不出来）

## 5. 分层诊断决策树（按顺序走，别跳）

部署链路有 6 层，404 可能出在任何一层。**按下面三个问题逐层缩小范围：**

```
Q1 文件在远程吗? ──否──▶ 代码层问题      → 见 6.A
   │是
   ▼
Q2 Pages 部署成功吗? ──否──▶ Actions/账号层 → 见 6.C / 6.D
   │是(横幅绿色 live)
   ▼
Q3 首页能开、只是部分页 404? ──是──▶ 缓存/路径层 → 见 6.E / 6.F
```

### Q1 · 文件真的在远程吗？（先排除代码层）
- **查**：浏览器开 `github.com/<用户名>/<仓库名>`，直接看文件在不在；
  或命令 `git ls-tree -r --name-only origin/main`（先 `git fetch`）。
- 文件**缺** → 代码没推上去，去 **6.A**。
- 文件**都在** → 代码没问题，往 Q2。

### Q2 · Pages 部署成功了吗？
- **查横幅**：Settings → Pages 顶部：
  - 🟢 `Your site is live at …`（带时间）→ 已部署成功，去 **Q3**。
  - ⏳ 转圈 / `ready to be published` / 空白 → **没部署成功**，去 Actions 排查 **6.C / 6.D**。
  - 🔴 红字报错 → 直接读报错，去 **6.C**。
- **查 Actions**：Actions 标签页最新 "Deploy static site" run 是什么状态？（绿✅/红✗/一直 queued）

### Q3 · 站点能开但部分页面 404？
→ 多半是 **缓存(6.E)**、**路径(6.F)**，或部署的是**旧版本(6.C 并发)**。

---

## 6. 各层根因详解

### 6.A 代码层 · 文件没到远程
**现象**：github 网页上看不到文件 / 新文件大面积缺失。
- `git push` 真成功了吗？**看输出里有没有 `xx..yy main -> main`（ref 真前进），别只看退出码。**
- commit 是否真的包含了文件？`git ls-tree -r origin/main` 与本地核对。
- **解决**：`git add -A && git commit && git push`，再用 `git ls-tree -r origin/main` 确认远程有了。

### 6.B 配置层 · Source 设置不对 / 两模式混用
- Source 必须与部署方式**配套**：
  - 仓库里有 `deploy.yml`(Actions) → Source = **GitHub Actions**。
  - 没有 workflow、想直接发布分支 → Source = **Deploy from a branch** → main / root，**并删掉 deploy.yml**。
- ❌ **最常见的坑**：Source 设成 "Deploy from a branch" 却又留着 Actions workflow（或反之）→ 两者打架，**持续 404**。
- **查**：Settings → Pages → Build and deployment → Source。

### 6.C 运行层 · Actions workflow 失败
Actions 标签 → 点开最新 run，看**哪一步红✗**，对照：

| 失败的步骤 / 报错 | 根因 | 解决 |
|---|---|---|
| `configure-pages` 失败 | Pages 没启用 / Source 没设成 GitHub Actions | 先去 Settings→Pages 把 Source 设为 GitHub Actions |
| `deploy-pages` 报 `Failed to create deployment (status: 404)` | 同上，Pages 未正确启用 | 设 Source = GitHub Actions 后重跑 |
| `deploy-pages` 报 `403 / Permission` | workflow 缺权限 | 确认 deploy.yml 有 `permissions: pages: write, id-token: write` |
| `upload-pages-artifact` 失败 | `path` 不对或目录为空 | 确认 `path: '.'` 且根目录有文件 |
| run 显示 **cancelled** | 被下一次 push 取消（concurrency `cancel-in-progress`） | 频繁 push 所致；**停手，等最后一次 push 的 run 自己跑完** |

### 6.D 账号/组织层 · run 一直 queued、0 个 job（最隐蔽，重点！）
**现象**：最新 run 一直 `Queued`（排队）几分钟不动，点进去**一个 job 都没有**。
这**不是你代码的问题** —— 是 GitHub 没给它分配 runner。逐条排查：

1. **邮箱未验证** → Settings → Emails，未验证的账号 Actions 不分配 runner。验证后重新触发（`workflow_dispatch` 或推个空 commit）。
2. **Actions 被禁用** → 仓库 Settings → Actions → General → 确认选了 "Allow all actions and reusable workflows"。
3. **Actions 额度用尽** → Settings → Billing → 看 Actions 用量。私有仓库每月有免费分钟数，用完会卡（公开仓库通常免费）。
4. **新账号 / 风控限制** → 刚注册或被标记的账号，Actions 可能被暂时限制，需要时间或联系 GitHub 支持。
5. **Spending limit = 0** 且触发计费时会卡住。

> **一招判断是不是账号层**：去 Actions 标签页看——
> - **所有** workflow（不只 Pages）都跑不起来 / 都 queued → 基本就是账号层（上面 1–5）。
> - **只有** Pages 这个不行、别的 workflow 能正常跑 → 不是账号层，回 6.B / 6.C。

### 6.E 缓存 / CDN 层
**现象**：横幅显示已 live，但浏览器还是旧内容 / 404。
- 硬刷新 `Ctrl+Shift+R`，或开**无痕窗口**。
- Pages 有 CDN 缓存，新部署后等几分钟再看。
- 核对 URL 没拼错（**区分大小写**）。

### 6.F 路径层
**现象**：首页能开，子页面或资源 404。
- 项目站 URL 带 `/<仓库名>/` 前缀，页面里链接一律用**相对路径** `foo.html`，**不要** `/foo.html`。
- 文件名大小写要和实际**完全一致**（Pages 区分大小写）。

---

## 7. 现象 → 根因 → 解决 速查表

| 现象 | 最可能根因 | 去看 |
|------|-----------|------|
| github 网页都看不到文件 | push 没成功 | 6.A |
| 文件在远程，但全站 404，横幅非绿 | Source 设错 / 没设 | 6.B |
| Actions run 一直 queued、0 job | 账号层（邮箱/额度/禁用） | 6.D |
| Actions run 红✗ | 某步失败，读报错 | 6.C |
| Actions 绿✅ 但仍 404 | 缓存 / 路径 / Source 不匹配 | 6.E / 6.F / 6.B |
| 首页能开、子页 404 | 路径 / 大小写 / 缓存 | 6.F / 6.E |
| 改了好几次才推，部署的是旧版 | 并发取消 | 6.C(cancelled 行) |

## 8. 一分钟自检清单
- [ ] github 网页能看到文件？（否→6.A）
- [ ] Settings→Pages 的 Source 和部署方式一致？（否→6.B）
- [ ] Settings→Pages 横幅是绿色 live？
- [ ] Actions 最新 run：绿✅ / 红✗ / queued不动0job？
      - queued不动 → 6.D 账号层
      - 红✗ → 6.C 读报错
- [ ] 硬刷新 / 无痕试过？（→6.E）
- [ ] 子页面用相对路径、大小写对？（→6.F）

## 9. 给 AI agent 的执行 + 诊断清单
**执行部署时**：
- [ ] 确认纯静态站；写入 `deploy.yml`（不加 enablement）+ 空 `.nojekyll`；push 到 main。
- [ ] **明确告知用户**手动设 Settings→Pages→Source = GitHub Actions（agent 无法代劳）。
- [ ] 让用户在自己浏览器确认；**不要仅凭工具返回的 HTTP 结果就断言已上线**。

**诊断 404 时**：严格按第 5 节决策树，先分清「代码层 / 配置层 / 运行层 / 账号层 / 缓存层 / 路径层」，再对症。**不要在没定位到层之前乱改设置**（尤其别在 Actions 与分支部署之间反复横跳，那本身就会造成 6.B 的冲突 404）。