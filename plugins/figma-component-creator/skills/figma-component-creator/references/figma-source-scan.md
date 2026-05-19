# Figma 设计稿源头 · 扫描与提取规范

当源头是 Figma 设计稿时（不是 demo 代码 / iOS / Android），Step 1 走这套流程：先识别已有 Components / Variables，再扫描候选元素让用户裁决。

---

## 阶段 1 · 识别"已存在"

目标：把 Figma 文件里**已经做成 Component / Variable / Style 的东西**捞出来，直接登记到 mapping.json，**不重建**。

### 1.1 遍历所有 page + 节点

```javascript
// 用 use_figma 跑
const allPages = figma.root.children;
const allComponents = [];
const allComponentSets = [];

for (const page of allPages) {
  await figma.setCurrentPageAsync(page);
  const components = page.findAll(n => n.type === 'COMPONENT');
  const sets = page.findAll(n => n.type === 'COMPONENT_SET');
  allComponents.push(...components);
  allComponentSets.push(...sets);
}
```

### 1.2 提取 Component / ComponentSet 元信息

每个 Component / ComponentSet 输出:
```json
{
  "id": "1234:5678",
  "name": "Atom/Button/Primary",
  "description": "主按钮",
  "type": "ComponentSet",  // 或 "Component"
  "page": "Components",
  "variants": [
    { "Size": "sm", "Type": "primary" },
    { "Size": "md", "Type": "primary" }
  ],
  "properties": {
    "disabled": "BOOLEAN",
    "loading": "BOOLEAN",
    "icon": "INSTANCE_SWAP"
  },
  "usageCount": 14  // 全文件中该组件被实例化的次数
}
```

### 1.3 提取 Variables

```javascript
const collections = figma.variables.getLocalVariableCollections();
const variables = figma.variables.getLocalVariables();

const byCollection = {};
for (const c of collections) {
  byCollection[c.name] = {
    id: c.id,
    modes: c.modes,
    variables: variables
      .filter(v => v.variableCollectionId === c.id)
      .map(v => ({
        id: v.id,
        name: v.name,
        type: v.resolvedType,  // 'COLOR' | 'FLOAT' | 'STRING' | 'BOOLEAN'
        values: v.valuesByMode,
      })),
  };
}
```

### 1.4 提取 legacy Styles

```javascript
// Figma 老版本 styles，新文件可能没有
const paintStyles = figma.getLocalPaintStyles();
const textStyles = figma.getLocalTextStyles();
const effectStyles = figma.getLocalEffectStyles();
```

如果有 legacy styles 且没对应 Variable，提醒用户：建议迁移到 Variable（不强制）。

### 1.5 登记到 mapping.json

```json
{
  "sources": [{ "type": "figma", "fileKey": "abc123" }],
  "tokens": {
    "Color/label": { "status": "existing", "variableId": "VariableID:1234" },
    "Spacing/md": { "status": "existing", "variableId": "VariableID:5678" }
  },
  "components": {
    "Atom/Button/Primary": { "status": "existing", "nodeId": "1234:5678", "componentSetId": "1234:5678" }
  }
}
```

---

## 阶段 2 · 扫描"候选提取"

目标：把图层里**反复出现但还不是 Component / Variable 的东西**找出来，列给用户裁决。

### 2.1 反复出现的颜色 → 候选 Color Variable

```javascript
const colorMap = new Map();  // key: 'r,g,b,a', value: { count, nodes: [] }

const allNodes = [];
for (const page of figma.root.children) {
  await figma.setCurrentPageAsync(page);
  allNodes.push(...page.findAll(n => 'fills' in n && Array.isArray(n.fills)));
}

for (const node of allNodes) {
  for (const fill of node.fills) {
    if (fill.type !== 'SOLID') continue;
    // 已经 bind variable 的跳过
    if (fill.boundVariables?.color) continue;
    const { r, g, b } = fill.color;
    const a = fill.opacity ?? 1;
    const key = `${r.toFixed(3)},${g.toFixed(3)},${b.toFixed(3)},${a.toFixed(2)}`;
    if (!colorMap.has(key)) colorMap.set(key, { count: 0, nodes: [], color: { r, g, b, a } });
    const entry = colorMap.get(key);
    entry.count++;
    if (entry.nodes.length < 5) entry.nodes.push(node.id);  // 只存前 5 个示例
  }
}

const candidates = [...colorMap.entries()]
  .filter(([_, v]) => v.count >= 3)
  .map(([key, v]) => ({
    type: 'Color',
    hex: rgbToHex(v.color),
    occurrence: v.count,
    sampleNodes: v.nodes,
    suggestedName: suggestColorName(v.color),  // e.g. "Color/blue-500"
  }));
```

