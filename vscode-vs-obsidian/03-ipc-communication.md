# 03. IPC 通信

> **核心问题**：进程间如何通信？

---

## 概述

IPC（Inter-Process Communication）通信是多进程架构的核心问题：
- 如何在主进程和渲染进程间传递数据
- 如何在渲染进程和 Extension Host 间通信
- 如何保证通信的类型安全和性能

---

## VSCode 的 IPC 通信

### 通信架构

```
┌─────────────────┐     ┌─────────────────┐     ┌─────────────────┐
│  Main Process   │     │ Renderer Process│     │ Extension Host  │
│                 │     │                 │     │                 │
│  ┌───────────┐  │     │  ┌───────────┐  │     │  ┌───────────┐  │
│  │ IPC Server│◄─┼─────┼─►│ IPC Client│◄─┼─────┼─►│ IPC Client│  │
│  └───────────┘  │     │  └───────────┘  │     │  └───────────┘  │
│                 │     │                 │     │                 │
│  - 文件系统     │     │  - UI 渲染      │     │  - 插件运行     │
│  - 原生对话框   │     │  - 编辑器       │     │  - LSP 代理     │
└─────────────────┘     └─────────────────┘     └─────────────────┘
```

### 自定义 RPC 协议

VSCode 实现了自己的 RPC 协议，而非直接使用 Electron IPC：

```typescript
// src/vs/base/parts/ipc/common/ipc.ts

// 消息类型
const enum MessageType {
  RequestType = 100,
  ResponseType = 101,
  EventType = 102,
  CancelType = 103,
}

// 消息结构
interface RequestMessage {
  type: MessageType.RequestType;
  id: number;           // 请求 ID，用于匹配响应
  channelName: string;  // 通道名
  name: string;         // 方法名
  arg: any;             // 参数
}

interface ResponseMessage {
  type: MessageType.ResponseType;
  id: number;           // 对应的请求 ID
  data: any;            // 响应数据
  error?: any;          // 错误信息
}
```

### Channel 抽象

```typescript
// 定义 Channel 接口
// src/vs/base/parts/ipc/common/ipc.ts
export interface IChannel {
  call<T>(command: string, arg?: any): Promise<T>;
  listen<T>(event: string): Event<T>;
}

// 服务端 Channel 实现
export interface IServerChannel<TContext = string> {
  call<T>(ctx: TContext, command: string, arg?: any): Promise<T>;
  listen<T>(ctx: TContext, event: string): Event<T>;
}
```

### 实际使用示例

```typescript
// 1. 定义服务接口
// src/vs/platform/files/common/files.ts
export interface IFileService {
  readFile(resource: URI): Promise<IFileContent>;
  writeFile(resource: URI, content: VSBuffer): Promise<void>;
}

// 2. 主进程实现 Channel
// src/vs/platform/files/node/fileIpc.ts
export class FileChannel implements IServerChannel {
  constructor(private fileService: IFileService) {}

  async call(ctx: string, command: string, arg: any): Promise<any> {
    switch (command) {
      case 'readFile':
        return this.fileService.readFile(URI.revive(arg));
      case 'writeFile':
        return this.fileService.writeFile(URI.revive(arg[0]), VSBuffer.wrap(arg[1]));
    }
  }

  listen(ctx: string, event: string): Event<any> {
    switch (event) {
      case 'onDidFilesChange':
        return this.fileService.onDidFilesChange;
    }
  }
}

// 3. 渲染进程使用 Channel
// src/vs/platform/files/browser/fileService.ts
export class FileServiceClient implements IFileService {
  constructor(private channel: IChannel) {}

  async readFile(resource: URI): Promise<IFileContent> {
    return this.channel.call('readFile', resource);
  }

  async writeFile(resource: URI, content: VSBuffer): Promise<void> {
    return this.channel.call('writeFile', [resource, content.buffer]);
  }

  get onDidFilesChange(): Event<IFilesChangeEvent> {
    return this.channel.listen('onDidFilesChange');
  }
}
```

### 序列化处理

