# 02. 模块与依赖系统

> **核心问题**：模块如何组织和协作？

---

## 概述

模块系统决定了：
- 代码如何组织和分层
- 模块之间如何发现和调用
- 依赖如何注入和管理
- 循环依赖如何处理

---

## VSCode 的模块系统

### 分层架构

VSCode 采用严格的分层架构，代码按层次组织：

```
src/vs/
├── base/           # 基础工具层（无依赖）
│   ├── common/     # 纯 JS，可在任何环境运行
│   ├── browser/    # 浏览器特定
│   ├── node/       # Node.js 特定
│   └── parts/      # UI 基础组件
│
├── platform/       # 平台服务层（依赖 base）
│   ├── configuration/
│   ├── files/
│   ├── storage/
│   └── ...
│
├── editor/         # Monaco 编辑器（依赖 base）
│   ├── common/
│   ├── browser/
│   └── contrib/
│
└── workbench/      # 工作台层（依赖 platform + editor）
    ├── api/        # 插件 API 实现
    ├── contrib/    # 内置功能
    ├── services/   # 工作台服务
    └── browser/    # UI 组件
```

### 分层规则

```
┌─────────────────────────────────────────────────────────────┐
│                     workbench                                │
│              (可以依赖下面所有层)                             │
├─────────────────────────────────────────────────────────────┤
│          platform              editor                        │
│      (可以依赖 base)        (可以依赖 base)                  │
├─────────────────────────────────────────────────────────────┤
│                        base                                  │
│                    (不能依赖任何层)                          │
└─────────────────────────────────────────────────────────────┘

规则：上层可以依赖下层，下层不能依赖上层
```

### 依赖注入系统

VSCode 使用装饰器实现依赖注入：

```typescript
// 1. 定义服务标识符
// src/vs/platform/files/common/files.ts
export const IFileService = createDecorator<IFileService>('fileService');

export interface IFileService {
  readonly _serviceBrand: undefined;
  readFile(resource: URI): Promise<IFileContent>;
  writeFile(resource: URI, content: VSBuffer): Promise<void>;
  // ...
}
```

```typescript
// 2. 实现服务
// src/vs/platform/files/node/fileService.ts
export class FileService implements IFileService {
  readonly _serviceBrand: undefined;

  constructor(
    @ILogService private readonly logService: ILogService
  ) {}

  async readFile(resource: URI): Promise<IFileContent> {
    this.logService.trace('Reading file:', resource.toString());
    // 实现...
  }
}
```

```typescript
// 3. 注册服务
// src/vs/workbench/electron-sandbox/desktop.main.ts
const services = new ServiceCollection();
services.set(IFileService, new SyncDescriptor(FileService));
services.set(ILogService, new SyncDescriptor(LogService));

const instantiationService = new InstantiationService(services);
```

```typescript
// 4. 使用服务（自动注入）
// src/vs/workbench/contrib/files/browser/fileActions.ts
class OpenFileAction extends Action {
  constructor(
    @IFileService private readonly fileService: IFileService,
    @IEditorService private readonly editorService: IEditorService
  ) {
    super();
  }

  async run(): Promise<void> {
    const content = await this.fileService.readFile(this.uri);
    await this.editorService.openEditor({ resource: this.uri });
  }
}
```

### createDecorator 实现原理

```typescript
// src/vs/platform/instantiation/common/instantiation.ts
export function createDecorator<T>(serviceId: string): ServiceIdentifier<T> {
  const id = function (target: any, key: string, index: number): void {
    // 存储依赖信息到元数据
    const dependencies = getServiceDependencies(target);
    dependencies.push({ id, index });
  };
  id.toString = () => serviceId;
  return id as ServiceIdentifier<T>;
}
```

### Contribution 机制

除了服务，VSCode 还有 Contribution 机制用于扩展功能：

```typescript
// 定义 Contribution Point
// src/vs/workbench/services/actions/common/menusExtensionPoint.ts
export const menusExtensionPoint = ExtensionsRegistry.registerExtensionPoint<MenusContribution>({
  extensionPoint: 'menus',
  jsonSchema: menusSchema,
  activationEventsGenerator: generateMenuActivationEvents
});

// 插件通过 package.json 声明
{
  "contributes": {
    "menus": {
      "editor/context": [
        {
          "command": "myExtension.myCommand",
          "when": "editorTextFocus"
        }
      ]
    }
  }
}
```

