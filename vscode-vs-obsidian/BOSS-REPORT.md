# RFC-001: Agent Coding App 架构设计

- **RFC 编号**: 001
- **标题**: Agent Coding App 架构设计
- **作者**: 架构组
- **状态**: 🟡 Draft
- **创建日期**: 2026-01-19
- **最后更新**: 2026-01-19

---

## 概述

基于 Electron 构建的 Agent Coding 桌面应用，具备 AI 对话与代码编辑能力。

---

## 动机

### 产品定位

我们要构建一个 **Agent Coding App**：

- **核心能力**：AI Agent 驱动的编程体验，用户通过对话完成编码任务
- **编辑器**：具备基础的代码编辑能力（语法高亮、编辑、Diff 预览）
- **可扩展性**：架构设计需要支持未来的功能扩展和插件生态

### 为什么需要架构决策

为了实现可扩展性，我们需要在开发前确定核心架构。我们研究了两个成熟的 Electron 应用作为参考：

| 参考对象 | 特点 | 研究价值 |
|----------|------|----------|
| **VSCode** | 企业级架构、多进程、丰富插件生态 | 学习可扩展性设计 |
| **Obsidian** | 轻量快速、单进程、社区驱动 | 学习简洁架构 |

共完成 **35 个技术领域**的深度研究（详见 `research/vscode-vs-obsidian/`），为以下架构决策提供依据。

---

## 目标

1. **MVP 可用**：3 个月内交付可用的桌面应用
2. **核心体验**：流畅的 AI 对话 + 专业的代码编辑
3. **可扩展**：架构预留插件能力
4. **跨平台**：支持 Windows / macOS / Linux

## 非目标

1. ❌ 本阶段不做 Web 版本
2. ❌ 本阶段不做移动端
3. ❌ 本阶段不对外开放插件系统
4. ❌ 不追求功能对标 VSCode

---

## 提案

按研究领域分类，共 35 个技术决策点。

---

### 一、基础架构

#### 1. 进程架构
> 📖 [01-process-architecture.md](./01-process-architecture.md)

| VSCode | Obsidian |
|--------|----------|
| 多进程：Main + Renderer + Extension Host | 单进程：Main + Renderer |
| 插件独立进程，崩溃隔离 | 插件主线程运行 |

**推荐**：混合架构（UI/插件单进程 + AI Worker 独立）
**理由**：AI 耗时操作独立，插件保持简单

---

#### 2. 模块系统
> 📖 [02-module-system.md](./02-module-system.md)

| VSCode | Obsidian |
|--------|----------|
| AMD (require) 历史遗留 + ESM 迁移中 | ESM 原生 |
| 复杂的模块加载器 | 简单直接 |

**推荐**：纯 ESM
**理由**：现代标准，工具链支持好，无历史包袱

---

#### 3. IPC 通信
> 📖 [03-ipc-communication.md](./03-ipc-communication.md)

| VSCode | Obsidian |
|--------|----------|
| MessagePort + Protocol Buffer | EventEmitter |
| 复杂的 RPC 协议 | 简单事件 |

**推荐**：Electron IPC + MessagePort（AI Worker）
**理由**：Main-Renderer 用 Electron IPC，AI Worker 用 MessagePort

---

### 二、插件系统

#### 4. 插件 API 设计
> 📖 [04-plugin-api-design.md](./04-plugin-api-design.md)

| VSCode | Obsidian |
|--------|----------|
| 命名空间分组 `vscode.window.*` | 单一 App 对象入口 |
| 数百个 API | 几十个核心 API |

**推荐**：Obsidian 风格，单一 App 入口
**理由**：API 简洁，学习成本低

---

#### 5. 扩展点机制
> 📖 [05-extension-points.md](./05-extension-points.md)

| VSCode | Obsidian |
|--------|----------|
| package.json 声明式 contributes | 代码运行时注册 |
| 支持懒加载 | 无懒加载 |

**推荐**：运行时注册为主，简单声明为辅
**理由**：灵活性优先，MVP 不需要复杂的懒加载

---

#### 6. 插件生命周期
> 📖 [06-plugin-lifecycle.md](./06-plugin-lifecycle.md)