### 2.2 反复出现的字号 / 字重组合 → 候选 Typography Variable

```javascript
const typoMap = new Map();
const textNodes = allNodes.filter(n => n.type === 'TEXT');

for (const t of textNodes) {
  const key = `${t.fontSize}px / ${t.fontName.family} ${t.fontName.style} / lh=${t.lineHeight.value || 'auto'}`;
  if (!typoMap.has(key)) typoMap.set(key, { count: 0, nodes: [], spec: { fontSize: t.fontSize, fontName: t.fontName, lineHeight: t.lineHeight } });
  const entry = typoMap.get(key);
  entry.count++;
  if (entry.nodes.length < 5) entry.nodes.push(t.id);
}

const typoCandidates = [...typoMap.entries()]
  .filter(([_, v]) => v.count >= 3)
  .map(([key, v]) => ({
    type: 'Typography',
    spec: v.spec,
    occurrence: v.count,
    sampleNodes: v.nodes,
    suggestedName: suggestTypoName(v.spec),  // e.g. "Typography/body"
  }));
```

### 2.3 反复出现的圆角值 → 候选 Radius Variable

```javascript
const radiusMap = new Map();
const radiusNodes = allNodes.filter(n => 'cornerRadius' in n && typeof n.cornerRadius === 'number' && n.cornerRadius > 0);

for (const n of radiusNodes) {
  if (n.boundVariables?.topLeftRadius) continue;  // 已 bind
  const r = n.cornerRadius;
  if (!radiusMap.has(r)) radiusMap.set(r, { count: 0, nodes: [] });
  const entry = radiusMap.get(r);
  entry.count++;
  if (entry.nodes.length < 5) entry.nodes.push(n.id);
}

const radiusCandidates = [...radiusMap.entries()]
  .filter(([_, v]) => v.count >= 3)
  .map(([r, v]) => ({
    type: 'Radius',
    value: r,
    occurrence: v.count,
    sampleNodes: v.nodes,
    suggestedName: `Radius/r${r}`,
  }));
```

### 2.4 反复出现的 padding / itemSpacing → 候选 Spacing Variable

```javascript
const spacingMap = new Map();
const frames = allNodes.filter(n => n.layoutMode && n.layoutMode !== 'NONE');

for (const f of frames) {
  for (const attr of ['paddingTop', 'paddingBottom', 'paddingLeft', 'paddingRight', 'itemSpacing']) {
    const val = f[attr];
    if (typeof val !== 'number' || val === 0) continue;
    if (f.boundVariables?.[attr]) continue;
    if (!spacingMap.has(val)) spacingMap.set(val, { count: 0, nodes: [] });
    const entry = spacingMap.get(val);
    entry.count++;
    if (entry.nodes.length < 5) entry.nodes.push(f.id);
  }
}

const spacingCandidates = [...spacingMap.entries()]
  .filter(([_, v]) => v.count >= 3)
  .map(([s, v]) => ({
    type: 'Spacing',
    value: s,
    occurrence: v.count,
    sampleNodes: v.nodes,
    suggestedName: `Spacing/s${s}`,
  }));
```

### 2.5 反复出现的图层组合 → 候选 Component

最难的一步。判定标准:

- 多个 Frame 有**结构上相似**的 children（type、顺序、层级深度近似）
- 视觉尺寸近似（容差 ±10%）
- 出现位置 ≥ 3

简化做法（先 v0）:
- 找所有顶层 Frame，按 children 类型签名分组（如 `"FRAME(TEXT,FRAME(RECT,TEXT))"`）
- 同签名 ≥ 3 个 → 候选 Atom 或 Molecule

```javascript
function signatureOf(node, depth = 2) {
  if (depth === 0 || !('children' in node)) return node.type;
  return `${node.type}(${node.children.map(c => signatureOf(c, depth - 1)).join(',')})`;
}

const sigMap = new Map();
const candidateNodes = allNodes.filter(n =>
  n.type === 'FRAME' &&
  n.layoutMode !== 'NONE' &&
  n.width < 400 && n.height < 200  // 限制在 atom / molecule 大小
);

for (const n of candidateNodes) {
  const sig = signatureOf(n);
  if (!sigMap.has(sig)) sigMap.set(sig, { count: 0, nodes: [] });
  const entry = sigMap.get(sig);
  entry.count++;
  if (entry.nodes.length < 5) entry.nodes.push(n.id);
}

const componentCandidates = [...sigMap.entries()]
  .filter(([_, v]) => v.count >= 3)
  .map(([sig, v]) => ({
    type: 'Component',
    layer: guessLayer(sig),  // 'Atom' | 'Molecule'
    signature: sig,
    occurrence: v.count,
    sampleNodes: v.nodes,
    suggestedName: '⚠️ 待用户命名',  // 模型起名不准，让用户定
  }));
```

