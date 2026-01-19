# 06. 插件生命周期

> **核心问题**：插件如何加载和卸载？

---

## 概述

插件生命周期管理决定了：
- 插件何时被加载和激活
- 插件如何初始化资源
- 插件如何清理资源
- 插件如何更新和重载

---

## VSCode 的插件生命周期

### 生命周期阶段

```
┌─────────────────────────────────────────────────────────────┐
│                    Extension Lifecycle                       │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  1. Discovery (发现)                                         │
│     └── 扫描 extensions 目录，解析 package.json              │
│                                                              │
│  2. Loading (加载)                                           │
│     └── 验证 manifest，注册 contribution points             │
│                                                              │
│  3. Activation (激活) - 按需触发                             │
│     └── 执行 activate() 函数                                │
│                                                              │
│  4. Running (运行)                                           │
│     └── 插件正常工作                                         │
│                                                              │
│  5. Deactivation (停用)                                      │
│     └── 执行 deactivate() 函数                              │
│                                                              │
│  6. Uninstall (卸载)                                         │
│     └── 删除文件，清理存储                                   │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### 入口函数

```typescript
// extension.ts
import * as vscode from 'vscode';

// 激活时调用
export function activate(context: vscode.ExtensionContext) {
  console.log('Extension activated!');

  // 注册命令
  const disposable = vscode.commands.registerCommand('myExt.hello', () => {
    vscode.window.showInformationMessage('Hello!');
  });

  // 添加到订阅列表，会在停用时自动清理
  context.subscriptions.push(disposable);

  // 可以返回 API 供其他插件使用
  return {
    getVersion: () => '1.0.0'
  };
}

// 停用时调用
export function deactivate() {
  console.log('Extension deactivated!');
  // 清理全局资源
}
```

### ExtensionContext

```typescript
interface ExtensionContext {
  // 订阅管理（自动清理）
  subscriptions: Disposable[];

  // 存储路径
  globalStoragePath: string;      // 全局存储
  storagePath: string | undefined; // 工作区存储
  logPath: string;                 // 日志路径

  // 插件路径
  extensionPath: string;           // 插件安装路径
  extensionUri: Uri;

  // 状态存储
  globalState: Memento;            // 全局状态（跨工作区）
  workspaceState: Memento;         // 工作区状态

  // 密钥存储
  secrets: SecretStorage;

  // 插件模式
  extensionMode: ExtensionMode;    // Production | Development | Test

  // 环境信息
  environmentVariableCollection: EnvironmentVariableCollection;
}
```

### 延迟激活

VSCode 支持多种激活事件实现按需加载：

```json
{
  "activationEvents": [
    "onLanguage:javascript",
    "onCommand:myExt.hello",
    "onView:myTreeView",
    "workspaceContains:**/.myconfig",
    "onStartupFinished"
  ]
}
```

```typescript
// 激活流程
// 1. VSCode 启动
// 2. 扫描所有插件的 activationEvents
// 3. 当触发条件满足时，才加载并执行 activate()
```

### Disposable 模式

```typescript
export function activate(context: vscode.ExtensionContext) {
  // 所有注册都返回 Disposable
  const cmd = vscode.commands.registerCommand('...', () => {});
  const provider = vscode.languages.registerCompletionItemProvider('...', provider);
  const watcher = vscode.workspace.createFileSystemWatcher('**/*.md');

  // 方式1：添加到 subscriptions（推荐）
  context.subscriptions.push(cmd, provider, watcher);

  // 方式2：手动管理
  const disposables: vscode.Disposable[] = [];
  disposables.push(cmd);

  // 创建组合 Disposable
  return vscode.Disposable.from(...disposables);
}
```

### 状态持久化

```typescript
export function activate(context: vscode.ExtensionContext) {
  // 全局状态（所有工作区共享）
  const count = context.globalState.get<number>('activationCount', 0);
  context.globalState.update('activationCount', count + 1);

  // 工作区状态
  const lastFile = context.workspaceState.get<string>('lastOpenedFile');

  // 密钥存储
  await context.secrets.store('apiKey', 'secret-value');
  const apiKey = await context.secrets.get('apiKey');
}
```

---

## Obsidian 的插件生命周期

### 生命周期阶段

```
┌─────────────────────────────────────────────────────────────┐
│                     Plugin Lifecycle                         │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  1. Discovery (发现)                                         │
│     └── 扫描 .obsidian/plugins 目录                         │
│                                                              │
│  2. Loading (加载) - 启动时立即执行                          │
│     └── 执行 onload() 函数                                  │
│                                                              │
│  3. Layout Ready (布局就绪)                                  │
│     └── UI 完全加载后                                        │
│                                                              │
│  4. Running (运行)                                           │
│     └── 插件正常工作                                         │
│                                                              │
│  5. Unloading (卸载)                                         │
│     └── 执行 onunload() 函数                                │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### 入口函数