| VSCode | Obsidian |
|--------|----------|
| 懒激活（onCommand/onLanguage 等） | 启动时全部加载 |
| 复杂的激活事件 | 简单的 onload/onunload |

**推荐**：启动时加载，简单生命周期
**理由**：MVP 插件少，无需懒加载优化

---

#### 7. 插件间通信
> 📖 [07-plugin-communication.md](./07-plugin-communication.md)

| VSCode | Obsidian |
|--------|----------|
| 通过 API 暴露服务 | 全局事件 + App 对象 |
| 显式依赖声明 | 隐式共享 |

**推荐**：事件总线 + 共享服务
**理由**：简单直接，满足内部插件通信需求

---

### 三、UI 系统

#### 8. UI 布局
> 📖 [08-ui-layout.md](./08-ui-layout.md)

| VSCode | Obsidian |
|--------|----------|
| 固定区域（侧边栏+编辑器+面板） | 灵活分栏（Workspace Leaf） |
| 布局受限但清晰 | 高度灵活但复杂 |

**推荐**：固定三栏（侧边栏 + Chat + Editor）
**理由**：场景明确，降低复杂度

---

#### 9. 视图系统
> 📖 [09-view-system.md](./09-view-system.md)

| VSCode | Obsidian |
|--------|----------|
| ViewContainer + View 注册 | ItemView 类继承 |
| Webview 支持 | 原生 DOM |

**推荐**：React 组件注册
**理由**：现代前端开发体验，组件化

---

#### 10. 主题系统
> 📖 [10-theme-system.md](./10-theme-system.md)

| VSCode | Obsidian |
|--------|----------|
| JSON 主题 + TextMate 语法 | CSS 变量 + CSS 片段 |
| 细粒度（数百变量） | 中等粒度 |

**推荐**：CSS 变量 + 深色/浅色双主题
**理由**：简单实用，满足基本需求

---

### 四、交互系统

#### 11. 命令系统
> 📖 [11-command-system.md](./11-command-system.md)

| VSCode | Obsidian |
|--------|----------|
| Command Palette 核心 | Command Palette |
| 命令 ID + 参数 | 命令 ID + 回调 |

**推荐**：命令注册 + Command Palette
**理由**：标准交互模式，用户熟悉

---

#### 12. 快捷键系统
> 📖 [12-keybinding-system.md](./12-keybinding-system.md)

| VSCode | Obsidian |
|--------|----------|
| JSON 配置 + when 条件 | Hotkey 管理器 |
| 复杂的上下文条件 | 简单绑定 |

**推荐**：简单快捷键绑定，支持自定义
**理由**：MVP 无需复杂的上下文条件

---

#### 13. 右键菜单
> 📖 [13-context-menu.md](./13-context-menu.md)

| VSCode | Obsidian |
|--------|----------|
| menu contributes 声明式 | Menu 类 API |
| 按区域分组 | 运行时构建 |

**推荐**：运行时注册菜单项
**理由**：灵活，与插件系统一致

---

### 五、编辑器核心

#### 14. 编辑器引擎
> 📖 [14-editor-engine.md](./14-editor-engine.md)

| Monaco (VSCode) | CodeMirror (Obsidian) |
|-----------------|----------------------|
| 包大小 ~2.5MB | 包大小 ~500KB |
| VSCode 同款体验 | 轻量可定制 |
| 大文件优秀 | 移动端友好 |

**推荐**：Monaco Editor
**理由**：产品定位需要专业代码编辑体验

---

#### 15. 编辑器扩展
> 📖 [15-editor-extension.md](./15-editor-extension.md)

| VSCode/Monaco | CodeMirror |
|---------------|------------|
| Language API + Providers | Extension + Facet |
| 丰富的语言支持 | 模块化扩展 |

**推荐**：Monaco Language API
**理由**：与编辑器引擎选择一致

---

#### 16. 装饰系统
> 📖 [16-decoration-system.md](./16-decoration-system.md)

| VSCode/Monaco | CodeMirror |
|---------------|------------|
| deltaDecorations API | Decoration + StateField |
| 高性能增量更新 | 响应式更新 |

