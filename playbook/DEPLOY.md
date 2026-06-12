# 部署作业书 · 静态站点 → GitHub Pages

> 适用于任何「纯静态前端工程」（HTML/CSS/JS，无构建）。已验证可用。

---

## 0. 前提
- 仓库根目录有 `index.html`，打开即用，无需 npm build。
- 访问地址：`https://<用户名>.github.io/<仓库名>/`。
- 部署方式：**GitHub Actions**（唯一推荐）。

## 1. 放置文件
- `.github/workflows/deploy.yml`（用 [`templates/deploy.yml`](templates/deploy.yml) 原样内容）。
- 根目录加空文件 `.nojekyll`（防止 Jekyll 忽略下划线开头文件）。

## 2. 推送
```bash
git init -b main            # 已是 git 仓库则跳过
git add -A
git commit -m "init"
git remote add origin https://github.com/<用户名>/<仓库名>.git
git push -u origin main
```

## 3. ⭐ 仓库设置（最关键，90% 失败在这）
**Settings → Pages → Build and deployment → Source 选 `GitHub Actions`。**
❌ 不要选 "Deploy from a branch"——它与 Actions workflow 不匹配，会导致站点持续 404。

## 4. 验证
1. Actions 标签页 → 最新 run 变**绿 ✅**。
2. 浏览器访问 `https://<用户名>.github.io/<仓库名>/`。
3. 首次部署 + CDN 生效需 1–3 分钟，期间 404 属正常，多刷新。

## 5. 踩坑表
| # | 坑 | 现象 | 解决 |
|---|----|------|------|
| 1 | `configure-pages` 加了 `enablement: true` | 该步报错失败 | 删掉，Source 设对后不需要它 |
| 2 | Source 选了 "Deploy from a branch" | 持续 404 | 改成 "GitHub Actions" |
| 3 | 缺 `.nojekyll` | 下划线文件 404 | 根目录加空 `.nojekyll` |
| 4 | 把首次延迟当失败 | 刚 push 就 404 | 等 1–3 分钟看 Actions 变绿 |
| 5 | 子页面用绝对路径 `/foo.html` | 子页 404 | 项目站有 `/<仓库名>/` 前缀，**一律相对路径** `foo.html` |
| 6 | 想用 API/token 自动开 Pages | `POST /pages` 返回 403 | 普通仓库 token 无 Pages 管理权限，手动设一次 Source 即可 |

## 6. 给 AI agent
- 写好 `deploy.yml`（**不加 enablement**）+ `.nojekyll`，push。
- **明确告知用户**：必须手动把 Settings→Pages→Source 设为 "GitHub Actions"（agent 无法代劳）。
- 让用户在**自己浏览器**确认 URL；**不要仅凭工具返回就断言已上线**（见 AGENT-GUIDE 验证纪律）。
- 失败时让用户贴 Actions 里失败 run 的**红色步骤报错文本**，据此定位。
