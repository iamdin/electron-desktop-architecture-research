# 07. 插件间通信

> **核心问题**：插件间如何协作？

---

## 概述

插件间通信决定了：
- 插件如何共享功能和数据
- 如何避免重复实现
- 如何构建插件生态系统
- 依赖关系如何管理

---

## VSCode 的插件间通信

### 1. 导出 API

VSCode 支持插件显式导出 API 供其他插件使用：

```typescript
// 插件 A：提供 API
// extension.ts
export function activate(context: vscode.ExtensionContext) {
  // 返回公开的 API
  return {
    getVersion: () => '1.0.0',
    doSomething: (input: string) => {
      return `Processed: ${input}`;
    },
    onDataChanged: new vscode.EventEmitter<string>().event
  };
}

// 类型定义（供其他插件使用）
// api.d.ts
export interface MyExtensionApi {
  getVersion(): string;
  doSomething(input: string): string;
  onDataChanged: vscode.Event<string>;
}
```

```typescript
// 插件 B：使用 API
import * as vscode from 'vscode';
import type { MyExtensionApi } from 'extension-a';

export async function activate(context: vscode.ExtensionContext) {
  // 获取其他插件
  const extensionA = vscode.extensions.getExtension<MyExtensionApi>('publisher.extension-a');

  if (extensionA) {
    // 确保插件已激活
    const api = await extensionA.activate();

    // 使用 API
    console.log(api.getVersion());
    console.log(api.doSomething('hello'));

    // 订阅事件
    api.onDataChanged(data => {
      console.log('Data changed:', data);
    });
  }
}
```

### 2. 依赖声明

```json
// 插件 B 的 package.json
{
  "name": "extension-b",
  "extensionDependencies": [
    "publisher.extension-a"
  ]
}
```

有了依赖声明：
- VSCode 会先激活依赖插件
- 安装时自动安装依赖
- 禁用依赖时会警告

### 3. 通过命令通信

```typescript
// 插件 A：注册命令
vscode.commands.registerCommand('extensionA.getData', () => {
  return { value: 42 };
});

// 插件 B：执行命令
const result = await vscode.commands.executeCommand('extensionA.getData');
console.log(result.value); // 42
```

### 4. 通过配置共享

```json
// 插件 A 提供配置
{
  "contributes": {
    "configuration": {
      "properties": {
        "sharedConfig.apiEndpoint": {
          "type": "string",
          "default": "https://api.example.com"
        }
      }
    }
  }
}
```

```typescript
// 插件 B 读取配置
const config = vscode.workspace.getConfiguration('sharedConfig');
const endpoint = config.get<string>('apiEndpoint');
```

### 5. 通过工作区状态共享

```typescript
// 插件 A：设置状态
vscode.commands.executeCommand('setContext', 'myExtension.isActive', true);

// 插件 B：在 when clause 中使用
// package.json
{
  "menus": {
    "editor/context": [{
      "command": "extensionB.action",
      "when": "myExtension.isActive"
    }]
  }
}
```

---

## Obsidian 的插件间通信

### 1. 直接访问其他插件

```typescript
// 插件 B：直接访问插件 A
export default class PluginB extends Plugin {
  onload() {
    // 通过 app.plugins 访问
    const pluginA = this.app.plugins.plugins['plugin-a'];

    if (pluginA) {
      // 直接调用方法（如果暴露了的话）
      pluginA.somePublicMethod();

      // 访问属性
      console.log(pluginA.settings);
    }
  }
}
```

### 2. 等待其他插件加载

```typescript
export default class PluginB extends Plugin {
  async onload() {
    // 等待布局就绪（通常其他插件也已加载）
    this.app.workspace.onLayoutReady(() => {
      this.initWithDependencies();
    });
  }

  initWithDependencies() {
    const pluginA = this.app.plugins.plugins['plugin-a'];
    if (pluginA && pluginA.settings) {
      // 使用插件 A 的功能
    }
  }
}
```

### 3. 通过事件通信

```typescript
// 插件 A：触发自定义事件
export default class PluginA extends Plugin {
  triggerCustomEvent(data: any) {
    this.app.workspace.trigger('plugin-a:custom-event', data);
  }
}

// 插件 B：监听事件
export default class PluginB extends Plugin {
  onload() {
    this.registerEvent(
      this.app.workspace.on('plugin-a:custom-event' as any, (data) => {
        console.log('Received:', data);
      })
    );
  }
}
```

### 4. 通过全局对象共享

```typescript
// 插件 A：暴露到全局
export default class PluginA extends Plugin {
  onload() {
    // @ts-ignore
    window.pluginAApi = {
      getVersion: () => this.manifest.version,
      doSomething: (input: string) => `Processed: ${input}`
    };
  }

  onunload() {
    // @ts-ignore
    delete window.pluginAApi;
  }
}

// 插件 B：使用全局 API
export default class PluginB extends Plugin {
  onload() {
    // @ts-ignore
    if (window.pluginAApi) {
      console.log(window.pluginAApi.getVersion());
    }
  }
}
```

