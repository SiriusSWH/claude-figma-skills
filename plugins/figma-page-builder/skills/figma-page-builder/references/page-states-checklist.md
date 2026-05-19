# 页面状态枚举 checklist

每个页面必须按这个表枚举状态，漏一个回退到"清单不完整"。

---

## 通用最小集（所有页面都要）

- **Default**: 默认数据态、常规渲染
- **Empty**: 无数据、首次进入空白态
- **Loading**: 数据请求中、骨架屏 / spinner
- **Error**: 请求失败 / 网络错误 / 服务端错误
- **浮层叠加态**: 该页可能弹出的 modal / sheet / actionsheet / toast 各算一个状态

## 按页面类型的附加状态

### 表单页（登录 / 注册 / 设置）
- Default（空表单）
- Filled（用户填了一部分）
- Validating（实时校验中）
- Error（字段错误，含具体错误文案）
- Success（提交成功，可能跳转或显示成功态）
- Disabled（提交按钮置灰）

### 列表页
- WithData（多种数据量：1 条 / 多条 / 很多条 / 接近上限）
- Empty（无数据，含 onboarding 提示）
- Loading（首次 + 加载更多）
- Error（含 retry 按钮）
- Refreshing（下拉刷新中）
- Editing / Selecting（多选模式，如适用）
- Searching（搜索结果态）

### 详情页
- Default（完整内容）
- Loading（骨架屏）
- Error（请求失败）
- NoPermission（无权限查看）
- NotFound（资源不存在）
- + 内容相关的状态（如笔记编辑中 / 已发布 / 草稿）

### 流程页（如付款 / 提交）
- Default（待操作）
- Submitting（提交中，按钮 loading）
- Success（成功页 / 成功 toast 后跳转）
- Failure（失败页 / 失败 toast）
- TimedOut（超时）

### 设置 / 偏好页
- 每个 setting 项的 toggle 不同组合（开 / 关 / 不可用）
- WithUnsavedChanges（有未保存改动）

### 引导页 / Onboarding
- Step 1 / Step 2 / ... Step N
- Skip 按钮态

### 权限相关
- Granted（已授权）
- Denied（被拒绝）
- NotDetermined（未询问）
- Prompting（询问中的对话框）

---

## 浮层态怎么算

浮层（modal / sheet / actionsheet / toast / popover）有几种处理方式：

### 方式 A：单独 frame（推荐）
浮层 + 其下方"原页面冻结态"作为一个组合 frame：
```
Page/Note/Edit-Default
Page/Note/Edit-DeleteConfirm    ← 弹出删除确认 modal 的态
Page/Note/Edit-MoreActions      ← 弹出 actionsheet 的态
```

好处：用户能直接看到上下文。

### 方式 B：浮层独立组件
浮层本身作为 organism 在组件库里，页面 frame 只表示触发条件。但需要在流程图里标注"点击 X 弹出 Y"。

→ 默认用方式 A。除非用户明确说浮层多 / 想节省 frame 数量。

---

## First-time / Onboarding 态

如果页面首次进入时有特殊引导（手把手提示 / 默认数据填充 / 教学卡片），单独列：
```
Page/Home-Default        ← 老用户日常态
Page/Home-FirstTime      ← 新用户首次态
```

不要塞进 Default。

---

## Permission denied 态

涉及隐私 / 摄像头 / 相册 / 位置 / 通知的页面，必须列：
```
Page/Camera-Granted      ← 已授权，正常使用
Page/Camera-Denied       ← 被拒绝，含跳转设置引导
Page/Camera-NotDetermined ← 未询问，含权限请求 prompt
```

漏这条最常见，但用户实际跑业务时一定会撞到。

---

## 暗色模式

如果项目支持 Dark mode：
- 不要为每个状态都建 light + dark 两份 frame
- 让 Color Variables 自动跟随 mode 切换
- 单独建一个 "Theme Showcase" page，展示几个关键页的 light / dark 对比

---

## 给用户看清单的格式

不要 dump yaml。按下面的表给：

```
Page: Login/PhoneInput
├─ Default            手机号输入框为空，按钮 disabled
├─ Filled             手机号 11 位，按钮 enabled
├─ Validating         实时校验中，按钮 loading
├─ Error              手机号格式错，下方红色提示
└─ + 弹层: TermsModal  用户点"条款"弹出的协议 modal

Page: Login/SmsCode
├─ Default            收到验证码后输入，60s 倒计时
├─ Resending          重新发送中
├─ ResendDone         重发成功 toast
├─ Error              验证码错误
└─ Success            校验通过，准备跳转
```

这样用户一眼看出哪些状态全 / 哪些漏。
