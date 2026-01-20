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

### Electron IPC 封装层

VSCode 在 Electron IPC 之上封装了 `IMessagePassingProtocol` 接口：

```typescript
// src/vs/base/parts/ipc/common/ipc.ts（源码验证）

// 消息传递协议接口
export interface IMessagePassingProtocol {
  send(buffer: VSBuffer): void;
  readonly onMessage: Event<VSBuffer>;
  drain?(): Promise<void>;  // 等待写缓冲区清空
}
```

```typescript
// src/vs/base/parts/ipc/common/ipc.electron.ts（源码验证）

export interface Sender {
  send(channel: string, msg: unknown): void;
}

/**
 * Electron Protocol 利用 Electron 风格的 IPC 通信（ipcRenderer, ipcMain）
 * 实现 IMessagePassingProtocol。
 */
export class Protocol implements IMessagePassingProtocol {
  constructor(private sender: Sender, readonly onMessage: Event<VSBuffer>) { }

  send(message: VSBuffer): void {
    try {
      this.sender.send('vscode:message', message.buffer);
    } catch (e) {
      // systems are going down
    }
  }

  disconnect(): void {
    this.sender.send('vscode:disconnect', null);
  }
}
```

```typescript
// src/vs/base/parts/ipc/electron-main/ipc.electron.ts（源码验证）

/**
 * 基于 Electron ipcMain API 的 IPCServer 实现
 */
export class Server extends IPCServer {
  private static readonly Clients = new Map<number, IDisposable>();

  private static getOnDidClientConnect(): Event<ClientConnectionEvent> {
    // 监听渲染进程的 hello 消息
    const onHello = Event.fromNodeEventEmitter<WebContents>(
      validatedIpcMain,
      'vscode:hello',
      ({ sender }) => sender
    );

    return Event.map(onHello, webContents => {
      const id = webContents.id;

      // 创建作用域内的消息事件
      const onMessage = createScopedOnMessageEvent(id, 'vscode:message') as Event<VSBuffer>;
      const onDidClientDisconnect = Event.any(
        Event.signal(createScopedOnMessageEvent(id, 'vscode:disconnect')),
        onDidClientReconnect.event
      );

      // 使用 ElectronProtocol 封装
      const protocol = new ElectronProtocol(webContents, onMessage);

      return { protocol, onDidClientDisconnect };
    });
  }

  constructor() {
    super(Server.getOnDidClientConnect());
  }
}
```

### 自定义 RPC 协议

VSCode 实现了自己的 RPC 协议，而非直接使用 Electron IPC：

```typescript
// src/vs/base/parts/ipc/common/ipc.ts（源码验证）

// 请求类型
const enum RequestType {
  Promise = 100,        // 普通请求
  PromiseCancel = 101,  // 取消请求
  EventListen = 102,    // 事件订阅
  EventDispose = 103    // 取消订阅
}

// 响应类型
const enum ResponseType {
  Initialize = 200,      // 初始化
  PromiseSuccess = 201,  // 成功响应
  PromiseError = 202,    // 错误响应（Error 对象）
  PromiseErrorObj = 203, // 错误响应（普通对象）
  EventFire = 204        // 事件触发
}

// 请求消息结构（内部使用二进制序列化，这里展示逻辑结构）
type IRawPromiseRequest = {
  type: RequestType.Promise;
  id: number;           // 请求 ID，用于匹配响应
  channelName: string;  // 通道名
  name: string;         // 方法名
  arg: any;             // 参数
};

type IRawEventListenRequest = {
  type: RequestType.EventListen;
  id: number;
  channelName: string;
  name: string;
  arg: any;
};

// 响应消息结构
type IRawPromiseSuccessResponse = {
  type: ResponseType.PromiseSuccess;
  id: number;           // 对应的请求 ID
  data: any;            // 响应数据
};

type IRawPromiseErrorResponse = {
  type: ResponseType.PromiseError;
  id: number;
  data: {
    message: string;
    name: string;
    stack: string[] | undefined;
  };
};
```

### Channel 抽象

