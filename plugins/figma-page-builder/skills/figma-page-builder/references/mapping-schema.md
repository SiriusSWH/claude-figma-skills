# mapping.json 结构与用法

存放位置：`<demo-root>/mapping.json`，纳入 git，是 skill 的**基线**。

## 完整结构

```json
{
  "$schema": "figma-skills v0.1",
  "demoRoot": ".",
  "figmaFile": {
    "fileKey": "tzwh6TIG1iiSbz3LA8spTi",
    "componentLibraryPageId": "930:2",
    "name": "组件库 · Web 端"
  },
  "iconLibrary": {
    "kind": "lucide",
    "version": "0.395.0",
    "strokeWidth": 1.75
  },
  "tokens": {
    "color/accent": {
      "css": { "var": "--accent", "value": "#007AFF" },
      "figma": { "varCollection": "Brand", "var": "Primary", "value": "#007AFF" }
    },
    "color/text": {
      "css": { "var": "--text", "value": "#1d1d1f" },
      "figma": { "varCollection": "Ink", "var": "Ink1", "value": "#1d1d1f" }
    },
    "spacing/sidebar-row-padding": {
      "css": { "selector": ".nb-row", "prop": "padding", "value": "0 8px" },
      "figma": { "component": "Sidebar nav tree / nb-row", "prop": "padding", "value": "0 8" }
    },
    "typography/note-title": {
      "css": { "selector": ".note-title", "fontSize": "13px", "fontWeight": 500, "lineHeight": null },
      "figma": { "textStyle": "Note / Title", "fontSize": 13, "fontWeight": "Medium" }
    }
  },
  "components": {
    "Note card": {
      "css": ".note-card",
      "figma": { "id": "924:33", "type": "COMPONENT_SET" },
      "html_tag": "div",
      "variants_mode": "css_modifier",
      "variants": {
        "default": { "css": "", "figma": "State=default" },
        "hover": { "css": ":hover", "figma": "State=hover" },
        "selected": { "css": ".selected", "figma": "State=selected" },
        "batch-unchecked": { "css": ".batch-mode &", "figma": "State=batch-unchecked" },
        "batch-checked": { "css": ".batch-mode &.batch-checked", "figma": "State=batch-checked" }
      }
    },
    "Sidebar tb button": {
      "css": ".tb",
      "figma": null,
      "html_tag": "button",
      "variants_mode": "css_modifier",
      "variants": {
        "default": { "css": "", "figma": null },
        "hover": { "css": ":hover", "figma": null },
        "active": { "css": ".active", "figma": null }
      },
      "note": "tb 在 Figma 里目前没有独立 Component，是各 toolbar 里直接拼的;Phase 2 后续要不要提"
    }
  },
  "stateMap": {
    "state-01-normal": {
      "label": "普通态（基础页）",
      "figmaFrameId": "957:630",
      "demoUrl": "http://localhost:5173/",
      "demoStateScript": null,
      "lastVerified": "2026-05-14T08:30:00Z"
    },
    "state-02-batch-empty": {
      "label": "进入批量模式 0/200",
      "figmaFrameId": null,
      "demoUrl": "http://localhost:5173/?batch=on",
      "demoStateScript": "state.batchMode = true; state.selectedNoteIds.clear();",
      "lastVerified": null
    }
  },
  "acceptedDiffs": [
    {
      "id": "diff-2026-05-13-001",
      "where": "spacing/sidebar-row-padding",
      "demo": "0 8",
      "figma": "0 12",
      "reason": "Figma 设计师手动调过，比 demo 更宽松，移动端用",
      "approvedBy": "user",
      "approvedAt": "2026-05-13T14:20:00Z"
    }
  ],
  "lastSync": {
    "timestamp": "2026-05-14T08:30:00Z",
    "direction": "demo-to-figma",
    "changes": {
      "tokensUpdated": 4,
      "componentsUpdated": 2,
      "statesAdded": 1
    }
  },
  "ignore": {
    "cssFiles": ["styles.css"],
    "ignoredSelectors": [".pdf-render-stage *"],
    "ignoredFigmaPages": ["移动端", "PC端"]
  }
}
```

