---
name: figma-page-builder
display_name: Figma 页面拼装器
description: 根据 demo（HTML/iOS/Android）+ PRD 在指定 Figma 文件中拼装可交互的产品页面与流程。强制流程：识别源头 + PRD → 组件覆盖度预检 → 文本版流程图给用户确认 → Figma 流程图落地 → 分批拼页面 → 用户每批验收。所有页面必须用组件库现有 Component + Variable 拼，禁止凭印象自画。触发场景：用户说"按 PRD 在 Figma 里画页面"、"拼一套登录流程的 Figma"、"基于 demo + PRD 在 Figma 出页面"、"做交互流程图加页面"。前置依赖：mapping.json 必须存在且 components 表非空，否则建议先跑 figma-component-creator。
---

# Figma 页面拼装器

## 这个 skill 解决什么

`figma-component-creator` 建完组件库后，怎么把组件**拼成**真实页面 + 流程？这个 skill 处理的就是这一步：吃 demo + PRD，吐出 Figma 文件里完整的「流程图 + 页面状态全集」，并保证所有页面**只用现有组件**拼出来，不凭印象画。

是 `figma-component-creator` 的**下游 skill**。

## 不变规则（Guardrails，违反即停下）

1. **mapping.json 是基线**：开局必读。不存在 / `components` 表空 → 🛑 停，建议先跑 `figma-component-creator`
2. **组件覆盖度预检 = 硬闸门**：列出本次需要的组件 vs 现有组件，缺口必须用户决议（三选项）后才能继续
3. **流程图两步式**：先文本版（mermaid / 缩进树）→ 用户确认 → 才在 Figma 真画
4. **用户确认 = 硬闸门**：流程图未确认不画 Figma；完整页面清单未确认不开始拼
5. **批次大小**：一批 = 一个完整用户流程的所有页面 + 状态，或 ≤ 3 个独立页面 + 状态。跨流程不合批
6. **组件实例化纪律**：
   - 必须 `figma.createComponent.createInstance()`，禁止 `figma.createFrame` 自画可视元素
   - **禁止 detach instance**（detach 后下游同步链断，未来再做页面更新会失败）
   - 实例改动只能通过：`setProperties`（boolean/variant）、text content、instance swap
7. **变量引用强约束**：所有 fill / spacing / radius / typography **必须 bind Figma Variable**，硬编码 fail
8. **自验证强制**：每批结束自己 `get_screenshot` 对比 demo / PRD wireframe，视觉不接近不交付
9. **页面状态完整性**：每页必须枚举 Default / Empty / Loading / Error / 浮层态 / First-time（如适用）/ Permission（如适用），漏一个视为清单不完整

---

## 主流程（9 步）

