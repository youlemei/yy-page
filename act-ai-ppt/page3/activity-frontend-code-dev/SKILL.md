---
name: activity-frontend-code-dev
description: "活动前端代码开发和自测技能。用于用户已有前端技术方案、PRD 或明确组件需求，要求在 EMP/Astro 项目中创建组件、等待或接入 F2C 导入产物、开发弹窗/挂件/H5 页面、接入 HTTP/PB/MFApi、补 Mock、本地 pnpm dev 调试、Playwright 截图验证、pnpm build/build:unpkg 自测和排查 CI 构建问题时使用。触发语包括：按方案开发前端、开始前端编码、接入F2C、完善数据交互、本地自测、检查弹窗样式、EMP/Astro开发。"
---

# 活动前端开发自测

## 工作方式

从技术方案和仓库现状出发，完成“创建组件 -> 等待/接入 F2C -> 业务开发 -> Mock -> 本地浏览器自测 -> 构建验证”。不要停留在建议；除非用户只要求方案或提问，默认实际改代码。

涉及组件创建时先使用 `astro-development` 技能。涉及 F2C 职责边界时参考 `emp-astro-f2c-development`。如果用户说 F2C 还没导入，不要生成视图层代码；先完成外层结构或等待用户导入。

## 开发前检查

先做这些检查：
- `git status --short`，确认已有改动，不能回滚用户改动
- 当前分支和目标分支是否一致
- `package.json`、`emp.config.ts`、`showCaseConfig.ts`
- 目标组件目录是否已存在
- F2C 子目录是否已人工导入
- 参考项目是否只读
- 现有工具：`pnpm dev`、`pnpm build`、`pnpm build:unpkg`

如果本地 `rg` 不可用，改用 PowerShell `Get-ChildItem | Select-String`。

## 组件创建

在目标项目根目录执行：

```bash
npm init @astro/astro@latest --registry=https://npm-registry.yy.com new component <ComponentName> -- --useDefault
```

创建后检查：
- `src/components/<ComponentName>/`
- `emp.config.ts` exposes
- `showCaseConfig.ts`
- `form.json`
- `astro.desc.json`

Layout 组件要确认 `astro.desc.json`、`slot.json`。不要手写重复注册，除非 CLI 没有完成。

## F2C 接入规则

F2C 是人工从 Figma 插件导入的静态 React 代码：
- 不提前创建名为 `f2c` 的目录
- 子目录名使用实际 React 名称
- 不无故改 F2C 的 DOM/CSS 布局
- 用户重新导入 F2C 后，只重新接数据和事件，保留设计稿布局
- 如果必须改 CSS，先说明原因，优先局部追加，不大面积重排

接入方式：
- 小弹窗可由父组件传 `data/onClose/onConfirm/onOpenDetail`
- 大页面或复杂视图可在 F2C 子目录内直接读 Store、渲染列表、处理 tab、局部弹窗
- 根组件负责 URL 参数解析、MFApi/PB 初始化、跨组件打开、关闭窗口和生命周期

## 数据和交互实现

常见文件职责：

```text
index.tsx       # 外层编排、props/form、URL参数、渲染分发
store.ts        # Valtio状态、动作、防重复提交、reset/close
service.ts      # PB订阅、PB请求、回包处理
api.ts          # HTTP请求
types.ts        # 协议类型
mockData.ts     # 开发态mock
<F2CReactName>/ # 视图层数据绑定和局部交互
```

实现时注意：
- URL 参数里的 JSON/Base64 要防御式解析
- `noticeType` 后可能带 `:{cmptUseInx}`，按需求决定前缀匹配或精确匹配
- PB 广播要过滤 `actId/cmptUseInx/bannerId`
- 关闭窗口区分 PC/移动端已有项目写法
- 提交类动作要防重复
- 倒计时、动画、请求要在 unmount 或关闭时清理
- 本地开发无真实 `actId/serviceId` 时补 development mock，不影响生产路径

## Mock 约定

为每个可视状态准备可直接访问的 mock：
- 弹窗：`?mockType=intro|started|reward1|bingo|none`
- 挂件：无 `actId/serviceId` 时 fallback mock
- Layout：开发态无 children 时可注入代表性子组件
- 页面：当前、历史、空态、异常态

Mock 只在 `process.env.NODE_ENV === 'development'` 生效。

## 本地启动

优先：

```bash
pnpm dev
```

如果缓存异常，可在用户确认后删除：

```powershell
Remove-Item -LiteralPath node_modules\.cache -Recurse -Force
```

后台启动时用日志文件，便于排错：

```powershell
$log = Join-Path $env:TEMP 'widget-sample-pnpm-dev.log'
$cmd = 'cd /d "<ProjectRoot>" && "<pnpm.cmd>" dev > "' + $log + '" 2>&1'
Start-Process -FilePath "cmd.exe" -ArgumentList @('/c', $cmd) -WindowStyle Hidden
```

检查端口：

```powershell
Get-NetTCPConnection -LocalPort 8801,8802 -ErrorAction SilentlyContinue
```

如果有临时代理或旧 dev server，先确认进程和端口，不要误杀无关进程。

## 浏览器自测

本地 HTTPS 证书常会报错。不要在浏览器里替用户点击安全拦截。需要自动化时用 Playwright CLI：

```bash
npx playwright open --ignore-https-errors https://localhost:8802/<ComponentName>
npx playwright screenshot --ignore-https-errors --viewport-size 900,900 --wait-for-timeout 1800 "<url>" ".playwright-cli/<name>.png"
```

必须验证：
- 页面不是空白
- F2C 素材加载
- 文案换行和设计稿一致
- 主要按钮可点击
- 关闭行为可触发
- mock 状态覆盖完整
- 控制台没有业务错误；本地 websocket 证书错误可单独说明

对于 3D/canvas 或动画页面，还要用截图确认非空白、位置正确、移动端/桌面都可见。

## 构建和发布前检查

至少跑：

```bash
pnpm build
```

发布链路相关再跑：

```bash
pnpm build:unpkg
```

只把 asset size warning 当 warning 处理，除非需求要求优化体积。

CI 排查顺序：
- 找第一个失败命令，不要被后续级联错误误导
- `pkgcare inc` 失败会导致 `build:unpkg` 不执行，随后 `astro sync` 可能报缺 `emp.json`
- 分支名带 `_` 时注意内部 `pkgcare` 可能把版本里的 `_` 去掉，引发 tag 校验问题
- 不要本地执行 `npm publish`，除非用户明确要求

## 完成标准

交付前确认：
- 代码改动与目标项目一致，没有改参考项目
- F2C 布局未被无关改动破坏
- mock URL 可打开
- 核心交互走通
- `pnpm build` 通过
- 必要时 `pnpm build:unpkg` 通过
- final response 写明改动文件、验证命令、剩余风险