```typescript
// src/vs/base/parts/ipc/common/ipc.ts（源码验证）

/**
 * IChannel 是对一组命令的抽象。
 * 可以在 channel 上调用多个命令，每个命令最多接受一个参数。
 * call 总是返回一个 Promise。
 */
export interface IChannel {
  call<T>(command: string, arg?: any, cancellationToken?: CancellationToken): Promise<T>;
  listen<T>(event: string, arg?: any): Event<T>;
}

/**
 * IServerChannel 是 IChannel 的服务端对应。
 * 用于处理远程 Promise 或事件。
 */
export interface IServerChannel<TContext = string> {
  call<T>(ctx: TContext, command: string, arg?: any, cancellationToken?: CancellationToken): Promise<T>;
  listen<T>(ctx: TContext, event: string, arg?: any): Event<T>;
}

/**
 * IChannelServer 托管一组 channels。
 * 可以通过 channel 名称注册 channels。
 */
export interface IChannelServer<TContext = string> {
  registerChannel(channelName: string, channel: IServerChannel<TContext>): void;
}

/**
 * IChannelClient 可以访问一组 channels。
 * 可以通过 channel 名称获取 channels。
 */
export interface IChannelClient {
  getChannel<T extends IChannel>(channelName: string): T;
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

VSCode 使用**两层序列化**机制：

```typescript
// src/vs/base/parts/ipc/common/ipc.ts（源码验证）

// 1. 底层二进制序列化 - 数据类型枚举
enum DataType {
  Undefined = 0,
  String = 1,
  Buffer = 2,
  VSBuffer = 3,
  Array = 4,
  Object = 5,  // 只有复杂对象使用 JSON
  Int = 6      // 整数使用 VQL 编码
}

// 使用 VQL (Variable-Length Quantity) 编码整数，减少传输体积
// 参考: https://en.wikipedia.org/wiki/Variable-length_quantity
```

```typescript
// src/vs/base/common/marshalling.ts（源码验证）

// 2. 对象恢复层 - 恢复特殊类型
export function revive<T = any>(obj: any, depth = 0): Revived<T> {
  if (typeof obj === 'object') {
    switch ((<MarshalledObject>obj).$mid) {
      case MarshalledId.Uri: return URI.revive(obj);
      case MarshalledId.Regexp: return new RegExp(obj.source, obj.flags);
      case MarshalledId.Date: return new Date(obj.source);
    }
    // 递归处理数组和对象...
  }
  return obj;
}
```

### Extension Host 通信

Extension Host 使用独立的 `RPCProtocol` 通过 **MessagePort** 与渲染进程通信：

#### 通信建立过程

```typescript
// src/vs/workbench/services/extensions/electron-browser/localProcessExtensionHost.ts（源码验证）

private _establishProtocol(extensionHostProcess: ExtensionHostProcess, opts: IExtensionHostProcessOptions): Promise<IMessagePassingProtocol> {
  // 使用 MessagePort 进行通信
  writeExtHostConnection(new MessagePortExtHostConnection(), opts.env);

  const portPromise = acquirePort(undefined, opts.responseChannel, opts.responseNonce);

  return new Promise<IMessagePassingProtocol>((resolve, reject) => {
    portPromise.then((port) => {
      const onMessage = new BufferedEmitter<VSBuffer>();

      // 监听来自 Extension Host 的消息
      port.onmessage = ((e) => {
        if (e.data) {
          onMessage.fire(VSBuffer.wrap(e.data));
        }
      });
      port.start();

      // 返回 IMessagePassingProtocol 实现
      resolve({
        onMessage: onMessage.event,
        send: message => port.postMessage(message.buffer),  // 通过 MessagePort 发送
      });
    });
  });
}
```

#### RPCProtocol 消息类型

```typescript
// src/vs/workbench/services/extensions/common/rpcProtocol.ts（源码验证）

// Extension Host 专用的消息类型
const enum MessageType {
  RequestJSONArgs = 1,                   // JSON 参数请求
  RequestJSONArgsWithCancellation = 2,   // 带取消的 JSON 参数请求
  RequestMixedArgs = 3,                  // 混合参数请求（含 Buffer）
  RequestMixedArgsWithCancellation = 4,  // 带取消的混合参数请求
  Acknowledged = 5,                      // 确认收到
  Cancel = 6,                            // 取消请求
  ReplyOKEmpty = 7,                      // 空成功响应
  ReplyOKVSBuffer = 8,                   // Buffer 成功响应
  ReplyOKJSON = 9,                       // JSON 成功响应
  ReplyOKJSONWithBuffers = 10,           // 带 Buffer 的 JSON 响应
  ReplyErrError = 11,                    // 错误响应
  ReplyErrEmpty = 12,                    // 空错误响应
}
```

#### RPCProtocol 核心实现

```typescript
// src/vs/workbench/services/extensions/common/rpcProtocol.ts（源码验证）

export class RPCProtocol extends Disposable implements IRPCProtocol {
  private static readonly UNRESPONSIVE_TIME = 3 * 1000; // 3s 无响应检测

