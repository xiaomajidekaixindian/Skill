---
name: electron-frameless-window
description: "用于为 Electron 应用实现无边框（frameless）窗口与自绘标题栏，覆盖四条实现路线（frame:false 全自绘 / titleBarStyle:hidden + titleBarOverlay 系统按钮回填 / hiddenInset macOS 内嵌 / transparent 异形窗口）及平台分支、Windows 11 材质、macOS 毛玻璃、主题联动、全屏与降级处理。当任务涉及隐藏系统标题栏、自定义标题栏、把标题栏和界面融为一体、titleBarStyle、titleBarOverlay、trafficLightPosition、vibrancy、backgroundMaterial、-webkit-app-region 拖拽区，或排查窗口不能拖动、不能拉边缘改大小、双击标题栏不还原、还原后窗口过大、系统按钮与自绘内容重叠、全屏后多出一条空横栏等问题时使用。也用于把已有项目从 frame:false 或原生标题栏迁移到系统能力不受损的方案。"
agent_created: true
---

# Electron 无边框窗口

## 用途

把 Electron 窗口的标题栏去掉、与界面融为一体，同时**不牺牲系统能力**（拉边缘改大小、双击还原、Aero Snap、红黄绿灯、Win11 Snap Layouts）。

核心判断：无边框的难点不在「怎么去掉标题栏」，而在**去掉之后付出了什么代价、代价能不能不付**。

## 四条路线速查

| 路线 | 核心配置 | 保留 | 代价 | 何时选 |
|---|---|---|---|---|
| **A · 全自绘** | `frame: false` | 无，全部自己实现 | resize 边框、双击还原、Snap Layouts、macOS 红绿灯全丢 | 游戏启动器、异形窗口、完全自定义 |
| **B · 系统按钮回填** | `titleBarStyle: 'hidden'` + `titleBarOverlay` | 原生 resize、Aero Snap、双击还原、系统三按钮（可改色/高度） | 标题栏文字需自绘 | **默认首选**（Windows / Linux） |
| **C · 内嵌式** | `titleBarStyle: 'hiddenInset'` + `trafficLightPosition` | macOS 红绿灯保留并内缩 | 仅 macOS 有效 | macOS 风格产品 |
| **D · 异形窗口** | `transparent: true` + `frame: false` | 可做圆角 / 不规则 / 半透明 | 性能下降、Windows 上不能 resize、易闪烁 | 只有必须异形时才用 |

判据：**只要一句「看起来没标题栏」，选 B。** 选 A 必须有明确理由；选 D 先确认放弃 resize 可接受。

叠加项（与上面不互斥，按需组合）：

| 叠加项 | 作用 | 平台 |
|---|---|---|
| `titleBarStyle: 'customButtonsOnHover'` | 红绿灯平时隐藏，鼠标移到左上角才浮现 | 仅 macOS，需较新 Electron |
| `vibrancy: 'under-window'` + `visualEffectState: 'active'` | 毛玻璃背景 | 仅 macOS |
| `backgroundMaterial: 'mica' \| 'acrylic'` | Win11 原生材质 | Windows，需 Electron 30+ |
| `thickFrame: false` | 关闭 resize 边框与窗口阴影 | Windows |

## 工作流

### 第一步：侦察现状（只读，不改代码）

改代码前先确认以下事实，否则容易做出「两套按钮」或重复实现：

1. 主进程入口路径 + 现有 `BrowserWindow` 完整配置
2. Electron 版本（`titleBarOverlay` 需 22+ 稳定，`backgroundMaterial` 需 30+）
3. 脚手架类型（electron-vite / electron-forge / vite-plugin-electron / 纯 electron）
4. 渲染层根布局（是否已有标题栏组件、主体是否纵向 flex）
5. 是否已存在 `frame: false` 或自绘窗口按钮 —— **已有 `frame: false` 时必须先删掉自绘的最小化 / 最大化 / 关闭按钮**，否则换成路线 B 后会出现两套按钮
6. 是否已有 window 相关 IPC / preload 暴露

### 第二步：选路线并确认平台分支

按上表选路线。**平台分支不能只写 Windows**：

```
macOS     → hiddenInset / hidden + trafficLightPosition + 可选 vibrancy
Windows   → hidden + titleBarOverlay { color, symbolColor, height }
Linux     → hidden + titleBarOverlay（兼容性差时退回原生标题栏）
```

