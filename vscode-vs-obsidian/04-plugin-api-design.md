# 04. 插件 API 设计

> **核心问题**：暴露什么 API 给插件？

---

## 概述

插件 API 设计决定了：
- 插件能做什么、不能做什么
- 开发者的学习曲线
- API 的稳定性和向后兼容
- 宿主应用的安全性和可维护性

---

## VSCode 的插件 API 设计

### API 风格：命名空间

VSCode 使用 `vscode` 命名空间暴露所有 API：

```typescript
import * as vscode from 'vscode';

// 窗口相关
vscode.window.showInformationMessage('Hello');
vscode.window.createWebviewPanel(...);
vscode.window.createTreeView(...);

// 工作区相关
vscode.workspace.openTextDocument(uri);
vscode.workspace.fs.readFile(uri);
vscode.workspace.getConfiguration('myExtension');

// 命令相关
vscode.commands.registerCommand('myExtension.myCommand', () => {});
vscode.commands.executeCommand('vscode.open', uri);

// 语言相关
vscode.languages.registerCompletionItemProvider('javascript', provider);
vscode.languages.registerHoverProvider('javascript', provider);

// 编辑器相关
vscode.TextEdit.insert(position, 'text');
vscode.Range(start, end);
```

### API 分类

```typescript
// vscode.d.ts 结构

declare module 'vscode' {
  // 1. 命名空间 - 功能入口
  export namespace window { ... }
  export namespace workspace { ... }
  export namespace commands { ... }
  export namespace languages { ... }
  export namespace debug { ... }
  export namespace extensions { ... }
  export namespace env { ... }
  export namespace tasks { ... }
  export namespace notebooks { ... }
  export namespace scm { ... }
  export namespace tests { ... }
  export namespace comments { ... }
  export namespace authentication { ... }

  // 2. 类 - 数据结构
  export class Uri { ... }
  export class Position { ... }
  export class Range { ... }
  export class Selection { ... }
  export class TextEdit { ... }
  export class WorkspaceEdit { ... }
  export class Diagnostic { ... }
  export class CompletionItem { ... }
  export class TreeItem { ... }

  // 3. 接口 - 契约定义
  export interface TextDocument { ... }
  export interface TextEditor { ... }
  export interface WebviewPanel { ... }
  export interface TreeDataProvider<T> { ... }
  export interface CompletionItemProvider { ... }

  // 4. 枚举 - 常量值
  export enum DiagnosticSeverity { ... }
  export enum CompletionItemKind { ... }
  export enum TextEditorRevealType { ... }

  // 5. 类型别名
  export type ProviderResult<T> = T | undefined | null | Thenable<T | undefined | null>;
  export type Event<T> = (listener: (e: T) => any) => Disposable;
}
```

### 白名单机制

插件只能访问 `vscode` 命名空间暴露的 API，无法访问内部实现：

```typescript
// 插件代码
import * as vscode from 'vscode';

// ✅ 可以访问
vscode.window.showInformationMessage('Hello');

// ❌ 不能访问内部实现
// import { FileService } from 'vs/platform/files/common/fileService';
// 编译会失败，因为这些模块不在 node_modules 中
```

### Proposed API（实验性 API）

```json
// package.json
{
  "enabledApiProposals": ["inlineCompletions"]
}
```

```typescript
// 使用实验性 API
vscode.languages.registerInlineCompletionItemProvider('*', {
  provideInlineCompletionItems(document, position) {
    // Copilot 风格的内联补全
    return [{ insertText: 'suggested code' }];
  }
});
```

### 废弃策略

```typescript
// vscode.d.ts 中标记废弃
/**
 * @deprecated Use `vscode.workspace.fs.readFile` instead.
 */
export function readFile(uri: Uri): Thenable<Uint8Array>;
```

VSCode 会在多个版本中保留废弃 API，给开发者迁移时间。

---

## Obsidian 的插件 API 设计

### API 风格：类实例

Obsidian 通过 `Plugin` 基类和 `App` 对象暴露 API：

```typescript
import { Plugin, App, TFile, MarkdownView } from 'obsidian';

export default class MyPlugin extends Plugin {
  // this.app 是核心入口
  async onload() {
    // Vault（文件系统）
    this.app.vault.read(file);
    this.app.vault.modify(file, content);
    this.app.vault.getMarkdownFiles();

    // Workspace（布局）
    this.app.workspace.getActiveViewOfType(MarkdownView);
    this.app.workspace.getLeaf();
    this.app.workspace.onLayoutReady(() => {});

    // MetadataCache（元数据）
    this.app.metadataCache.getFileCache(file);
    this.app.metadataCache.getFirstLinkpathDest(linkpath, sourcePath);

    // Plugin 基类方法
    this.addCommand({ id: 'my-command', name: 'My Command', callback: () => {} });
    this.addSettingTab(new MySettingTab(this.app, this));
    this.registerView('my-view', (leaf) => new MyView(leaf));
    this.registerEvent(this.app.vault.on('create', () => {}));
  }
}
```

