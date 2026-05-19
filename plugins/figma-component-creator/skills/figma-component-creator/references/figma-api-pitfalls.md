# Figma Plugin API 陷阱（来自实战）

这些坑都是真踩过的。任何 `use_figma` 调用前先读一遍。

---

## 1. HORIZONTAL frame 设 width 时容易锁错轴

### 错误：
```javascript
const card = figma.createFrame();
card.layoutMode = 'HORIZONTAL';
card.counterAxisSizingMode = 'FIXED';  // 这是 height 轴！
card.resize(268, card.height);
// 结果：card 高度被锁死成默认 100，子内容塞不下
```

### 正确：
```javascript
const card = figma.createFrame();
card.layoutMode = 'HORIZONTAL';
// 对 HORIZONTAL：primary = 水平（width）, counter = 垂直（height）
card.primaryAxisSizingMode = 'FIXED';     // width 锁
card.counterAxisSizingMode = 'AUTO';      // height hug 内容
card.resize(268, card.height);
```

记忆方法：
- `HORIZONTAL` 布局 → primary 是水平
- `VERTICAL` 布局 → primary 是垂直
- 你想 fix 哪边宽就锁该边对应的轴

---

## 2. `layoutSizingHorizontal = 'FILL'` 必须先有 parent

### 错误：
```javascript
const child = figma.createText();
child.layoutSizingHorizontal = 'FILL';  // ❌ child 还没 append，没有 parent
parent.appendChild(child);
```

### 正确：
```javascript
const child = figma.createText();
parent.appendChild(child);
child.layoutSizingHorizontal = 'FILL';  // ✅
```

或封装成 helper:
```javascript
function fillH(parent, child) {
  parent.appendChild(child);
  child.layoutSizingHorizontal = 'FILL';
}
```

---

## 3. TEXT 节点没有 `children` 属性

### 错误：
```javascript
function walk(n) {
  if (n.name === 'target') do();
  n.children.forEach(walk);  // ❌ TEXT 节点没 children
}
```

### 正确：
```javascript
const PARENT_TYPES = new Set([
  'PAGE', 'FRAME', 'GROUP',
  'COMPONENT', 'COMPONENT_SET', 'INSTANCE',
  'BOOLEAN_OPERATION',
]);

function walk(n) {
  if (n.name === 'target') do();
  if (PARENT_TYPES.has(n.type) && n.children) {
    n.children.forEach(walk);
  }
}
```

---

## 4. 非当前 page 需要 `loadAsync` 才能访问 children

### 错误（沉默失败）：
```javascript
const page = figma.root.children.find(p => p.name === 'X');
const kids = page.children;  // 可能是空数组（未加载）
```

### 正确：
```javascript
const page = figma.root.children.find(p => p.name === 'X');
await page.loadAsync();
const kids = page.children;  // 现在有了
```

或：
```javascript
await figma.setCurrentPageAsync(page);
// 当前 page 自动 load
```

`figma.getNodeById('<id>')` 对未加载 page 上的节点也返回 null —— 先 `loadAsync` 再调用。

---

## 5. `fills = []` vs `fills = 'transparent'`

### 错误：
```javascript
f.fills = 'transparent';  // ❌ Figma 期望对象数组或空数组
```

### 正确（要透明）：
```javascript
f.fills = [];
```

### 正确（要白色但能 toggle 掉）：
```javascript
f.fills = [{ type: 'SOLID', color: { r: 1, g: 1, b: 1 } }];
```

### 检测白色默认 fill（新建的 frame 默认是白底）：
```javascript
function isWhite(fills) {
  if (!Array.isArray(fills) || fills.length !== 1) return false;
  const f = fills[0];
  if (f.type !== 'SOLID') return false;
  return f.color.r === 1 && f.color.g === 1 && f.color.b === 1;
}
```

新建 Frame 都有默认白底；如果你想要"透明 group 容器"，必须显式 `f.fills = []`。