```typescript
// src/vs/base/common/marshalling.ts

// 自定义序列化，处理特殊类型
export function stringify(obj: any): string {
  return JSON.stringify(obj, (key, value) => {
    if (value instanceof URI) {
      return { $mid: MarshalledId.Uri, ...value.toJSON() };
    }
    if (value instanceof RegExp) {
      return { $mid: MarshalledId.Regexp, source: value.source, flags: value.flags };
    }
    if (ArrayBuffer.isView(value)) {
      return { $mid: MarshalledId.Buffer, data: Array.from(value as Uint8Array) };
    }
    return value;
  });
}

export function parse(text: string): any {
  return JSON.parse(text, (key, value) => {
    if (value && typeof value === 'object') {
      switch (value.$mid) {
        case MarshalledId.Uri:
          return URI.revive(value);
        case MarshalledId.Regexp:
          return new RegExp(value.source, value.flags);
        case MarshalledId.Buffer:
          return new Uint8Array(value.data);
      }
    }
    return value;
  });
}
```

### Extension Host 通信

```typescript
// src/vs/workbench/services/extensions/common/extensionHostProtocol.ts

// 定义协议
export interface IExtensionHostProtocol {
  // 主线程 -> Extension Host
  $executeContributedCommand(id: string, ...args: any[]): Promise<any>;
  $activateExtension(extensionId: string): Promise<void>;

  // Extension Host -> 主线程
  $registerCommand(id: string): void;
  $showInformationMessage(message: string): Promise<void>;
}

// 代理实现（渲染进程侧）
export class MainThreadCommands implements MainThreadCommandsShape {
  private readonly _proxy: ExtHostCommandsShape;

  constructor(extHostContext: IExtHostContext) {
    this._proxy = extHostContext.getProxy(ExtHostContext.ExtHostCommands);
  }

  async $executeCommand(id: string, ...args: any[]): Promise<any> {
    // 执行命令
    return this._commandService.executeCommand(id, ...args);
  }
}

// 代理实现（Extension Host 侧）
export class ExtHostCommands implements ExtHostCommandsShape {
  private readonly _proxy: MainThreadCommandsShape;

  constructor(mainContext: IMainContext) {
    this._proxy = mainContext.getProxy(MainContext.MainThreadCommands);
  }

  registerCommand(id: string, callback: Function): Disposable {
    this._commands.set(id, callback);
    this._proxy.$registerCommand(id);
    return { dispose: () => this._commands.delete(id) };
  }
}
```

---

## Obsidian 的 IPC 通信

### 简化的单进程模型

由于 Obsidian 插件运行在渲染进程，大部分情况不需要 IPC：

```typescript
// 直接调用，无需 IPC
export default class MyPlugin extends Plugin {
  async onload() {
    // 直接访问 Vault（文件系统抽象）
    const content = await this.app.vault.read(file);

    // 直接操作 DOM
    document.body.addClass('my-class');

    // 直接访问编辑器
    const editor = this.app.workspace.activeEditor?.editor;
    if (editor) {
      editor.replaceSelection('Hello');
    }
  }
}
```

### 需要 IPC 的场景

Obsidian 仍需要与主进程通信的场景：

```typescript
// 1. 原生对话框
const { remote } = require('electron');
const result = await remote.dialog.showOpenDialog({
  properties: ['openFile']
});

// 2. 系统菜单
const { Menu, MenuItem } = remote;
const menu = new Menu();
menu.append(new MenuItem({ label: 'Custom Item' }));

// 3. 原生通知
new Notification('Title', { body: 'Body' });
```

### Electron IPC 直接使用

```typescript
// Obsidian 插件中使用 Electron IPC
const { ipcRenderer } = require('electron');

// 发送消息到主进程
ipcRenderer.send('my-channel', { data: 'hello' });

// 接收主进程响应
ipcRenderer.on('my-channel-reply', (event, response) => {
  console.log('Response:', response);
});

// 同步调用（阻塞）
const result = ipcRenderer.sendSync('sync-channel', 'data');
```

---

## 对比分析

### 架构对比表

| 方面 | VSCode | Obsidian |
|------|--------|----------|
| **通信协议** | 自定义 RPC | Electron IPC |
| **序列化** | 自定义（支持 URI、Buffer 等） | JSON |
| **类型安全** | 编译时类型检查 | 运行时，弱类型 |
| **通信频率** | 高（所有插件 API 调用） | 低（仅系统调用） |
| **双向通信** | 支持（call + listen） | 支持 |
| **流式传输** | 支持 | 不直接支持 |
| **错误处理** | 结构化错误传递 | 基本错误传递 |

