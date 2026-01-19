# 01. 进程架构

> **核心问题**：插件跑在哪个进程？

---

## 概述

进程架构是插件化应用最基础的决策之一，它决定了：
- 插件崩溃是否影响主应用
- 插件能访问哪些系统能力
- 插件与宿主的通信成本
- 应用的整体性能和资源占用

---

## VSCode 的进程架构

### 多进程模型

```
┌─────────────────────────────────────────────────────────────┐
│                      Main Process                            │
│  - 窗口管理、菜单、生命周期                                    │
│  - 文件系统访问、原生对话框                                    │
└─────────────────────────────────────────────────────────────┘
           │                              │
           ▼                              ▼
┌─────────────────────┐      ┌─────────────────────────────────┐
│   Renderer Process  │      │   Extension Host Process        │
│   (Workbench UI)    │      │   (插件运行环境)                 │
│                     │      │                                 │
│  - Monaco Editor    │◄────►│  - 所有插件代码                  │
│  - 侧边栏、面板     │ IPC  │  - Node.js 完整能力              │
│  - 状态栏          │      │  - 隔离的 V8 实例                │
└─────────────────────┘      └─────────────────────────────────┘
                                          │
                                          ▼
                             ┌─────────────────────────────────┐
                             │   Language Server Processes     │
                             │   (LSP 语言服务)                 │
                             │                                 │
                             │  - TypeScript Server            │
                             │  - Python Language Server       │
                             │  - 每个语言独立进程              │
                             └─────────────────────────────────┘
```

### 关键设计点

#### 1. Extension Host 独立进程

```typescript
// VSCode 启动 Extension Host 的简化逻辑
// src/vs/workbench/services/extensions/electron-sandbox/extensionHostProcessManager.ts

class ExtensionHostProcessManager {
  private _extensionHostProcess: ChildProcess;

  start() {
    // 使用 fork 创建独立 Node.js 进程
    this._extensionHostProcess = fork(
      'extensionHostProcess.js',
      {
        execArgv: ['--nolazy'],
        env: { VSCODE_HANDLES_UNCAUGHT_ERRORS: 'true' }
      }
    );

    // 建立 IPC 通道
    this._extensionHostProcess.on('message', this._onMessage.bind(this));
  }
}
```

#### 2. 崩溃隔离

```typescript
// Extension Host 崩溃时的处理
this._extensionHostProcess.on('exit', (code, signal) => {
  if (code !== 0) {
    // 插件崩溃不影响主界面
    // 显示错误提示，允许用户重启 Extension Host
    this._showExtensionHostCrashedDialog();
  }
});
```

#### 3. 多窗口共享

```
Window 1 ─────┐
              │
Window 2 ─────┼────► Shared Extension Host
              │
Window 3 ─────┘
```

多个窗口共享同一个 Extension Host，减少内存占用。

---

## Obsidian 的进程架构

### 单进程模型