---

## 6. 字体加载

### 错误：
```javascript
const t = figma.createText();
t.fontName = { family: 'Noto Sans SC', style: 'Medium' };  // ❌ 没加载就用
```

### 正确：
```javascript
await figma.loadFontAsync({ family: 'Noto Sans SC', style: 'Medium' });
const t = figma.createText();
t.fontName = { family: 'Noto Sans SC', style: 'Medium' };
```

每个用到的字重都要单独 load:
```javascript
await figma.loadFontAsync({ family: 'Noto Sans SC', style: 'Regular' });
await figma.loadFontAsync({ family: 'Noto Sans SC', style: 'Medium' });
await figma.loadFontAsync({ family: 'Noto Sans SC', style: 'Bold' });
// Inter 也是单独
await figma.loadFontAsync({ family: 'Inter', style: 'Regular' });
```

**特别注意**：`Inter` 的 "Semi Bold" 写法是 `'Semi Bold'`（带空格），不是 `'SemiBold'`。

---

## 7. 创建 ComponentSet 的正确姿势

### 错误（缺步骤）：
```javascript
const c1 = figma.createComponent();
c1.name = 'Variant=primary';
// ...
const c2 = figma.createComponent();
c2.name = 'Variant=secondary';
// 不能直接放在一起，必须用 combineAsVariants
```

### 正确：
```javascript
const variants = ['primary', 'secondary', 'danger'].map(v => {
  const c = figma.createComponent();
  c.name = `Variant=${v}`;
  setupContent(c, v);
  return c;
});

const set = figma.combineAsVariants(variants, page);  // page 是放置位置
set.name = 'Button';
// set 现在是 COMPONENT_SET，把它放进你想要的容器
container.appendChild(set);
```

注意 `combineAsVariants(nodes, parent)` 的 parent 是 ComponentSet 被放置的容器，不是 variants 共同父。variants 会自动被移到新创建的 ComponentSet 内。

---

## 8. layoutPositioning ABSOLUTE 必须在 auto-layout frame 内

### 错误：
```javascript
const frame = figma.createFrame();
frame.layoutMode = 'NONE';  // 不是 auto-layout
const child = figma.createRectangle();
frame.appendChild(child);
child.layoutPositioning = 'ABSOLUTE';  // ❌ 报错
```

### 正确：
```javascript
const frame = figma.createFrame();
frame.layoutMode = 'VERTICAL';  // 必须是 auto-layout
const child = figma.createRectangle();
frame.appendChild(child);
child.layoutPositioning = 'ABSOLUTE';  // ✅
child.x = 10; child.y = 20;
```

ABSOLUTE 让 child 不参与 parent 的 auto-layout 流，自由定位。

---

## 9. `combineAsVariants` 之后再改 ComponentSet 属性

```javascript
const set = figma.combineAsVariants(variants, page);
set.name = 'Button';
set.layoutMode = 'HORIZONTAL';  // 这是 ComponentSet 自己的布局（variants 如何排列）
set.itemSpacing = 16;
set.paddingLeft = 16; set.paddingRight = 16;
set.paddingTop = 16; set.paddingBottom = 16;
```

ComponentSet 本身也支持 auto-layout，控制 variants 在画布上如何排列展示。

---

## 10. 创建 SVG 节点 + **strokeWeight 必须手动按比例缩放**

### 用 createNodeFromSvg：
```javascript
const svgString = `<svg xmlns="..." viewBox="0 0 24 24" stroke-width="1.75" ...><path d="..."/></svg>`;
const node = figma.createNodeFromSvg(svgString);
node.resize(16, 16);  // 设你想要的尺寸
parent.appendChild(node);
```

### ⚠️ 大坑：strokeWeight 是绝对值，不会随 resize 缩放

SVG 的 `stroke-width="1.75"` 是在 `viewBox="0 0 24 24"` 坐标系里的，**渲染时浏览器会按比例缩**（24→16 时 stroke 实际显示约 1.17）。

