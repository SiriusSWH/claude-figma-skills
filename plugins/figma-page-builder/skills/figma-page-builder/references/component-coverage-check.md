# 组件覆盖度预检流程

Step 3 的具体执行规范。这步做不好，后面拼页面就会"凭印象造组件"。

---

## 输入

- `mapping.json.components` 现有组件清单
- 本次涉及的页面 + 状态清单（来自 Step 2 的统一目标清单）
- demo / PRD 的原始 UI 元素

## 步骤

### 1. 列出本次需要的所有 UI 元素

按页面遍历，从 demo HTML / iOS XIB / Android XML / PRD wireframe 中抠：

```
Page: Login/PhoneInput
├─ Input（手机号输入框，带国家码前缀）
├─ Button（"下一步"，主按钮）
├─ Text Link（"忘记密码"）
├─ Checkbox（同意协议）
├─ NavigationBar（顶部）
├─ Toast（错误提示）
└─ ...
```

### 2. 跟现有组件对照

```
| 需要的 | mapping.json 现有 | 状态 |
| Atom/Input/Phone | Atom/Input | ❓ 现有 Input 是否支持国家码前缀？查 variants |
| Atom/Button/Primary | Atom/Button/Primary | ✅ |
| Atom/Link | ❌ 没有 | ❌ 缺 |
| Atom/Checkbox | Atom/Checkbox | ✅ |
| Organism/NavigationBar | Organism/NavigationBar | ✅ |
| Molecule/Toast | Molecule/Toast | ✅ |
```

### 3. 缺口分类

| 类型 | 处理 |
|---|---|
| **完全没有的组件** | 强制建议 a) 暂停去跑 figma-component-creator |
| **有但 variant 不够** | 建议 a) 给现有组件加 variant（也是 figma-component-creator 的工作） |
| **有但 property 不够**（如 Input 不支持 prefix slot） | 建议 a) 扩展现有组件 |
| **完全等价的现有可替代**（如 Link 用 Button/Tertiary 代替） | 给 b) 选项，标 acceptedGap |
| **一次性特殊变体**（只此一页用） | 建议 b) 替代 + acceptedGap |

### 4. 给用户的缺口报告格式

```
=== 组件覆盖度报告 ===

✅ 现有组件可用（5 个）:
  - Atom/Button/Primary
  - Atom/Checkbox
  - Atom/Input (满足 Default / Filled / Error)
  - Organism/NavigationBar
  - Molecule/Toast

⚠️  需要扩展（2 个）:
  - Atom/Input 缺 prefix slot（用于显示国家码 +86）
    建议: 给 Atom/Input 加 instance swap property "prefix"
  - Atom/Button 缺 variant=loading 状态
    建议: 给 Atom/Button 加 boolean property "loading"

❌ 完全缺失（1 个）:
  - Atom/Link 文字链接组件
    建议方案:
    a) 暂停本流程，去跑 figma-component-creator 把 Atom/Link 建出来（推荐）
    b) 用 Atom/Button/Tertiary 代替（视觉接近，但语义不对）
    c) 你给具体方案

🛑 请你回复其中一项，再继续下一步。
```

### 5. 用户回复后

| 用户说 | 行动 |
|---|---|
| "去补建" | 暂停本 skill，告诉用户切到 figma-component-creator，建完回来 resume |
| "用 Button/Tertiary 代替 Link，其他都去建" | 部分用 b)、部分暂停补建 |
| "全部去补建" | 暂停本 skill |
| "全部用现有代替" | 写入 mapping.json.acceptedGaps，继续 |
| "改成 PRD 里删掉 Link" | 修订目标清单，回 Step 2 校验 |

### 6. acceptedGaps schema

```json
{
  "acceptedGaps": [
    {
      "needed": "Atom/Link",
      "substituted": "Atom/Button/Tertiary",
      "rationale": "视觉接近，本期暂用，未来补建",
      "affectedPages": ["Login/PhoneInput", "Settings/Privacy"],
      "createdAt": "2026-05-19",
      "dismissAt": null
    }
  ]
}
```

---

## 不要做的事

- ❌ 看到缺组件就静悄悄"用 frame 自己画一个"（违反 Guardrail 6）
- ❌ "用 Button 代替"但不写 acceptedGaps（未来 sync 时会出错）
- ❌ 不告诉用户就跳过缺口
- ❌ 强行说"差不多就行"

## 触发警觉

用户说"差不多就行" / "你看着办" → **拒绝**。明确回复："我不能凭感觉决定用哪个组件替代，因为 mapping.json 是后续所有页面 / 同步工作的精确映射基线，硬上一个差不多的会污染基线。请明确回复 a / b / c 三选项之一。"
