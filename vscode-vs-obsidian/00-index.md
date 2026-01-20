# VSCode vs Obsidian 架构对比

为 Coding Agent Desktop 的 Electron 应用提供架构设计参考。

两者都是 Electron + TypeScript 技术栈，都有成熟的插件系统，但做出了不同的架构取舍。

---

## 章节目录

### 一、基础架构

| # | 主题 | 核心问题 |
|---|------|---------|
| [01](./01-process-architecture.md) | 进程架构 | 插件跑在哪个进程？ |
| [02](./02-module-system.md) | 模块系统 | 模块如何组织和协作？ |
| [03](./03-ipc-communication.md) | IPC 通信 | 进程间如何通信？ |

### 二、插件系统

| # | 主题 | 核心问题 |
|---|------|---------|
| [04](./04-plugin-api-design.md) | 插件 API | 暴露什么能力给插件？ |
| [05](./05-extension-points.md) | 扩展点 | 插件如何扩展宿主？ |
| [06](./06-plugin-lifecycle.md) | 生命周期 | 插件如何加载和卸载？ |
| [07](./07-plugin-communication.md) | 插件通信 | 插件间如何协作？ |

### 三、UI 与交互

| # | 主题 | 核心问题 |
|---|------|---------|
| [08](./08-ui-layout.md) | UI 布局 | 界面如何组织？ |
| [09](./09-view-system.md) | 视图系统 | 自定义视图怎么做？ |
| [10](./10-theme-system.md) | 主题系统 | 主题如何实现？ |
| [11](./11-command-system.md) | 命令系统 | 命令如何注册和执行？ |
| [12](./12-keybinding-system.md) | 快捷键 | 快捷键如何绑定？ |
| [13](./13-context-menu.md) | 上下文菜单 | 右键菜单如何扩展？ |

### 四、编辑器

| # | 主题 | 核心问题 |
|---|------|---------|
| [14](./14-editor-engine.md) | 编辑器引擎 | 用什么编辑器引擎？ |
| [15](./15-editor-extension.md) | 编辑器扩展 | 编辑器功能如何扩展？ |
| [16](./16-decoration-system.md) | 装饰器系统 | 编辑器装饰如何实现？ |

### 五、数据与状态

| # | 主题 | 核心问题 |
|---|------|---------|
| [17](./17-state-management.md) | 状态管理 | 状态如何管理？ |
| [18](./18-settings-system.md) | 配置系统 | 配置如何存储和访问？ |
| [19](./19-file-system.md) | 文件系统 | 文件系统如何抽象？ |
| [20](./20-cache-persistence.md) | 缓存持久化 | 数据如何持久化？ |

### 六、特定能力

| # | 主题 | 核心问题 |
|---|------|---------|
| [21](./21-language-service.md) | 语言服务 | 语言服务如何实现？ |
| [22](./22-debug-capability.md) | 调试能力 | 调试能力如何实现？ |
| [23](./23-terminal-integration.md) | 终端集成 | 终端如何集成？ |
| [24](./24-search-capability.md) | 搜索能力 | 搜索如何实现？ |
| [25](./25-version-control.md) | 版本控制 | 版本控制如何集成？ |

### 七、生态与治理

| # | 主题 | 核心问题 |
|---|------|---------|
| [26](./26-plugin-marketplace.md) | 插件市场 | 插件如何分发？ |
| [27](./27-developer-experience.md) | 开发者体验 | 开发者体验如何？ |
| [28](./28-security-model.md) | 安全模型 | 安全边界在哪里？ |

### 八、性能与优化

| # | 主题 | 核心问题 |
|---|------|---------|
| [29](./29-startup-performance.md) | 启动性能 | 启动如何优化？ |
| [30](./30-runtime-performance.md) | 运行时性能 | 运行时如何优化？ |

### 九、平台适配

| # | 主题 | 核心问题 |
|---|------|---------|
| [31](./31-multi-platform.md) | 多平台支持 | 如何跨平台？ |
| [32](./32-internationalization.md) | 国际化 | 如何国际化？ |
| [33](./33-accessibility.md) | 无障碍 | 如何无障碍？ |

### 十、特色功能

| # | 主题 | 核心问题 |
|---|------|---------|
| [34](./34-vscode-specific.md) | VSCode 特有 | VSCode 有什么特有能力？ |
| [35](./35-obsidian-specific.md) | Obsidian 特有 | Obsidian 有什么特有能力？ |

---

## 最终决策

完成技术对比后，参见 [技术决策文档](../technical-decisions.md)
