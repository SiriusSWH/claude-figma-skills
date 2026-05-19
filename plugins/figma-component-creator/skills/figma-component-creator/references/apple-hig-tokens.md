# Apple HIG + 字体硬数据

建组件时直接抄这份。不要凭印象写"我猜 iOS 应该是这样"。

---

## 字体可用性（macOS Figma desktop 实测，2026-05）

### 实际可调用的字体

| family | 字重 | 备注 |
|---|---|---|
| **SF Pro** | Black / Heavy / Bold / Semibold / Medium / Regular / Light / Thin / Ultralight (+Italic) | 完整。还有 Compressed/Condensed/Expanded 变体 |
| **SF Pro Rounded** | 9 级（Ultralight ~ Black） | 用于圆润感场景 |
| **SF Mono** | （未单独测，按需探测） | 等宽 |
| **Noto Sans SC** | Black / Bold / Medium / Regular / DemiLight / Light / Thin | 中文唯一可靠选项 |
| **Noto Sans TC / HK / JP / KR** | 同上 | 繁中 / 日 / 韩 |
| **Inter** | 全字重 | 跨平台兜底英文 |

### 不可用的字体（坑）

| family | 状态 | 原因 |
|---|---|---|
| **PingFang SC / HK / TC** | ❌ Figma 客户端**完全不识别** | 字体版权屏蔽；即便 macOS 系统装了也调不到 |
| **苹方** | ❌ | 同上 |
| **San Francisco**（旧名） | ❌ | 已改名 SF Pro |

### 字体决策默认值

- **单人 / macOS-only 项目**: 英文 SF Pro / 中文 Noto Sans SC
- **跨平台协作**: 英文 Inter / 中文 Noto Sans SC
- **永远禁止**: PingFang (Figma 不识别)
- **图标字体**: 不要用，icon 走 vector instance swap

### 字重映射（保证中英一致视觉重量）

| 英文 SF Pro | 中文 Noto Sans SC |
|---|---|
| Regular | Regular |
| Medium | Medium |
| Semibold | Medium 或 Bold（Noto Sans SC 无 Semibold）|
| Bold | Bold |
| Heavy / Black | Black |

→ 注意：Noto Sans SC **没有 Semibold**。中文要么用 Medium 要么用 Bold，跟英文 Semibold 对齐看视觉感受决定。

### 加载字体的代码

```javascript
// 加载前必须 loadFontAsync
await figma.loadFontAsync({ family: "SF Pro", style: "Regular" });
await figma.loadFontAsync({ family: "Noto Sans SC", style: "Regular" });

// 在 text 节点上设 fontName
text.fontName = { family: "SF Pro", style: "Regular" };
```

混排中英文时，要建两段 text range：

```javascript
text.setRangeFontName(0, 3, { family: "SF Pro", style: "Regular" });   // 英文段
text.setRangeFontName(3, 6, { family: "Noto Sans SC", style: "Regular" });  // 中文段
```

---

## iOS Typography Scale（HIG）

| Style | Size (pt) | Weight | LineHeight |
|---|---|---|---|
| Large Title | 34 | Regular / Bold | 41 |
| Title 1 | 28 | Regular / Bold | 34 |
| Title 2 | 22 | Regular / Bold | 28 |
| Title 3 | 20 | Regular / Semibold | 25 |
| Headline | 17 | Semibold | 22 |
| Body | 17 | Regular | 22 |
| Callout | 16 | Regular | 21 |
| Subhead | 15 | Regular | 20 |
| Footnote | 13 | Regular | 18 |
| Caption 1 | 12 | Regular | 16 |
| Caption 2 | 11 | Regular | 13 |

→ Figma Typography Variables 用这套作为默认。Demo 不匹配的，**抠 demo 真实 px 写入**。

---

## iOS Spacing（8pt grid）

| 用途 | 推荐值 |
|---|---|
| 紧凑内边距 | 4, 8 |
| 标准内边距 | 12, 16 |
| 区块间距 | 20, 24 |
| 大区块间距 | 32, 40, 48 |
| 屏幕边缘 padding | 16, 20 |

→ Figma Spacing Variables 至少建 4/8/12/16/20/24/32/40/48 这 9 级。

---

## iOS Radius

| 用途 | 值 |
|---|---|
| 小按钮 / Tag | 6, 8 |
| 标准 Card | 10, 12 |
| 大 Card / Modal | 16, 20 |
| Full pill | 全圆（height/2） |
| 仅 sheet 顶部圆角 | 16（左上 + 右上）|

---

## 触控目标（Touch Target）

- **最小 44 × 44 pt**（HIG 硬约束）
- 建按钮 / IconButton 时 frame size 不能小于 44×44，即使内容是 24×24 icon

---

## 颜色（语义化 token）

iOS 系统语义色，建 Color Variables 至少含：

### Label（文字）
- `label` / `label-secondary` / `label-tertiary` / `label-quaternary`

### Background
- `system-background` / `secondary-system-background` / `tertiary-system-background`
- `system-grouped-background` / `secondary-system-grouped-background`

### Fill（控件填充）
- `system-fill` / `secondary-system-fill` / `tertiary-system-fill` / `quaternary-system-fill`

### Separator
- `separator` / `opaque-separator`

### 状态色
- `system-blue`（主品牌色 fallback）
- `system-red`（destructive）
- `system-green`（success）
- `system-orange`（warning）
- `system-yellow`、`system-purple`、`system-pink` 按需

→ 每个语义色都要有 **Light** 和 **Dark** 两个值（Figma Color Variables 用 Mode 区分）。

---

## SafeArea

iPhone 通用：
- 顶部（带 notch / Dynamic Island）: 47-59pt
- 底部（带 home indicator）: 34pt
- 左右: 0（人像）

→ Templates 层做页面布局时必须留 SafeArea，不要把内容贴边。

---

## Shadow（HIG 标准）

iOS 用得克制，主要在 Modal / FAB / 抬起的 Card：

| 用途 | Y offset | Blur | Spread | Color |
|---|---|---|---|---|
| 浮动 Card | 0 | 8 | 0 | rgba(0,0,0,0.08) |
| Modal / Sheet | 0 | 16 | 0 | rgba(0,0,0,0.12) |
| FAB | 0 | 12 | 0 | rgba(0,0,0,0.16) |

不要给所有 Card 都加 shadow，除非源头 demo 用了。

---

## 字重 ≤ Medium 约束（已废止？）

历史 global CLAUDE.md 写过"字重 ≤ Medium，除非用户授权"。这条**只对 PingFang 有效**：

- PingFang Semibold 在 Figma 不可用 → 才回退到 Medium 或 Inter Semi Bold
- **SF Pro 全字重可用，不受此约束**
- Noto Sans SC 无 Semibold，写 Semibold 时按上面的映射降到 Medium 或升到 Bold

所以新的规则：
- 用 SF Pro → 直接写 Semibold/Bold/Heavy，无限制
- 用 PingFang → 不要用，改 Noto Sans SC
- 用 Noto Sans SC → 没 Semibold，按视觉感受选 Medium 或 Bold
