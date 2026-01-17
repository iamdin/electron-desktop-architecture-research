# Obsidian 架构分析

> 来源：Obsidian 开发者文档、API 类型定义及社区分析

## 1. 整体架构

Obsidian 采用**单 Renderer 进程**架构，插件直接运行在主渲染进程中：

```
┌─────────────────────────────────────────────────────────────┐
│                     Main Process (Electron)                  │
│    - 应用生命周期管理                                         │
│    - 窗口管理                                                │
│    - 原生 OS 交互                                            │
│    - 文件系统访问                                            │
└────────────────────────┬────────────────────────────────────┘
                         │ contextBridge / preload
┌────────────────────────┼────────────────────────────────────┐
│                  Renderer Process                            │
│  ┌────────────────────────────────────────────────────────┐ │
│  │                    App 核心实例                         │ │
│  │  ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────────┐  │ │
│  │  │  Vault  │ │Workspace│ │MetaCache│ │FileManager  │  │ │
│  │  └─────────┘ └─────────┘ └─────────┘ └─────────────┘  │ │
│  └────────────────────────┬───────────────────────────────┘ │
│                           │                                  │
│      ┌────────────────────┼────────────────────┐            │
│      ▼                    ▼                    ▼            │
│  ┌────────┐          ┌────────┐          ┌────────┐        │
│  │Plugin 1│          │Plugin 2│          │Plugin 3│        │
│  └────────┘          └────────┘          └────────┘        │
│                                                             │
│  ┌─────────────────────────────────────────────────────┐   │
│  │                 CodeMirror 6 Editor                  │   │
│  └─────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
```

---

## 2. 核心设计理念

### 2.1 单一入口模式

所有功能通过一个 `App` 实例访问：

```typescript
class MyPlugin extends Plugin {
  async onload() {
    const vault = this.app.vault;           // 文件系统
    const workspace = this.app.workspace;   // UI 布局
    const metadataCache = this.app.metadataCache; // 元数据缓存
    const fileManager = this.app.fileManager;     // 文件管理
  }
}
```

**设计动机：**
- **简化心智模型** — 开发者只需记住 `this.app`
- **依赖关系清晰** — 不需要 import 多个模块，减少循环依赖
- **热重载友好** — 插件 reload 时只需替换 Plugin 实例，App 保持稳定

### 2.2 继承式组件模式

采用类继承扩展功能：

| 扩展类型 | 基类 | 用途 |
|---------|------|------|
| 插件 | `Plugin` | 插件入口，管理生命周期 |
| 视图 | `ItemView` | 自定义标签页视图 |
| 设置 | `PluginSettingTab` | 插件设置界面 |
| 模态框 | `Modal` | 弹窗交互 |
| 建议 | `EditorSuggest` | 编辑器自动补全 |

**继承式的优势：**
- 更直观的生命周期管理（`onOpen`, `onClose`）
- 可以直接操作 DOM（`this.containerEl`）
- 类型推断更友好

### 2.3 完全信任模式

> "Obsidian is a browser application, not a node application."

- 插件可访问完整的浏览器 API
- 桌面版提供有限的 Node.js API 模拟
- **无沙箱隔离**，插件与应用同进程
- 用户主动安装即表示信任

### 2.4 声明式注册

采用声明式模式注册扩展：

- 命令、视图、事件处理器在 `onload` 阶段注册
- 框架自动跟踪所有注册项
- `onunload` 时自动清理

---

## 3. 资源管理模式

### 3.1 EventRef 模式

每次订阅事件返回一个 `EventRef`，用于后续取消订阅。

**问题场景：** 插件卸载时忘记取消事件订阅会导致内存泄漏。

**解决方案：**
```typescript
// 方式 1: 手动管理（容易忘记）
const ref = this.app.vault.on('create', (file) => {...});
this.app.vault.offref(ref);  // 必须手动取消

// 方式 2: 自动管理（推荐）
this.registerEvent(
  this.app.vault.on('create', (file) => {...})
);
// 不需要手动清理！
```

### 3.2 自动清理 API

```typescript
// 注册定时器（自动清理）
this.registerInterval(window.setInterval(() => {...}, 1000));

// 注册 DOM 事件（自动清理）
this.registerDomEvent(document, 'click', (evt) => {...});

// 添加子组件（自动清理）
this.addChild(new ChildComponent());
```

---