---

## Obsidian 的模块系统

### 扁平结构

Obsidian 采用相对扁平的结构，以 `App` 对象为中心：

```typescript
// App 对象是全局单例，包含所有核心服务
interface App {
  // 核心服务
  vault: Vault;              // 文件管理
  workspace: Workspace;       // 布局管理
  metadataCache: MetadataCache; // 元数据缓存

  // 插件系统
  plugins: {
    enabledPlugins: Set<string>;
    plugins: Record<string, Plugin>;
  };

  // 内部状态（未公开但可访问）
  internalPlugins: InternalPlugins;
  commands: Commands;
  // ...
}
```

### 全局单例模式

```typescript
// 插件通过 this.app 访问所有服务
export default class MyPlugin extends Plugin {
  async onload() {
    // 访问 Vault 服务
    const files = this.app.vault.getMarkdownFiles();

    // 访问 Workspace 服务
    const activeView = this.app.workspace.getActiveViewOfType(MarkdownView);

    // 访问 MetadataCache 服务
    const cache = this.app.metadataCache.getFileCache(file);

    // 甚至可以访问其他插件
    const otherPlugin = this.app.plugins.plugins['other-plugin'];
  }
}
```

### 服务注册（内部机制）

虽然 Obsidian 闭源，但从 API 可以推断其内部结构：

```typescript
// 推测的内部结构
class App {
  vault: Vault;
  workspace: Workspace;
  metadataCache: MetadataCache;

  constructor() {
    // 按顺序初始化服务
    this.vault = new Vault(this);
    this.metadataCache = new MetadataCache(this);
    this.workspace = new Workspace(this);

    // 每个服务都持有 app 引用，形成双向关联
  }
}

class Vault {
  constructor(private app: App) {}

  // 可以访问其他服务
  async getFileCache(file: TFile) {
    return this.app.metadataCache.getFileCache(file);
  }
}
```

### 事件驱动协作

Obsidian 模块间主要通过事件通信：

```typescript
// 监听 Vault 事件
this.registerEvent(
  this.app.vault.on('create', (file) => {
    console.log('File created:', file.path);
  })
);

// 监听 Workspace 事件
this.registerEvent(
  this.app.workspace.on('active-leaf-change', (leaf) => {
    console.log('Active leaf changed:', leaf);
  })
);

// 监听 MetadataCache 事件
this.registerEvent(
  this.app.metadataCache.on('changed', (file) => {
    console.log('Metadata changed:', file.path);
  })
);
```

---

## 对比分析

### 架构对比表

| 方面 | VSCode | Obsidian |
|------|--------|----------|
| **代码组织** | 严格分层 (base/platform/workbench) | 扁平结构 |
| **依赖管理** | DI 容器 + 装饰器 | 全局单例 (app) |
| **服务发现** | 静态注册 + ServiceIdentifier | 直接访问 app 属性 |
| **模块边界** | 严格隔离，编译时检查 | 松散，运行时访问 |
| **循环依赖** | 通过接口抽象避免 | 双向引用（app ↔ service） |
| **扩展机制** | Contribution Points | 事件 + 直接调用 |
| **可测试性** | 高（可 mock 依赖） | 一般（依赖全局状态） |

### 优劣分析

#### VSCode DI 的优势

1. **可测试性**：轻松 mock 依赖
   ```typescript
   // 测试时注入 mock 服务
   const mockFileService = { readFile: jest.fn() };
   const action = instantiationService.createInstance(
     OpenFileAction,
     mockFileService
   );
   ```

2. **明确的依赖关系**：通过构造函数声明
3. **编译时检查**：TypeScript 类型安全
4. **解耦**：服务实现可替换

#### VSCode DI 的劣势

1. **学习成本**：需要理解 DI 概念
2. **样板代码**：每个服务需要定义 ID、接口、实现
3. **调试困难**：依赖注入链可能很长
4. **过度设计**：对小型项目来说过于复杂