### API 分类

```typescript
// obsidian.d.ts 结构

declare module 'obsidian' {
  // 1. 核心类
  export class App { ... }
  export class Plugin { ... }
  export class Component { ... }

  // 2. 文件系统
  export class Vault { ... }
  export abstract class TAbstractFile { ... }
  export class TFile extends TAbstractFile { ... }
  export class TFolder extends TAbstractFile { ... }

  // 3. 工作区
  export class Workspace { ... }
  export class WorkspaceLeaf { ... }
  export abstract class View { ... }
  export class ItemView extends View { ... }
  export class MarkdownView extends ItemView { ... }

  // 4. 编辑器
  export class Editor { ... }
  export class MarkdownRenderer { ... }
  export abstract class EditorSuggest<T> { ... }

  // 5. UI 组件
  export class Modal { ... }
  export class Notice { ... }
  export class Menu { ... }
  export class Setting { ... }
  export class PluginSettingTab { ... }

  // 6. 元数据
  export class MetadataCache { ... }
  export interface CachedMetadata { ... }
  export interface FrontMatterCache { ... }
}
```

### 近乎无限制的能力

```typescript
export default class MyPlugin extends Plugin {
  onload() {
    // 1. 直接访问 DOM
    document.body.addClass('my-class');

    // 2. 访问未公开的内部 API
    // @ts-ignore
    const internalPlugins = this.app.internalPlugins;

    // 3. 修改全局对象
    // @ts-ignore
    window.myPlugin = this;

    // 4. 访问 Electron API
    const { ipcRenderer } = require('electron');

    // 5. 访问 Node.js API
    const fs = require('fs');
    const path = require('path');

    // 6. 访问其他插件
    const otherPlugin = this.app.plugins.plugins['other-plugin'];
    if (otherPlugin) {
      // 调用其他插件的方法
      otherPlugin.someMethod();
    }
  }
}
```

### 类型定义质量

```typescript
// obsidian.d.ts 提供完整类型
export class Vault {
  read(file: TFile): Promise<string>;
  cachedRead(file: TFile): Promise<string>;
  modify(file: TFile, data: string, options?: DataWriteOptions): Promise<void>;
  create(path: string, data: string, options?: DataWriteOptions): Promise<TFile>;
  delete(file: TAbstractFile, force?: boolean): Promise<void>;
  rename(file: TAbstractFile, newPath: string): Promise<void>;
  // ...
}
```

---

## 对比分析

### API 设计对比表

| 方面 | VSCode | Obsidian |
|------|--------|----------|
| **风格** | 命名空间 (`vscode.*`) | 类实例 (`this.app.*`) |
| **能力边界** | 严格白名单 | 近乎无限制 |
| **类型支持** | 完整 (`@types/vscode`) | 完整 (`obsidian.d.ts`) |
| **API 数量** | 非常多（~30 个命名空间） | 相对较少 |
| **抽象级别** | 高（屏蔽实现细节） | 中（部分暴露内部） |
| **稳定性承诺** | 严格（语义化版本） | 一般 |
| **实验性 API** | 有 (Proposed API) | 无正式机制 |
| **废弃策略** | 标记废弃，长期保留 | 较少废弃 |

### API 设计哲学

| 哲学 | VSCode | Obsidian |
|------|--------|----------|
| **信任模型** | 不信任插件 | 信任插件 |
| **复杂度** | 由宿主承担 | 由插件承担 |
| **灵活性** | 受控灵活 | 完全灵活 |
| **学习曲线** | 陡峭 | 平缓 |

### 代码示例对比

#### 注册命令

**VSCode**：
```typescript
import * as vscode from 'vscode';

export function activate(context: vscode.ExtensionContext) {
  const disposable = vscode.commands.registerCommand('myExtension.helloWorld', () => {
    vscode.window.showInformationMessage('Hello World!');
  });
  context.subscriptions.push(disposable);
}
```

**Obsidian**：
```typescript
import { Plugin } from 'obsidian';

export default class MyPlugin extends Plugin {
  onload() {
    this.addCommand({
      id: 'hello-world',
      name: 'Hello World',
      callback: () => {
        new Notice('Hello World!');
      }
    });
  }
}
```

#### 读取文件