## 4. 插件生命周期

### 4.1 生命周期流程

```
Plugin 创建
    │
    ▼
onload() ─────────────── 初始化：注册命令、视图、事件
    │
    ▼
onUserEnable() ───────── 首次启用时调用（v1.10.0+）
    │
    │  ┌──────────────── 运行中：处理用户交互
    │  │
    ▼  │
[活跃状态] ◄────────────┘
    │
    ▼
onunload() ───────────── 清理：移除非注册资源
    │
    ▼
Plugin 销毁（自动清理注册资源）
```

### 4.2 加载时序

```
应用启动
    │
    ▼
读取 .obsidian/plugins/ 目录
    │
    ▼
解析每个插件的 manifest.json
    │
    ▼
检查 minAppVersion 兼容性
    │
    ▼
动态加载 main.js
    │
    ▼
创建 Plugin 实例
    │
    ▼
调用 plugin.onload()
    │
    ▼
插件激活完成
```

### 4.3 加载机制特点

- **同步加载** — 所有插件在应用启动时加载
- **无懒加载** — 不像 VS Code 有激活事件
- **无隔离** — 插件直接运行在渲染进程
- **共享上下文** — 所有插件共享同一个 App 实例

---

## 5. 插件结构

每个插件位于 `.obsidian/plugins/[plugin-id]/`：

| 文件 | 必需 | 说明 |
|-----|-----|------|
| manifest.json | 是 | 插件元数据 |
| main.js | 是 | 编译后的入口 |
| styles.css | 否 | 插件样式 |
| data.json | 自动 | 持久化数据 |

### manifest.json

```json
{
  "id": "my-plugin",
  "name": "My Plugin",
  "version": "1.0.0",
  "minAppVersion": "1.0.0",
  "description": "Plugin description",
  "author": "Author Name",
  "authorUrl": "https://example.com",
  "isDesktopOnly": false
}
```

---

## 6. UI 层架构

### 6.1 UI 技术栈

Obsidian 的 UI 采用**原生 DOM + CSS**方案，没有使用 React/Vue 等框架：

```
┌─────────────────────────────────────────────────────────────┐
│                      UI 层架构                               │
├─────────────────────────────────────────────────────────────┤
│  ┌─────────────────────────────────────────────────────┐   │
│  │                 Workspace（工作区）                   │   │
│  │  ┌──────────┐  ┌──────────────────────────────┐    │   │
│  │  │LeftSplit │  │         RootSplit            │    │   │
│  │  │(侧边栏)  │  │  ┌────────┐  ┌────────┐     │    │   │
│  │  │          │  │  │  Leaf  │  │  Leaf  │     │    │   │
│  │  │ - 文件树 │  │  │ (标签) │  │ (标签) │     │    │   │
│  │  │ - 搜索   │  │  └────────┘  └────────┘     │    │   │
│  │  │ - 书签   │  │                              │    │   │
│  │  └──────────┘  └──────────────────────────────┘    │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                             │
│  ┌─────────────────────────────────────────────────────┐   │
│  │                   组件层                              │   │
│  │  ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌──────────┐  │   │
│  │  │ Modal   │ │ Notice  │ │ Menu    │ │ Setting  │  │   │
│  │  │ (模态框)│ │ (通知)  │ │ (菜单)  │ │ (设置)   │  │   │
│  │  └─────────┘ └─────────┘ └─────────┘ └──────────┘  │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                             │
│  ┌─────────────────────────────────────────────────────┐   │
│  │                   底层渲染                            │   │
│  │  ┌─────────────────────┐  ┌──────────────────────┐  │   │
│  │  │   CodeMirror 6      │  │   原生 DOM 操作       │  │   │
│  │  │   (编辑器)          │  │   (containerEl)      │  │   │
│  │  └─────────────────────┘  └──────────────────────┘  │   │
│  └─────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
```

### 6.2 Workspace 工作区模型

```typescript
interface Workspace extends Events {
  leftSplit: WorkspaceSidedock;   // 左侧边栏
  rightSplit: WorkspaceSidedock;  // 右侧边栏
  leftRibbon: WorkspaceRibbon;    // 左侧图标栏
  rightRibbon: WorkspaceRibbon;   // 右侧图标栏
  rootSplit: WorkspaceSplit;      // 主编辑区
  activeLeaf: WorkspaceLeaf;      // 当前激活的标签页
}
```