```
┌─────────────────────────────────────────────────────────────┐
│                      Main Process                            │
│  - 窗口管理、菜单、文件系统                                   │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                    Renderer Process                          │
│                                                              │
│  ┌─────────────────┐  ┌─────────────────┐                   │
│  │   Obsidian Core │  │   Plugin A      │                   │
│  │                 │  │                 │                   │
│  │  - App 对象     │  │  - onload()     │                   │
│  │  - Workspace    │  │  - 直接访问 DOM │                   │
│  │  - Vault        │  │  - 共享内存空间 │                   │
│  └─────────────────┘  └─────────────────┘                   │
│                                                              │
│  ┌─────────────────┐  ┌─────────────────┐                   │
│  │   Plugin B      │  │   Plugin C      │                   │
│  │                 │  │                 │                   │
│  │  - 同进程运行   │  │  - 无隔离边界   │                   │
│  │  - 可访问一切   │  │  - 崩溃影响全局 │                   │
│  └─────────────────┘  └─────────────────┘                   │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### 关键设计点

#### 1. 插件直接在渲染进程运行

```typescript
// Obsidian 插件的 main.ts
export default class MyPlugin extends Plugin {
  async onload() {
    // 直接访问 DOM
    document.body.addClass('my-plugin-loaded');

    // 直接访问 app 对象（全局单例）
    this.app.workspace.onLayoutReady(() => {
      console.log('Layout ready');
    });

    // 可以访问任何 Obsidian 内部对象
    // @ts-ignore - 访问未暴露的内部 API
    const internalState = this.app.internalPlugins;
  }
}
```

#### 2. 无沙箱隔离

```typescript
// 插件可以做任何事
export default class DangerousPlugin extends Plugin {
  async onload() {
    // 修改原型链
    Array.prototype.customMethod = function() { /* ... */ };

    // 拦截全局事件
    const originalFetch = window.fetch;
    window.fetch = async (...args) => {
      console.log('Intercepted:', args);
      return originalFetch(...args);
    };

    // 访问 Node.js API（通过 Electron）
    const { ipcRenderer } = require('electron');
  }
}
```

#### 3. 每个窗口独立

```
Window 1 ──► Renderer Process 1 ──► Plugin Instance A1, B1, C1
Window 2 ──► Renderer Process 2 ──► Plugin Instance A2, B2, C2
```

每个窗口有独立的插件实例。

---

## 对比分析

### 架构对比表

| 方面 | VSCode | Obsidian |
|------|--------|----------|
| **插件运行位置** | 独立进程 (Extension Host) | 渲染进程 (同进程) |
| **隔离级别** | 进程隔离 | 无隔离 |
| **崩溃影响** | 插件崩溃不影响 UI | 插件崩溃可能导致整个应用崩溃 |
| **内存占用** | 较高（多进程开销） | 较低（单进程） |
| **通信成本** | IPC 序列化开销 | 直接函数调用，零开销 |
| **DOM 访问** | 需通过 Webview | 直接访问 |
| **Node.js 能力** | 完整支持 | 完整支持 |
| **多窗口** | 共享 Extension Host | 独立实例 |
| **Web 兼容** | 支持 (Web Worker) | 不支持 |

### 优劣分析

#### VSCode 进程隔离的优势

1. **稳定性**：插件崩溃不影响主界面
2. **安全性**：插件无法直接操作 UI DOM
3. **可调试**：可以单独调试 Extension Host
4. **Web 兼容**：可以在浏览器中运行（vscode.dev）

#### VSCode 进程隔离的劣势

1. **性能开销**：IPC 通信需要序列化/反序列化
2. **复杂度**：架构复杂，API 设计受限
3. **实时性**：无法同步操作 DOM
4. **内存占用**：多进程导致更高内存占用

#### Obsidian 同进程的优势

1. **性能**：直接函数调用，无通信开销
2. **灵活性**：插件可以做任何事
3. **简单**：API 设计简单直接
4. **低延迟**：可同步操作 DOM

#### Obsidian 同进程的劣势

1. **稳定性**：一个插件崩溃可能影响全局
2. **安全性**：插件可以访问任何内部状态
3. **可维护性**：插件可能依赖未公开 API
4. **Web 兼容**：难以移植到浏览器

---

## 代码示例对比

### 读取文件内容

**VSCode**：
```typescript
// 需要通过 API，内部走 IPC
import * as vscode from 'vscode';

const uri = vscode.Uri.file('/path/to/file.txt');
const content = await vscode.workspace.fs.readFile(uri);
const text = new TextDecoder().decode(content);
```

**Obsidian**：
```typescript
// 直接调用，无 IPC 开销
const content = await this.app.vault.read(
  this.app.vault.getAbstractFileByPath('file.md')
);
```

### 修改 UI

**VSCode**：
```typescript
// 必须通过 Webview，隔离的 iframe
const panel = vscode.window.createWebviewPanel(
  'myView',
  'My Panel',
  vscode.ViewColumn.One,
  { enableScripts: true }
);
panel.webview.html = '<html>...</html>';

// 通过 postMessage 通信
panel.webview.postMessage({ type: 'update', data: '...' });
```

**Obsidian**：
```typescript
// 直接操作 DOM
class MyView extends ItemView {
  async onOpen() {
    this.contentEl.createEl('h1', { text: 'Hello' });
    this.contentEl.createEl('button', {
      text: 'Click me',
      onclick: () => this.handleClick()
    });
  }
}
```

### 监听编辑器事件

**VSCode**：
```typescript
// 通过 API 订阅，事件通过 IPC 传递
vscode.workspace.onDidChangeTextDocument(event => {
  // event 是序列化后的对象
  console.log(event.document.uri.fsPath);
});
```

**Obsidian**：
```typescript
// 直接订阅 CodeMirror 事件
this.registerEditorExtension(
  EditorView.updateListener.of(update => {
    // 直接访问 EditorView 实例
    if (update.docChanged) {
      console.log(update.state.doc.toString());
    }
  })
);
```

---

## 中间方案

除了 VSCode 的完全隔离和 Obsidian 的完全开放，还有一些中间方案：

### 1. VM 沙箱（Node.js vm 模块）

```typescript
import * as vm from 'vm';