⚠️ Component 候选**不要自动起名**，模型猜的名字 80% 不对。请用户看完示例 node 后命名。

---

## 阶段 3 · 输出给用户的清单格式

```
=== Figma 源头扫描报告 ===

【已存在】（共 18 项，直接登记，不重建）
Components (12):
  - Atom/Button/Primary (使用 14 次)
  - Atom/Button/Secondary (使用 8 次)
  - ...
Variables (6):
  - Color/label (语义色)
  - Color/secondaryLabel
  - Spacing/md (8pt)
  - ...

【候选提取】（共 9 项，请逐项裁决 a/b/c）

  [c-001] Color · #007AFF
    出现 14 次 · sample nodes: 1234:1, 1234:5, 1234:9
    建议命名: Color/system-blue
    你的选择: ___
  
  [c-002] Color · #FF3B30
    出现 6 次 · sample nodes: ...
    建议命名: Color/system-red
    你的选择: ___
  
  [c-003] Typography · 17px / SF Pro Regular / lh=22
    出现 23 次
    建议命名: Typography/body
    你的选择: ___
  
  [c-004] Radius · 10
    出现 7 次
    建议命名: Radius/r10
    你的选择: ___
  
  [c-005] Spacing · 12
    出现 31 次
    建议命名: Spacing/s12
    你的选择: ___
  
  [c-006] Component · FRAME(TEXT,FRAME(RECT))
    出现 5 次 · sample nodes: 1234:88, 1234:90, ...
    猜测层级: Molecule
    待你命名 → ___
    你的选择: ___
  
  ...

🛑 请回复每项 a / b / c 后再继续:
   a) 提取（指定命名）
   b) 暂不提取（写 acceptedGap）
   c) 跳过（不登记）
```

---

## 阶段 4 · 候选转 Component / Variable 的实际操作

用户授权 a) 后:

### 4.1 候选 → Color Variable

```javascript
const collection = figma.variables.getLocalVariableCollections()
  .find(c => c.name === 'Color') || figma.variables.createVariableCollection('Color');

const v = figma.variables.createVariable(suggestedName, collection.id, 'COLOR');
v.setValueForMode(collection.defaultModeId, { r, g, b, a });

// 把所有 sampleNodes 改成 bind 这个 variable
for (const nodeId of allNodesWithThisColor) {
  const node = await figma.getNodeByIdAsync(nodeId);
  const fills = JSON.parse(JSON.stringify(node.fills));
  fills[0] = figma.variables.setBoundVariableForPaint(fills[0], 'color', v);
  node.fills = fills;
}
```

### 4.2 候选 → Spacing / Radius Variable

```javascript
const v = figma.variables.createVariable(suggestedName, collection.id, 'FLOAT');
v.setValueForMode(collection.defaultModeId, value);

// 把所有 sampleNodes 的相应属性改成 bind
for (const nodeId of sampleNodes) {
  const node = await figma.getNodeByIdAsync(nodeId);
  node.setBoundVariable('itemSpacing', v);  // 或 'cornerRadius' / 'paddingLeft' 等
}
```

### 4.3 候选 → Component

```javascript
const template = await figma.getNodeByIdAsync(sampleNodes[0]);  // 选第一个作模板
const component = figma.createComponent();
component.name = userProvidedName;
component.resize(template.width, template.height);

// 把 template 的 children 转移过去
for (const child of [...template.children]) {
  component.appendChild(child);
}
// 把 template 转成 instance
const newInstance = component.createInstance();
template.parent.insertChild(template.parent.children.indexOf(template), newInstance);
newInstance.x = template.x;
newInstance.y = template.y;
template.remove();

// 其它 sampleNodes 同样替换为 instance（如果用户同意全替换）
```

⚠️ 替换其它位置前**必须用户明确同意**。如果用户只想提取不替换其它位置，把这一步跳过，记 acceptedGap。

---

## 不变规则

- ❌ 单次出现的图层不进候选清单
- ❌ 自动起名 Component（必须让用户命名）
- ❌ 未经用户许可就把原图层 reparent 成 Component
- ❌ 替换原图层时 detach 后重新画（必须 createInstance 复用 Component）
- ✅ 所有候选必须标 `出现次数` 和 `sampleNodes`
- ✅ 用户裁决 a/b/c 必须逐项收，不能合并"全选 a"（除非用户主动说"全部提取"）
