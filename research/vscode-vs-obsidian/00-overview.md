# VSCode vs Obsidian 架构对比

> **目标**：为 AI Chat + Editor 的 Electron 应用提供架构设计参考

---

## 为什么选择这两个产品对比

| 产品 | 借鉴价值 |
|------|----------|
| **VSCode** | 业界最成熟的插件化编辑器，架构严谨，扩展性极强 |
| **Obsidian** | 轻量但功能丰富，插件生态活跃，开发体验友好 |

两者都是 Electron + TypeScript 技术栈，都有成熟的插件系统，但做出了不同的架构取舍。

---

## 基本信息

|  | VSCode | Obsidian |
|--|--------|----------|
| **定位** | 通用代码编辑器 | 知识管理/笔记工具 |
| **目标用户** | 开发者 | 知识工作者 |
| **开源情况** | MIT 开源 (Code OSS) | 闭源 (插件 API 公开) |
| **技术栈** | Electron + TypeScript | Electron + TypeScript |
| **编辑器引擎** | Monaco Editor | CodeMirror 6 |
| **首次发布** | 2015 | 2020 |
| **插件数量** | 50,000+ | 2,000+ |
| **代码规模** | ~100万行 | 闭源，估计 10-20万行 |

---

## 核心架构决策清单

以下是构建插件化 Electron 应用需要做的**关键架构决策**：

---

### 一、基础架构层

#### 1. 进程架构

> 决定应用的稳定性、性能和安全边界

| 决策点 | 选项 | VSCode | Obsidian |
|--------|------|--------|----------|
| 插件运行位置 | 主进程 / 渲染进程 / 独立进程 | 独立进程 (Extension Host) | 渲染进程 (同进程) |
| 插件隔离级别 | 进程隔离 / VM 沙箱 / 无隔离 | 进程隔离 | 无隔离 |
| 崩溃影响范围 | 插件崩溃是否影响主应用 | 不影响 | 影响 |
| 多窗口架构 | 共享进程 / 独立进程 | 共享 Extension Host | 独立窗口 |
| Web 兼容性 | 是否支持浏览器运行 | 支持 (vscode.dev) | 不支持 |

**详见**: [01-process-architecture.md](./01-process-architecture.md)

---

#### 2. 模块与依赖系统

> 决定代码组织方式和模块间的协作模式

| 决策点 | 选项 | VSCode | Obsidian |
|--------|------|--------|----------|
| 依赖注入 | DI 容器 / 手动注入 / 全局单例 | DI 容器 (装饰器) | 全局单例 (app) |
| 服务发现 | 静态注册 / 动态发现 | 静态 + Contribution | 静态注册 |
| 模块边界 | 严格隔离 / 松散耦合 | 严格 (按层分离) | 松散 |
| 循环依赖处理 | 禁止 / 延迟加载 / 接口抽象 | 接口抽象 | 不严格限制 |
| 代码分层 | 分层架构 / 扁平结构 | 严格分层 (base/platform/workbench) | 扁平 |

**详见**: [02-module-system.md](./02-module-system.md)

---

#### 3. IPC 通信

> 决定进程间如何通信

| 决策点 | 选项 | VSCode | Obsidian |
|--------|------|--------|----------|
| 通信协议 | Electron IPC / MessagePort / 自定义 | 自定义 RPC | Electron IPC |
| 序列化方式 | JSON / 结构化克隆 / 自定义 | 自定义序列化 | JSON |
| 调用模式 | 请求-响应 / 事件广播 / 双向流 | 全部支持 | 请求-响应为主 |
| 类型安全 | 运行时校验 / 编译时类型 | 编译时类型 | 弱类型 |

**详见**: [03-ipc-communication.md](./03-ipc-communication.md)

---

### 二、插件系统

#### 4. 插件 API 设计

> 决定插件能做什么、不能做什么