**推荐**：Monaco Decorations
**理由**：用于 AI 修改高亮、错误标记等

---

### 六、数据与状态

#### 17. 状态管理
> 📖 [17-state-management.md](./17-state-management.md)

| VSCode | Obsidian |
|--------|----------|
| 服务类 + 依赖注入 | 全局 App 对象 |
| 复杂但可测试 | 简单但耦合 |

**推荐**：Zustand
**理由**：现代方案，简洁且可维护

---

#### 18. 设置系统
> 📖 [18-settings-system.md](./18-settings-system.md)

| VSCode | Obsidian |
|--------|----------|
| JSON Schema 定义 | Setting Tab UI |
| 配置贡献点 | 插件自定义 |

**推荐**：Settings UI + JSON 存储
**理由**：用户友好，开发简单

---

#### 19. 文件系统
> 📖 [19-file-system.md](./19-file-system.md)

| VSCode | Obsidian |
|--------|----------|
| FileSystemProvider 抽象 | Vault API |
| 支持远程（SSH/WSL） | 仅本地 |

**推荐**：直接文件操作（Node.js fs）
**理由**：MVP 仅本地，无需抽象层

---

#### 20. 缓存与持久化
> 📖 [20-cache-persistence.md](./20-cache-persistence.md)

| VSCode | Obsidian |
|--------|----------|
| Memento API + SQLite | localStorage + 文件 |
| 工作区/全局分离 | 简单键值 |

**推荐**：Electron Store + 文件系统
**理由**：简单可靠，满足基本需求

---

### 七、开发者功能

#### 21. 语言服务
> 📖 [21-language-service.md](./21-language-service.md)

| VSCode | Obsidian |
|--------|----------|
| LSP 完整支持 | 无标准协议 |
| 丰富的语言功能 | 基础补全 |

**推荐**：Monaco 内置 + AI 补全
**理由**：基础语言功能用 Monaco，智能补全用 AI

---

#### 22. 调试功能
> 📖 [22-debug-capability.md](./22-debug-capability.md)

| VSCode | Obsidian |
|--------|----------|
| DAP 完整支持 | 无 |
| 丰富的调试 UI | - |

**推荐**：MVP 不做，后续考虑
**理由**：非核心功能，AI 可协助调试

---

#### 23. 终端集成
> 📖 [23-terminal-integration.md](./23-terminal-integration.md)

| VSCode | Obsidian |
|--------|----------|
| xterm.js 完整集成 | 无 |
| 多终端/分屏 | - |

**推荐**：AI 代理执行命令
**理由**：符合 Agent 产品理念，用户通过对话执行命令

---

#### 24. 搜索功能
> 📖 [24-search-capability.md](./24-search-capability.md)

| VSCode | Obsidian |
|--------|----------|
| ripgrep 集成 | 内置搜索 |
| 正则/文件过滤 | 全文搜索 |

**推荐**：基础文件搜索 + AI 语义搜索
**理由**：结合传统搜索和 AI 能力

---

#### 25. 版本控制
> 📖 [25-version-control.md](./25-version-control.md)

| VSCode | Obsidian |
|--------|----------|
| SCM API + Git 扩展 | 社区插件 |
| 完整 Git UI | 基础支持 |

**推荐**：基础 Git 状态显示 + AI 辅助
**理由**：AI 可以帮助执行 Git 命令

---

### 八、生态系统

#### 26. 插件市场
> 📖 [26-plugin-marketplace.md](./26-plugin-marketplace.md)

| VSCode | Obsidian |
|--------|----------|
| 官方市场 + 审核 | 社区市场 |
| 企业级基础设施 | 轻量运营 |

**推荐**：MVP 不做，后续考虑
**理由**：先专注核心功能

---

#### 27. 开发者体验
> 📖 [27-developer-experience.md](./27-developer-experience.md)

| VSCode | Obsidian |
|--------|----------|
| 完善文档 + 脚手架 | 示例 + 社区文档 |
| yo generator | 模板仓库 |

**推荐**：内部文档 + 插件模板
**理由**：MVP 阶段内部开发为主

---

### 九、安全与性能

#### 28. 安全模型
> 📖 [28-security-model.md](./28-security-model.md)

