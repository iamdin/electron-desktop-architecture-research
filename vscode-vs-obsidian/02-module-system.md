# 02. 模块划分与代码组织

> **核心问题**：代码如何划分和组织？模块如何融合？

---

## 概述

模块划分决定了：
- 代码的层次结构和边界
- 团队协作的分工方式
- 代码的可维护性和可测试性
- 依赖关系的清晰程度

本文聚焦于：
- 系统的模块划分方式
- 文件组织约定
- 模块融合机制（依赖注入 vs 全局单例）

> **注意**：模块间的事件通信在 03-ipc-communication.md 和 07-plugin-communication.md 中详述，扩展点机制（Contribution Points）在 05-extension-points.md 中详述。

---

## VSCode 的模块划分

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

规则：
1. 上层可以依赖下层，下层不能依赖上层
2. 同层之间：platform 和 editor 互不依赖
3. 层内依赖：同层模块可以相互依赖，但需谨慎
```

### 环境隔离

每个层内部进一步按运行环境划分：

```
platform/files/
├── common/              # 跨环境代码（接口定义、纯逻辑）
│   ├── files.ts         # IFileService 接口
│   └── fileService.ts   # 通用实现
│
├── browser/             # 浏览器环境
│   └── fileService.ts   # 使用 fetch/IndexedDB
│
├── node/                # Node.js 环境
│   └── fileService.ts   # 使用 fs 模块
│
└── electron-sandbox/    # Electron 沙箱渲染进程
    └── fileService.ts   # 通过 IPC 调用主进程
```

**环境隔离规则**：
- `common/` 可被所有环境导入
- `browser/` 只能被浏览器环境导入
- `node/` 只能被 Node.js 环境导入
- `electron-sandbox/` 只能被 Electron 渲染进程导入

### 文件组织约定

**目录命名**：
- 使用 kebab-case（如 `file-service`）
- 功能模块放在独立目录
- 测试文件放在 `test/` 子目录

**文件命名**：
- 接口定义：`{name}.ts`（如 `files.ts` 定义 `IFileService`）
- 实现：`{name}Service.ts` 或 `{name}Impl.ts`
- 类型：`{name}.ts` 或 `{name}Types.ts`

**导出约定**：
```typescript
// platform/files/common/files.ts
// 1. 先导出接口和类型
export interface IFileService { ... }
export interface IFileContent { ... }

// 2. 再导出服务标识符
export const IFileService = createDecorator<IFileService>('fileService');
```

---

## Obsidian 的模块划分

### 扁平结构

Obsidian 采用以 `App` 为中心的扁平结构：

```
┌─────────────────────────────────────────────────────────────┐
│                         App                                  │
│                    (全局单例，持有所有服务)                    │
├─────────────────────────────────────────────────────────────┤
│  vault         workspace        metadataCache               │
│  (文件管理)     (布局管理)        (元数据缓存)                 │
├─────────────────────────────────────────────────────────────┤
│  plugins       commands         keymap                      │
│  (插件管理)     (命令系统)        (快捷键)                    │
└─────────────────────────────────────────────────────────────┘
```

### 核心服务挂载方式

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
}
```

### 文件组织（推测）

由于 Obsidian 闭源，从 API 推断其内部结构：

```
obsidian/
├── app.ts              # App 类，持有所有服务
├── vault.ts            # 文件管理
├── workspace.ts        # 布局管理
├── metadata-cache.ts   # 元数据缓存
├── plugin.ts           # 插件基类
├── view.ts             # 视图基类
└── ...
```

**特点**：
- 无明显分层，服务间可相互引用
- App 作为服务定位器
- 服务通过构造函数接收 App 引用

---

## 模块融合方式

### VSCode: 依赖注入 (DI)

VSCode 使用装饰器实现依赖注入：

#### 1. 定义服务标识符

```typescript
// src/vs/platform/files/common/files.ts
export const IFileService = createDecorator<IFileService>('fileService');

export interface IFileService {
  readonly _serviceBrand: undefined;
  readFile(resource: URI): Promise<IFileContent>;
  writeFile(resource: URI, content: VSBuffer): Promise<void>;
}
```