```typescript
// main.ts
import { Plugin, PluginManifest } from 'obsidian';

export default class MyPlugin extends Plugin {
  settings: MyPluginSettings;

  // 加载时调用（可能在布局就绪前）
  async onload() {
    console.log('Loading plugin: ' + this.manifest.name);

    // 加载设置
    await this.loadSettings();

    // 等待布局就绪
    this.app.workspace.onLayoutReady(() => {
      this.initializeAfterLayout();
    });

    // 注册命令
    this.addCommand({
      id: 'my-command',
      name: 'My Command',
      callback: () => {}
    });

    // 注册事件（自动在 unload 时清理）
    this.registerEvent(
      this.app.vault.on('create', (file) => {})
    );

    // 注册 DOM 事件
    this.registerDomEvent(document, 'click', (evt) => {});

    // 注册定时器
    this.registerInterval(
      window.setInterval(() => console.log('tick'), 5000)
    );
  }

  // 卸载时调用
  onunload() {
    console.log('Unloading plugin: ' + this.manifest.name);
    // 注册的事件/定时器会自动清理
    // 这里清理其他资源
  }

  async loadSettings() {
    this.settings = Object.assign({}, DEFAULT_SETTINGS, await this.loadData());
  }

  async saveSettings() {
    await this.saveData(this.settings);
  }
}
```

### Component 基类

Obsidian 的 `Plugin` 继承自 `Component`，提供自动资源管理：

```typescript
abstract class Component {
  // 子组件管理
  addChild<T extends Component>(child: T): T;
  removeChild(child: Component): void;

  // 事件注册（自动清理）
  registerEvent(event: EventRef): void;

  // DOM 事件注册（自动清理）
  registerDomEvent<K extends keyof WindowEventMap>(
    el: Window,
    type: K,
    callback: (ev: WindowEventMap[K]) => any
  ): void;

  // 定时器注册（自动清理）
  registerInterval(id: number): number;

  // 生命周期
  load(): void;
  onload(): void;
  unload(): void;
  onunload(): void;
}
```

### 数据持久化

```typescript
export default class MyPlugin extends Plugin {
  settings: MyPluginSettings;

  async onload() {
    // 加载数据（从 data.json）
    const data = await this.loadData();
    this.settings = Object.assign({}, DEFAULT_SETTINGS, data);
  }

  async saveSettings() {
    // 保存数据（到 data.json）
    await this.saveData(this.settings);
  }
}

// 存储位置：.obsidian/plugins/my-plugin/data.json
```

### 布局就绪

```typescript
export default class MyPlugin extends Plugin {
  async onload() {
    // onload 时 workspace 可能还没准备好
    // 某些操作需要等待布局就绪

    // 方式1：回调
    this.app.workspace.onLayoutReady(() => {
      this.initUI();
    });

    // 方式2：条件检查
    if (this.app.workspace.layoutReady) {
      this.initUI();
    } else {
      this.app.workspace.onLayoutReady(() => this.initUI());
    }
  }

  initUI() {
    // 安全地操作 workspace
    const leaf = this.app.workspace.getLeaf();
  }
}
```

---

## 对比分析

### 生命周期对比表

| 阶段 | VSCode | Obsidian |
|------|--------|----------|
| **发现** | 扫描 extensions 目录 | 扫描 .obsidian/plugins |
| **加载时机** | 按需（activationEvents） | 启动时立即 |
| **入口函数** | activate() / deactivate() | onload() / onunload() |
| **资源管理** | context.subscriptions | registerEvent/Interval |
| **状态存储** | globalState / workspaceState | loadData() / saveData() |
| **热重载** | 需重启 Extension Host | Cmd+R 刷新窗口 |

### 激活策略对比