```
Step 0 · 启动校验（必读）
  ├─ Read <demo-root>/CLAUDE.md
  ├─ Read <demo-root>/mapping.json
  │   ├─ 不存在 / components 空 → 🛑 停，建议:
  │   │   "本 skill 需要组件库基线。建议先跑 figma-component-creator 把
  │   │    所需组件建完，得到 mapping.json.components 后再回来。
  │   │    或者你确认要继续，列出哪些组件预计在哪里？"
  │   └─ 存在 → 记录已有组件清单
  ├─ Read <demo-root>/.figma-skills/lessons.md
  └─ 从用户拿到 Figma 文件 URL（提取 fileKey）+ PRD 输入形式

Step 1 · PRD 输入形式确认
  ├─ 识别 PRD 类型: Word / Markdown / 飞书 / Notion / 截图 / 口头
  ├─ 拿到结构化文本:
  │   - Word / Markdown / 飞书 / Notion → Read 或让用户粘贴
  │   - 截图 → 🛑 让用户先转成文字描述
  │   - 口头 → 🛑 让用户先写成 Markdown / 用 dictate
  └─ 列 PRD 涉及的功能模块清单

Step 2 · 源头 + PRD 一致性校验
  ├─ 对照 demo 实际页面 vs PRD 描述
  ├─ 列差异表（demo 显示 X，PRD 写 Y，建议方向）
  ├─ 默认: demo = source of truth，但 PRD 是产品意图
  └─ 🛑 让用户裁决冲突 → 形成统一的"目标页面 + 流程"清单

Step 3 · 组件覆盖度预检（HARD STEP）
  ├─ 列本次涉及的所有 UI 元素（从 demo + PRD 摸出来）
  ├─ 跟 mapping.json.components 表比对
  ├─ 输出缺口表:
  │   | 需要的组件 | 现有 | 建议方案 |
  │   | Atom/SegmentControl | ❌ | 用 Atom/Tabs 代替 / 或新建 |
  │   | Molecule/Toast | ✅ | 直接用 |
  ├─ 🛑 缺组件时强制给用户三选项:
  │   a) 暂停本 skill，去跑 figma-component-creator 补建（推荐）
  │   b) 用现有最接近的代替（写进 mapping.json.acceptedGaps）
  │   c) 你给具体方案
  └─ 用户明确回复才继续。禁止"那我就先继续做"

Step 4 · 目标设备 + 页面尺寸
  ├─ 询问用户: iOS / Android / Web / 多端？
  ├─ 定 frame size:
  │   - iOS: 393×852 (iPhone 16 Pro) 或 375×667 (SE)
  │   - Android: 412×915 (Pixel 8)
  │   - Web: 1440×900 / 1280×800
  ├─ 多端 → 每端独立 page，页面平铺
  └─ 写入决策

Step 5 · 文本版流程图（不动 Figma）
  ├─ 用 mermaid 或缩进树画流程图（在终端给用户看，便于快速改）
  ├─ 必须标明:
  │   - 入口页（用户从哪来）
  │   - 每个 action 触发后跳哪页
  │   - 异常分支（错误 / 空 / 加载 / 无权限）
  │   - 终止页
  ├─ 列每页状态枚举（按 references/page-states-checklist.md 最小集）
  └─ 🛑 用户确认流程结构后才进 Step 6
  详见 references/flow-diagram-spec.md

Step 6 · Figma 流程图落地（一次性完成，非分批）
  ├─ 在 Figma 文件单独建 page: `Flow/<FlowName>/Diagram`
  ├─ 用 FigJam 风格节点（圆角矩形 + 箭头）画
  ├─ 节点内显示页面名称 + 状态标签
  ├─ 节点位置: 左→右 或 上→下，按层级自动布局
  └─ 自验证 get_screenshot → 对比文本版流程图

Step 7 · 🛑 完整页面清单 + 用户确认（HARD GATE）
  输出表给用户:
  | Page | State | 用到的 Components | 用到的 Variables | 源头 |
  | Login/PhoneInput | Default | Atom/Input, Atom/Button, ... | Color/label, ... | demo/login.html |
  | Login/PhoneInput | Error | Atom/Input(state=error), ... | ... | PRD §3.2 |
  | ... | ... | ... | ... | ... |
  
  + 分批方案:
  批次 1: Login 流程 (4 页 × 平均 3 状态 = 12 frames)
  批次 2: ...
  
  - 强制 stop，等用户回复
  - 用户可挑刺 / 增删 / 重组
  - 用户明确说"开始拼页面"才进 Step 8

Step 8 · 分批拼页面（严格按批次顺序）
  每批执行流程（不可省）:
    1. 在 Figma 对应 page 建 frame（尺寸按 Step 4 决策）
    2. 严格用 createInstance:
       const inst = component.createInstance();
       inst.setProperties({ state: 'error', size: 'lg' });
       inst.children[0].characters = '错误文案';  // text 改法
    3. 文本/boolean/variant/swap 全部通过 properties 改 → 禁止 detach
    4. 所有 fill/spacing/radius bind Variable:
       node.setBoundVariable('fills', variableId);
    5. 自己 get_screenshot 取本批
    6. 对照 demo / PRD wireframe 自查:
       - 视觉不接近 → 自己修 → 再对照（循环到匹配）
       - 视觉接近 → 走下一步
    7. 报告"已拼 X 页，Page Y，node id Z，请去 Figma 检查"
    8. 🛑 等用户回复:
         ├─ "继续" → 进下一批
         ├─ "改 X" → 修这批
         ├─ "重做这批" → 删掉重建
         └─ "先停" → 保存状态等用户

Step 9 · 收尾
  ├─ 写 <demo-root>/mapping.json:
  │   ├─ pages: 每页 Figma node ID + demo 路径 + PRD 章节
  │   ├─ flows: 流程节点 + 跳转 + 起止页
  │   ├─ acceptedGaps: 替代方案记录（如果有）
  │   └─ lastSync 更新
  ├─ 追加 <demo-root>/.figma-skills/lessons.md
  └─ 总结给用户:
       ├─ 已拼页面数 / 流程数
       ├─ 用到的组件 / 变量统计
       ├─ acceptedGaps 列表（提醒未来要补建）
       └─ 下一步建议（启用 prototype 模式 / 继续拼其他流程）
```