### 5. 通过命令通信

```typescript
// 插件 A：注册命令并返回结果
export default class PluginA extends Plugin {
  onload() {
    this.addCommand({
      id: 'get-data',
      name: 'Get Data',
      callback: () => {
        return { value: 42 };  // 返回值可以被获取
      }
    });
  }
}

// 插件 B：执行命令
export default class PluginB extends Plugin {
  async getData() {
    // 注意：Obsidian 的命令不直接支持返回值
    // 需要通过其他方式（如事件）传递数据
    this.app.commands.executeCommandById('plugin-a:get-data');
  }
}
```

---

## 对比分析

### 通信方式对比表

| 通信方式 | VSCode | Obsidian |
|---------|--------|----------|
| **导出 API** | activate() 返回值 | 无正式机制 |
| **依赖声明** | extensionDependencies | 无 |
| **直接访问** | 不支持 | app.plugins.plugins |
| **命令调用** | executeCommand（支持返回值） | executeCommandById（不支持返回值） |
| **事件通信** | 通过导出的 Event | workspace.trigger |
| **全局对象** | 不推荐 | 常用 |
| **类型共享** | 独立 d.ts 包 | 无正式机制 |

### 安全性对比

| 方面 | VSCode | Obsidian |
|------|--------|----------|
| **访问控制** | 只能访问导出的 API | 可访问任何内容 |
| **类型安全** | 通过类型声明保证 | 无保证 |
| **版本兼容** | 可以检查版本 | 需自行处理 |
| **依赖顺序** | 宿主保证 | 需自行处理 |

### 代码示例对比

#### 提供和使用 API

**VSCode**：
```typescript
// 提供方
export function activate(context: vscode.ExtensionContext) {
  return {
    version: '1.0.0',
    processText: (text: string) => text.toUpperCase()
  };
}

// 使用方
const ext = vscode.extensions.getExtension('publisher.provider');
if (ext) {
  const api = await ext.activate();
  const result = api.processText('hello');
}
```

**Obsidian**：
```typescript
// 提供方
export default class Provider extends Plugin {
  public api = {
    version: '1.0.0',
    processText: (text: string) => text.toUpperCase()
  };
}

// 使用方
export default class Consumer extends Plugin {
  onload() {
    const provider = this.app.plugins.plugins['provider'] as Provider;
    if (provider?.api) {
      const result = provider.api.processText('hello');
    }
  }
}
```

---

## 插件生态模式

### 1. 核心 + 扩展模式

```
┌─────────────────────────────────────────┐
│              Core Plugin                 │
│  - 提供基础 API                          │
│  - 定义扩展点                            │
└─────────────────────────────────────────┘
         ▲         ▲         ▲
         │         │         │
    ┌────┴────┐ ┌──┴──┐ ┌────┴────┐
    │ Ext A   │ │Ext B│ │  Ext C  │
    │ 主题扩展 │ │语言包│ │功能扩展  │
    └─────────┘ └─────┘ └─────────┘
```

### 2. 松耦合协作模式

```
┌─────────────────┐     ┌─────────────────┐
│    Plugin A     │     │    Plugin B     │
│   Git 插件      │     │   AI 插件       │
└────────┬────────┘     └────────┬────────┘
         │                       │
         │  通过命令/事件通信     │
         └───────────┬───────────┘
                     ▼
              ┌──────────────┐
              │   用户工作流  │
              │  AI 辅助提交  │
              └──────────────┘
```

### 3. 服务提供者模式

```typescript
// VSCode 的 LSP 就是这种模式
// 多个语言服务器提供同类型服务

interface LanguageService {
  provideCompletions(document: Document, position: Position): CompletionItem[];
  provideHover(document: Document, position: Position): Hover;
}

// TypeScript 插件提供 JS/TS 支持
// Python 插件提供 Python 支持
// 统一的接口，不同的实现
```

---

## 对 Coding Agent Desktop 应用的建议

### 推荐的插件间通信设计

```typescript
// ==================== 1. API 注册机制 ====================

interface PluginAPI {
  /** API 版本 */
  readonly version: string;
  /** API 方法 */
  [key: string]: any;
}

// 插件注册 API
class MyPlugin extends BasePlugin {
  async onActivate() {
    // 注册公开 API
    this.context.registerAPI('my-plugin', {
      version: '1.0.0',
      processText: (text: string) => text.toUpperCase(),
      onDataChanged: this.dataChangedEmitter.event
    });
  }
}

// 其他插件使用 API
class ConsumerPlugin extends BasePlugin {
  async onActivate() {
    // 获取 API（类型安全）
    const api = await this.context.getAPI<MyPluginAPI>('my-plugin');
    if (api) {
      const result = api.processText('hello');
      api.onDataChanged(data => console.log(data));
    }
  }
}
```

### API 获取机制