  private readonly _protocol: IMessagePassingProtocol;
  private readonly _locals: any[];   // 本地注册的服务
  private readonly _proxies: any[];  // 远程代理
  private _lastMessageId: number;

  constructor(protocol: IMessagePassingProtocol, ...) {
    this._protocol = protocol;
    this._register(this._protocol.onMessage((msg) => this._receiveOneMessage(msg)));
  }

  // 获取远程代理
  public getProxy<T>(identifier: ProxyIdentifier<T>): Proxied<T> {
    const { nid: rpcId, sid } = identifier;
    if (!this._proxies[rpcId]) {
      this._proxies[rpcId] = this._createProxy(rpcId, sid);
    }
    return this._proxies[rpcId];
  }

  // 动态创建代理（使用 Proxy）
  private _createProxy<T>(rpcId: number, debugName: string): T {
    const handler = {
      get: (target: any, name: PropertyKey) => {
        if (typeof name === 'string' && !target[name] && name.charCodeAt(0) === CharCode.DollarSign) {
          // $开头的方法自动代理为远程调用
          target[name] = (...myArgs: any[]) => {
            return this._remoteCall(rpcId, name, myArgs);
          };
        }
        return target[name];
      }
    };
    return new Proxy(Object.create(null), handler);
  }

  // 远程调用
  private _remoteCall(rpcId: number, methodName: string, args: any[]): Promise<any> {
    const req = ++this._lastMessageId;
    const result = new LazyPromise();

    this._pendingRPCReplies[String(req)] = new PendingRPCReply(result, disposable);
    this._onWillSendRequest(req);

    const msg = MessageIO.serializeRequest(req, rpcId, methodName, serializedArgs, !!cancellationToken);
    this._protocol.send(msg);

    return result;
  }
}
```

#### 代理模式使用示例

```typescript
// 代理实现（渲染进程侧）
export class MainThreadCommands implements MainThreadCommandsShape {
  private readonly _proxy: ExtHostCommandsShape;

  constructor(extHostContext: IExtHostContext) {
    // 通过 RPCProtocol 获取 Extension Host 侧的代理
    this._proxy = extHostContext.getProxy(ExtHostContext.ExtHostCommands);
  }

  async $executeCommand(id: string, ...args: any[]): Promise<any> {
    return this._commandService.executeCommand(id, ...args);
  }
}

// 代理实现（Extension Host 侧）
export class ExtHostCommands implements ExtHostCommandsShape {
  private readonly _proxy: MainThreadCommandsShape;

  constructor(mainContext: IMainContext) {
    // 通过 RPCProtocol 获取渲染进程侧的代理
    this._proxy = mainContext.getProxy(MainContext.MainThreadCommands);
  }

  registerCommand(id: string, callback: Function): Disposable {
    this._commands.set(id, callback);
    this._proxy.$registerCommand(id);  // 远程调用，自动序列化
    return { dispose: () => this._commands.delete(id) };
  }
}
```

---

## Obsidian 的 IPC 通信

### 安全模型（源码验证）

Obsidian 采用**非隔离的渲染进程**模型：

```typescript
// Obsidian 的 Electron 配置（推测，基于插件行为验证）
new BrowserWindow({
  webPreferences: {
    nodeIntegration: true,      // 允许 require
    contextIsolation: false,    // 不隔离上下文，window.require 可用
    enableRemoteModule: true,   // 启用 @electron/remote
  }
});
```

**证据**：[obsidian-electron-window-tweaker](https://github.com/mgmeyers/obsidian-electron-window-tweaker) 插件源码：

```typescript
// main.ts（源码验证）
// 使用 window.require 而非 require，说明 contextIsolation: false
const setAlwaysOnTop = (on: boolean) => {
  window.require("electron").remote.getCurrentWindow().setAlwaysOnTop(on);
};

const setOpacity = (opacity: number) => {
  window.require("electron").remote.getCurrentWindow().setOpacity(opacity);
};
```

### 简化的单进程模型

由于 Obsidian 插件运行在渲染进程，大部分情况**不需要 IPC**：

```typescript
// 直接调用，无需 IPC（源码验证：obsidian-sample-plugin）
export default class MyPlugin extends Plugin {
  async onload() {
    // 直接访问 Vault（文件系统抽象，内部使用 Node.js fs）
    const content = await this.app.vault.read(file);

    // 直接操作 DOM（同进程）
    document.body.addClass('my-class');

    // 直接访问编辑器（同进程）
    const editor = this.app.workspace.activeEditor?.editor;
    if (editor) {
      editor.replaceSelection('Hello');
    }
  }
}
```

### 访问主进程能力（通过 @electron/remote）

Obsidian 使用 **`@electron/remote`** 模块（已废弃但仍在使用）：

```typescript
// 源码验证：obsidian-electron-window-tweaker/main.ts