**VSCode**：
```typescript
const uri = vscode.Uri.file('/path/to/file.txt');
const content = await vscode.workspace.fs.readFile(uri);
const text = new TextDecoder().decode(content);
```

**Obsidian**：
```typescript
const file = this.app.vault.getAbstractFileByPath('file.md');
if (file instanceof TFile) {
  const content = await this.app.vault.read(file);
}
```

#### 创建视图

**VSCode**：
```typescript
// 1. 在 package.json 声明
{
  "contributes": {
    "views": {
      "explorer": [{
        "id": "myView",
        "name": "My View"
      }]
    }
  }
}

// 2. 实现 TreeDataProvider
class MyTreeDataProvider implements vscode.TreeDataProvider<MyItem> {
  getTreeItem(element: MyItem): vscode.TreeItem {
    return element;
  }
  getChildren(element?: MyItem): MyItem[] {
    return [];
  }
}

// 3. 注册
vscode.window.registerTreeDataProvider('myView', new MyTreeDataProvider());
```

**Obsidian**：
```typescript
// 1. 实现 View
class MyView extends ItemView {
  getViewType() { return 'my-view'; }
  getDisplayText() { return 'My View'; }

  async onOpen() {
    this.contentEl.createEl('h1', { text: 'My View' });
  }
}

// 2. 注册
this.registerView('my-view', (leaf) => new MyView(leaf));

// 3. 激活
this.app.workspace.getRightLeaf(false)?.setViewState({
  type: 'my-view',
  active: true,
});
```

---

## API 设计模式

### 1. Provider 模式（VSCode）

```typescript
// 定义 Provider 接口
interface CompletionItemProvider {
  provideCompletionItems(
    document: TextDocument,
    position: Position,
    token: CancellationToken
  ): ProviderResult<CompletionItem[]>;
}

// 实现 Provider
class MyCompletionProvider implements CompletionItemProvider {
  provideCompletionItems(document, position, token) {
    return [
      new CompletionItem('hello'),
      new CompletionItem('world'),
    ];
  }
}

// 注册 Provider
vscode.languages.registerCompletionItemProvider('javascript', new MyCompletionProvider());
```

**优点**：
- 标准化接口，易于理解
- 支持取消（CancellationToken）
- 支持异步和同步返回

### 2. 事件注册模式（Obsidian）

```typescript
// 注册事件监听
this.registerEvent(
  this.app.vault.on('create', (file) => {
    console.log('File created:', file.path);
  })
);

// registerEvent 会自动在插件卸载时清理
```

**优点**：
- 简单直接
- 自动生命周期管理

### 3. Disposable 模式（VSCode）

```typescript
// 所有注册都返回 Disposable
const disposable = vscode.commands.registerCommand('...', () => {});

// 手动清理
disposable.dispose();

// 或添加到 subscriptions 自动清理
context.subscriptions.push(disposable);
```

### 4. Builder 模式（两者都有）

```typescript
// VSCode: WorkspaceEdit
const edit = new vscode.WorkspaceEdit();
edit.insert(uri, position, 'text');
edit.delete(uri, range);
edit.replace(uri, range, 'new text');
await vscode.workspace.applyEdit(edit);

// Obsidian: Setting
new Setting(containerEl)
  .setName('My Setting')
  .setDesc('Description')
  .addText(text => text
    .setPlaceholder('Enter value')
    .setValue(this.settings.myValue)
    .onChange(async (value) => {
      this.settings.myValue = value;
      await this.saveSettings();
    }));
```

---

## 对 Coding Agent Desktop 应用的建议

### 推荐：分层 API 设计

```
┌─────────────────────────────────────────────────────────────┐
│                     Public API (Level 1)                     │
│  - 文档化、稳定、语义化版本                                    │
│  - 所有插件可用                                               │
├─────────────────────────────────────────────────────────────┤
│                    Extended API (Level 2)                    │
│  - 文档化、相对稳定                                           │
│  - 需要声明权限                                               │
├─────────────────────────────────────────────────────────────┤
│                   Internal API (Level 3)                     │
│  - 不文档化、随时可变                                         │
│  - 仅官方插件使用                                             │
└─────────────────────────────────────────────────────────────┘
```

### API 设计示例