```typescript
interface PluginContext {
  /**
   * 注册插件公开 API
   */
  registerAPI<T extends PluginAPI>(id: string, api: T): void;

  /**
   * 获取其他插件的 API
   * @param id 插件 ID
   * @param options 选项
   */
  getAPI<T extends PluginAPI>(
    id: string,
    options?: {
      /** 是否等待插件激活 */
      waitForActivation?: boolean;
      /** 超时时间 */
      timeout?: number;
      /** 最低版本要求 */
      minVersion?: string;
    }
  ): Promise<T | undefined>;

  /**
   * 检查插件是否已安装
   */
  hasPlugin(id: string): boolean;

  /**
   * 监听插件激活
   */
  onPluginActivated(id: string, callback: (api: PluginAPI) => void): Disposable;
}
```

### 事件总线

```typescript
// 应用级事件总线
interface EventBus {
  /**
   * 发布事件
   */
  emit<T>(event: string, data: T): void;

  /**
   * 订阅事件
   */
  on<T>(event: string, handler: (data: T) => void): Disposable;

  /**
   * 一次性订阅
   */
  once<T>(event: string, handler: (data: T) => void): Disposable;
}

// 使用
class PluginA extends BasePlugin {
  async onActivate() {
    // 发布事件
    this.context.eventBus.emit('plugin-a:data-updated', { value: 42 });
  }
}

class PluginB extends BasePlugin {
  async onActivate() {
    // 订阅事件
    this.context.registerDisposable(
      this.context.eventBus.on('plugin-a:data-updated', (data) => {
        console.log('Received:', data);
      })
    );
  }
}
```

### 服务扩展点

```typescript
// 定义服务接口
interface AIToolProvider {
  /** 工具名称 */
  name: string;
  /** 工具描述 */
  description: string;
  /** 执行工具 */
  execute(params: Record<string, any>): Promise<any>;
}

// 宿主定义扩展点
class AIService {
  private toolProviders: Map<string, AIToolProvider> = new Map();

  registerToolProvider(provider: AIToolProvider): Disposable {
    this.toolProviders.set(provider.name, provider);
    return {
      dispose: () => this.toolProviders.delete(provider.name)
    };
  }

  async executeTool(name: string, params: Record<string, any>) {
    const provider = this.toolProviders.get(name);
    if (!provider) throw new Error(`Tool not found: ${name}`);
    return provider.execute(params);
  }
}

// 插件提供工具
class WebSearchPlugin extends BasePlugin {
  async onActivate() {
    this.context.registerDisposable(
      this.context.services.ai.registerToolProvider({
        name: 'web_search',
        description: 'Search the web',
        execute: async (params) => {
          const results = await this.search(params.query);
          return results;
        }
      })
    );
  }
}
```

### 依赖管理

```typescript
// manifest.json
{
  "name": "plugin-b",
  "dependencies": {
    "plugin-a": "^1.0.0"  // 支持语义化版本
  },
  "optionalDependencies": {
    "plugin-c": "^2.0.0"
  }
}

// 运行时检查
class PluginB extends BasePlugin {
  async onActivate() {
    // 必需依赖（由框架保证已激活）
    const pluginA = await this.context.getAPI<PluginAAPI>('plugin-a');

    // 可选依赖
    const pluginC = await this.context.getAPI<PluginCAPI>('plugin-c');
    if (pluginC) {
      // 使用可选功能
    }
  }
}
```

### 版本协商

```typescript
// API 版本检查
const api = await this.context.getAPI<MyAPI>('my-plugin', {
  minVersion: '1.2.0'
});

if (!api) {
  // 版本不满足或插件未安装
  this.context.ui.showWarning(
    'This plugin requires my-plugin v1.2.0 or higher'
  );
  return;
}

// API 内部版本检查
interface VersionedAPI {
  version: string;
  // v1.0.0
  basicMethod(): void;
  // v1.2.0
  advancedMethod?(): void;
}

const api = await this.context.getAPI<VersionedAPI>('my-plugin');
if (api && semver.gte(api.version, '1.2.0')) {
  api.advancedMethod?.();
}
```

---

## 关键决策清单

1. **是否支持插件间直接访问？**
   - 开放：灵活但难以维护
   - 限制：安全但需要 API 设计

2. **API 如何暴露？**
   - 返回值方式（VSCode）
   - 注册方式（推荐）
   - 全局对象（不推荐）

3. **依赖如何声明和管理？**
   - 必需依赖 vs 可选依赖
   - 版本要求

4. **激活顺序如何保证？**
   - 框架保证依赖先激活
   - 开发者自行等待

5. **类型如何共享？**
   - 独立类型包
   - 内联类型声明

---

## 参考资料

- [VSCode Extension API - getExtension](https://code.visualstudio.com/api/references/vscode-api#extensions.getExtension)
- [VSCode Extension Dependencies](https://code.visualstudio.com/api/references/extension-manifest#extension-dependencies)
- [Obsidian Plugin API](https://docs.obsidian.md/Plugins/Getting+started/Build+a+plugin)