---

## 决策快查

### 组件覆盖缺口的三选项处理（HARD）

| 缺口程度 | 推荐方案 |
|---|---|
| 缺核心组件（页面主体元素） | a) 暂停去跑 figma-component-creator |
| 缺小配饰（icon / badge 等） | a) 或 b) 用现有最接近替代 |
| 缺一次性变体（特殊状态） | b) 替代 + acceptedGaps 标记 |
| 完全缺类（如没建过 Toast 体系） | a) 强制先补建 |

### 组件改动的合法手段

| 想改什么 | 合法手段 | 禁止 |
|---|---|---|
| 视觉风格 | `setProperties({ variant: 'primary' })` | 改 fill 直接覆盖 |
| 二态切换 | `setProperties({ disabled: true })` | 加 overlay 假装 disabled |
| 文字内容 | `text.characters = '...'` | detach 后改 |
| 替换图标/头像 | `setProperties({ icon: <componentKey> })` | createNodeFromSvg 自画 |
| 颜色微调 | 改 Variable 值，所有 instance 联动 | 单 instance 改 fill |

### 状态枚举最小集

按页面类型至少要列的状态：

- **表单页**: Default / Empty / Filled / Validating / Error / Success
- **列表页**: WithData / Empty / Loading / Error / Refreshing
- **详情页**: Default / Loading / Error / NoPermission
- **流程页**: Default / Submitting / Success / Failure
- **设置页**: Default / 各 setting toggle 不同组合
- **通用**: + 所有该页可能弹出的 modal / sheet / actionsheet

详见 `references/page-states-checklist.md`。

---

## 错误恢复

允许用户在批次间说：
- "重做第 N 批" → 删掉该批已拼 frame，重建
- "跳过页面 X" → 标记 skipped，写进 mapping.json
- "改流程图重走" → 回 Step 5
- "缺的组件先建" → 暂停本 skill，提示去 figma-component-creator，建完再 resume
- "先停下" → 保存当前批次进度等用户

## 触发警觉

用户说以下话立刻停下：
- "这不是 demo 那样的" → 没做自验证，停下检查源头
- "这个组件不对" → 可能用了错的 component，或 detach 了，回查
- "为什么换个组件" → 没按 acceptedGaps 处理，回查
- "单页查看修改" → 之前批量拼的质量差，逐页对照

## 不要做的事

- ❌ mapping.json 缺时继续做
- ❌ 缺组件不给三选项，自作主张代替
- ❌ 直接画 Figma 流程图（不先做文本版）
- ❌ createFrame 自画可视元素（必须 createInstance）
- ❌ detach instance 后改内部结构
- ❌ fill / spacing / radius 写死数值（必须 bind Variable）
- ❌ 跨流程合批
- ❌ 漏列页面状态（如忘了 Empty / Error）
- ❌ 用户没说"开始拼页面"就调 use_figma 写

## 引用的姐妹 skill / 本地 references

- **前置 skill**: `figma-component-creator`（建组件库 + 变量）
- **本地 references**:
  - `references/figma-api-pitfalls.md`（Plugin API 常见坑）
  - `references/mapping-schema.md`（mapping.json schema）