| 决策点 | 选项 | VSCode | Obsidian |
|--------|------|--------|----------|
| API 风格 | 命名空间 / 类实例 / 函数式 | 命名空间 (`vscode.*`) | 类实例 (`this.app.*`) |
| 能力边界 | 白名单 / 黑名单 / 无限制 | 白名单 (只暴露 API) | 近乎无限制 |
| 类型支持 | 完整类型 / 部分类型 | 完整 `@types/vscode` | 完整 `obsidian.d.ts` |
| API 稳定性 | 严格版本控制 / 随时变更 | 严格，少 breaking change | 相对稳定 |
| Proposed API | 有 / 无 | 有 (需 opt-in) | 无 |
| 废弃策略 | 标记废弃 / 直接移除 | 标记废弃，长期保留 | 较少废弃 |

**详见**: [04-plugin-api-design.md](./04-plugin-api-design.md)

---

#### 5. 扩展点机制

> 决定插件如何与宿主应用集成

| 决策点 | 选项 | VSCode | Obsidian |
|--------|------|--------|----------|
| 声明方式 | 声明式 (JSON) / 命令式 (代码) | 声明式 (package.json) | 命令式 (代码注册) |
| UI 扩展点 | 固定槽位 / 自由注入 | 固定槽位 (contribution points) | 半固定 (View 类型) |
| 菜单系统 | 声明式 / 代码注册 | 声明式 (when clause) | 代码注册 |
| 激活时机 | 懒加载 / 立即加载 | 懒加载 (activationEvents) | 启动时加载 |
| 扩展点数量 | 多 / 少 | 非常多 (~30+) | 较少 (~10) |
| 自定义扩展点 | 插件可定义 / 仅宿主定义 | 插件可定义 | 仅宿主定义 |

**详见**: [05-extension-points.md](./05-extension-points.md)

---

#### 6. 插件生命周期

> 决定插件的加载、更新和卸载方式

| 决策点 | 选项 | VSCode | Obsidian |
|--------|------|--------|----------|
| 加载时机 | 启动时 / 按需 / 延迟 | 按需 (activationEvents) | 启动时全部 |
| 热重载 | 支持 / 不支持 | 需重启 Host | Cmd+R 刷新 |
| 资源清理 | 自动 / 手动 | 手动 (dispose) | 手动 (unload) |
| 版本管理 | 内置更新 / 外部管理 | 内置市场 | 内置 + BRAT |
| 依赖管理 | 独立 node_modules / 共享 | 独立 | 共享运行时 |
| 插件间依赖 | 支持 / 不支持 | 支持 (extensionDependencies) | 不支持 |

**详见**: [06-plugin-lifecycle.md](./06-plugin-lifecycle.md)

---

#### 7. 插件间通信

> 决定插件如何协作

| 决策点 | 选项 | VSCode | Obsidian |
|--------|------|--------|----------|
| 通信方式 | 导出 API / 事件 / 命令 | 导出 API + 命令 | 全局事件 + 直接访问 |
| API 导出 | 显式导出 / 全局暴露 | 显式 (activate return) | 无正式机制 |
| 类型共享 | 独立类型包 / 内联 | 独立类型包 | 无 |
| 版本协商 | 有 / 无 | 有 (semver) | 无 |

**详见**: [07-plugin-communication.md](./07-plugin-communication.md)

---

### 三、UI 与交互

#### 8. UI 布局架构

> 决定界面的整体结构

| 决策点 | 选项 | VSCode | Obsidian |
|--------|------|--------|----------|
| 布局模型 | 固定区域 / 灵活分割 | 固定区域 | 灵活 Workspace |
| 区域类型 | Activity Bar / Side Bar / Editor / Panel / Status Bar | 全部 | Ribbon / Sidebar / Workspace / Status |
| 分屏支持 | 编辑器分组 / 任意分割 | 编辑器分组 | 任意分割 (Leaf) |
| 标签页 | 单行 / 多行 / 可关闭 | 单行可滚动 | 单行 |
| 布局持久化 | 自动保存 / 手动保存 | 自动 | 自动 |
| 响应式 | 固定断点 / 流式 | 固定断点 | 流式 |

**详见**: [08-ui-layout.md](./08-ui-layout.md)

---

#### 9. 视图系统

> 决定自定义视图的能力边界

