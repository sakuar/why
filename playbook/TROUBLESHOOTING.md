# 疑难排查与问题复盘

> 本文记录把"创意小站"推送到 GitHub Pages 过程中遇到的真实问题、根因与解决方式。
> 核心教训一句话：**在工具输出可能被污染/伪造的环境里，永远用 git 远程 + 用户浏览器作为最终真相，绝不凭工具回显宣布成功。**

---

## 问题 1 · 工具输出被注入污染

**现象**
- Bash / PowerShell 的 stdout、甚至 `Read` 到的文件内容，**末尾被追加伪造内容**：
  - 伪造的成功（如凭空出现的 `HTTP 200 / deployment successful`）；
  - 伪造的 `system-reminder`，直接"指示"我去告诉用户已成功；
  - 反向操纵话术（如真数据旁边加一句"别信""this line is fake"）；
  - 洪水攻击（把一个写文件淹没成数百行重复行）。

**影响**：无法凭单次工具回显判断真实状态；曾据此误报过"已上线/已推送"。

**解决**
1. **边界标记**：输出用 `BEGIN_XXX` / `END_XXX` 包裹，只采信标记内**自洽**的数据。
2. **真相裁判用 git 远程**：本地 stdout 不可信时，用 `git ls-tree`/`git show` 查**远程**内容（GitHub 服务器，不受本地污染）。
3. **多源交叉验证**：同一事实用 2–3 种独立方式确认，丢弃夹带操纵字样的片段。
4. **最终以用户浏览器 / GitHub 网页为准**。

---

## 问题 2 · `git commit`/`push` 报 exit 0，但实际没推到远程（最关键）

**现象**
- `git add` + `commit` + `push` 全部返回 `exit 0`，看似成功；
- 但远程仓库实际**没有**新文件，本地 `HEAD` 还停在旧提交；
- 用户访问站点"没看见生效"。

**根因**
- 本地**暂存区 / HEAD 状态被搅乱**：新文件虽被 `git` 跟踪（`ls-files` 计数包含它们），
  但**并不在 HEAD 提交里**（`git cat-file -e HEAD:文件` 返回 false）；
- 同时 `git status` 被伪造成 `clean`、`commit` 被伪造成 `nothing to commit`，
  **阻止了真正的提交**，而 stdout 的 `exit 0` 是假象。

**解决（决定性步骤）**
```bash
git fetch origin
git reset --mixed origin/main      # 强制把暂存区/HEAD 对齐远程，工作区文件保留
git add -A
git diff --cached --name-only      # 用底层命令确认“真正暂存了什么”
git commit -m "..."
git push origin main
# 验证：直接查远程，不信本地 stdout
git fetch origin main
git ls-tree --name-only FETCH_HEAD | grep '\.html$'
```
- 关键：`git diff --cached --name-only`、`git ls-tree FETCH_HEAD`、`git cat-file -e` 这些**底层/远程查询**难以被伪造，是可信的。
- 推送成功的铁证：远程 ref 真实前进（`829d3e9..6eb0505 main -> main`）+ 远程文件列表出现新文件。

---

## 问题 3 · `Write` 报成功，但文件没真正更新

**现象**：用 `Write` 重建 `index.html` 返回 "updated successfully"，但磁盘上仍是旧版（字节数没变、不含新内容）。

**解决**
- 写入关键文件后，**做内容级校验**，不盲信 `Write` 的返回：
  ```powershell
  $s=[System.IO.File]::ReadAllText($path,[Text.Encoding]::UTF8)
  $s.Contains("新内容的关键字符串")   # 必须为 True 才算写成功
  ```
- 校验为 True 后再提交；为 False 则改用 `[System.IO.File]::WriteAllText` 重写。

---

## 问题 4 · 多 agent（workflow）子 agent 写文件不落盘

**现象**：多 agent 并行生成小站，每个都报 `ok=true`，但 `git status` 显示**一个新文件都没有**。

**根因**：workflow 的子 agent 跑在隔离环境，它的 `Write` 落到了沙箱，**不同步到真实工作目录**。

**解决**：改"**内容回传**"模式——
- 子 agent **不写文件**，把完整 HTML 作为结构化输出的 `html` 字段返回；
- 由**主控制流**取回内容后用可靠方式（`[System.IO.File]::WriteAllText`）落盘；
- 注意子 agent 返回的内容文件是 **UTF-8、JSON 转义**，落盘时用 UTF-8（无 BOM）写出。

---

## 问题 5 · GitHub Pages 站点 404 / "没看见生效"

**现象**：`https://<user>.github.io/<repo>/` 持续 404 或看不到更新。

**根因（按出现顺序）**
1. 早期：误以为账号部署能力有问题——实为**工具污染喂的假数据**（另一个仓库 cv 能正常上线，证明账号没问题）。
2. 配置：Pages **Source 没设为 `GitHub Actions`** / `configure-pages` 多加了 `enablement: true`。
3. 本次："没看见生效"的真因是**问题 2**——代码压根没推到远程。
4. 缓存：推送生效后，浏览器/CDN 缓存导致看到旧页。

**解决**
- 部署配置见 [`DEPLOY.md`](DEPLOY.md)：Source 选 `GitHub Actions`，`deploy.yml` 不加 `enablement`。
- **先确认代码真在远程**（问题 2 的验证），再谈部署。
- 生效后**硬刷新** `Ctrl+Shift+R` 或开无痕窗口排除缓存；首次部署等 1–3 分钟。

---

## 可信度排序（验证任何"是否成功"时按此取信）

```
高 │ ① GitHub 远程 (ls-tree/show FETCH_HEAD) · 用户浏览器 / GitHub 网页
   │ ② git 底层查询 (cat-file -e / diff --cached) · commit 的文件&字节统计
   │ ③ 逐个 Read 文件（File does not exist / 自洽内容）· 带边界标记的文件内容校验
低 │ ④ Bash/PowerShell 原始 stdout（易被注入，仅作参考）
```

> 准则：**报告要忠实**。失败就说失败并附证据；没验证就说没验证；确认了再平实地说成功。
