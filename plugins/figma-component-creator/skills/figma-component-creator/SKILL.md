---
name: figma-component-creator
display_name: Figma 组件创建器
description: 从 HTML demo / iOS / Android / Figma 设计稿 四类源头按原子设计 + Apple HIG 在指定 Figma 文件中一次性生成组件库。当源头是 Figma 设计稿时，先识别已有 Components / Variables 直接登记，再扫描候选元素（反复出现的图层 / 颜色 / 字号 / 间距）让用户裁决要不要提取为组件 / 变量。强制流程：识别源 → 抠 token / 扫描已有 → 列清单 → 用户确认 → 分批生成 → 每批用户验收。触发场景：用户说"从 demo 生成 Figma 组件库"、"把 iOS 应用翻译成 Figma 组件"、"基于 X 在 Figma 建组件库"、"从这份 Figma 设计稿提取组件"、"整理这份 Figma 的组件和变量"、"生成 Figma design system"。即使用户没明说"组件库"，只要要从源头到 Figma 的一次性建库 / 整理，就主动用这个 skill。
---

# Figma 组件创建器

## 这个 skill 解决什么

从一个源头一次性在指定 Figma 文件中按**原子设计 + Apple HIG**建出组件库。这个 skill 处理的是**初次建库**，结束后可继续用 `figma-page-builder` 拼页面。

支持四类源头：

| 源头 | 工作模式 |
|---|---|
| HTML demo | 从 CSS / HTML 抠真实数值 → 生成 Component + Variable |
| iOS 工程 | 从 Swift / Asset Catalog 抠 UIColor / UIFont / 间距 → 生成 |
| Android 工程 | 从 colors.xml / dimens.xml / styles.xml 抠 → 生成 |
| **Figma 设计稿** | **识别 + 提取双模式**：① 已有 Components / Variables → 直接登记到 mapping.json，不重建；② 候选元素（反复出现的图层 / 颜色 / 字号 / 间距）→ 列给用户裁决要不要提取成组件 / 变量 |

## 不变规则（Guardrails，违反即停下）

1. **Demo 是 source of truth**：所有数值通过 `grep` / `Read` 从源头抠真实值，禁止凭印象写
2. **Token 先行**：组件之前先建 Figma Variables；所有组件必须**引用变量**，禁止 magic number
3. **Atomic 严格依赖顺序**：Variables → Atoms → Molecules → Organisms → Templates，不许跳级
4. **用户确认 = 硬闸门**：清单未确认不开始生成；批次未验收不进下一批
5. **批次大小**：≤ 5 个独立组件 / 1 个组件家族（如所有 Button variants 算一批）
6. **字体规则**（基于实测，见 `references/apple-hig-tokens.md`）：
   - 英文 → SF Pro（全字重可用）
   - 中文 → **Noto Sans SC**（PingFang Figma 不识别，硬约束）
7. **自验证强制**：每批结束自己 `get_screenshot` 对照源头 demo，视觉不接近不交付
8. **命名**：Figma name 英文 `Atom/Button/Primary` 格式；Description 字段写中文
9. **不擅自动用户原图层**（Figma 源头特有）：已存在的 Component / Variable **不重建只登记**；候选提取必须用户明确许可才把原图层转成 Component
10. **提取候选不靠感觉**（Figma 源头特有）：候选必须标"出现次数 ≥ 3 / 出现位置"作为依据，单次出现的图层不进候选清单

---

## 主流程（7 步）