| 方面 | VSCode | Obsidian |
|------|--------|----------|
| **默认行为** | 延迟激活 | 立即加载 |
| **启动性能** | 好（按需加载） | 一般（全部加载） |
| **首次交互** | 可能有延迟 | 无延迟 |
| **内存占用** | 低（未激活不占用） | 高（全部加载） |

### 资源清理对比

**VSCode**：
```typescript
export function activate(context: vscode.ExtensionContext) {
  // 必须手动添加到 subscriptions
  context.subscriptions.push(
    vscode.commands.registerCommand('...', () => {}),
    vscode.workspace.createFileSystemWatcher('**/*.md'),
    new MyService()  // 需要实现 Disposable
  );
}

export function deactivate() {
  // subscriptions 会自动 dispose
  // 这里清理不在 subscriptions 中的资源
}
```

**Obsidian**：
```typescript
export default class MyPlugin extends Plugin {
  onload() {
    // 使用 register* 方法自动管理
    this.registerEvent(this.app.vault.on('create', () => {}));
    this.registerDomEvent(document, 'click', () => {});
    this.registerInterval(window.setInterval(() => {}, 1000));

    // 添加子组件
    this.addChild(new MyComponent());
  }

  onunload() {
    // register* 和 addChild 添加的资源自动清理
    // 这里清理其他资源
  }
}
```

---

## 热重载

### VSCode

```typescript
// VSCode 不支持真正的热重载
// 需要重启 Extension Host

// 开发时按 F5 启动调试会创建新的 Extension Host
// 或使用命令 "Developer: Restart Extension Host"
```

### Obsidian

```typescript
// 可以禁用/启用插件实现重载
// 或按 Cmd+R 刷新整个窗口

// 插件需要正确实现 onunload() 清理资源
export default class MyPlugin extends Plugin {
  private observer: MutationObserver;

  onload() {
    this.observer = new MutationObserver(() => {});
    this.observer.observe(document.body, { childList: true });
  }

  onunload() {
    // 必须清理，否则重载后会有多个 observer
    this.observer.disconnect();
  }
}
```

---

## 对 AI Chat + Editor 应用的建议

### 推荐的生命周期设计

```typescript
// 插件生命周期接口
interface Plugin {
  /** 插件清单 */
  readonly manifest: PluginManifest;

  /** 插件上下文 */
  readonly context: PluginContext;

  /**
   * 激活时调用
   * - 在此注册命令、视图等
   * - 可以是异步的
   */
  onActivate(): Promise<void> | void;

  /**
   * 布局就绪后调用
   * - 可以安全地操作 UI
   */
  onReady?(): Promise<void> | void;

  /**
   * 停用时调用
   * - 清理资源
   */
  onDeactivate?(): Promise<void> | void;
}
```

### PluginContext 设计

```typescript
interface PluginContext {
  // ==================== 资源管理 ====================

  /** 注册一次性资源，停用时自动清理 */
  registerDisposable(disposable: Disposable): void;

  /** 注册事件监听，停用时自动清理 */
  registerEvent<T>(
    emitter: EventEmitter<T>,
    event: string,
    handler: (data: T) => void
  ): void;

  /** 注册定时器，停用时自动清理 */
  registerInterval(callback: () => void, ms: number): void;
  registerTimeout(callback: () => void, ms: number): void;

  // ==================== 存储 ====================

  /** 全局状态（跨会话、跨工作区） */
  globalState: StateStorage;

  /** 会话状态（当前工作区） */
  sessionState: StateStorage;

  /** 持久化数据 */
  loadData<T>(): Promise<T | null>;
  saveData<T>(data: T): Promise<void>;

  /** 密钥存储 */
  secrets: SecretStorage;

  // ==================== 路径 ====================

  /** 插件安装目录 */
  readonly extensionPath: string;

  /** 数据存储目录 */
  readonly dataPath: string;

  /** 日志目录 */
  readonly logPath: string;

  // ==================== 注册 ====================

  /** 注册命令 */
  registerCommand(id: string, handler: CommandHandler): Disposable;

  /** 注册视图 */
  registerView(id: string, factory: ViewFactory): Disposable;

  /** 注册设置页 */
  registerSettingTab(tab: SettingTab): Disposable;

  // ==================== 工具 ====================

  /** 日志 */
  readonly logger: Logger;

  /** 配置 */
  getConfiguration<T>(section: string): T;
}
```

### 激活策略