### 性能对比

| 场景 | VSCode | Obsidian |
|------|--------|----------|
| **读取文件** | IPC 序列化开销 | 直接调用 |
| **修改 DOM** | 不可能 / 需 Webview | 直接操作 |
| **事件监听** | IPC 事件代理 | 直接订阅 |
| **大数据传输** | 需优化（分块、流式） | 直接传递引用 |

### 代码量对比

**VSCode 调用文件服务**：
```typescript
// 1. 定义接口 (files.ts)
// 2. 实现 Channel (fileIpc.ts)
// 3. 实现 Client (fileService.ts)
// 4. 注册服务 (main.ts)
// 5. 调用
const content = await fileService.readFile(uri);
```

**Obsidian 调用文件服务**：
```typescript
// 直接调用
const content = await this.app.vault.read(file);
```

---

## VSCode IPC 优化技巧

### 1. 批量请求

```typescript
// 不好：多次 IPC
for (const uri of uris) {
  await fileService.readFile(uri);
}

// 好：批量 IPC
const contents = await fileService.readFiles(uris);
```

### 2. 事件节流

```typescript
// src/vs/base/common/async.ts
export function throttle<T>(fn: (arg: T) => void, delay: number): (arg: T) => void {
  let pending = false;
  let lastArg: T;

  return (arg: T) => {
    lastArg = arg;
    if (!pending) {
      pending = true;
      setTimeout(() => {
        pending = false;
        fn(lastArg);
      }, delay);
    }
  };
}

// 使用
const throttledUpdate = throttle((changes) => {
  this._proxy.$onDidChangeTextDocument(changes);
}, 100);
```

### 3. 延迟初始化

```typescript
// 只在需要时才创建 Channel
class LazyChannel implements IChannel {
  private _channel: IChannel | null = null;

  private get channel(): IChannel {
    if (!this._channel) {
      this._channel = this.createChannel();
    }
    return this._channel;
  }

  call<T>(command: string, arg?: any): Promise<T> {
    return this.channel.call(command, arg);
  }
}
```

### 4. 使用 Transfer 传递大数据

```typescript
// 使用 Transferable 避免拷贝
const buffer = new ArrayBuffer(1024 * 1024);
worker.postMessage({ buffer }, [buffer]); // buffer 被转移，不是拷贝
```

---

## 对 AI Chat + Editor 应用的建议

### 推荐架构

```
┌─────────────────────────────────────────────────────────────┐
│                      Main Process                            │
│                                                              │
│  ┌─────────────────────────────────────────────────────┐    │
│  │                    IPC Server                        │    │
│  │  - FileChannel                                       │    │
│  │  - DialogChannel                                     │    │
│  │  - SystemChannel                                     │    │
│  └─────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                    Renderer Process                          │
│                                                              │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐       │
│  │   UI Layer   │  │  IPC Client  │  │  AI Worker   │       │
│  │              │◄─┤              │  │              │       │
│  │  - React     │  │  - Services  │  │  - Streaming │       │
│  │  - Editor    │  │  - Proxy     │  │  - LLM calls │       │
│  └──────────────┘  └──────────────┘  └──────────────┘       │
│                                                              │
│  ┌─────────────────────────────────────────────────────┐    │
│  │                   Plugin Runtime                     │    │
│  │  - 核心插件（同进程）                                  │    │
│  │  - 第三方插件（可选隔离）                              │    │
│  └─────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────┘
```

### IPC 层设计