#### Obsidian 全局单例的优势

1. **简单直观**：直接通过 `app.xxx` 访问
2. **低学习成本**：无需理解 DI
3. **快速开发**：代码量少
4. **灵活性**：可以访问任何东西

#### Obsidian 全局单例的劣势

1. **难以测试**：依赖全局状态
2. **耦合度高**：模块间可随意访问
3. **隐式依赖**：不清楚模块依赖了什么
4. **重构困难**：改动可能影响很多地方

---

## 代码示例对比

### 定义和使用服务

**VSCode**：
```typescript
// 1. 定义服务接口
export const INotificationService = createDecorator<INotificationService>('notificationService');

export interface INotificationService {
  info(message: string): void;
  warn(message: string): void;
  error(message: string): void;
}

// 2. 实现服务
export class NotificationService implements INotificationService {
  constructor(
    @ILogService private readonly logService: ILogService
  ) {}

  info(message: string): void {
    this.logService.info(message);
    // 显示通知...
  }
}

// 3. 使用服务
class MyFeature {
  constructor(
    @INotificationService private readonly notificationService: INotificationService
  ) {}

  doSomething(): void {
    this.notificationService.info('Operation completed');
  }
}
```

**Obsidian**：
```typescript
// 直接使用内置的 Notice
export default class MyPlugin extends Plugin {
  doSomething(): void {
    new Notice('Operation completed');
  }
}

// 如果要自定义通知服务
class NotificationService {
  constructor(private app: App) {}

  info(message: string): void {
    new Notice(message);
  }
}

export default class MyPlugin extends Plugin {
  private notificationService: NotificationService;

  onload() {
    this.notificationService = new NotificationService(this.app);
  }

  doSomething(): void {
    this.notificationService.info('Operation completed');
  }
}
```

### 模块间通信

**VSCode**：
```typescript
// 通过事件服务
export class FileWatcher {
  constructor(
    @IFileService private readonly fileService: IFileService,
    @IEventService private readonly eventService: IEventService
  ) {
    this.fileService.onDidFilesChange(e => {
      this.eventService.fire('files:changed', e);
    });
  }
}

export class FileExplorer {
  constructor(
    @IEventService private readonly eventService: IEventService
  ) {
    this.eventService.on('files:changed', this.refresh.bind(this));
  }
}
```

**Obsidian**：
```typescript
// 直接监听 vault 事件
export default class MyPlugin extends Plugin {
  onload() {
    this.registerEvent(
      this.app.vault.on('modify', (file) => {
        this.handleFileChange(file);
      })
    );
  }
}
```

---

## 中间方案

### 1. 轻量级 DI（无装饰器）

```typescript
// 服务容器
class Container {
  private services = new Map<string, any>();

  register<T>(id: string, factory: () => T): void {
    this.services.set(id, factory);
  }

  get<T>(id: string): T {
    const factory = this.services.get(id);
    if (!factory) throw new Error(`Service not found: ${id}`);
    return factory();
  }
}

// 使用
const container = new Container();
container.register('fileService', () => new FileService());
container.register('editorService', () => new EditorService(container.get('fileService')));

const editorService = container.get<EditorService>('editorService');
```

### 2. 服务定位器模式

```typescript
// 服务定位器
class ServiceLocator {
  private static instance: ServiceLocator;
  private services = new Map<string, any>();

  static getInstance(): ServiceLocator {
    if (!ServiceLocator.instance) {
      ServiceLocator.instance = new ServiceLocator();
    }
    return ServiceLocator.instance;
  }

  register<T>(id: string, service: T): void {
    this.services.set(id, service);
  }

  get<T>(id: string): T {
    return this.services.get(id);
  }
}

// 使用
const locator = ServiceLocator.getInstance();
locator.register('fileService', new FileService());

class MyFeature {
  private fileService = ServiceLocator.getInstance().get<FileService>('fileService');
}
```

### 3. 基于 Context 的依赖传递（React 风格）