| VSCode | Obsidian |
|--------|----------|
| Workspace Trust + 进程隔离 | 最小沙箱 |
| 细粒度权限 | 信任用户 |

**推荐**：简单确认机制
**理由**：AI 操作需确认，插件暂不开放

---

#### 29. 启动性能
> 📖 [29-startup-performance.md](./29-startup-performance.md)

| VSCode | Obsidian |
|--------|----------|
| 2-4 秒（懒加载优化） | <1 秒 |
| 复杂的启动优化 | 轻量架构 |

**推荐**：保持简单架构
**理由**：架构简单自然启动快

---

#### 30. 运行时性能
> 📖 [30-runtime-performance.md](./30-runtime-performance.md)

| VSCode | Obsidian |
|--------|----------|
| 虚拟列表/懒渲染 | 按需渲染 |
| 复杂的性能监控 | 轻量实现 |

**推荐**：按需优化
**理由**：先保证功能，出现问题再针对性优化

---

### 十、平台与体验

#### 31. 多平台支持
> 📖 [31-multi-platform.md](./31-multi-platform.md)

| VSCode | Obsidian |
|--------|----------|
| 桌面 + Web | 桌面 + 移动 |
| vscode.dev | iOS/Android |

**推荐**：桌面三端（Win/Mac/Linux）
**理由**：Electron 原生支持，MVP 专注

---

#### 32. 国际化
> 📖 [32-internationalization.md](./32-internationalization.md)

| VSCode | Obsidian |
|--------|----------|
| vscode-nls + 语言包扩展 | 内置多语言 |
| 完善的翻译流程 | 社区贡献 |

**推荐**：中英文双语
**理由**：核心市场优先，预留 i18n 扩展

---

#### 33. 无障碍
> 📖 [33-accessibility.md](./33-accessibility.md)

| VSCode | Obsidian |
|--------|----------|
| 完善的 ARIA + 屏幕阅读器 | 基础支持 |
| 高对比度主题 | 社区主题 |

**推荐**：基础无障碍支持
**理由**：语义化 HTML + ARIA 基础

---

### 十一、特色参考

#### 34. VSCode 特色
> 📖 [34-vscode-specific.md](./34-vscode-specific.md)

值得借鉴：
- Remote Development（SSH/Container）→ 后续考虑
- Live Share → 后续考虑
- Copilot 集成 → 我们的核心功能

---

#### 35. Obsidian 特色
> 📖 [35-obsidian-specific.md](./35-obsidian-specific.md)

值得借鉴：
- 本地优先 → 采纳
- 双向链接 → 可用于知识管理场景
- Graph View → 可视化代码关系（后续）

---

## 决策汇总

**状态说明**：
- 📝 已提议（有明确推荐，待确认）
- ⚠️ 需重点讨论（有推荐但影响大）
- ✅ 已确定（讨论后更新）