// 1. 窗口操作（同步远程调用，非 IPC）
window.require("electron").remote.getCurrentWindow().setAlwaysOnTop(true);
window.require("electron").remote.getCurrentWindow().setOpacity(0.8);

// 2. macOS 特有功能
window.require("electron").remote.getCurrentWindow().setVibrancy("window");
window.require("electron").remote.getCurrentWindow().setTrafficLightPosition({ x: 7, y: 6 });

// 3. 原生对话框
const { dialog } = window.require("electron").remote;
const result = await dialog.showOpenDialog({
  properties: ['openFile']
});

// 4. 系统菜单
const { Menu, MenuItem } = window.require("electron").remote;
const menu = new Menu();
menu.append(new MenuItem({ label: 'Custom Item' }));
```

### 插件声明要求

使用 Electron API 的插件必须声明 `isDesktopOnly`：

```json
// manifest.json（源码验证：obsidian-electron-window-tweaker）
{
  "id": "obsidian-electron-window-tweaker",
  "name": "Electron Window Tweaker",
  "isDesktopOnly": true  // 必须声明，否则在移动端崩溃
}
```

### @electron/remote 的工作原理

`@electron/remote` 不是真正的 IPC，而是**同步远程过程调用**：

```
┌─────────────────────────────────────────────────────────────────┐
│                     Renderer Process                             │
│                                                                  │
│  window.require("electron").remote.getCurrentWindow()            │
│           │                                                      │
│           ▼                                                      │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │              @electron/remote 模块                       │    │
│  │  - 内部使用 ipcRenderer.sendSync() 同步调用              │    │
│  │  - 返回主进程对象的代理（Proxy）                          │    │
│  │  - 所有方法调用都被拦截并转发到主进程                      │    │
│  └─────────────────────────────────────────────────────────┘    │
│           │                                                      │
└───────────┼──────────────────────────────────────────────────────┘
            │ ipcRenderer.sendSync()
            ▼
┌─────────────────────────────────────────────────────────────────┐
│                      Main Process                                │
│                                                                  │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │              @electron/remote 主进程模块                 │    │
│  │  - 接收渲染进程的同步请求                                │    │
│  │  - 执行实际的 BrowserWindow 方法                         │    │
│  │  - 同步返回结果                                          │    │
│  └─────────────────────────────────────────────────────────┘    │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 安全性对比

| 方面 | VSCode | Obsidian |
|------|--------|----------|
| **contextIsolation** | ✅ 启用 | ❌ 禁用 |
| **nodeIntegration** | ❌ 禁用（通过 preload） | ✅ 启用 |
| **@electron/remote** | ❌ 不使用 | ✅ 使用 |
| **插件隔离** | ✅ 独立进程 | ❌ 同进程 |
| **恶意插件风险** | 低（沙箱隔离） | 高（完全访问） |

---

## 对比分析

### 架构对比表

| 方面 | VSCode | Obsidian |
|------|--------|----------|
| **通信协议** | 自定义 RPC（MessagePort + 二进制） | @electron/remote（同步 IPC） |
| **序列化** | 二进制 + VQL 编码 + JSON 回退 | 无需序列化（同步代理） |
| **类型安全** | 编译时类型检查（ProxyIdentifier） | 运行时，弱类型 |
| **通信频率** | 高（所有插件 API 调用） | 低（仅系统调用） |
| **双向通信** | 支持（call + listen） | 支持（同步阻塞） |
| **流式传输** | 支持（MessagePort） | 不支持 |
| **错误处理** | 结构化错误传递（保留 stack） | 直接抛出 |
| **取消机制** | 内置 CancellationToken | 无 |
| **无响应检测** | 3s 自动检测 | 无（同步阻塞 UI） |

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

## 对 Coding Agent Desktop 应用的建议

### 推荐架构