```typescript
// manifest.json
{
  "name": "my-plugin",
  "activationPolicy": "lazy",  // "immediate" | "lazy" | "onDemand"
  "activationEvents": [
    "onCommand:my-plugin.activate",
    "onView:my-plugin.sidebar",
    "onChat"
  ]
}
```

```typescript
// 激活策略实现
enum ActivationPolicy {
  /** 启动时立即激活 */
  Immediate = 'immediate',
  /** 延迟激活（启动完成后） */
  Lazy = 'lazy',
  /** 按需激活（等待触发条件） */
  OnDemand = 'onDemand'
}

class PluginManager {
  async loadPlugins() {
    const plugins = await this.discoverPlugins();

    // 按策略分组
    const immediate = plugins.filter(p => p.manifest.activationPolicy === 'immediate');
    const lazy = plugins.filter(p => p.manifest.activationPolicy === 'lazy');
    const onDemand = plugins.filter(p => p.manifest.activationPolicy === 'onDemand');

    // 立即激活
    await Promise.all(immediate.map(p => this.activatePlugin(p)));

    // 启动完成后激活
    this.onStartupFinished(() => {
      lazy.forEach(p => this.activatePlugin(p));
    });

    // 按需激活 - 注册触发器
    onDemand.forEach(p => {
      this.registerActivationTriggers(p);
    });
  }
}
```

### 资源管理最佳实践

```typescript
// 插件基类
abstract class BasePlugin implements Plugin {
  private disposables: Disposable[] = [];
  private intervals: number[] = [];
  private timeouts: number[] = [];

  /** 注册可清理资源 */
  protected register(disposable: Disposable): void {
    this.disposables.push(disposable);
  }

  /** 注册定时器 */
  protected registerInterval(callback: () => void, ms: number): number {
    const id = window.setInterval(callback, ms);
    this.intervals.push(id);
    return id;
  }

  /** 自动清理 */
  async onDeactivate(): Promise<void> {
    // 清理 disposables
    this.disposables.forEach(d => d.dispose());
    this.disposables = [];

    // 清理定时器
    this.intervals.forEach(id => window.clearInterval(id));
    this.intervals = [];

    this.timeouts.forEach(id => window.clearTimeout(id));
    this.timeouts = [];
  }
}

// 使用
class MyPlugin extends BasePlugin {
  async onActivate() {
    // 自动管理生命周期
    this.register(
      this.context.registerCommand('my-command', () => {})
    );

    this.registerInterval(() => {
      console.log('tick');
    }, 5000);
  }

  // 不需要手动实现 onDeactivate，除非有额外清理
}
```

### 热重载支持

```typescript
class PluginManager {
  /** 重载单个插件 */
  async reloadPlugin(id: string): Promise<void> {
    const plugin = this.plugins.get(id);
    if (!plugin) return;

    // 1. 停用
    await this.deactivatePlugin(plugin);

    // 2. 重新加载代码（开发模式）
    if (this.isDevelopment) {
      delete require.cache[plugin.mainPath];
    }

    // 3. 重新激活
    await this.activatePlugin(plugin);

    this.emit('plugin:reloaded', id);
  }

  /** 开发模式下的文件监听 */
  watchPluginChanges(pluginPath: string): void {
    const watcher = fs.watch(pluginPath, { recursive: true });
    watcher.on('change', debounce(() => {
      this.reloadPlugin(this.getPluginIdFromPath(pluginPath));
    }, 500));
  }
}
```

---

## 关键决策清单

1. **激活策略是什么？**
   - 立即激活：简单，但影响启动性能
   - 按需激活：复杂，但启动快

2. **如何管理资源？**
   - 显式 Disposable（VSCode 风格）
   - 自动注册清理（Obsidian 风格）

3. **状态存储在哪里？**
   - 全局 vs 工作区
   - 内存 vs 持久化

4. **是否支持热重载？**
   - 开发体验 vs 复杂度
   - 需要插件正确实现清理

5. **如何处理异步初始化？**
   - 同步 onload vs 异步 onActivate
   - 布局就绪回调

---

## 参考资料

- [VSCode Extension Anatomy](https://code.visualstudio.com/api/get-started/extension-anatomy)
- [VSCode Extension Context](https://code.visualstudio.com/api/references/vscode-api#ExtensionContext)
- [Obsidian Plugin Lifecycle](https://docs.obsidian.md/Plugins/Getting+started/Build+a+plugin)