const sandbox = {
  console: console,
  api: exposedApi,  // 只暴露安全的 API
};

const context = vm.createContext(sandbox);
const script = new vm.Script(pluginCode);
script.runInContext(context);
```

**优点**：同进程，低开销，但有一定隔离
**缺点**：vm 模块有安全漏洞，可以逃逸

### 2. iframe 沙箱

```typescript
const iframe = document.createElement('iframe');
iframe.sandbox = 'allow-scripts';
iframe.srcdoc = `
  <script>
    window.addEventListener('message', (event) => {
      // 处理消息
    });
  </script>
`;
```

**优点**：浏览器级别隔离，较安全
**缺点**：通信开销大，功能受限

### 3. Web Worker

```typescript
const worker = new Worker('plugin-worker.js');
worker.postMessage({ type: 'init', config: pluginConfig });
worker.onmessage = (event) => {
  // 处理插件消息
};
```

**优点**：独立线程，不阻塞 UI
**缺点**：无法访问 DOM，通信有开销

### 4. Realm（TC39 提案）

```typescript
const realm = new Realm();
const result = realm.evaluate(`
  // 插件代码在隔离的 Realm 中运行
  globalThis.myApi.doSomething();
`);
```

**优点**：同进程，轻量隔离
**缺点**：提案阶段，浏览器支持有限

---

## 对 AI Chat + Editor 应用的建议

### 推荐方案：分层隔离

```
┌─────────────────────────────────────────────────────────────┐
│                      Main Process                            │
│  - 窗口管理、系统集成                                         │
└─────────────────────────────────────────────────────────────┘
                              │
          ┌───────────────────┼───────────────────┐
          ▼                   ▼                   ▼
┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐
│ Renderer Process│  │  AI Worker      │  │ Plugin Host     │
│ (UI)            │  │  (AI 推理)      │  │ (插件运行)      │
│                 │  │                 │  │                 │
│ - React/Vue UI  │  │ - LLM 调用     │  │ - 核心插件同进程│
│ - 编辑器       │  │ - 流式响应      │  │ - 第三方隔离   │
└─────────────────┘  └─────────────────┘  └─────────────────┘
```

### 具体建议

| 决策点 | 建议 | 理由 |
|--------|------|------|
| **核心插件** | 同进程运行 | 性能优先，由官方维护，可信任 |
| **第三方插件** | 可选隔离 | 提供两种模式，让用户选择 |
| **AI 能力** | 独立 Worker | 避免阻塞 UI，方便流式处理 |
| **UI 插件** | 同进程 + 约束 | 需要操作 DOM，通过代码审查约束 |

### 隔离级别选项

```typescript
// manifest.json
{
  "name": "my-plugin",
  "isolation": "none" | "worker" | "iframe" | "process"
}
```

- `none`：同进程，完全信任（官方插件、知名插件）
- `worker`：Web Worker 隔离（计算密集型插件）
- `iframe`：iframe 沙箱（UI 插件）
- `process`：独立进程（高风险插件）

---

## 关键决策清单

在设计进程架构时，需要回答以下问题：

1. **插件的信任级别如何划分？**
   - 官方插件 vs 第三方插件
   - 审核过的 vs 未审核的

2. **性能和安全的平衡点在哪里？**
   - 对 AI Chat 应用，响应延迟是否关键？
   - 用户数据的敏感程度如何？

3. **是否需要 Web 兼容？**
   - 如果需要 vscode.dev 类似的 Web 版本，必须考虑进程隔离

4. **多窗口如何处理？**
   - 插件状态是否需要跨窗口共享？

5. **崩溃恢复策略是什么？**
   - 插件崩溃后如何恢复？
   - 是否需要自动重启机制？

---

## 参考资料

- [VSCode Extension Host 架构](https://code.visualstudio.com/api/advanced-topics/extension-host)
- [Electron 进程模型](https://www.electronjs.org/docs/latest/tutorial/process-model)
- [Web Worker API](https://developer.mozilla.org/en-US/docs/Web/API/Web_Workers_API)
- [TC39 Realm 提案](https://github.com/tc39/proposal-shadowrealm)