```typescript
// 创建 Context
interface AppContext {
  fileService: FileService;
  editorService: EditorService;
  notificationService: NotificationService;
}

const AppContext = createContext<AppContext | null>(null);

// Provider
function AppProvider({ children }) {
  const context = useMemo(() => ({
    fileService: new FileService(),
    editorService: new EditorService(),
    notificationService: new NotificationService(),
  }), []);

  return <AppContext.Provider value={context}>{children}</AppContext.Provider>;
}

// 使用
function useFileService() {
  const context = useContext(AppContext);
  if (!context) throw new Error('Must be used within AppProvider');
  return context.fileService;
}
```

---

## 对 AI Chat + Editor 应用的建议

### 推荐方案：分层 + 轻量 DI

```
┌─────────────────────────────────────────────────────────────┐
│                       app (应用层)                           │
│  - UI 组件、页面                                             │
│  - 依赖 core 和 services                                    │
├─────────────────────────────────────────────────────────────┤
│                    services (服务层)                         │
│  - ChatService, EditorService, PluginService                │
│  - 依赖 core                                                │
├─────────────────────────────────────────────────────────────┤
│                      core (核心层)                           │
│  - 基础工具、类型定义                                        │
│  - 不依赖任何层                                              │
└─────────────────────────────────────────────────────────────┘
```

### 具体实现

```typescript
// core/types.ts - 服务接口定义
export interface IChatService {
  sendMessage(content: string): Promise<ChatResponse>;
  streamMessage(content: string): AsyncIterable<string>;
}

export interface IEditorService {
  openFile(path: string): Promise<void>;
  getContent(): string;
}

// core/di.ts - 轻量级 DI
export class ServiceContainer {
  private services = new Map<symbol, any>();
  private factories = new Map<symbol, () => any>();

  register<T>(id: symbol, factory: () => T): void {
    this.factories.set(id, factory);
  }

  get<T>(id: symbol): T {
    if (!this.services.has(id)) {
      const factory = this.factories.get(id);
      if (!factory) throw new Error(`Service not found`);
      this.services.set(id, factory());
    }
    return this.services.get(id);
  }
}

// services/chat.ts
export const IChatService = Symbol('ChatService');

export class ChatService implements IChatService {
  constructor(private container: ServiceContainer) {}

  async sendMessage(content: string): Promise<ChatResponse> {
    // 实现...
  }
}

// app/index.ts - 初始化
const container = new ServiceContainer();
container.register(IChatService, () => new ChatService(container));
container.register(IEditorService, () => new EditorService(container));

// 导出供插件使用
export const app = {
  chat: container.get<IChatService>(IChatService),
  editor: container.get<IEditorService>(IEditorService),
};
```

### 插件如何访问服务

```typescript
// 插件通过注入的 app 对象访问服务
export default class MyPlugin implements Plugin {
  constructor(private app: App) {}

  async onload() {
    // 类似 Obsidian 的简洁访问方式
    const response = await this.app.chat.sendMessage('Hello');

    // 但背后是 DI 管理的服务
  }
}
```

### 设计原则

| 原则 | 说明 |
|------|------|
| **接口隔离** | 插件只能访问定义好的接口，不能访问内部实现 |
| **明确依赖** | 服务的依赖在构造函数中声明 |
| **可替换** | 服务实现可以在不同环境替换（测试、生产） |
| **简洁 API** | 对外暴露简洁的 `app.xxx` 访问方式 |

---

## 关键决策清单

1. **选择什么级别的分层？**
   - 严格分层（VSCode 风格）：适合大型团队、长期维护
   - 扁平结构（Obsidian 风格）：适合快速迭代、小型团队

2. **选择什么依赖管理方式？**
   - DI 容器：可测试性好，但学习成本高
   - 全局单例：简单直接，但难以测试
   - 轻量级 DI：平衡方案

3. **模块边界如何强制执行？**
   - 编译时检查（ESLint 规则）
   - 目录结构约束
   - 代码审查

4. **循环依赖如何处理？**
   - 提取接口
   - 延迟加载
   - 事件解耦

---

## 参考资料

- [VSCode 依赖注入实现](https://github.com/microsoft/vscode/tree/main/src/vs/platform/instantiation)
- [InversifyJS - TypeScript DI 框架](https://inversify.io/)
- [依赖注入设计模式](https://en.wikipedia.org/wiki/Dependency_injection)
