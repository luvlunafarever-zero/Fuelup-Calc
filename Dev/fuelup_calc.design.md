# Fuelup Calc 详细设计书

| 项目 | 内容 |
| --- | --- |
| 文档名称 | Fuelup Calc 详细设计书 |
| 对应源码 | `Dev/fuelup_calc.dev.html` |
| 软件版本 | 2.5.2 |
| 文档日期 | 2026-11-05 |
| 原作者 | luvluna |
| 技术栈 | HTML5 + CSS3 + 原生 JavaScript（ES6+，单文件应用） |

---

## 目录

1. [项目概述](#1-项目概述)
2. [总体设计](#2-总体设计)
3. [功能需求说明](#3-功能需求说明)
4. [页面布局与 UI 设计](#4-页面布局与-ui-设计)
5. [HTML 结构设计](#5-html-结构设计)
6. [CSS 样式设计](#6-css-样式设计)
7. [JavaScript 逻辑设计](#7-javascript-逻辑设计)
8. [核心计算逻辑](#8-核心计算逻辑)
9. [输入校验规则](#9-输入校验规则)
10. [数据持久化设计](#10-数据持久化设计)
11. [事件流与交互流程](#11-事件流与交互流程)
12. [移动端适配设计](#12-移动端适配设计)
13. [错误处理设计](#13-错误处理设计)
14. [初始化流程](#14-初始化流程)
15. [已知限制与优化建议](#15-已知限制与优化建议)
16. [附录](#16-附录)

---

## 1. 项目概述

### 1.1 项目背景

Fuelup Calc 是一款面向手机屏幕尺寸的网页计算器工具，用于帮助车主在加油前估算：

- **可加油量**（升）：基于油箱总容积、隐藏（备用）油箱容积、剩余续航里程和平均油耗推算；
- **加油总费用**（元）：可加油量 × 汽油单价；
- **每公里花费**（元/km）：汽油单价 × 平均油耗 ÷ 100，有折扣时同时显示折扣后每公里花费；
- **折扣优惠**：根据每升优惠价和加油机输入金额，计算实际支付金额与加油机实际加油体积。

### 1.2 设计目标

| 目标 | 说明 |
| --- | --- |
| 移动优先 | 以手机竖屏为主要使用场景，最大内容宽度限制为 600px |
| 即时计算 | 输入数据后自动计算并显示结果，无需"计算"按钮 |
| 数据持久化 | 所有输入自动保存到 `localStorage`，刷新/重开页面不丢失 |
| 结果常驻可见 | 计算结果固定在页面顶部（fixed header），输入时始终可见 |
| 键盘友好 | 手机虚拟键盘弹起时，顶部结果区域自动调整位置避免被遮挡 |
| 零依赖 | 不依赖任何第三方框架/库，单 HTML 文件即可运行 |

### 1.3 UI 设计参考

界面设计参考了 **Vuetify** 的基础样式风格（参见 `.clinerules`），采用手工实现的类 Vuetify 组件：

- `v-card`（卡片）
- `v-expansion-panel`（可折叠面板）
- `v-text-field`（带浮动标签的输入框）
- `v-btn`（带 Ripple 波纹效果的按钮）
- Bottom Sheet（底部弹出确认框）
- Chip（折扣结果显示胶囊）

---

## 2. 总体设计

### 2.1 文件结构

本项目为**单文件应用**，全部代码集中在一个 HTML 文件中：

```
fuelup_calc.dev.html
├── <head>
│   ├── meta 信息（charset / author / version / viewport）
│   ├── 页面标题与内联 SVG favicon
│   └── <style> 全部 CSS 样式（约 660 行）
└── <body>
    ├── fixed-header（固定顶部结果区）
    ├── fixed-header-placeholder（占位元素）
    ├── main-content（主内容区）
    │   └── v-app > v-card > v-card__text
    │       ├── v-expansion-panels
    │       │   ├── configPanel（配置信息面板）
    │       │   └── discountPanel（折扣计算面板）
    │       └── button-wrapper
    │           ├── remainingRange 输入框
    │           ├── fuelPrice 输入框
    │           └── resetBtn 重置按钮
    ├── resetConfirmSheet（Bottom Sheet 确认对话框）
    └── <script> 全部 JavaScript 逻辑（约 670 行，IIFE 封装）
```

### 2.2 工作区约定

| 目录 | 用途 |
| --- | --- |
| `./Dev` | 开发中文件，可编辑（本设计书对应 `Dev/fuelup_calc.dev.html`） |
| `./Release` | 发布文件，只读（历史版本：`fuelup_calc.2.1.html`、`fuelup_calc.2.5.html`） |

### 2.3 模块划分

JavaScript 逻辑整体封装在一个 IIFE（立即执行函数）中，避免污染全局作用域。内部按职责划分为以下逻辑模块：

```
┌─────────────────────────────────────────────────────┐
│                    IIFE 主模块                        │
├─────────────────────────────────────────────────────┤
│  ① 键盘适配模块        （visualViewport 监听与调整）   │
│  ② 占位高度模块        （fixed-header 高度补偿）       │
│  ③ 面板折叠模块        （配置面板/折扣面板展开收起）     │
│  ④ 浮动标签模块        （输入框 label 浮动效果）        │
│  ⑤ 持久化模块          （localStorage 读写）           │
│  ⑥ 重置流程模块        （Bottom Sheet 确认对话框）      │
│  ⑦ 核心计算模块        （加油量/费用/折扣计算）         │
│  ⑧ 清空按钮模块        （输入框一键清空）              │
│  ⑨ 初始化模块          （状态恢复与首次渲染）           │
└─────────────────────────────────────────────────────┘
```

---

## 3. 功能需求说明

### 3.1 功能清单

| 编号 | 功能 | 说明 |
| --- | --- | --- |
| F-01 | 加油量计算 | 根据油箱容积、隐藏容积、剩余续航、平均油耗计算可加油量 |
| F-02 | 费用计算 | 可加油量 × 汽油单价 |
| F-03 | 折扣计算 | 根据每升优惠价与油价计算折扣百分比；根据加油机金额计算实际加油体积与实际支付金额 |
| F-04 | 结果常驻显示 | 结果固定在顶部，包含升数、金额、明细行 |
| F-05 | 自动保存 | 任意输入变化后自动保存到 localStorage |
| F-06 | 状态恢复 | 页面加载时自动恢复上次输入与面板展开状态 |
| F-07 | 一键清空输入框 | 每个输入框在悬停/聚焦且有值时显示清空按钮 |
| F-08 | 全量重置 | 重置按钮弹出 Bottom Sheet 二次确认，确认后清空全部输入（隐藏容积恢复默认 5） |
| F-09 | 输入校验 | 非法输入时在顶部结果区显示错误提示，并将对应面板标题标记为错误底色 |
| F-10 | 键盘适配 | 虚拟键盘弹起遮挡结果区时，结果区自动跟随可视视口移动 |
| F-11 | 面板折叠记忆 | 配置面板与折扣面板的展开/收起状态持久化 |
| F-12 | 每公里花费显示 | 明细行显示每公里花费（元/km），有折扣且每升优惠价 > 0 时同时显示折扣前后对比 |

### 3.2 输入项一览

| 输入项 | 元素 ID | 所属区域 | 单位 | step | 默认值 | 是否必填 |
| --- | --- | --- | --- | --- | --- | --- |
| 油箱总容积 | `tankCapacity` | 配置面板 | 升 | 0.5 | 空 | 是（> 0） |
| 隐藏油箱容积 | `reserveFuel` | 配置面板 | 升 | 0.5 | `5` | 是（≥ 0） |
| 平均油耗 | `avgFuelConsumption` | 配置面板 | L/100km | 0.1 | 空 | 是（> 0） |
| 每升优惠价 | `discountPerLiter` | 折扣面板 | 元 | 0.01 | 空 | 否（≥ 0） |
| 加油机输入金额 | `pumpAmount` | 折扣面板 | 元 | 0.1 | 空 | 否（> 0） |
| 剩余续航 | `remainingRange` | 主输入区 | km | 1 | 空 | 是（≥ 0） |
| 汽油单价 | `fuelPrice` | 主输入区 | 元/升 | 0.1 | 空 | 否（< 0 视为 0） |

### 3.3 输出项一览

| 输出项 | 元素 ID | 位置 | 格式 |
| --- | --- | --- | --- |
| 可加油量 | `resultLitersFixed` | 固定头部左侧 | `{数值} <small>升</small>`，1 位小数 |
| 加油总费用 | `resultMoneyFixed` | 固定头部右侧 | 2 位小数 |
| 明细行 | `detailTextFixed` | 固定头部底部 | `X.XX 元/km`（有折扣且每升优惠价 > 0 时为 `X.XX (X.XX) 元/km`）+（可选）折扣百分比；油价为 0 时左侧为空 |
| 实际支付 | `dscActualPay` | 折扣面板 Chip | `¥ {数值} CNY`，2 位小数 |
| 加油机体积 | `dscPumpVolume` | 折扣面板 Chip | `{数值} L`，1 位小数 |

---

## 4. 页面布局与 UI 设计

### 4.1 整体布局

页面采用 Flex 纵向布局，`body` 最小高度为 `100dvh`（动态视口高度，兼容移动端地址栏收缩），内容水平居中，最大宽度 600px。

### 4.2 配色方案

| 用途 | 颜色值 | 说明 |
| --- | --- | --- |
| 主题色（头部背景/按钮） | `#546E7A` / `#607D8B` | Material Blue Grey 系列 |
| 页面背景 | 渐变 `#f5f7fa → #e9eef3`（135deg） | 浅灰渐变 |
| 卡片背景 | `#ffffff` | 白色卡片 + 阴影 |
| 输入框边框 | `#B0BEC5`（默认）/ `#455A64`（聚焦） | |
| 浮动标签 | `#78909C`（默认）/ `#01579B`（聚焦） | |
| 面板标题错误态 | 文字 `#9b1a1a`，背景 `#fff5ea` | 输入校验失败提示 |
| 明细行文字 | `#e7ffce` | 头部深色背景上的浅色文字 |
| 折扣百分比 | `#ffa3ab` | 头部明细行右侧 |
| 实付 Chip | 背景 `#E8F5E9`，文字 `#2E7D32` | 绿色系 |
| 体积 Chip | 背景 `#E3F2FD`，文字 `#1565C0` | 蓝色系 |

### 4.3 组件设计

#### 4.3.1 固定结果头部（fixed-header）

- `position: fixed; top: 0; z-index: 100`，带 `0 4px 12px` 投影与 12px 圆角；
- 内部结构：
  - `.result-stats`：左侧大号升数（2.2rem 粗体）+ 右侧金额（1.6rem 粗体），`flex-wrap` 支持换行；
  - `.result-detail`：明细行，上方有半透明白色分隔线，左侧 `.detail-left` 支持文本溢出省略（`text-overflow: ellipsis`），右侧 `.detail-discount` 显示折扣百分比；
- 键盘弹起时通过 JS 动态修改 `top` 值（见 [12. 移动端适配设计](#12-移动端适配设计)）。

#### 4.3.2 可折叠面板（v-expansion-panel）

- 标题区 `.v-expansion-panel__title`：左侧图标 + 文字，右侧箭头图标（展开时旋转 180°）；
- 内容区 `.v-expansion-panel__content`：通过 `max-height: 0 → 500px` 过渡实现展开/收起动画（`cubic-bezier(0.4, 0, 0.2, 1)`，0.3s）；
- 激活态类：`.v-expansion-panel--active`；
- 错误态类：`.v-expansion-panel-err`（应用于标题区，提示该面板内存在非法输入）；
- 共两个面板：
  - **配置信息**（齿轮图标）：油箱总容积、隐藏油箱容积、平均油耗；
  - **折扣计算**（标签图标）：顶部为折扣结果 Chip 行（默认 `display: none`，有有效数据时加 `.visible` 显示），下方为每升优惠价、加油机输入金额两个输入框。

#### 4.3.3 浮动标签输入框（v-text-field）

DOM 结构（每个输入框统一）：

```html
<div class="v-text-field" id="field-xxx">
    <div class="v-field">
        <div class="v-field__outline">
            <div class="v-field__outline__start"></div>  <!-- 2px 边框 -->
        </div>
        <div class="v-field__field">
            <input type="number" id="xxx" step="..." placeholder=" ">
            <span class="v-field__clearable" data-target="xxx">
                <svg class="clear-icon__svg">...</svg>   <!-- 清空按钮 -->
            </span>
        </div>
    </div>
    <label class="v-label" for="xxx">标签文字</label>
</div>
```

浮动标签行为：

| 状态 | 类名 | 效果 |
| --- | --- | --- |
| 无值且未聚焦 | （默认） | label 居中显示在输入框内（`top: 50%`） |
| 有值或聚焦 | `v-text-field--has-value` / `v-text-field--focused` | label 上浮至边框处（`top: 0`），缩小 85%，白色背景遮挡边框 |
| 聚焦 | `v-field--focused` | 边框变为 `#01579B`，输入文字变色 |

#### 4.3.4 清空按钮（v-field__clearable）

- 绝对定位于输入框右侧，圆形带叉图标；
- 默认 `opacity: 0; visibility: hidden; pointer-events: none`；
- 显示条件（CSS 控制）：输入框容器有 `.has-value` 类，且（鼠标悬停在输入框上 或 输入框聚焦）；
- 点击时播放向右滑出动画（`.v-field__clearable--clearing`，0.15s）；
- 使用 `pointerdown` 事件（而非 `click`）并 `preventDefault()`，避免点击时输入框失焦导致移动端键盘收起。

#### 4.3.5 按钮（v-btn）

- 全宽、圆角 8px、蓝灰色背景，带 Material 风格三层阴影；
- `::after` 伪元素实现 Ripple 波纹点击效果（`:active` 时放大并淡出）；
- 重置按钮包含一个旋转箭头 SVG 图标 + "Reset" 文字。

#### 4.3.6 Bottom Sheet 确认对话框

- `.bottom-sheet-overlay`：全屏半透明黑色遮罩（`rgba(0,0,0,0.5)`），`z-index: 1000`，默认 `visibility: hidden; opacity: 0`；
- `.bottom-sheet-panel`：底部白色面板，顶部圆角，默认 `translateY(100%)`，激活时滑入（0.3s）；
- 激活态类：`.active`（同时作用于遮罩层）；
- 内容：标题"确认清空" + 提示文字 + 取消/确认两个按钮；
- 点击遮罩空白处关闭。

---

## 5. HTML 结构设计

### 5.1 文档元信息

```html
<meta charset="UTF-8">
<meta name="author" content="luvluna">
<meta name="version" content="2.5.1">
<meta name="viewport" content="width=device-width, initial-scale=1.0, user-scalable=yes">
```

- favicon 使用内联 SVG data URI（加油泵图标，颜色 `#455A64`），无需外部资源。

### 5.2 DOM 结构树

```
body
├── div.fixed-header#fixedHeader
│   ├── div.result-stats
│   │   ├── div.result-liters#resultLitersFixed   → "— 升"
│   │   └── div.result-money#resultMoneyFixed     → "—"
│   └── div.result-detail#detailTextFixed         → "输入数据后自动计算"
│
├── div.fixed-header-placeholder#headerPlaceholder
│
├── div.main-content
│   └── div.v-app > div.v-card > div.v-card__text
│       ├── div.v-expansion-panels
│       │   ├── div.v-expansion-panel#configPanel
│       │   │   ├── div.v-expansion-panel__title#configPanelHeader
│       │   │   │   ├── 左侧：齿轮图标 + "配置信息"
│       │   │   │   └── 右侧：箭头图标（展开指示）
│       │   │   └── div.v-expansion-panel__content
│       │   │       ├── field-tank       → input#tankCapacity
│       │   │       ├── field-reserve    → input#reserveFuel (value="5")
│       │   │       └── field-consumption→ input#avgFuelConsumption
│       │   └── div.v-expansion-panel#discountPanel
│       │       ├── div.v-expansion-panel__title#discountPanelHeader
│       │       │   ├── 左侧：标签图标 + "折扣计算"
│       │       │   └── 右侧：箭头图标
│       │       └── div.v-expansion-panel__content
│       │           ├── div.discount-results#discountResults
│       │           │   ├── chip--pay    → span#dscActualPay  "¥ —"
│       │           │   └── chip--volume → span#dscPumpVolume "— L"
│       │           ├── field-discount   → input#discountPerLiter
│       │           └── field-pump       → input#pumpAmount
│       └── div.button-wrapper
│           ├── field-range  → input#remainingRange
│           ├── field-price  → input#fuelPrice
│           └── button.v-btn#resetBtn（图标 + "Reset"）
│
└── div.bottom-sheet-overlay#resetConfirmSheet
    └── div.bottom-sheet-panel
        ├── div.bottom-sheet-title    "确认清空"
        ├── div.bottom-sheet-message  "确定要清空所有输入数据吗？此操作不可撤销。"
        └── div.bottom-sheet-actions
            ├── button#cancelResetBtn   "取消"
            └── button#confirmResetBtn  "确认清空"
```

### 5.3 关键元素 ID 索引

| ID | 类型 | 用途 |
| --- | --- | --- |
| `fixedHeader` | 容器 | 固定结果区根元素 |
| `headerPlaceholder` | 容器 | 固定头部占位元素 |
| `resultLitersFixed` | 显示 | 可加油量 |
| `resultMoneyFixed` | 显示 | 总费用 |
| `detailTextFixed` | 显示 | 明细行 |
| `configPanel` / `configPanelHeader` | 面板 | 配置信息面板及其标题 |
| `discountPanel` / `discountPanelHeader` | 面板 | 折扣计算面板及其标题 |
| `discountResults` / `dscActualPay` / `dscPumpVolume` | 显示 | 折扣结果 Chip |
| `tankCapacity` / `reserveFuel` / `avgFuelConsumption` | 输入 | 配置项 |
| `discountPerLiter` / `pumpAmount` | 输入 | 折扣项 |
| `remainingRange` / `fuelPrice` | 输入 | 常用项 |
| `resetBtn` / `resetConfirmSheet` / `cancelResetBtn` / `confirmResetBtn` | 交互 | 重置流程 |

---

## 6. CSS 样式设计

### 6.1 全局基础

```css
* { margin: 0; padding: 0; box-sizing: border-box; }

body {
    font-family: 'Roboto', -apple-system, BlinkMacSystemFont, 'Segoe UI', 'Helvetica Neue', sans-serif;
    background: linear-gradient(135deg, #f5f7fa 0%, #e9eef3 100%);
    min-height: 100dvh;                 /* 动态视口高度，适配移动端 */
    padding: 12px 24px 24px 24px;
    display: flex; flex-direction: column;
    align-items: center; justify-content: space-between;
}
```

### 6.2 关键类说明表

| 类名 | 职责 | 关键属性 |
| --- | --- | --- |
| `.main-content` | 主内容容器 | `max-width: 600px; flex: 1 1 auto` |
| `.fixed-header` | 固定结果区 | `position: fixed; top: 0; z-index: 100; transition: top 0.1s ease-out` |
| `.fixed-header-placeholder` | 占位元素 | 高度由 JS 动态设置 |
| `.v-card` | 白色卡片 | 圆角 12px，双层阴影 |
| `.v-expansion-panel` | 折叠面板 | 边框 `#e2e8f0`，阴影，非首个面板 `margin-top: 16px` |
| `.v-expansion-panel__content` | 面板内容 | `max-height: 0; overflow: hidden`，激活时 `max-height: 500px` |
| `.v-expansion-panel--active` | 面板激活态 | 箭头旋转 180°，内容展开 |
| `.v-expansion-panel-err` | 面板错误态 | 标题文字 `#9b1a1a`，背景 `#fff5ea` |
| `.discount-results` | 折扣结果行 | 默认 `display: none`，`.visible` 时 `display: flex` |
| `.discount-chip` | 结果胶囊 | `border-radius: 999px`，`--pay` / `--volume` 变体 |
| `.v-text-field` | 输入框容器 | `margin-bottom: 24px; position: relative` |
| `.v-field__outline__start` | 输入框边框 | `border: 2px solid #B0BEC5`，聚焦变色 |
| `.v-label` | 浮动标签 | 绝对定位，`transition: all 0.15s` |
| `.v-field__clearable` | 清空按钮 | 默认隐藏，`.has-value` + 悬停/聚焦时显示 |
| `.v-field__clearable--clearing` | 清空动画态 | 向右平移 20px 并淡出 |
| `.v-btn` | 按钮 | Material 阴影 + Ripple 伪元素 |
| `.bottom-sheet-overlay` | 遮罩层 | `z-index: 1000`，`.active` 时可见 |
| `.bottom-sheet-panel` | 底部面板 | `translateY(100%)` → `.active` 时 `translateY(0)` |

### 6.3 动画与过渡一览

| 元素 | 动画 | 时长/曲线 |
| --- | --- | --- |
| 面板内容展开 | `max-height: 0 → 500px` | 0.3s `cubic-bezier(0.4, 0, 0.2, 1)` |
| 面板箭头 | `rotate(0 → 180deg)` | 0.2s ease |
| 浮动标签 | 位置 + 缩放 + 颜色 | 0.15s `cubic-bezier(0.4, 0, 0.2, 1)` |
| 清空按钮显隐 | `opacity + visibility + transform` | 0.2s ease |
| 清空按钮点击 | 向右滑出 20px + 淡出 | 0.15s ease-out |
| 按钮 Ripple | `scale(0 → 2)` + 淡出 | 0.3s / 0.5s |
| Bottom Sheet | 遮罩淡入 + 面板上滑 | 0.2s / 0.3s `cubic-bezier(0.4, 0, 0.2, 1)` |
| fixed-header 位置 | `top` 值变化 | 0.1s ease-out |

### 6.4 响应式设计

`@media (max-width: 480px)` 断点下的调整：

| 调整项 | 桌面/默认 | 小屏 |
| --- | --- | --- |
| 结果升数字号 | 2.2rem | 1.8rem |
| 结果金额字号 | 1.6rem | 1.8rem |
| fixed-header 内边距 | 20px 24px | 16px 20px |
| 输入框最小高度 | 56px | 52px |
| 面板内容展开内边距 | 20px 20px 8px | 25px 16px 0 |
| Bottom Sheet 面板内边距 | 28px 20px 30px | 24px 16px 20dvh（底部预留安全区） |

---

## 7. JavaScript 逻辑设计

### 7.1 代码组织

全部逻辑封装于 IIFE 中：

```javascript
(function() {
    const STORAGE_KEY = 'fuel_calculator_pro';
    // ... 全部逻辑
})();
```

### 7.2 DOM 引用

脚本开头集中获取所有需要操作的 DOM 元素引用，分为以下组：

| 分组 | 元素 |
| --- | --- |
| 输入元素 | `tankInput`、`reserveInput`、`rangeInput`、`consumptionInput`、`priceInput`、`discountPerLiterInput`、`pumpAmountInput` |
| 结果显示 | `resultLitersFixed`、`resultMoneyFixed`、`detailTextFixed` |
| 布局元素 | `fixedHeader`、`headerPlaceholder` |
| 面板元素 | `configPanel`、`panelHeader`、`discountPanel`、`discountPanelHeader`、`discountResults`、`dscActualPay`、`dscPumpVolume` |
| 重置流程 | `resetBtn`、`confirmSheet`、`cancelResetBtn`、`confirmResetBtn` |

### 7.3 函数清单

| 函数 | 所属模块 | 职责 |
| --- | --- | --- |
| `updatePlaceholderHeight()` | 占位高度 | 将占位元素高度设为 `fixedHeader.offsetHeight`，防止内容被遮挡 |
| `getKeyboardHeight()` | 键盘适配 | 返回 `window.innerHeight - visualViewport.height`（键盘高度估算） |
| `isHeaderFullyVisible()` | 键盘适配 | 判断结果区是否仍在可视视口内（`visualViewport.offsetTop >= originalMsgBottom/3` 视为被遮挡） |
| `adjustForKeyboard()` | 键盘适配 | 键盘弹起且结果区被遮挡时，将 `fixedHeader.style.top` 设为 `visualViewport.offsetTop + 8`（并做边界钳制）；否则恢复 `0` |
| `toggleConfigPanel()` / `toggleDiscountPanel()` | 面板折叠 | 切换面板激活类，并将状态写入 localStorage |
| `initConfigPanelState()` / `initDiscountPanelState()` | 面板折叠 | 从 localStorage 恢复面板展开状态（默认展开） |
| `errInConfigPanel(hasErr)` / `errInDiscountPanel(hasErr)` | 错误提示 | 为面板标题添加/移除错误态类 |
| `updateFloatingLabel(fieldId, input)` | 浮动标签 | 根据是否有值/是否聚焦更新容器与边框类名 |
| `bindFloatingLabel(fieldId, input)` | 浮动标签 | 为输入框绑定 focus/blur/input 事件以驱动浮动标签 |
| `saveToLocalStorage()` | 持久化 | 序列化全部输入值 + 时间戳写入 localStorage |
| `loadFromLocalStorage()` | 持久化 | 读取并恢复输入值，刷新浮动标签状态；无数据时设置默认值 |
| `showResetConfirm()` / `hideResetConfirm()` | 重置流程 | 显示/隐藏 Bottom Sheet |
| `performReset()` | 重置流程 | 清空全部输入（隐藏容积恢复 `5`），刷新标签、保存并重算 |
| `formatNumber(value, decimals)` | 工具 | 数字格式化，非法值返回 `'0.0'` |
| `updateDiscountResults(fuelPrice)` | 折扣计算 | 计算并显示折扣面板结果（无返回值） |
| `calculateAndDisplay()` | 核心计算 | 主计算入口：校验 → 计算 → 渲染结果 |
| `showError(msg)` | 错误提示 | 在固定头部渲染错误状态 |
| `onInputChange()` | 事件处理 | 输入变化统一回调：重算 + 保存 |
| `initClearableButtons()` | 清空按钮 | 为所有清空按钮绑定 `pointerdown` 事件 |
| `handleClearPointerDown(event)` | 清空按钮 | 清空目标输入框并触发 input 事件，保持焦点，播放动画 |
| `handleClearClick(event)` | 清空按钮 | （备用 click 实现，当前未绑定） |
| `updateFloatingLabelForField(container, input)` | 浮动标签 | 面向单个容器更新浮动标签状态 |
| `observeInputValueChanges()` | 清空按钮 | 为所有输入框绑定 input 事件，驱动 `.has-value` 类 |
| `updateClearableVisibility()` | 清空按钮 | 根据输入值有无切换 `.has-value` 类 |
| `initAllClearable()` | 清空按钮 | 组合初始化清空按钮功能 |

---

## 8. 核心计算逻辑

### 8.1 主计算流程（`calculateAndDisplay`）

```
输入：tankCapacity, reserveFuel, avgConsumption, remainingRange, fuelPrice
```

**步骤：**

1. **校验配置项**（任一失败即调用 `showError` 并终止）：
   - `tankCapacity` 必须 > 0；
   - `reserveFuel` 必须 ≥ 0；
   - `avgConsumption` 必须 > 0；
   - 校验通过后清除配置面板错误态。
2. **校验常用项**：
   - `remainingRange` 必须 ≥ 0，否则报错终止；
   - `fuelPrice` 为 NaN 或 < 0 时按 0 处理（不报错）。
3. **计算**：
   - 续航油量 `remainingFuelFromRange = remainingRange × avgConsumption ÷ 100`
   - 剩余总油量 `totalRemainingFuel = reserveFuel + remainingFuelFromRange`
   - 可加油量 `maxFillable = tankCapacity − totalRemainingFuel`，并钳制到 `[0, tankCapacity]`
   - 总费用 `totalCost = maxFillable × fuelPrice`
   - 每公里花费 `pricePerKm = fuelPrice × avgConsumption ÷ 100`（仅当 `fuelPrice > 0`，否则为 0）
4. **折扣与每公里花费**（可选）：若 `discountPerLiter ≥ 0` 且 `fuelPrice > 0`：
   - `discountRate = discountPerLiter ÷ fuelPrice × 100`
   - `payPercent = (100 − discountRate).toFixed(1)`
   - 折扣每公里花费 `dscPricePerKm = (fuelPrice − discountPerLiter) × avgConsumption ÷ 100`
   - 若 `discountPerLiter > 0`，明细左侧显示原价与折扣价对比
5. **渲染**：
   - 升数区：`{maxFillable 保留1位小数} <small>升</small>`
   - 金额区：`{totalCost 保留2位小数}`
   - 明细区左侧（`detail-left`）：
     - `fuelPrice > 0` 且（无有效折扣或每升优惠价为 0）：`X.XX 元/km`（`pricePerKm`，2 位小数）
     - `fuelPrice > 0` 且 `discountPerLiter > 0`：`X.XX (X.XX) 元/km`（`pricePerKm` 与 `dscPricePerKm`，2 位小数）
     - `fuelPrice ≤ 0`：空
   - 明细区右侧（`detail-discount`）：若有折扣则追加 `{payPercent}%`
6. 调用 `updateDiscountResults(fuelPrice)` 更新折扣面板。

### 8.2 折扣计算流程（`updateDiscountResults`）

```
输入：fuelPrice（由主计算传入），discountPerLiter，pumpAmount
```

**显示条件：**

- `pumpAmount > 0`（有效加油机金额）；
- `discountPerLiter ≥ 0` 且 `fuelPrice > 0`。

两者同时满足才显示结果 Chip（`.visible`），否则隐藏并清除面板错误态。

**计算公式：**

| 输出 | 公式 |
| --- | --- |
| 折扣率 | `discountRate = discountPerLiter ÷ fuelPrice × 100`（当前计算后未使用，支付百分比改由 `calculateAndDisplay` 计算） |
| 加油机体积 | `pumpVolume = pumpAmount ÷ fuelPrice`（1 位小数，显示为 `— L`） |
| 实际支付 | `actualPay = pumpVolume × (fuelPrice − discountPerLiter)`（2 位小数，显示为 `¥ — CNY`） |

### 8.3 公式汇总表

| 名称 | 公式 | 单位 |
| --- | --- | --- |
| 续航油量 | `剩余续航 × 平均油耗 ÷ 100` | L |
| 剩余总油量 | `隐藏油箱容积 + 续航油量` | L |
| 可加油量 | `油箱总容积 − 剩余总油量`（钳制 `[0, 油箱总容积]`） | L |
| 加油总费用 | `可加油量 × 汽油单价` | 元 |
| 每公里花费 | `汽油单价 × 平均油耗 ÷ 100` | 元/km |
| 折扣每公里花费 | `(汽油单价 − 每升优惠价) × 平均油耗 ÷ 100` | 元/km |
| 折扣率 | `每升优惠价 ÷ 汽油单价 × 100%` | % |
| 支付百分比 | `100% − 折扣率` | % |
| 加油机体积 | `加油机输入金额 ÷ 汽油单价` | L |
| 实际支付 | `加油机体积 × (汽油单价 − 每升优惠价)` | 元 |

### 8.4 数值格式化

`formatNumber(value, decimals = 1)`：

- `isNaN(value)` 或 `value === null` → 返回 `'0.0'`；
- 否则返回 `value.toFixed(decimals)`。

---

## 9. 输入校验规则

### 9.1 校验规则表

| 输入项 | 规则 | 失败处理 | 错误提示文案 |
| --- | --- | --- | --- |
| 油箱总容积 | 必填，数值 > 0 | 报错终止，配置面板标红 | 请输入有效的油箱总容积（大于0） |
| 隐藏油箱容积 | 必填，数值 ≥ 0 | 报错终止，配置面板标红 | 隐藏容积无效，请输入≥0的数值 |
| 平均油耗 | 必填，数值 > 0 | 报错终止，配置面板标红 | 平均油耗无效，请输入大于0的数值 |
| 剩余续航 | 数值 ≥ 0 | 报错终止 | 剩余续航无效，请输入≥0的数值 |
| 汽油单价 | 数值 ≥ 0 | 不报错，负值/NaN 按 0 处理 | — |
| 每升优惠价 | ≥ 0 才参与折扣计算 | 不满足时隐藏折扣结果 | — |
| 加油机输入金额 | > 0 才参与折扣计算 | 不满足时隐藏折扣结果 | — |

### 9.2 错误状态渲染（`showError`）

错误发生时固定头部显示：

| 区域 | 内容 |
| --- | --- |
| 升数区 | `⚠️ <small>ERROR</small>` |
| 金额区 | `请检查输入` |
| 明细区 | 具体错误文案 |

同时对应面板标题加上 `.v-expansion-panel-err` 错误底色，帮助用户定位问题所在面板。

---

## 10. 数据持久化设计

### 10.1 存储键

| localStorage 键 | 内容 | 写入时机 |
| --- | --- | --- |
| `fuel_calculator_pro` | 全部输入值的 JSON | 每次输入变化 / 重置后 |
| `config_panel_open` | `'true'` / `'false'` | 配置面板展开/收起时 |
| `discount_panel_open` | `'true'` / `'false'` | 折扣面板展开/收起时 |

### 10.2 数据结构

`fuel_calculator_pro` 存储的 JSON 结构：

```json
{
    "tankCapacity": "45",
    "reserveFuel": "5",
    "remainingRange": "120",
    "avgFuelConsumption": "7.5",
    "fuelPrice": "7.89",
    "discountPerLiter": "0.5",
    "pumpAmount": "200",
    "lastUpdated": "2026-10-27T07:00:00.000Z"
}
```

> 注：所有输入值以**字符串**形式保存（直接取自 `input.value`）；`lastUpdated` 为 ISO 8601 时间戳，用于记录最后更新时间。

### 10.3 读取与容错

- `loadFromLocalStorage()` 使用 `try/catch` 包裹 `JSON.parse`，解析失败时打印警告并回退默认值；
- 各字段使用 `!== undefined` 判断，缺失字段回退默认值（隐藏容积默认 `'5'`，其余为空）；
- 恢复数据后逐个调用 `updateFloatingLabel` 刷新浮动标签状态；
- 所有 `localStorage.setItem` 均包裹 `try/catch`（隐私模式等场景下可能抛出异常）。

---

## 11. 事件流与交互流程

### 11.1 事件绑定总表

| 目标 | 事件 | 处理函数 | 说明 |
| --- | --- | --- | --- |
| 7 个输入框 | `input` | `onInputChange` / 匿名（折扣项） | 触发重算 + 保存 |
| 7 个输入框 | `focus` / `blur` / `input` | `updateFloatingLabel`（经 `bindFloatingLabel`） | 浮动标签 |
| 5 个主输入框 | `focus` / `blur` | `setTimeout(adjustForKeyboard, 50)` | 键盘适配 |
| `configPanelHeader` | `click` | `toggleConfigPanel` | 折叠/展开 |
| `discountPanelHeader` | `click` | `toggleDiscountPanel` | 折叠/展开 |
| `resetBtn` | `click` | `showResetConfirm` | 打开确认框 |
| `cancelResetBtn` | `click` | `hideResetConfirm` | 关闭确认框 |
| `confirmResetBtn` | `click` | `performReset` | 执行重置 |
| `resetConfirmSheet` | `click`（仅点击遮罩自身） | `hideResetConfirm` | 点击空白关闭 |
| 清空按钮（`.v-field__clearable`） | `pointerdown` | `handleClearPointerDown` | 清空对应输入框 |
| `window` | `resize` / `load` | `updatePlaceholderHeight` | 重算占位高度 |
| `window.visualViewport` | `resize` / `scroll` | `requestAnimationFrame(adjustForKeyboard)` | 键盘适配 |
| `fixedHeader` | MutationObserver | `updatePlaceholderHeight` | 结果区高度变化时同步占位 |
| `document` | `DOMContentLoaded` | `initAllClearable` | 初始化清空按钮 |

### 11.2 清空按钮交互流程（`handleClearPointerDown`）

```
pointerdown
  ├── preventDefault + stopPropagation（防止输入框失焦/键盘收起）
  ├── 读取 data-target 找到目标输入框
  ├── 记录目标当前是否聚焦（wasFocused）
  ├── 清空输入框值
  ├── dispatch input 事件（冒泡）→ 触发重算 + 保存 + 可见性更新
  ├── 更新浮动标签状态
  ├── 若原先聚焦 → 重新 focus（保持键盘）
  └── 播放滑出动画（150ms 后移除动画类）
```

### 11.3 重置流程

```
点击重置按钮
  → showResetConfirm()：Bottom Sheet 加 .active（遮罩淡入 + 面板上滑）
      ├── 点击"取消" / 点击遮罩空白 → hideResetConfirm()
      └── 点击"确认清空" → performReset()
            ├── 清空全部输入（reserveFuel 恢复 '5'）
            ├── 刷新全部浮动标签
            ├── saveToLocalStorage()
            ├── calculateAndDisplay()
            └── hideResetConfirm()
```

---

## 12. 移动端适配设计

### 12.1 虚拟键盘适配（核心难点）

移动端虚拟键盘弹起时会压缩可视视口，可能导致固定结果区被推出屏幕。本应用基于 **`window.visualViewport` API** 实现自适应：

**关键变量：**

- `originalMsgBottom`：页面加载时记录的 `fixedHeader.getBoundingClientRect().bottom`（结果区原始底部位置）。

**判断逻辑（`isHeaderFullyVisible`）：**

- 若 `visualViewport.offsetTop >= originalMsgBottom / 3`，认为结果区已被键盘推出视口（偏移量超过结果区高度的 1/3）。

**调整逻辑（`adjustForKeyboard`）：**

```
键盘高度 = window.innerHeight − visualViewport.height

若 键盘高度 > 100 且 结果区被推出视口：
    newTop = visualViewport.offsetTop + 8（补偿 margin-top）
    边界钳制：若 newTop + 头部高度 + 20 > 可视高度，
              则 newTop = max(0, 可视高度 − 头部高度 − 20)
    fixedHeader.style.top = newTop
否则：
    fixedHeader.style.top = 0（恢复原位）
```

**触发时机：**

- `visualViewport` 的 `resize` 与 `scroll` 事件（均经 `requestAnimationFrame` 节流）；
- 输入框 `focus` / `blur` 后延迟 50ms；
- 初始化后延迟 100ms。

### 12.2 占位高度同步

由于 `fixed-header` 为固定定位会脱离文档流，使用 `headerPlaceholder` 占位元素撑开相应空间：

- 触发时机：`window.resize`、`window.load`、`fixedHeader` 的 MutationObserver（属性/子节点变化）、初始化后 100ms；
- 实现：`headerPlaceholder.style.height = fixedHeader.offsetHeight + 'px'`。

### 12.3 其他移动端细节

| 细节 | 实现 |
| --- | --- |
| 视口高度 | `min-height: 100dvh` 兼容地址栏收缩 |
| 触摸高亮 | 清空按钮 `-webkit-tap-highlight-color: transparent` |
| 数字键盘 | 输入框 `type="number"` + `step` 控制精度 |
| 旋转按钮 | 保留 `::-webkit-inner/outer-spin-button`（`opacity: 0.5`） |
| 安全区 | 小屏下 Bottom Sheet 底部预留 `20dvh` |

---

## 13. 错误处理设计

| 场景 | 处理方式 |
| --- | --- |
| 配置项非法（容积/油耗） | 顶部显示 ⚠️ ERROR + 具体文案，配置面板标题标红，折扣结果隐藏（传入油价 0） |
| 剩余续航非法 | 顶部显示 ⚠️ ERROR + 文案，折扣结果仍尝试按当前油价更新 |
| 油价非法 | 静默按 0 处理，不报错 |
| 折扣数据不足 | 隐藏折扣结果 Chip，清除折扣面板错误态 |
| `localStorage` 读写失败 | `try/catch` 静默吞掉（读取失败额外 `console.warn`），回退默认值 |
| `visualViewport` 不支持 | 相关判断直接返回安全默认值，不做调整 |

---

## 14. 初始化流程

页面脚本执行顺序：

```
1. 获取全部 DOM 引用
2. 记录 originalMsgBottom（结果区初始底部位置）
3. 绑定 visualViewport resize/scroll → adjustForKeyboard
4. 绑定 5 个主输入框 focus/blur → adjustForKeyboard
5. setTimeout(adjustForKeyboard, 100)
6. 绑定 window resize/load → updatePlaceholderHeight
7. 创建 MutationObserver 监听 fixedHeader 变化 → updatePlaceholderHeight
8. 绑定配置面板/折扣面板标题点击事件
9. 绑定全部 7 个输入框的浮动标签事件
10. 绑定输入变化 → onInputChange（重算 + 保存）
11. 绑定重置流程事件
12. document.addEventListener('DOMContentLoaded', initAllClearable)
13. initConfigPanelState()    ← 恢复配置面板展开状态
14. initDiscountPanelState()  ← 恢复折扣面板展开状态
15. loadFromLocalStorage()    ← 恢复输入数据
16. calculateAndDisplay()     ← 首次计算渲染
17. 若无任何关键数据 → 显示欢迎文案：
    "✨ 请在上方输入车辆数据，将自动保存"
18. setTimeout(updatePlaceholderHeight, 100)
```

**初始默认状态：**

| 项 | 默认值 |
| --- | --- |
| 隐藏油箱容积 | `5` |
| 其余输入框 | 空 |
| 两个面板 | 默认展开（除非 localStorage 记录为 `'false'`） |
| 结果区 | `— 升` / `—` / "输入数据后自动计算" |

---

## 15. 已知限制与优化建议

| 编号 | 现状 | 建议 |
| --- | --- | --- |
| 1 | `updateDiscountResults` 中 `fuelPrice <= 0` 的分支因前置条件 `hasDiscountData` 已要求 `fuelPrice > 0` 而不可达（死代码） | 可移除该分支或重构判断顺序 |
| 2 | `handleClearClick`（click 版清空实现）已定义但未绑定 | 确认无用后可删除 |
| 3 | 键盘适配的输入框列表仅包含 5 个主输入框，未含折扣面板的 2 个输入框 | 如需一致体验可将折扣输入框加入 `allInputs` |
| 4 | 面板内容 `max-height: 500px` 为硬编码上限 | 内容增多时可改用 `scrollHeight` 动态设置 |
| 5 | 金额显示区未带货币符号（仅数字） | 可按需补充 `¥` 前缀 |
| 6 | 无单元测试 | 可将计算函数抽离为纯函数并补充测试 |

---

## 16. 附录

### 16.1 版本历史（依据 Release 目录）

| 版本 | 文件 |
| --- | --- |
| 2.1 | `Release/fuelup_calc.2.1.html` |
| 2.5 | `Release/fuelup_calc.2.5.html` |

### 16.2 图标资源

全部图标为内联 SVG（Material Design 风格 path 数据），无外部依赖：

| 图标 | 使用位置 |
| --- | --- |
| 齿轮（settings） | 配置面板标题 |
| 标签（tag/label） | 折扣面板标题 |
| 箭头（chevron-up） | 面板展开指示 |
| 圆形叉（close-circle） | 输入框清空按钮 |
| 旋转箭头（refresh） | 重置按钮 |
| 加油泵（favicon） | 页面图标 |

### 16.3 术语表

| 术语 | 说明 |
| --- | --- |
| 隐藏油箱容积 | 油表显示为空时油箱内实际剩余的备用燃油量 |
| 续航油量 | 依据剩余续航里程与平均油耗推算的当前油箱内燃油量 |
| 可加油量 | 油箱总容积 − 剩余总油量，即加满所需油量 |
| 每公里花费 | 汽油单价 × 平均油耗 ÷ 100，即每行驶一公里的燃油成本 |
| 折扣每公里花费 | (汽油单价 − 每升优惠价) × 平均油耗 ÷ 100，即享受优惠后每公里燃油成本 |
| 支付百分比 | 享受每升优惠后实际支付金额占原价的比例 |
| visualViewport | 浏览器提供的可视视口 API，可感知虚拟键盘引起的视口变化 |
| Bottom Sheet | 从屏幕底部滑出的面板，常用于确认类交互 |

---

*文档结束*