| 决策点 | 选项 | VSCode | Obsidian |
|--------|------|--------|----------|
| 视图类型 | TreeView / Webview / 自定义 | TreeView + Webview | ItemView |
| Webview 隔离 | iframe / Shadow DOM / 无 | iframe (完全隔离) | 无 (直接 DOM) |
| 视图位置 | 固定 / 可拖拽 | 固定槽位 | 可拖拽 |
| 视图持久化 | 自动 / 手动 | 手动 (serializer) | 自动 |
| 视图通信 | postMessage / 直接调用 | postMessage | 直接调用 |

**详见**: [09-view-system.md](./09-view-system.md)

---

#### 10. 主题系统

> 决定视觉定制的灵活性

| 决策点 | 选项 | VSCode | Obsidian |
|--------|------|--------|----------|
| 主题格式 | JSON / CSS / 混合 | JSON (Token 颜色) + CSS | CSS + CSS 变量 |
| 变量命名 | 语义化 / 组件化 | 语义化 (editor.background) | 混合 |
| 暗色/亮色 | 自动检测 / 手动切换 | 手动切换 | 手动切换 |
| 图标主题 | 支持 / 不支持 | 支持 | 不支持 |
| 产品主题 | 支持 / 不支持 | 支持 (Product Icon Theme) | 不支持 |
| 主题继承 | 支持 / 不支持 | 支持 | 不支持 |

**详见**: [10-theme-system.md](./10-theme-system.md)

---

#### 11. 命令系统

> 决定用户交互的核心方式

| 决策点 | 选项 | VSCode | Obsidian |
|--------|------|--------|----------|
| 命令注册 | 声明式 / 代码注册 | 混合 | 代码注册 |
| 命令 ID | 全局唯一 / 带前缀 | 全局唯一 | 插件前缀 |
| 命令参数 | 类型化 / any | 类型化 | any |
| 命令面板 | 内置 / 可扩展 | 高度可扩展 | 基础功能 |
| 命令分类 | 有 / 无 | 有 (category) | 无 |
| 命令搜索 | 模糊匹配 / 精确匹配 | 模糊匹配 | 模糊匹配 |

**详见**: [11-command-system.md](./11-command-system.md)

---

#### 12. 快捷键系统

> 决定键盘交互的灵活性

| 决策点 | 选项 | VSCode | Obsidian |
|--------|------|--------|----------|
| 绑定方式 | 声明式 / 代码注册 | 声明式 + 用户覆盖 | 用户配置为主 |
| 上下文条件 | when clause / 代码判断 | when clause | 代码判断 |
| 冲突处理 | 优先级 / 覆盖 / 提示 | 优先级 + 提示 | 覆盖 |
| 键序列 | 支持 (chord) / 不支持 | 支持 | 不支持 |
| 平台差异 | 自动映射 / 手动配置 | 自动映射 | 自动映射 |
| Vim 模式 | 内置 / 插件 | 插件 | 插件 |

**详见**: [12-keybinding-system.md](./12-keybinding-system.md)

---

#### 13. 上下文菜单

> 决定右键菜单的扩展方式

| 决策点 | 选项 | VSCode | Obsidian |
|--------|------|--------|----------|
| 注册方式 | 声明式 / 代码注册 | 声明式 (menus) | 代码注册 (registerEvent) |
| 菜单位置 | 固定 / 动态 | 固定位置 (contributes.menus) | 动态注入 |
| 条件显示 | when clause / 代码判断 | when clause | 代码判断 |
| 分组排序 | group + order | group + order | 无内置机制 |
| 子菜单 | 支持 / 不支持 | 支持 | 支持 |

**详见**: [13-context-menu.md](./13-context-menu.md)

---

### 四、编辑器

#### 14. 编辑器引擎

> 决定核心编辑能力

| 决策点 | 选项 | VSCode | Obsidian |
|--------|------|--------|----------|
| 引擎选择 | Monaco / CodeMirror / ProseMirror / 自研 | Monaco | CodeMirror 6 |
| 文档模型 | 行模型 / 字符模型 / AST | 行模型 | 字符模型 |
| 虚拟滚动 | 支持 / 不支持 | 支持 | 支持 |
| 大文件支持 | 好 / 一般 / 差 | 好 | 一般 |
| 协同编辑 | 支持 / 不支持 | 支持 (Live Share) | 不支持 |