**但 Figma 不会自动缩**！`createNodeFromSvg` 把 `stroke-width="1.75"` 直接当成 Figma 绝对单位的 strokeWeight。你 `resize(16,16)` 后，几何被压缩了，stroke 还是 1.75 绝对宽 → 在 16×16 的 icon 上 stroke 比 demo 浏览器渲染**粗 50%**，肉眼立刻能看出"图标不对"。

### 正确做法：

```javascript
function mkIcon(svgString, targetPx, color) {
  const node = figma.createNodeFromSvg(svgString);
  node.resize(targetPx, targetPx);
  // 从 SVG 里解析出 viewBox 大小（一般 24），算缩放后的 stroke
  // 假设源 stroke-width = 1.75, viewBox 24x24
  const SRC_VIEWBOX = 24;
  const SRC_STROKE = 1.75;
  const scaledStroke = SRC_STROKE * targetPx / SRC_VIEWBOX;
  function paint(n) {
    if ((n.type === 'VECTOR' || n.type === 'LINE') && 'strokeWeight' in n) {
      n.strokes = [color];
      n.strokeWeight = scaledStroke;
    }
    if ('children' in n) n.children.forEach(paint);
  }
  paint(node);
  return node;
}
```

对照表（源 stroke 1.75 / viewBox 24）:

| 目标尺寸 | 缩放后 strokeWeight |
|---|---|
| 12 | 0.88 |
| 14 | 1.02 |
| 16 | 1.17 |
| 18 | 1.31 |
| 20 | 1.46 |
| 24 | 1.75 (原值) |

### 自查方法

建完 icon master 后, dump 一下 strokeWeight 看是不是预期值:
```javascript
const icon = figma.getNodeById('xxx').findOne(n => n.type === 'VECTOR');
console.log(icon.width, icon.strokeWeight);  // 应该 ≈ width * 1.75 / 24
```

如果 strokeWeight === 1.75 不论 width 多少 → 中招了, 跑一次 rescale fix。

---

### ⚠️ 第二大坑：半透明 stroke 在路径重叠处会 alpha 叠加变暗

`createNodeFromSvg` 把 SVG 里的每个 `<path>` 拆成独立的 VECTOR 节点。如果 stroke 颜色是半透明（如 `rgba(0,0,0,0.58)` 即 text-secondary），多个 path 重叠的地方会**alpha 叠加**变得明显更暗（文件图标的折角、箭头与杆汇合处都是典型重灾区）。

浏览器渲染 demo 也有同样行为，但 16px 渲染下重叠在亚像素以下看不见；Figma master 在画布上往往以更大尺寸预览，重叠斑点立即暴露。

**正确做法：图标 stroke 用等效不透明色**

`rgba(0,0,0,a)` 在白底渲染等效于 `rgb(255*(1-a))` 三通道、opacity=1。

| 原 rgba | 白底等效 opaque | hex |
|---|---|---|
| `rgba(0,0,0,0.58)` (text-secondary) | `rgb(107,107,107)` | `#6B6B6B` |
| `rgba(0,0,0,0.40)` (text-tertiary) | `rgb(153,153,153)` | `#999999` |
| `rgba(0,0,0,0.25)` (border) | `rgb(191,191,191)` | `#BFBFBF` |

通用换算：把 paint 从 `{color: {r,g,b}, opacity: a}` 改成 `{color: {r*a + (1-a), g*a + (1-a), b*a + (1-a)}, opacity: 1}`。

注意：这只在 icon 几乎总是渲染在**白色或近白背景**时正确。如果同一个 icon 还要叠在 accent 蓝底或深色面板上，用不同 token 的 opaque 等效色（或直接用 `WHITE`，那本来就是 opaque）。

**自查方法**：master 高倍 zoom 看, 折角 / 箭头汇合处有没有明显黑斑。