```typescript
// shared/ipc/protocol.ts
export interface IIPCProtocol {
  // 文件操作
  'file:read': (path: string) => Promise<string>;
  'file:write': (path: string, content: string) => Promise<void>;
  'file:watch': (path: string) => void;

  // 对话框
  'dialog:open': (options: OpenDialogOptions) => Promise<string[]>;
  'dialog:save': (options: SaveDialogOptions) => Promise<string>;

  // AI 相关
  'ai:complete': (prompt: string) => Promise<string>;
  'ai:stream': (prompt: string) => AsyncIterable<string>;
}

// main/ipc/server.ts
import { ipcMain } from 'electron';

export function setupIPCServer() {
  // 类型安全的 IPC 处理
  ipcMain.handle('file:read', async (event, path: string) => {
    return fs.readFile(path, 'utf-8');
  });

  ipcMain.handle('file:write', async (event, path: string, content: string) => {
    return fs.writeFile(path, content, 'utf-8');
  });
}

// renderer/ipc/client.ts
import { ipcRenderer } from 'electron';

export const ipc = {
  file: {
    read: (path: string) => ipcRenderer.invoke('file:read', path),
    write: (path: string, content: string) => ipcRenderer.invoke('file:write', path, content),
  },
  dialog: {
    open: (options: OpenDialogOptions) => ipcRenderer.invoke('dialog:open', options),
  },
};
```

### AI 流式响应处理

```typescript
// 使用 MessagePort 进行流式通信
// main/ai/stream.ts
ipcMain.on('ai:stream:start', (event, { prompt, portId }) => {
  const port = event.ports[0];

  const stream = llm.stream(prompt);
  for await (const chunk of stream) {
    port.postMessage({ type: 'chunk', data: chunk });
  }
  port.postMessage({ type: 'end' });
  port.close();
});

// renderer/ai/client.ts
async function* streamAI(prompt: string): AsyncIterable<string> {
  const channel = new MessageChannel();

  ipcRenderer.postMessage('ai:stream:start', { prompt }, [channel.port2]);

  const port = channel.port1;
  port.start();

  while (true) {
    const message = await new Promise<any>((resolve) => {
      port.onmessage = (e) => resolve(e.data);
    });

    if (message.type === 'end') break;
    yield message.data;
  }

  port.close();
}

// 使用
for await (const chunk of streamAI('Hello')) {
  console.log(chunk);
}
```

### 类型安全的 IPC 封装

```typescript
// shared/ipc/typed-ipc.ts
type IPCHandler<T extends Record<string, (...args: any[]) => any>> = {
  [K in keyof T]: (...args: Parameters<T[K]>) => ReturnType<T[K]>;
};

// 定义协议
interface MainIPCProtocol {
  'file:read': (path: string) => Promise<string>;
  'file:write': (path: string, content: string) => Promise<void>;
}

// 主进程注册
function registerMainHandlers(handlers: IPCHandler<MainIPCProtocol>) {
  for (const [channel, handler] of Object.entries(handlers)) {
    ipcMain.handle(channel, (event, ...args) => handler(...args));
  }
}

// 渲染进程调用
function createIPCClient<T extends Record<string, (...args: any[]) => any>>(): IPCHandler<T> {
  return new Proxy({} as IPCHandler<T>, {
    get(target, prop: string) {
      return (...args: any[]) => ipcRenderer.invoke(prop, ...args);
    },
  });
}

const ipc = createIPCClient<MainIPCProtocol>();
const content = await ipc['file:read']('/path/to/file'); // 类型安全
```

---

## 关键决策清单

1. **是否需要自定义 RPC 协议？**
   - 如果通信频繁，建议自定义（像 VSCode）
   - 如果通信少，直接用 Electron IPC

2. **如何处理大数据传输？**
   - 使用 Transferable 避免拷贝
   - 分块传输
   - 流式传输

3. **如何保证类型安全？**
   - 定义共享的协议接口
   - 使用 TypeScript 泛型

4. **如何处理流式响应（AI 场景）？**
   - MessagePort
   - Server-Sent Events 风格
   - WebSocket

5. **如何优化性能？**
   - 批量请求
   - 事件节流
   - 延迟初始化

---

## 参考资料

- [VSCode IPC 实现](https://github.com/microsoft/vscode/tree/main/src/vs/base/parts/ipc)
- [Electron IPC 文档](https://www.electronjs.org/docs/latest/tutorial/ipc)
- [MessageChannel API](https://developer.mozilla.org/en-US/docs/Web/API/MessageChannel)
- [Transferable Objects](https://developer.mozilla.org/en-US/docs/Web/API/Web_Workers_API/Transferable_objects)