**详见**: [14-editor-engine.md](./14-editor-engine.md)

---

#### 15. 编辑器扩展

> 决定编辑器功能的扩展方式

| 决策点 | 选项 | VSCode | Obsidian |
|--------|------|--------|----------|
| 扩展机制 | Language Server / 内置 API | LSP + 内置 | CM Extension |
| 语法高亮 | TextMate / Tree-sitter / 自定义 | TextMate | Lezer (CM6) |
| 代码补全 | Provider 模式 / 直接注入 | Provider | EditorSuggest |
| 代码诊断 | Diagnostic API / 直接标记 | Diagnostic API | CM Linter |
| 代码操作 | Code Action / Quick Fix | Code Action | 无内置 |
| 悬停提示 | Hover Provider / 直接注入 | Hover Provider | CM Tooltip |

**详见**: [15-editor-extension.md](./15-editor-extension.md)

---

#### 16. 装饰器系统

> 决定编辑器视觉增强方式

| 决策点 | 选项 | VSCode | Obsidian |
|--------|------|--------|----------|
| 装饰类型 | 行级 / 字符级 / 自定义 Widget | 全部支持 | CM Decoration |
| 性能优化 | 按需渲染 / 全量渲染 | 按需渲染 | 按需渲染 |
| 装饰持久化 | 跟随文档 / 独立管理 | 独立管理 | 跟随文档 |
| 交互装饰 | 支持 / 不支持 | 有限支持 | Widget 支持 |

**详见**: [16-decoration-system.md](./16-decoration-system.md)

---

### 五、数据与状态

#### 17. 状态管理

> 决定数据如何流动

| 决策点 | 选项 | VSCode | Obsidian |
|--------|------|--------|----------|
| 全局状态 | 集中式 / 分散式 | 分散 (各服务管理) | 集中 (app 对象) |
| 事件系统 | EventEmitter / Observable / 信号 | EventEmitter | EventRef |
| 状态变更 | 不可变 / 可变 | 混合 | 可变 |
| 状态同步 | 主动推送 / 拉取 | 主动推送 | 拉取 + 事件 |

**详见**: [17-state-management.md](./17-state-management.md)

---

#### 18. 配置系统

> 决定设置的存储和访问方式

| 决策点 | 选项 | VSCode | Obsidian |
|--------|------|--------|----------|
| 配置格式 | JSON / YAML / 自定义 | JSON | JSON |
| 配置作用域 | 用户 / 工作区 / 文件夹 / 语言 | 多级作用域 | 单一作用域 |
| Schema 定义 | JSON Schema / 代码定义 | JSON Schema | 代码定义 |
| 配置 UI | 自动生成 / 手动编写 | 自动生成 | 手动编写 (PluginSettingTab) |
| 配置迁移 | 有机制 / 无机制 | 无内置机制 | 无内置机制 |
| 配置同步 | 内置 / 第三方 | 内置 Settings Sync | 第三方 |

**详见**: [18-settings-system.md](./18-settings-system.md)

---

#### 19. 文件系统

> 决定文件访问的抽象程度

| 决策点 | 选项 | VSCode | Obsidian |
|--------|------|--------|----------|
| 抽象层 | FileSystemProvider / 直接访问 | FileSystemProvider | Vault API |
| 虚拟文件系统 | 支持 / 不支持 | 支持 | 不支持 |
| 远程文件 | 支持 / 不支持 | 支持 (Remote) | 不支持 |
| 文件监听 | 内置 / 第三方 | 内置 | 内置 |
| 二进制文件 | 支持 / 不支持 | 支持 | 有限支持 |

**详见**: [19-file-system.md](./19-file-system.md)

---

#### 20. 缓存与持久化

> 决定数据的持久化策略

| 决策点 | 选项 | VSCode | Obsidian |
|--------|------|--------|----------|
| 缓存存储 | IndexedDB / LocalStorage / 文件 | IndexedDB + 文件 | LocalStorage + 文件 |
| 插件数据 | 隔离存储 / 共享存储 | 隔离 (globalState) | 隔离 (data.json) |
| 工作区状态 | 自动保存 / 手动保存 | 自动保存 | 自动保存 |
| 撤销历史 | 持久化 / 内存 | 内存 | 内存 |