**修复脚本片段**:
```javascript
function toOpaqueEquiv(paint) {
  if (!paint || paint.type !== 'SOLID') return paint;
  const op = paint.opacity == null ? 1 : paint.opacity;
  if (op >= 0.99) return paint;
  const c = paint.color;
  return {
    type: 'SOLID',
    color: {
      r: c.r * op + (1 - op),
      g: c.g * op + (1 - op),
      b: c.b * op + (1 - op),
    },
    opacity: 1,
  };
}
// 然后 walk 所有 vector,n.strokes = n.strokes.map(toOpaqueEquiv)
```

---

## 11. 处理批量错误回滚

### 现象：
一个 `use_figma` 调用里中途报错，**整批操作回滚**（包括前面成功的）。

### 应对：
- 一次性大量改动时，拆成多个 `use_figma` 调用
- 每次调用末尾返回 status，确认成功后再发下一个
- 关键步骤（如建组件）单独成一次调用

---

## 12. `getNodeById` 在懒加载场景下返回 null

见 #4。**总是先 `await page.loadAsync()`** 再 getNodeById。

---

## 13. Variables API

### 创建 Variable Collection
```javascript
const coll = figma.variables.createVariableCollection('Brand');
coll.addMode('default');  // 默认就有一个 mode，按需添加
```

### 创建 Color Variable
```javascript
const v = figma.variables.createVariable('Primary', coll.id, 'COLOR');
v.setValueForMode(coll.modes[0].modeId, { r: 0, g: 0.478, b: 1 });  // #007AFF
```

### 修改值
```javascript
const v = figma.variables.getVariableById('VariableID:xxx');
v.setValueForMode(modeId, newRgb);
```

### 注意
- `resolvedType`: 'COLOR' | 'FLOAT' | 'STRING' | 'BOOLEAN'
- COLOR 值是 `{r,g,b,a}` 都 0-1
- FLOAT 值是数字

---

## 14. 写后验证

每次大型操作后，**立刻**截图验证。不要等到最后才发现问题。

```javascript
// 操作完成后
return JSON.stringify({ status: 'ok', createdIds: [...] });
```

然后用 `mcp__pencil__get_screenshot` 或类似工具截图对比。

---

## 15. Shared Plugin Data 的写入

要在 Figma 文件里持久存元数据（不是 Figma 节点属性）：

```javascript
// 必须用 sharedPluginData（pluginData 需要 manifest id）
node.setSharedPluginData('figma_skills', 'lastSyncTimestamp', '2026-05-14T08:30:00Z');
const v = node.getSharedPluginData('figma_skills', 'lastSyncTimestamp');
```

namespace（第一个参数）必须 ≥3 字符且只能 a-z / 0-9 / _ / .

---

## 16. 检测元素是否在 Figma 文件中存在

```javascript
const n = figma.getNodeById('123:456');
if (!n) {
  // 不存在或所在 page 未加载
}
```

更稳的版本：
```javascript
async function safeGet(id) {
  const n = figma.getNodeById(id);
  if (n) return n;
  // 尝试 loadAsync 各 page
  for (const page of figma.root.children) {
    await page.loadAsync();
    const n2 = figma.getNodeById(id);
    if (n2) return n2;
  }
  return null;
}
```

---

## 17. `figma.setCurrentPageAsync` 后必须 await

```javascript
await figma.setCurrentPageAsync(page);
// 这之后可以直接用 figma.currentPage
```

不 await 会沉默失败。

---

## 18. 字体的 "Semi Bold" 写法

Figma 字体 style 严格匹配字符串。对 Inter：
- `'Regular'`
- `'Medium'`
- `'Semi Bold'`（带空格！）
- `'Bold'`
- `'Extra Bold'`（带空格！）

对 Noto Sans SC:
- `'Regular'`
- `'Medium'`
- `'Bold'`
（无 Semi Bold）

写错 style 名字直接报错。