### 第三步：实施

复制 `assets/` 下的模板按需改，不要从零写：

| 文件 | 用途 |
|---|---|
| `assets/main.basic.ts` | 路线 B + C 最小可用版，窗口工厂 |
| `assets/main.pro.ts` | 生产级：主题联动、全屏事件、降级开关、窗口状态持久化、材质 |
| `assets/preload.ts` | 白名单 IPC（平台标识、全屏切换、窗口控制） |
| `assets/TitleBar.vue` | Vue 3 + TS 自绘标题栏，含拖拽区与系统按钮留白 |
| `assets/titlebar.css` | 非 Vue 项目的纯 CSS 等价实现 |

实施时守五条铁律：

1. **高度三处一致**：`titleBarOverlay.height` = 标题栏 CSS `height` = 布局预留高度。任何一处不一致，自绘内容与系统按钮就会错位 —— 最高频翻车点。
2. **颜色同源**：`titleBarOverlay.color` 与标题栏背景用同一常量，否则顶部出现接缝。支持深浅主题时用 `nativeTheme.shouldUseDarkColors` 驱动，并在 `nativeTheme.on('updated')` 里调 `win.setTitleBarOverlay()` 刷新。
3. **拖拽区**：标题栏容器 `-webkit-app-region: drag`；容器内所有可点元素（按钮、输入框、下拉）必须 `-webkit-app-region: no-drag`；标题栏加 `user-select: none`。
4. **为系统按钮留白**：Windows 右侧显式预留约 140px，macOS 左侧预留约 82px；写进布局，不要靠肉眼调。
5. **安全基线不降级**：`contextIsolation: true`、`nodeIntegration: false`，窗口能力走 preload + `contextBridge` 白名单，主进程用 `BrowserWindow.fromWebContents(event.sender)` 取窗口。

### 第四步：验收

改完必须逐条验证，不以「编译通过」或「看起来对」结案。详见 `references/checklist.md`，含手动清单与 Playwright 自动化断言片段。

必查九条：能拉边缘改大小、双击标题栏还原、还原尺寸与初始一致、左侧按钮可点、顶部无接缝、系统按钮不重叠、全屏后不多出一条空横栏、macOS 红绿灯未被遮挡、`window.require === undefined`。

## 翻车点速查

| 症状 | 根因 | 修法 |
|---|---|---|
| 窗口不能拉边缘改大小 / 不能双击还原 | 裸用 `frame: false` | 改用 `titleBarStyle: 'hidden'` + `titleBarOverlay` |
| 顶部有一条色差接缝 | `titleBarOverlay.color` 与标题栏背景不同 | 抽成同一颜色常量 |
| 系统按钮和自绘内容重叠 | 未预留右侧空间 / 高度不一致 | 布局预留 + 三处高度统一 |
| 自绘按钮点不动 | 父级 `drag` 覆盖了按钮 | 按钮加 `no-drag` |
| 出现两套最小化 / 关闭按钮 | 从 `frame: false` 迁移时没删旧按钮 | 删除自绘按钮 |
| 全屏后顶部多一条空横栏 | 未处理 `enter-full-screen`，标题栏仍渲染 | 主进程转发全屏状态，渲染层 `v-if` 隐藏 |
| 还原后窗口尺寸过大 | 初始尺寸按屏幕写死，未按工作区收缩 | `workAreaSize` + `clamp` + `minWidth/minHeight` |
| Linux 上按钮错位或消失 | `titleBarOverlay` 在 Linux 支持不完整 | 检测失败时退回原生标题栏 |
| 窗口拖动几次后内容拖不动 | 通常不是窗口问题，是业务组件自身状态锁死 | 先排除业务组件，再做单变量实验 |

## 资源索引

- `references/routes.md` —— 四条路线完整配置、平台矩阵、进阶叠加项、路线 A 自绘按钮实现
- `references/checklist.md` —— 验收清单 + Playwright 自动化断言
- `assets/` —— 可直接复制的窗口工厂、preload、标题栏组件与样式

## 边界

- 不做「无边框 + 透明异形」之外的窗口特效；毛玻璃与材质只在本机平台原生支持时启用
- 不引入 UI 框架依赖，模板只用 Electron 原生 API
- 页面内路由与外壳装配不在本职责内，涉及多页面骨架时配合 `page-registry-hash-router` 使用