**详见**: [20-cache-persistence.md](./20-cache-persistence.md)

---

### 六、特定能力

#### 21. 语言服务

> 决定编程语言支持方式

| 决策点 | 选项 | VSCode | Obsidian |
|--------|------|--------|----------|
| 协议 | LSP / 自定义 / 无 | LSP | 无 |
| 运行位置 | 独立进程 / Worker / 同进程 | 独立进程 | N/A |
| 语言支持 | 多语言 / 单语言 | 多语言 | Markdown 为主 |

**详见**: [21-language-service.md](./21-language-service.md)

---

#### 22. 调试能力

> 决定调试支持方式

| 决策点 | 选项 | VSCode | Obsidian |
|--------|------|--------|----------|
| 协议 | DAP / 自定义 / 无 | DAP | 无 |
| 调试 UI | 内置 / 可扩展 | 内置 + 可扩展 | N/A |

**详见**: [22-debug-capability.md](./22-debug-capability.md)

---

#### 23. 终端集成

> 决定终端能力

| 决策点 | 选项 | VSCode | Obsidian |
|--------|------|--------|----------|
| 终端引擎 | xterm.js / 其他 / 无 | xterm.js | 无内置 |
| Shell 集成 | 支持 / 不支持 | 支持 | N/A |
| 链接检测 | 支持 / 不支持 | 支持 | N/A |

**详见**: [23-terminal-integration.md](./23-terminal-integration.md)

---

#### 24. 搜索能力

> 决定全局搜索实现

| 决策点 | 选项 | VSCode | Obsidian |
|--------|------|--------|----------|
| 搜索引擎 | ripgrep / 自研 / 浏览器 API | ripgrep | 自研 |
| 搜索范围 | 文件内容 / 文件名 / 符号 | 全部 | 全部 |
| 正则支持 | 完整 / 基础 / 无 | 完整 | 基础 |
| 替换功能 | 支持 / 不支持 | 支持 | 有限支持 |

**详见**: [24-search-capability.md](./24-search-capability.md)

---

#### 25. 版本控制

> 决定 Git 集成方式

| 决策点 | 选项 | VSCode | Obsidian |
|--------|------|--------|----------|
| Git 集成 | 内置 / 插件 / 无 | 内置 + SCM API | 插件 |
| Diff 视图 | 内置 / 插件 | 内置 | 插件 |
| 多 SCM | 支持 / 不支持 | 支持 | N/A |

**详见**: [25-version-control.md](./25-version-control.md)

---

### 七、生态与治理

#### 26. 插件市场

> 决定插件分发方式

| 决策点 | 选项 | VSCode | Obsidian |
|--------|------|--------|----------|
| 市场形态 | 官方商店 / 社区列表 / 无 | 官方商店 | 社区列表 |
| 审核机制 | 自动 / 人工 / 无 | 自动 + 人工 | 基础检查 |
| 付费插件 | 支持 / 不支持 | 不支持 | 不支持 |
| 私有插件 | 支持 / 不支持 | 支持 | 支持 (手动安装) |

**详见**: [26-plugin-marketplace.md](./26-plugin-marketplace.md)

---

#### 27. 开发者体验

> 决定插件生态的繁荣程度

| 决策点 | 选项 | VSCode | Obsidian |
|--------|------|--------|----------|
| 脚手架 | 官方 / 社区 / 无 | 官方 (yo code) | 社区 |
| 调试支持 | 断点 / console / DevTools | 完整断点调试 | DevTools |
| 文档质量 | 完整 / 基础 / 社区驱动 | 完整官方文档 | 社区驱动 |
| 示例代码 | 官方示例 / 社区示例 | 丰富官方示例 | 社区示例 |
| API Playground | 有 / 无 | 无 | 无 |

**详见**: [27-developer-experience.md](./27-developer-experience.md)

---

#### 28. 安全模型

> 决定插件的安全边界