| # | 领域 | 推荐方案 | 状态 |
|---|------|----------|------|
| 1 | 进程架构 | 混合（UI 单进程 + AI Worker） | ⚠️ |
| 2 | 模块系统 | 纯 ESM | 📝 |
| 3 | IPC 通信 | Electron IPC + MessagePort | 📝 |
| 4 | 插件 API | Obsidian 风格，单一 App 入口 | ⚠️ |
| 5 | 扩展点 | 运行时注册为主 | 📝 |
| 6 | 插件生命周期 | 启动时加载 | 📝 |
| 7 | 插件通信 | 事件总线 + 共享服务 | 📝 |
| 8 | UI 布局 | 固定三栏 | ⚠️ |
| 9 | 视图系统 | React 组件注册 | 📝 |
| 10 | 主题系统 | CSS 变量 + 双主题 | 📝 |
| 11 | 命令系统 | Command Palette | 📝 |
| 12 | 快捷键 | 简单绑定 | 📝 |
| 13 | 右键菜单 | 运行时注册 | 📝 |
| 14 | 编辑器引擎 | Monaco Editor | 📝 |
| 15 | 编辑器扩展 | Monaco Language API | 📝 |
| 16 | 装饰系统 | Monaco Decorations | 📝 |
| 17 | 状态管理 | Zustand | ⚠️ |
| 18 | 设置系统 | Settings UI + JSON | 📝 |
| 19 | 文件系统 | Node.js fs 直接操作 | 📝 |
| 20 | 缓存持久化 | Electron Store | 📝 |
| 21 | 语言服务 | Monaco 内置 + AI 补全 | ⚠️ |
| 22 | 调试功能 | MVP 不做 | 📝 |
| 23 | 终端集成 | AI 代理执行 | ⚠️ |
| 24 | 搜索功能 | 基础搜索 + AI 语义 | 📝 |
| 25 | 版本控制 | Git 状态 + AI 辅助 | 📝 |
| 26 | 插件市场 | MVP 不做 | 📝 |
| 27 | 开发者体验 | 内部文档 + 模板 | 📝 |
| 28 | 安全模型 | 简单确认机制 | ⚠️ |
| 29 | 启动性能 | 保持简单架构 | 📝 |
| 30 | 运行时性能 | 按需优化 | 📝 |
| 31 | 多平台 | 桌面三端 | 📝 |
| 32 | 国际化 | 中英文 | 📝 |
| 33 | 无障碍 | 基础支持 | 📝 |
| 34 | VSCode 特色 | Copilot 集成是核心 | 📝 |
| 35 | Obsidian 特色 | 本地优先 | 📝 |

**汇总**：📝 28 项 | ⚠️ 7 项 | ✅ 0 项（待讨论后确认）

---

## 未解决问题

### 需要讨论

| # | 问题 | 选项 | 备注 |
|---|------|------|------|
| 1 | AI Worker 实现方式 | Web Worker / Child Process | Worker 轻量，Process 更隔离 |
| 2 | 多窗口支持 | 是 / 否 | 建议 MVP 暂不支持 |
| 3 | 对话历史存储 | SQLite / JSON 文件 | 需要评估数据量 |

### 技术验证

- [ ] Monaco + React 集成性能
- [ ] AI 流式响应实现
- [ ] Electron 自动更新方案
- [ ] 打包体积优化

---

## 参考资料

### 研究文档

所有详细研究位于 `research/vscode-vs-obsidian/` 目录：

| 类别 | 文档 |
|------|------|
| 基础架构 | `01-process-architecture.md`, `02-module-system.md`, `03-ipc-communication.md` |
| 插件系统 | `04-plugin-api-design.md`, `05-extension-points.md`, `06-plugin-lifecycle.md`, `07-plugin-communication.md` |
| UI 系统 | `08-ui-layout.md`, `09-view-system.md`, `10-theme-system.md` |
| 交互系统 | `11-command-system.md`, `12-keybinding-system.md`, `13-context-menu.md` |
| 编辑器 | `14-editor-engine.md`, `15-editor-extension.md`, `16-decoration-system.md` |
| 数据状态 | `17-state-management.md`, `18-settings-system.md`, `19-file-system.md`, `20-cache-persistence.md` |
| 开发功能 | `21-language-service.md`, `22-debug-capability.md`, `23-terminal-integration.md`, `24-search-capability.md`, `25-version-control.md` |
| 生态系统 | `26-plugin-marketplace.md`, `27-developer-experience.md` |
| 安全性能 | `28-security-model.md`, `29-startup-performance.md`, `30-runtime-performance.md` |
| 平台体验 | `31-multi-platform.md`, `32-internationalization.md`, `33-accessibility.md` |
| 特色功能 | `34-vscode-specific.md`, `35-obsidian-specific.md` |

### 外部参考

- [Electron 官方文档](https://www.electronjs.org/docs)
- [Monaco Editor](https://microsoft.github.io/monaco-editor/)
- [VSCode 源码](https://github.com/microsoft/vscode)
- [Obsidian API](https://docs.obsidian.md/)

---

## 讨论记录

> 此部分记录 RFC 讨论过程中的重要问题和决策

### 2026-01-19 初稿

- 创建 RFC 初稿
- 待老板审阅并决策未解决问题

<!--
讨论模板：
### YYYY-MM-DD 讨论主题
- 参与者：
- 讨论内容：
- 结论：
-->