```
Step 0 · 启动校验（必读）
  ├─ Read <demo-root>/CLAUDE.md（如果存在）
  ├─ Read <demo-root>/mapping.json（如果存在）—— 已有 token / 已有组件
  ├─ Read <demo-root>/.figma-skills/lessons.md（如果存在）—— 历史踩坑
  └─ 从用户拿到 Figma 文件 URL，提取 fileKey

Step 1 · 源头摸底（按源头类型分支）

  分支 A · HTML demo
    ├─ grep `.css` / `:root` 取 CSS variables
    ├─ 列关键页面 HTML
    └─ 列已有 CSS class（如 `.btn-primary`）

  分支 B · iOS 工程
    ├─ 读 *.swift / Asset Catalog 取 UIColor + UIFont
    ├─ 列关键 ViewController / View
    └─ 列已有 component 类（如 `PrimaryButton`）

  分支 C · Android 工程
    ├─ 读 colors.xml / dimens.xml / styles.xml
    ├─ 列关键 Activity / Fragment
    └─ 列已有 View 类

  分支 D · Figma 设计稿（识别 + 提取双模式）
    详细方法见 references/figma-source-scan.md。简版流程:
    
    ① 扫"已存在":
      ├─ get_metadata + batch_get 遍历目标 Figma 文件
      ├─ 提取已存在的 Components / ComponentSets（含 properties / variants / description）
      ├─ 提取已存在的 Variables（Color / Spacing / Radius / Typography）
      ├─ 提取已存在的 Styles（legacy paint / text styles）
      └─ 标记为 [已存在] → 后续直接登记 mapping.json，不重建
    
    ② 扫"候选提取"（反复出现但还不是 Component / Variable 的图层）:
      ├─ 反复出现的图层组合（≥ 3 次） → 候选 Atom / Molecule
      ├─ 反复使用的 paint color（≥ 3 次） → 候选 Color variable
      ├─ 反复使用的字号 / 字重组合（≥ 3 次） → 候选 Typography variable
      ├─ 反复使用的圆角值（≥ 3 次） → 候选 Radius variable
      ├─ 反复使用的 padding / gap 值（≥ 3 次） → 候选 Spacing variable
      ├─ 每个候选必须标:
      │   - 出现次数
      │   - 出现位置（含 node id 列表）
      │   - 建议命名
      └─ 标记为 [候选提取] → 后续用户裁决
    
    ③ 列目标 page / frame（用户希望最终覆盖的页面）

Step 2 · 字体可用性探测（HARD STEP，不能跳）
  ├─ 在 Figma 中跑 `figma.listAvailableFontsAsync()`
  ├─ 过滤 SF Pro / PingFang / Noto Sans / Inter
  ├─ 决策（默认按下面规则，给用户看一次确认）:
  │   - 英文: SF Pro（如全字重可用）
  │   - 中文: Noto Sans SC
  │   - 跨平台分享场景: 改 Inter + Noto Sans SC
  └─ 把决策写入未来 mapping.json.fonts

Step 3 · Figma 文件预检
  ├─ get_metadata fileKey 看现有 page / ComponentSet
  ├─ 检测命名冲突
  └─ 询问用户两件事:
      1. 组件目录: 单 page 还是按 Atom/Molecule/Organism 分 page？
      2. 冲突处理: 覆盖 / 跳过 / 加版本号？

Step 4 · 完整组件清单（按 Atomic Design 分层）
  
  每个清单项必须标"状态分类":
    [已存在]  Figma 文件里已经有，直接登记到 mapping.json，不重建（仅 Figma 源头）
    [候选提取] 图层里反复出现但还不是 Component / Variable，建议提取（仅 Figma 源头，用户裁决）
    [新建]    Figma 文件里没有，源头要求要有 → 调 use_figma 创建
  
  Layer 0 · Tokens（Figma Variables）
    - Color: 语义色（label / secondaryLabel / systemBackground / separator…）+ 品牌色 + 状态色
    - Spacing: 4 / 8 / 12 / 16 / 20 / 24 / 32 / 40 / 48
    - Radius: 4 / 6 / 8 / 10 / 12 / 16 / 20 / 全圆（按 demo 实际用到的精简）
    - Typography: 字号 / 行高 / 字重（按 SF Pro 9 级，参考 HIG）
  
  Layer 1 · Atoms
    Button / Input / Icon / Avatar / Badge / Divider / Label / Tag / Switch / Checkbox / Radio
  
  Layer 2 · Molecules
    SearchBar / ListItem / FormField / Card / Cell / Toolbar Action
  
  Layer 3 · Organisms
    NavigationBar / TabBar / Header / List / Modal / ActionSheet / Toast / Sheet
  
  Layer 4 · Templates
    页面布局（含 SafeArea / StatusBar / 内容区比例）

  详细清单字段见 references/atomic-checklist.md

Step 5 · 🛑 用户确认完整清单（HARD GATE）
  - 强制 stop，等用户回复
  - 用户可挑刺 / 增加 / 删除 / 重组
  - 对 [候选提取] 项每个需明确（仅 Figma 源头）:
      a) 提取为 Component / Variable（按建议命名）
      b) 暂不提取，标 acceptedGap 写进 mapping.json（保留原图层不动）
      c) 跳过（不登记不提取）
  - 用户明确说"开始执行"才进 Step 6
  - 用户没说"开始"前，禁止调用 use_figma 写任何东西

Step 6 · 分批生成（严格按依赖顺序）
  批次顺序:
    批次 0:    Variables（Color + Spacing + Radius + Typography）
    批次 1…N:  Atoms（每批 ≤ 5 个或 1 个组件家族）
    批次 N+1…: Molecules
    批次 N+2…: Organisms
    批次 N+3…: Templates
  
  每批内部按"状态分类"分三种执行动作:
    A) [已存在] · 登记
       └─ 读 Figma 已有 node 的 id / properties / variants → 写入 mapping.json，不动 Figma
    
    B) [候选提取] · 提取
       ├─ 元素是 Variable（颜色 / 间距 / 圆角 / 字号）:
       │   ├─ 创建 Variable
       │   ├─ 把所有出现该原始值的 node 改成 bind Variable
       │   └─ 不删原 node，只重新绑定
       └─ 元素是 Component（图层组合）:
           ├─ 选最干净的一个原图层组合作为模板
           ├─ figma.createComponent → 把该图层 reparent 进 Component（保留视觉）
           ├─ 其它出现位置改成 instance（如用户同意一并替换）
           └─ 不动用户不希望动的位置（按 Step 5 用户回复处理）
    
    C) [新建] · 创建
       ├─ 写 use_figma 代码（严格用 token，禁止 magic number）
       ├─ figma.createComponent + auto-layout + bind Variables
       └─ 按 references/atomic-checklist.md 的 variant / boolean / swap 规范设属性
  
  每批执行流程（不可省）:
    1. 按上面 A / B / C 分类执行
    2. 自己 get_screenshot 取 Figma 当前批次截图
    3. 跟源头自对照:
       - 非 Figma 源头 → 对比 demo 截图
       - Figma 源头 → 提取前后对比，确认原图层视觉无回归
    4. 视觉不匹配 → 自己定位差异 → 修 → 再对照，循环到匹配
    5. 视觉匹配 → 报告"已 [登记/提取/新建] X 个，Page Y，node id Z，请去 Figma 检查"
    6. 🛑 等用户回复:
         ├─ "继续" → 进下一批
         ├─ "改 X" → 修这批
         ├─ "重做这批" → 删掉重建（仅 [新建] 项；[已存在] / [候选提取] 操作不可简单 undo，慎重）
         └─ "先停" → 保存状态等用户

Step 7 · 收尾
  ├─ 写 <demo-root>/mapping.json:
  │   ├─ sources: 本次源头类型（html / ios / android / figma）+ 路径或 fileKey
  │   ├─ fonts: 本次字体决策
  │   ├─ tokens: 变量 ID 表（标 [已存在] / [候选提取] / [新建] 来源）
  │   ├─ components: 每个组件的 Figma node ID + 源头映射（CSS class / Swift / xml / Figma node id）
  │   ├─ acceptedGaps: 用户选 b) 暂不提取的候选记录
  │   └─ lastSync: 时间戳
  ├─ 追加 <demo-root>/.figma-skills/lessons.md（本次新踩的坑，按现有 lessons 格式）
  └─ 总结给用户:
       ├─ 本次操作分布: [登记 X] [提取 Y] [新建 Z]
       ├─ 已建组件树
       ├─ 已建变量数量
       ├─ acceptedGaps 数量（提醒未来补提取）
       ├─ mapping.json diff
       └─ 下一步建议（哪些页面可以拼了 / 推荐启用 figma-page-builder）
```