## 字段说明

### `figmaFile`
锁定 Figma 文件 + 组件库 page。skill 启动时校验文件 key 一致，否则报错避免误改。

### `iconLibrary`
icon 库声明。Phase 2/5 创建图标时严格按此使用。

### `tokens`
扁平 map，key 用 `<category>/<name>` 格式。每个条目记录双边的"位置 + 值"。
- `css.var`：CSS 变量名（在 `:root` 里）
- `css.selector + prop`：直接定义在某 class 上的属性
- `figma.varCollection + var`：Figma Variables 路径
- `figma.textStyle` / `figma.paintStyle`：Figma Styles 路径
- `figma.component + prop`：组件内某个属性（不是顶层变量，是组件硬编码值）

### `components`
- `css`：CSS class 选择器（不带 `:hover` 之类修饰）
- `figma`：Figma 节点 id + type（如果还没建，写 null）
- `html_tag`：决定 Figma→HTML 写回时用什么标签（**很重要**，见 `element-mapping.md`）
- `variants_mode`：
  - `"css_modifier"`：变体在 HTML 里是同一元素加 class（默认）
  - `"separate_elements"`：变体是不同 HTML 元素（少见，如 List header normal vs batch 模式）
- `variants`：每个变体的 CSS 修饰类与 Figma variant property

### `stateMap`
demo 可达 UI 状态 → Figma frame。`demoStateScript` 是把 demo 切到该状态的 JS 片段（用户提供或 skill 推断），方便每次同步前能"跳到这个状态截图"。

### `acceptedDiffs`
两边值不一致但用户主动放行的项。skill 每次跑都会跳过这些。如果值再变 → 重新触发确认（acceptedDiffs 失效，因为基线变了）。

### `lastSync`
最近一次同步的元数据。用于冲突检测：如果某条 token 现在的 demo 值 / Figma 值跟 `lastSync` 时记录的不一样 → 该侧"变了"。

### `ignore`
排除清单。常见用法：忽略 PDF 渲染相关的 CSS、忽略 Figma 里的移动端 page。

---

## 首次运行：如何生成

进入 Phase 6 时，把 Phase 0-3 收集的信息整合写入 `mapping.json`：

1. 扫描 demo CSS 抽出所有 token → `tokens`
2. 扫描 Figma 抽出所有 Variables/Styles/Components → 跟 demo token 做模糊匹配建立关联
3. 任何模糊匹配 🛑 给用户确认
4. 用户在 Phase 1-3 的确认结果写入 `acceptedDiffs`
5. `lastSync.timestamp = now()`

## 增量运行：如何更新

只更新本轮变化的条目。未变条目原封不动（保留之前的双边值）。

## 冲突检测算法

对每条 token / component：

```
current_demo = 从 demo 现状读到的值
current_figma = 从 Figma 现状读到的值
baseline_demo = mapping.json 里上次的 demo 值
baseline_figma = mapping.json 里上次的 Figma 值

demo_changed = (current_demo != baseline_demo)
figma_changed = (current_figma != baseline_figma)

if demo_changed && !figma_changed:
  → 默认 push 到 Figma（demo 是真相）
if !demo_changed && figma_changed:
  → 默认 push 到 demo（Figma 是真相）
if demo_changed && figma_changed:
  → 🛑 三路合并，让用户选
if !demo_changed && !figma_changed:
  → 这次同步不动这条
```

## 校验

skill 每次启动时：
1. 检查 `figmaFile.fileKey` 与当前打开的 Figma 文件是否一致；不一致 🛑 报错
2. 检查每个 `tokens.*.figma.var` 在 Figma 里是否还存在；如果消失 → 提示用户（可能是手动删除）
3. 检查每个 `components.*.figma.id` 节点是否还存在；如果消失 → 提示用户