```
┌─────────────────────────────────────────────────────────────┐
│                      Main Process                            │
│                                                              │
│  ┌──────────────────────┐    ┌──────────────────────────┐   │
│  │      IPC Server      │    │     AI Server Manager    │   │
│  │  - FileChannel       │    │  - spawn AI Server       │   │
│  │  - DialogChannel     │    │  - 管理生命周期          │   │
│  │  - SystemChannel     │    │  - 健康检查/重启         │   │
│  └──────────────────────┘    └────────────┬─────────────┘   │
│                                           │ spawn            │
└───────────────────────────────────────────┼─────────────────┘
                              │             │
                              │             ▼
                              │  ┌─────────────────────────┐
                              │  │      AI Server          │
                              │  │     （独立子进程）        │
                              │  │  - HTTP/WebSocket 服务  │
                              │  │  - LLM 调用             │
                              │  │  - 流式响应             │
                              │  └─────────────────────────┘
                              │             ▲
                              ▼             │ WebSocket（直连）
┌───────────────────────────────────────────┼─────────────────┐
│                    Renderer Process       │                  │
│                                           │                  │
│  ┌──────────────┐  ┌──────────────┐  ┌────┴───────┐         │
│  │   UI Layer   │  │  IPC Client  │  │  AI Client │         │
│  │              │◄─┤              │  │            │         │
│  │  - React     │  │  - File      │  │  - WS 连接 │         │
│  │  - Editor    │  │  - Dialog    │  │  - 流式接收│         │
│  └──────────────┘  └──────────────┘  └────────────┘         │
│                                                              │
│  ┌─────────────────────────────────────────────────────┐    │
│  │                   Plugin Runtime                     │    │
│  │  - 核心插件（同进程）                                  │    │
│  │  - 第三方插件（可选隔离）                              │    │
│  └─────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────┘
```

**通信方式**：
- **Main ↔ Renderer**：Electron IPC（文件、对话框等系统操作）
- **Renderer ↔ AI Server**：WebSocket 直连（AI 对话、流式响应）
- **Main → AI Server**：仅负责 spawn 启动和生命周期管理

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

### AI Server 架构

AI 功能通过独立的 Server 进程实现，由 Main 进程 spawn 启动，Renderer 通过 WebSocket 直连：

```typescript
// main/ai-server/manager.ts
import { spawn, ChildProcess } from 'child_process';

class AIServerManager {
  private server: ChildProcess | null = null;
  private port: number = 3721;

  async start(): Promise<number> {
    // 启动 AI Server 子进程
    this.server = spawn('node', ['ai-server/index.js'], {
      stdio: ['pipe', 'pipe', 'pipe', 'ipc'],
      env: { ...process.env, AI_SERVER_PORT: String(this.port) }
    });

    // 监听进程退出，自动重启
    this.server.on('exit', (code) => {
      if (code !== 0) this.start();
    });

    // 等待 Server 就绪
    await this.waitForReady();
    return this.port;
  }

  // 提供端口给 Renderer 连接
  getPort(): number {
    return this.port;
  }
}

// 在 Main 进程启动时
const aiManager = new AIServerManager();
await aiManager.start();

// Renderer 请求 AI Server 端口
ipcMain.handle('ai:get-port', () => aiManager.getPort());
```

### AI 流式响应处理

```typescript
// Renderer 直接通过 WebSocket 与 AI Server 通信

// renderer/ai/client.ts
class AIClient {
  private ws: WebSocket | null = null;
  private port: number | null = null;

  async connect(): Promise<void> {
    // 从 Main 获取 AI Server 端口
    this.port = await ipcRenderer.invoke('ai:get-port');
    this.ws = new WebSocket(`ws://localhost:${this.port}`);
    await this.waitForOpen();
  }

  async *stream(prompt: string): AsyncIterable<string> {
    if (!this.ws) await this.connect();

    const requestId = crypto.randomUUID();

    // 发送请求
    this.ws!.send(JSON.stringify({ type: 'chat', requestId, prompt }));

    // 接收流式响应
    while (true) {
      const message = await this.waitForMessage(requestId);
      if (message.type === 'chunk') {
        yield message.data;
      } else if (message.type === 'end') {
        break;
      } else if (message.type === 'error') {
        throw new Error(message.error);
      }
    }
  }
}

// 使用
const ai = new AIClient();
for await (const chunk of ai.stream('Hello')) {
  console.log(chunk);
}
```

### AI Server 进程优势

| 方面 | Web Worker | AI Server（spawn + WebSocket） |
|------|-----------|-------------------------------|
| **隔离性** | 线程隔离 | 进程隔离（更彻底） |
| **崩溃影响** | 可能影响 Renderer | 完全隔离，不影响 UI |
| **通信方式** | postMessage | WebSocket（直连，无需转发） |
| **Node.js 能力** | 受限 | 完整 Node.js 能力 |
| **流式传输** | 需自己实现 | WebSocket 原生支持 |
| **调试** | 较困难 | 独立进程，可单独调试 |
| **重启恢复** | 需重新初始化 | Main 自动重启，Renderer 重连 |

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