---

## 决策快查

### Variant vs Boolean Property vs Instance Swap

| 维度类型 | 用法 | 例 |
|---|---|---|
| 视觉风格变体 | Variant | `Type=primary/secondary/destructive`、`Size=sm/md/lg` |
| 二态开关 | Boolean property | `disabled`、`selected`、`loading`、`hasIcon` |
| 内容可替换 | Instance swap | `iconLeft`、`avatar` |
| 文字内容 | Text property | `label`、`title` |

### 字体硬约束（基于 macOS Figma desktop 实测）

| 字体 | 可用性 | 字重 |
|---|---|---|
| SF Pro | ✅ 全字重 + Italic + Compressed/Condensed/Expanded | Black/Heavy/Bold/Semibold/Medium/Regular/Light/Thin/Ultralight |
| SF Pro Rounded | ✅ 全字重 | 9 级 |
| PingFang SC | ❌ Figma 客户端不识别 | 必须 fallback |
| Noto Sans SC | ✅ | Black/Bold/Medium/Regular/DemiLight/Light/Thin |
| Inter | ✅ 跨平台兜底 | 全字重 |

→ **默认: 英文 SF Pro / 中文 Noto Sans SC**。跨平台/团队协作场景才用 Inter + Noto Sans SC。

详见 `references/apple-hig-tokens.md`。

