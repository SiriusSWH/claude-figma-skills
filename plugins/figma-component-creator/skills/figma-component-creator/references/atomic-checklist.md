# 组件清单 schema + 各层状态枚举模板

清单要按这个 schema 给用户看，便于一次性挑刺。

---

## 清单条目必填字段

```yaml
- id: A-001                          # 内部编号，方便用户挑刺时引用
  layer: Atom | Molecule | Organism | Template
  figma_name: "Atom/Button/Primary"   # 英文，强制层级前缀
  cn_name: "主按钮"                    # 中文名（强制）
  description: "用于页面主要操作的按钮，每个页面通常只有一个"  # 中文说明
  source:                              # 源头依据，必须可追踪
    type: html | ios | android
    path: "demo/components.css:.btn-primary"   # 或 swift 文件 / xml 路径
  variants:                            # 视觉风格维度（用 Figma Variant）
    - dim: Size
      values: [sm, md, lg]
    - dim: Type
      values: [primary, secondary, destructive]
  booleans:                            # 二态开关（用 Boolean Property）
    - name: disabled
      trigger: "用户无权限或前置条件未满足"
    - name: loading
      trigger: "点击后异步请求中"
    - name: hasIcon
      trigger: "左侧带 icon 的情况"
  swaps:                               # 可替换内容（Instance Swap）
    - name: iconLeft
      default: null
  text_props:
    - name: label
      default: "按钮"
  states:                              # 完整状态枚举（中文触发条件）
    - name: Default
      trigger: "组件默认渲染"
    - name: Pressed
      trigger: "用户按住瞬间"
    - name: Disabled
      trigger: "disabled=true 或前置条件未满足"
    - name: Loading
      trigger: "loading=true，文字隐藏显示 spinner"
  depends_on: [Variables/Color/brand, Variables/Spacing, Atoms/Icon]   # 依赖前置层
```

**注**: id 用 `A-`/`M-`/`O-`/`T-` 前缀方便用户引用，不进 Figma name。

---

## 状态枚举 checklist（按组件类型最低要求）

下表是各类组件**必须列出**的状态。漏一个就视为清单不完整。

### Button
- Default
- Pressed
- Disabled
- Loading（如果业务可能异步）
- + Hover（如果目标含 web）

### Input / TextField
- Empty (placeholder 可见)
- Filled (有用户输入)
- Focus (focus ring)
- Error (含错误提示)
- Disabled
- ReadOnly（如果有此需求）

### List / Cell
- Default
- Pressed
- Selected
- Disabled
- + Swipe action 展开态（iOS）

### List 整体
- WithData
- Empty
- Loading（首次加载）
- Error
- + Refreshing（下拉刷新）

### Modal / Sheet
- Entering / Visible / Exiting（动画的中间态可省略）
- 含 dismissable / non-dismissable 区分

### Toast / Snackbar
- Success / Error / Warning / Info
- + Loading（如果支持）

### Switch / Checkbox / Radio
- Off / On / Indeterminate（仅 Checkbox）
- Disabled (Off / On)

### Avatar / Badge
- WithImage / WithInitials / Empty
- Online / Offline status dot（如果含）

### NavigationBar / TabBar
- LightAppearance / DarkAppearance
- WithBackButton / WithoutBackButton
- WithRightAction / WithoutRightAction

---

## 命名规范

### Figma 节点 name
- ComponentSet: `<Layer>/<Family>/<Variant>` 例 `Atom/Button/Primary`
- 变量集: `Color/`、`Spacing/`、`Radius/`、`Typography/`
- 不要用空格，用 `/` 做层级

### Description（中文，强制）
- 1 句话功能定位
- 一行触发场景

例：
```
Atom/Button/Primary
中文名: 主按钮
Description: 页面主要操作按钮，承载唯一推荐动作。每个页面只应该有 1 个，多个会削弱视觉焦点。
```

---

## 分批策略

「一批」的定义（满足任一即可）：
- ≤ 5 个**独立**组件
- 1 个组件家族的全部 variants（如 Button 的 primary/secondary/destructive × sm/md/lg 算 1 批）
- 1 个 organism 的全部依赖组件（如 NavigationBar + 内部 IconButton + Title）

跨 layer 不可合批：Atom 必须先全部完成，才能开 Molecule 批次。

---

## 清单输出格式（给用户看的）

不要直接给 yaml dump。给用户看的格式：

```
=== Layer 0 · Tokens ===

批次 0  Variables
├─ Color: 12 个语义色变量（label / secondaryLabel / systemBackground...）
├─ Spacing: 9 级（4/8/12/16/20/24/32/40/48）
├─ Radius: 6 级（4/8/10/12/16/全圆）
└─ Typography: 9 级字号 × {Regular, Medium, Semibold, Bold}

=== Layer 1 · Atoms ===

批次 1  Button 家族（A-001 ~ A-003）
├─ A-001 Atom/Button/Primary    主按钮
│   Variants: Size=sm/md/lg, Type=primary/destructive
│   States: Default/Pressed/Disabled/Loading
│   Booleans: hasIcon
│   源头: demo/components.css:.btn-primary
└─ ...

批次 2  Input 家族（A-004 ~ A-005）
└─ ...
```

让用户一眼看出 layer / 批次 / 组件 / 状态全貌。