| 决策点 | 选项 | VSCode | Obsidian |
|--------|------|--------|----------|
| 权限系统 | 细粒度 / 粗粒度 / 无 | 粗粒度 (Workspace Trust) | 无 |
| 网络访问 | 受限 / 不受限 | 不受限 | 不受限 |
| 文件访问 | 受限 / 不受限 | Workspace Trust | 不受限 |
| 代码签名 | 有 / 无 | 有 (市场发布) | 无 |

**详见**: [28-security-model.md](./28-security-model.md)

---

### 八、性能与优化

#### 29. 启动性能

> 决定首屏加载速度

| 决策点 | 选项 | VSCode | Obsidian |
|--------|------|--------|----------|
| 代码分割 | 按需加载 / 全量加载 | 按需加载 | 全量加载 |
| 插件懒加载 | 支持 / 不支持 | 支持 (activationEvents) | 不支持 |
| 缓存策略 | Service Worker / 文件缓存 | 文件缓存 | 文件缓存 |
| 冷启动优化 | V8 快照 / 无 | V8 快照 | 无 |

**详见**: [29-startup-performance.md](./29-startup-performance.md)

---

#### 30. 运行时性能

> 决定运行时流畅度

| 决策点 | 选项 | VSCode | Obsidian |
|--------|------|--------|----------|
| 渲染优化 | 虚拟列表 / 增量更新 | 虚拟列表 | 增量更新 |
| 计算卸载 | Worker / 主线程 | Worker | 主线程 |
| 内存管理 | 手动释放 / GC 依赖 | 混合 | GC 依赖 |
| 性能监控 | 内置 / 无 | 内置 (Process Explorer) | 无 |

**详见**: [30-runtime-performance.md](./30-runtime-performance.md)

---

### 九、平台适配

#### 31. 多平台支持

> 决定跨平台策略

| 决策点 | 选项 | VSCode | Obsidian |
|--------|------|--------|----------|
| 桌面平台 | Windows / macOS / Linux | 全部 | 全部 |
| 移动平台 | iOS / Android / 无 | 无 (有 Web) | 全部 |
| Web 平台 | 支持 / 不支持 | 支持 | 不支持 |
| 平台特性 | 统一抽象 / 条件代码 | 统一抽象 | 条件代码 |

**详见**: [31-multi-platform.md](./31-multi-platform.md)

---

#### 32. 国际化

> 决定多语言支持方式

| 决策点 | 选项 | VSCode | Obsidian |
|--------|------|--------|----------|
| i18n 方案 | 内置 / 第三方 / 无 | 内置 (nls) | 内置 |
| 语言包 | 插件形式 / 内置 | 插件形式 | 内置 |
| 动态切换 | 支持 / 需重启 | 需重启 | 需重启 |
| RTL 支持 | 支持 / 不支持 | 部分支持 | 部分支持 |

**详见**: [32-internationalization.md](./32-internationalization.md)

---

#### 33. 无障碍

> 决定可访问性支持

| 决策点 | 选项 | VSCode | Obsidian |
|--------|------|--------|----------|
| 屏幕阅读器 | 完整支持 / 部分支持 / 不支持 | 完整支持 | 部分支持 |
| 键盘导航 | 完整 / 部分 | 完整 | 部分 |
| 高对比度 | 支持 / 不支持 | 支持 | 有限支持 |
| ARIA 标注 | 完整 / 部分 | 完整 | 部分 |

**详见**: [33-accessibility.md](./33-accessibility.md)

---

### 十、特色功能

#### 34. VSCode 特有

| 特性 | 说明 |
|------|------|
| **Remote Development** | SSH/Container/WSL 远程开发 |
| **Live Share** | 实时协作编辑 |
| **Notebook API** | Jupyter Notebook 支持 |
| **Testing API** | 测试框架集成 |
| **Tasks** | 任务运行系统 |
| **Multi-root Workspace** | 多根工作区 |

**详见**: [34-vscode-specific.md](./34-vscode-specific.md)

---

#### 35. Obsidian 特有

| 特性 | 说明 |
|------|------|
| **双向链接** | [[WikiLink]] 语法 |
| **Graph View** | 知识图谱可视化 |
| **Canvas** | 无限画布 |
| **Properties** | YAML Front Matter |
| **Publish/Sync** | 官方发布和同步服务 |
| **Daily Notes** | 日记功能 |