---

## 错误恢复

允许用户在批次间说：
- "重做第 N 批" → 删该批已建组件，重建
- "跳过组件 X" → 标记 skipped，写进 mapping.json.skipped
- "先停下" → 保存当前状态（已建到哪一批）等用户
- "改 token 后重新走" → 回 Layer 0 重建变量，下游全部失效需要重新生成

---

## 触发警觉

用户说以下话，**立刻停下**：
- "单个查看并修改" → 之前批量做的工作质量差，逐个对照不要继续
- "为什么这个不对" → 先承认 + 复盘根因，不要立即修
- "怎么和 demo 不一样" → 没做自验证，停下检查源头数值

---

## 不要做的事

- ❌ 用户没确认清单前调 use_figma 写
- ❌ 在 Layer N 完成前开始 Layer N+1
- ❌ 一批 > 5 个独立组件
- ❌ 用 PingFang（Figma 客户端不识别）
- ❌ 用 magic number（必须引用 Variables）
- ❌ 凭印象写"我猜 iOS 标准是这样"（必须抠源头）
- ❌ 用 free-form Rect/Vector 凑视觉（必须 Component + Auto-layout）
- ❌ 把 Figma 源头里已存在的 Component / Variable 重建一份（必须先 scan 再登记）
- ❌ 把 Figma 源头里单次出现的图层放进候选（出现次数 ≥ 3 才进候选）
- ❌ 未经用户 a/b/c 裁决就把原图层 reparent 成 Component（破坏用户原设计）

## 引用的姐妹 skill / 本地 references

- **下游 skill**: `figma-page-builder`（用建好的组件库 + PRD 在 Figma 拼页面流程）
- **本地 references**:
  - `references/figma-api-pitfalls.md` —— Plugin API 常见坑（fills=[]、combineAsVariants、Frame layout 等）。建组件前先扫一眼。
  - `references/atomic-checklist.md` —— 组件清单 schema + 状态枚举模板 + 命名规范。
  - `references/apple-hig-tokens.md` —— Apple HIG token 硬数据 + 字体实测可用性。
  - `references/figma-source-scan.md` —— Figma 设计稿源头模式：识别已有 + 扫描候选的具体 API 调用与判定规则。