#### 2. 实现服务

```typescript
// src/vs/platform/files/node/fileService.ts
export class FileService implements IFileService {
  readonly _serviceBrand: undefined;

  constructor(
    @ILogService private readonly logService: ILogService  // 依赖注入
  ) {}

  async readFile(resource: URI): Promise<IFileContent> {
    this.logService.trace('Reading file:', resource.toString());
    // 实现...
  }
}
```

#### 3. 注册服务

```typescript
// src/vs/workbench/electron-sandbox/desktop.main.ts
const services = new ServiceCollection();
services.set(IFileService, new SyncDescriptor(FileService));
services.set(ILogService, new SyncDescriptor(LogService));

const instantiationService = new InstantiationService(services);
```

#### 4. 使用服务（自动注入）

```typescript
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

#### createDecorator 实现原理

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

### Obsidian: 全局单例

Obsidian 使用全局单例模式：

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

**插件访问服务**：

```typescript
export default class MyPlugin extends Plugin {
  async onload() {
    // 通过 this.app 访问所有服务
    const files = this.app.vault.getMarkdownFiles();
    const activeView = this.app.workspace.getActiveViewOfType(MarkdownView);
    const cache = this.app.metadataCache.getFileCache(file);
  }
}
```

### 对比

| 方面 | VSCode DI | Obsidian 全局单例 |
|------|-----------|------------------|
| **可测试性** | 高（可 mock 依赖） | 低（依赖全局状态） |
| **学习成本** | 高（需理解 DI 概念） | 低（直接访问） |
| **依赖明确性** | 明确（构造函数声明） | 隐式（任意位置访问） |
| **灵活性** | 高（可替换实现） | 低（固定实现） |
| **代码量** | 多（接口 + 实现 + 注册） | 少（直接访问） |
| **循环依赖** | 通过接口抽象避免 | 双向引用（app ↔ service） |
| **重构难度** | 低（接口隔离） | 高（改动影响广） |

---

## 对 Coding Agent Desktop 应用的建议

### 推荐方案：分层 + 轻量级 DI

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

### 轻量级 DI 实现（无装饰器）

```typescript
// core/di.ts - 服务容器
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
```

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

// 服务标识符
export const IChatService = Symbol('ChatService');
export const IEditorService = Symbol('EditorService');
```

```typescript
// services/chat.ts - 服务实现
export class ChatService implements IChatService {
  constructor(private container: ServiceContainer) {}

  async sendMessage(content: string): Promise<ChatResponse> {
    // 实现...
  }

  async *streamMessage(content: string): AsyncIterable<string> {
    // 实现...
  }
}
```

```typescript
// app/index.ts - 初始化
const container = new ServiceContainer();
container.register(IChatService, () => new ChatService(container));
container.register(IEditorService, () => new EditorService(container));

// 对外暴露简洁的访问方式（类似 Obsidian）
export const app = {
  chat: container.get<IChatService>(IChatService),
  editor: container.get<IEditorService>(IEditorService),
};
```

```typescript
// 插件使用服务
export default class MyPlugin implements Plugin {
  constructor(private app: App) {}

  async onload() {
    // 类似 Obsidian 的简洁访问方式
    const response = await this.app.chat.sendMessage('Hello');

    // 但背后是 DI 管理的服务，可测试、可替换
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
   - **推荐**：三层结构（core / services / app）

2. **选择什么模块融合方式？**
   - DI 容器：可测试性好，但学习成本高
   - 全局单例：简单直接，但难以测试
   - **推荐**：轻量级 DI（无装饰器，使用 Symbol 标识）

3. **如何强制执行模块边界？**
   - ESLint 规则（如 `import/no-restricted-paths`）
   - TypeScript path mapping
   - 代码审查
   - **推荐**：ESLint + CI 检查

---

## 参考资料

- [VSCode 源码结构](https://github.com/microsoft/vscode/tree/main/src/vs)
- [VSCode 依赖注入实现](https://github.com/microsoft/vscode/tree/main/src/vs/platform/instantiation)
- [InversifyJS - TypeScript DI 框架](https://inversify.io/)