```typescript
// @neovate/plugin-api

// ==================== Public API ====================

export namespace chat {
  /** 发送消息并获取响应 */
  export function sendMessage(content: string): Promise<ChatResponse>;

  /** 流式发送消息 */
  export function streamMessage(content: string): AsyncIterable<string>;

  /** 获取当前会话历史 */
  export function getHistory(): ChatMessage[];

  /** 监听新消息 */
  export function onMessage(callback: (message: ChatMessage) => void): Disposable;
}

export namespace editor {
  /** 打开文件 */
  export function openFile(path: string): Promise<void>;

  /** 获取当前编辑器内容 */
  export function getContent(): string;

  /** 插入文本 */
  export function insert(position: Position, text: string): void;

  /** 注册补全提供者 */
  export function registerCompletionProvider(
    language: string,
    provider: CompletionProvider
  ): Disposable;
}

export namespace workspace {
  /** 获取工作区根路径 */
  export function getRootPath(): string | undefined;

  /** 读取文件 */
  export function readFile(path: string): Promise<string>;

  /** 写入文件 */
  export function writeFile(path: string, content: string): Promise<void>;

  /** 监听文件变化 */
  export function onDidChangeFile(callback: (event: FileChangeEvent) => void): Disposable;
}

export namespace ui {
  /** 显示通知 */
  export function showNotification(message: string, type?: 'info' | 'warning' | 'error'): void;

  /** 显示模态框 */
  export function showModal(options: ModalOptions): Promise<ModalResult>;

  /** 注册侧边栏视图 */
  export function registerSidebarView(id: string, factory: () => View): Disposable;
}

// ==================== Extended API ====================

export namespace extended {
  /** 需要 'ai.advanced' 权限 */
  export namespace ai {
    /** 直接访问 LLM */
    export function callLLM(prompt: string, options: LLMOptions): Promise<string>;

    /** 获取 embedding */
    export function getEmbedding(text: string): Promise<number[]>;
  }

  /** 需要 'system.access' 权限 */
  export namespace system {
    /** 执行系统命令 */
    export function exec(command: string): Promise<ExecResult>;

    /** 访问剪贴板 */
    export function clipboard: Clipboard;
  }
}
```

### 权限声明

```json
// manifest.json
{
  "name": "my-plugin",
  "permissions": [
    "chat.read",
    "chat.write",
    "editor.read",
    "workspace.readwrite",
    "extended.ai"  // 需要用户批准
  ]
}
```

### 类型定义最佳实践

```typescript
// types.d.ts

/** 聊天消息 */
export interface ChatMessage {
  /** 消息 ID */
  readonly id: string;
  /** 角色 */
  readonly role: 'user' | 'assistant' | 'system';
  /** 内容 */
  readonly content: string;
  /** 时间戳 */
  readonly timestamp: number;
}

/** 聊天响应 */
export interface ChatResponse {
  /** 响应消息 */
  readonly message: ChatMessage;
  /** 使用的 token 数 */
  readonly usage: {
    readonly promptTokens: number;
    readonly completionTokens: number;
  };
}

/** 补全提供者 */
export interface CompletionProvider {
  /**
   * 提供补全项
   * @param document 当前文档
   * @param position 光标位置
   * @param token 取消令牌
   */
  provideCompletionItems(
    document: TextDocument,
    position: Position,
    token: CancellationToken
  ): ProviderResult<CompletionItem[]>;
}

/** 资源清理接口 */
export interface Disposable {
  /** 清理资源 */
  dispose(): void;
}

/** Provider 返回类型 */
export type ProviderResult<T> = T | undefined | null | Thenable<T | undefined | null>;
```

### API 稳定性策略

```typescript
// 使用 JSDoc 标记 API 状态

/**
 * 发送消息到 AI
 * @since 1.0.0
 * @stable
 */
export function sendMessage(content: string): Promise<ChatResponse>;

/**
 * 获取 embedding 向量
 * @since 1.2.0
 * @experimental 此 API 可能在未来版本中变更
 */
export function getEmbedding(text: string): Promise<number[]>;

/**
 * @deprecated 使用 `chat.sendMessage` 替代
 * @since 1.0.0
 * @removed 2.0.0
 */
export function send(message: string): Promise<string>;
```

---

## 关键决策清单

1. **API 风格选择？**
   - 命名空间：更好的组织，但学习成本高
   - 类实例：简单直接，但不易扩展

2. **能力边界如何划定？**
   - 白名单：安全，但限制插件能力
   - 黑名单：灵活，但难以控制

3. **是否需要权限系统？**
   - 对 AI 应用，可能需要控制 AI 调用、网络访问等

4. **如何处理实验性 API？**
   - Proposed API 机制
   - 版本号标记（alpha/beta）

5. **废弃策略是什么？**
   - 多少版本后移除？
   - 如何通知开发者？

---

## 参考资料

- [VSCode Extension API](https://code.visualstudio.com/api/references/vscode-api)
- [Obsidian API](https://docs.obsidian.md/Reference/TypeScript+API)
- [API 设计最佳实践](https://cloud.google.com/apis/design)
- [语义化版本](https://semver.org/lang/zh-CN/)