**详见**: [35-obsidian-specific.md](./35-obsidian-specific.md)

---

## 对比矩阵

### 复杂度 vs 灵活性

```
                    灵活性/扩展性
                         ↑
                         │
              Obsidian   │   VSCode
             (简单灵活)   │  (复杂强大)
                         │
         ─────────────────────────────→ 架构复杂度
                         │
             简单但受限   │   复杂但受限
                         │
```

### 总体对比

| 维度 | VSCode 路线 | Obsidian 路线 |
|------|-------------|---------------|
| **复杂度** | 高 | 低 |
| **稳定性** | 极高 | 高 |
| **安全性** | 较高 | 一般 |
| **灵活性** | 高 (受控) | 极高 (无限制) |
| **学习曲线** | 陡峭 | 平缓 |
| **适合团队** | 大型团队 | 小型团队 |
| **适合阶段** | 长期维护 | 快速迭代 |

---

## 章节索引

| # | 类别 | 文件 | 核心问题 |
|---|------|------|----------|
| 01 | 基础架构 | process-architecture.md | 插件跑在哪个进程？ |
| 02 | 基础架构 | module-system.md | 模块如何组织和协作？ |
| 03 | 基础架构 | ipc-communication.md | 进程间如何通信？ |
| 04 | 插件系统 | plugin-api-design.md | 暴露什么 API 给插件？ |
| 05 | 插件系统 | extension-points.md | 插件如何扩展宿主？ |
| 06 | 插件系统 | plugin-lifecycle.md | 插件如何加载和卸载？ |
| 07 | 插件系统 | plugin-communication.md | 插件间如何协作？ |
| 08 | UI 交互 | ui-layout.md | 界面如何组织？ |
| 09 | UI 交互 | view-system.md | 自定义视图怎么做？ |
| 10 | UI 交互 | theme-system.md | 主题如何实现？ |
| 11 | UI 交互 | command-system.md | 命令如何注册和执行？ |
| 12 | UI 交互 | keybinding-system.md | 快捷键如何绑定？ |
| 13 | UI 交互 | context-menu.md | 右键菜单如何扩展？ |
| 14 | 编辑器 | editor-engine.md | 用什么编辑器引擎？ |
| 15 | 编辑器 | editor-extension.md | 编辑器功能如何扩展？ |
| 16 | 编辑器 | decoration-system.md | 编辑器装饰如何实现？ |
| 17 | 数据状态 | state-management.md | 状态如何管理？ |
| 18 | 数据状态 | settings-system.md | 配置如何存储和访问？ |
| 19 | 数据状态 | file-system.md | 文件系统如何抽象？ |
| 20 | 数据状态 | cache-persistence.md | 数据如何持久化？ |
| 21 | 特定能力 | language-service.md | 语言服务如何实现？ |
| 22 | 特定能力 | debug-capability.md | 调试能力如何实现？ |
| 23 | 特定能力 | terminal-integration.md | 终端如何集成？ |
| 24 | 特定能力 | search-capability.md | 搜索如何实现？ |
| 25 | 特定能力 | version-control.md | 版本控制如何集成？ |
| 26 | 生态治理 | plugin-marketplace.md | 插件如何分发？ |
| 27 | 生态治理 | developer-experience.md | 开发者体验如何？ |
| 28 | 生态治理 | security-model.md | 安全边界在哪里？ |
| 29 | 性能优化 | startup-performance.md | 启动如何优化？ |
| 30 | 性能优化 | runtime-performance.md | 运行时如何优化？ |
| 31 | 平台适配 | multi-platform.md | 如何跨平台？ |
| 32 | 平台适配 | internationalization.md | 如何国际化？ |
| 33 | 平台适配 | accessibility.md | 如何无障碍？ |
| 34 | 特色功能 | vscode-specific.md | VSCode 有什么特有能力？ |
| 35 | 特色功能 | obsidian-specific.md | Obsidian 有什么特有能力？ |

---

## 下一步

每个章节将深入对比两者的实现，并给出**对 AI Chat + Editor 应用的设计建议**。