**核心概念：**

| 概念 | 说明 |
|------|------|
| WorkspaceSplit | 可分割的容器，支持水平/垂直分割 |
| WorkspaceLeaf | 单个标签页，承载一个 View |
| View | 实际渲染内容的组件 |

### 6.3 View 视图系统

```typescript
abstract class ItemView extends View {
  containerEl: HTMLElement;      // 视图容器
  contentEl: HTMLElement;        // 内容区域

  abstract getViewType(): string;
  abstract getDisplayText(): string;

  onOpen(): Promise<void>;       // 视图打开时
  onClose(): Promise<void>;      // 视图关闭时
}
```

**内置视图类型：**
- `markdown` — Markdown 编辑/预览
- `file-explorer` — 文件浏览器
- `search` — 搜索结果
- `graph` — 关系图谱
- `outline` — 大纲视图

### 6.4 为什么不用 React/Vue？

| 考量 | 原生 DOM | React/Vue |
|------|---------|-----------|
| 包体积 | 无额外依赖 | 增加框架体积 |
| 性能 | 直接操作，无虚拟 DOM 开销 | 有 diffing 开销 |
| 插件兼容 | 插件可直接操作 DOM | 需要暴露框架 API |
| 灵活性 | 完全控制 | 受框架约束 |
| 学习曲线 | 插件开发者需懂原生 DOM | 需学习特定框架 |

**Obsidian 的选择：** 牺牲开发便利性，换取更小的包体积、更高的性能和更大的插件灵活性。

### 6.5 CSS 变量系统

Obsidian 通过 CSS 变量实现主题定制：

```css
body {
  --background-primary: #ffffff;
  --background-secondary: #f5f5f5;
  --text-normal: #2e3338;
  --text-muted: #888888;
  --interactive-accent: #7c3aed;
}
```

插件可通过覆盖 CSS 变量或直接注入样式来定制 UI。

---

## 7. 扩展点分类

### 7.1 命令系统

通过 `Ctrl/Cmd + P` 调出命令面板：

```typescript
this.addCommand({
  id: 'my-command',
  name: 'My Command',
  callback: () => {...}
});
```

四种回调类型：
1. `checkCallback` — 条件命令
2. `callback` — 无条件命令
3. `editorCheckCallback` — 需要编辑器且有条件
4. `editorCallback` — 需要编辑器

### 7.2 视图系统

```typescript
this.registerView(VIEW_TYPE, (leaf) => new MyView(leaf));
```

### 7.3 编辑器扩展

基于 CodeMirror 6：

```typescript
this.registerEditorExtension(myExtension);
```

### 7.4 Markdown 后处理

```typescript
this.registerMarkdownPostProcessor((el, ctx) => {...});
this.registerMarkdownCodeBlockProcessor('lang', (source, el, ctx) => {...});
```

### 7.5 UI 组件

- **Ribbon** — 左侧边栏图标
- **StatusBar** — 底部状态栏（仅桌面）
- **Modal** — 模态框
- **Notice** — 通知提示

---

## 8. 文件系统抽象

### 8.1 Vault 抽象层

**为什么不直接用 Node.js fs？**
- Obsidian 需要同时支持桌面（Electron）和移动端
- 移动端没有 Node.js，使用 Capacitor/原生文件 API
- Vault 统一抽象，插件不需要关心平台差异

```typescript
interface Vault extends Events {
  adapter: DataAdapter;  // 底层适配器
}
```

### 8.2 FileManager vs Vault

- `Vault.rename()` — 只重命名文件
- `FileManager.renameFile()` — 重命名并更新所有引用链接

---

## 9. 设计原则总结

| 原则 | 说明 |
|------|------|
| 简洁性 | 单一入口点、统一注册/清理模式、最小化 API |
| 灵活性 | 完全访问 DOM、可修改 UI、可注入自定义样式 |
| 信任模型 | 用户主动安装、完整权限、无运行时沙箱 |
| 稳定性 | Electron 版本更新适应、版本兼容性检查 |

---

## 参考

- [Obsidian Developer Documentation](https://docs.obsidian.md/)
- [Plugin Development | DeepWiki](https://deepwiki.com/obsidianmd/obsidian-api)
- [Obsidian Typings - Electron Changelog](https://fevol.github.io/obsidian-typings/resources/electron-changelog/)
