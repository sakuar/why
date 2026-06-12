# 单站构建规格 + 生成 Prompt 模板（核心话术）

> 这是生成**一个创意小站**的需求规格与可直接复用的 prompt。后续任务把 `{占位符}` 填上即可。

---

## 1. 硬性规格（每个站都必须满足）

1. **单文件自包含**：一个 `.html`，HTML/CSS/JS 全部内联；**严禁**引用任何外部 CDN、图片、字体、第三方库。断网可用。
2. **中文界面**，视觉现代精致：协调配色、圆角、留白、过渡/微动画。
3. **响应式**：窄屏（手机）也要可用。
4. **左上角返回入口**：`<a href="index.html">← 入口</a>`，样式低调不抢戏。
5. **交互完整**：真的能玩/能用，不是静态占位或半成品。需要持久化的用 `localStorage`。
6. **代码整洁**、关键处中文注释（适量）、无明显 bug。

## 2. 命名规范

- **slug**：小写英文 + 中划线，作为文件名 `{slug}.html`。例：`pixel-garden`、`game-2048`、`dinner-wheel`。
- 同一仓库内 slug 唯一。

## 3. 结构化输出 Schema（强制 agent 按此返回，便于自动落盘与建导航）

```json
{
  "type": "object",
  "additionalProperties": false,
  "properties": {
    "slug":  { "type": "string" },
    "title": { "type": "string" },
    "desc":  { "type": "string", "description": "12字以内简介" },
    "cat":   { "type": "string" },
    "html":  { "type": "string", "description": "完整HTML文件内容，从<!DOCTYPE html>到</html>" }
  },
  "required": ["slug", "title", "desc", "cat", "html"]
}
```

## 4. 生成 Prompt 模板（直接复用）

> 把 `{title}` `{cat}` `{spec}` `{slug}` 替换为具体值后，发给生成 agent（推荐 Sonnet）。
> **注意输出方式是"内容回传"**，不是写文件——原因见 AGENT-GUIDE 坑#1。

```text
你是资深前端工程师，请做一个精致、可直接运行的【纯前端单文件 HTML 小站】。

【主题】{title}（分类：{cat}）
【功能要求】{spec}

【硬性规格】
1. 单个 HTML 文件，HTML / CSS / JS 全部内联，严禁引用任何外部 CDN、图片、字体或第三方库（必须完全自包含、断网可用）。
2. 界面中文。视觉现代精致：协调配色、圆角、留白、过渡/微动画；窄屏（手机）也可用（响应式）。
3. 页面左上角放返回入口链接：<a href="index.html">← 入口</a>，样式低调。
4. 交互完整、真的能玩/能用，不能是占位或半成品。需要持久化用 localStorage。
5. 代码整洁，无明显 bug。

【输出方式】重要：
- 不要尝试写文件。把【完整的 HTML 文件内容】（从 <!DOCTYPE html> 到 </html> 的全部内容）作为结构化输出的 html 字段原样返回。
- 同时返回 slug="{slug}"、title、desc（12字内）、cat="{cat}"。

现在直接开始，不要提问。
```

## 5. `spec`（功能描述）写法建议

`spec` 要**具体到可执行**，包含：核心玩法/功能、关键交互、视觉氛围、是否持久化。示例：

> **晚饭转盘**：可编辑选项的幸运转盘（canvas 画彩色扇形），点击旋转有缓动减速动画，
> 指针停下弹出"今天就吃 XX"。可增删选项，localStorage 记住列表。

## 6. 落盘与导航（主控制流负责）

- 拿到 `html` 后：`Write` 到 `{目标目录}\{slug}.html`。
- 全部写完后**重建 `index.html`**：按 `cat` 分组列出所有站的卡片（标题 + desc + 链接 `{slug}.html`）。
- 仓库根目录保留空文件 `.nojekyll`。
