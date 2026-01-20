# 09. 视图系统

> **核心问题**：自定义视图怎么做？

---

## 概述

视图系统决定了：
- 插件如何创建自定义 UI
- 视图与宿主的集成方式
- 视图的隔离程度
- 视图的通信机制

---

## VSCode 的视图系统

### 视图类型

| 类型 | 说明 | 适用场景 |
|------|------|----------|
| **TreeView** | 树形列表视图 | 文件浏览器、大纲 |
| **WebviewView** | 侧边栏 Webview | 自定义侧边栏 UI |
| **WebviewPanel** | 编辑器区 Webview | 自定义编辑器、预览 |
| **CustomEditor** | 自定义编辑器 | 二进制文件编辑 |
| **Notebook** | 笔记本视图 | Jupyter 风格 |

### TreeView

```typescript
// 1. 声明视图位置
// package.json
{
  "contributes": {
    "views": {
      "explorer": [{
        "id": "myTreeView",
        "name": "My Tree View"
      }]
    }
  }
}

// 2. 定义数据项
class MyTreeItem extends vscode.TreeItem {
  constructor(
    public readonly label: string,
    public readonly collapsibleState: vscode.TreeItemCollapsibleState,
    public readonly children?: MyTreeItem[]
  ) {
    super(label, collapsibleState);
    this.tooltip = `${this.label}`;
    this.description = 'description';
  }
}

// 3. 实现 TreeDataProvider
class MyTreeDataProvider implements vscode.TreeDataProvider<MyTreeItem> {
  private _onDidChangeTreeData = new vscode.EventEmitter<MyTreeItem | undefined>();
  readonly onDidChangeTreeData = this._onDidChangeTreeData.event;

  getTreeItem(element: MyTreeItem): vscode.TreeItem {
    return element;
  }

  getChildren(element?: MyTreeItem): Thenable<MyTreeItem[]> {
    if (element) {
      return Promise.resolve(element.children || []);
    }
    return Promise.resolve(this.getRootItems());
  }

  private getRootItems(): MyTreeItem[] {
    return [
      new MyTreeItem('Item 1', vscode.TreeItemCollapsibleState.Collapsed, [
        new MyTreeItem('Child 1', vscode.TreeItemCollapsibleState.None),
        new MyTreeItem('Child 2', vscode.TreeItemCollapsibleState.None),
      ]),
      new MyTreeItem('Item 2', vscode.TreeItemCollapsibleState.None),
    ];
  }

  refresh(): void {
    this._onDidChangeTreeData.fire(undefined);
  }
}

// 4. 注册
const provider = new MyTreeDataProvider();
vscode.window.registerTreeDataProvider('myTreeView', provider);

// 或使用 createTreeView 获得更多控制
const treeView = vscode.window.createTreeView('myTreeView', {
  treeDataProvider: provider,
  showCollapseAll: true,
  canSelectMany: true
});
```

### WebviewPanel

```typescript
// 创建 Webview Panel
const panel = vscode.window.createWebviewPanel(
  'myWebview',           // 视图类型
  'My Webview',          // 标题
  vscode.ViewColumn.One, // 显示位置
  {
    enableScripts: true,              // 允许脚本
    retainContextWhenHidden: true,    // 隐藏时保持状态
    localResourceRoots: [             // 允许加载的本地资源
      vscode.Uri.joinPath(context.extensionUri, 'media')
    ]
  }
);

// 设置 HTML 内容
panel.webview.html = getWebviewContent(panel.webview, context.extensionUri);

// 获取 Webview HTML
function getWebviewContent(webview: vscode.Webview, extensionUri: vscode.Uri): string {
  const scriptUri = webview.asWebviewUri(
    vscode.Uri.joinPath(extensionUri, 'media', 'main.js')
  );
  const styleUri = webview.asWebviewUri(
    vscode.Uri.joinPath(extensionUri, 'media', 'style.css')
  );

  return `
    <!DOCTYPE html>
    <html>
    <head>
      <meta charset="UTF-8">
      <meta http-equiv="Content-Security-Policy"
            content="default-src 'none'; style-src ${webview.cspSource}; script-src ${webview.cspSource};">
      <link href="${styleUri}" rel="stylesheet">
    </head>
    <body>
      <h1>Hello Webview</h1>
      <script src="${scriptUri}"></script>
    </body>
    </html>
  `;
}
```

### Webview 通信

```typescript
// Extension 侧
panel.webview.postMessage({ type: 'update', data: 'hello' });

panel.webview.onDidReceiveMessage(
  message => {
    switch (message.type) {
      case 'alert':
        vscode.window.showInformationMessage(message.text);
        break;
      case 'getData':
        panel.webview.postMessage({ type: 'data', data: getData() });
        break;
    }
  },
  undefined,
  context.subscriptions
);

// Webview 侧（media/main.js）
const vscode = acquireVsCodeApi();

// 发送消息到 Extension
vscode.postMessage({ type: 'alert', text: 'Hello!' });

// 接收 Extension 消息
window.addEventListener('message', event => {
  const message = event.data;
  switch (message.type) {
    case 'update':
      document.body.textContent = message.data;
      break;
    case 'data':
      handleData(message.data);
      break;
  }
});

// 保存状态
vscode.setState({ count: 10 });
const state = vscode.getState(); // { count: 10 }
```

### WebviewView（侧边栏 Webview）

```typescript
// package.json
{
  "contributes": {
    "views": {
      "explorer": [{
        "type": "webview",
        "id": "myWebviewView",
        "name": "My Webview View"
      }]
    }
  }
}

// 实现 WebviewViewProvider
class MyWebviewViewProvider implements vscode.WebviewViewProvider {
  resolveWebviewView(
    webviewView: vscode.WebviewView,
    context: vscode.WebviewViewResolveContext,
    token: vscode.CancellationToken
  ) {
    webviewView.webview.options = {
      enableScripts: true
    };
    webviewView.webview.html = this.getHtmlContent();
  }

  private getHtmlContent(): string {
    return `<html><body><h1>Sidebar Webview</h1></body></html>`;
  }
}

// 注册
vscode.window.registerWebviewViewProvider('myWebviewView', new MyWebviewViewProvider());
```

---

## Obsidian 的视图系统

### 视图类型

| 类型 | 说明 | 适用场景 |
|------|------|----------|
| **ItemView** | 基础视图 | 自定义面板 |
| **FileView** | 文件视图 | 文件相关面板 |
| **MarkdownView** | Markdown 视图 | 编辑器 |
| **Modal** | 模态框 | 弹窗交互 |

### ItemView

```typescript
// 定义视图
class MyView extends ItemView {
  static VIEW_TYPE = 'my-view';

  getViewType(): string {
    return MyView.VIEW_TYPE;
  }

  getDisplayText(): string {
    return 'My View';
  }

  getIcon(): string {
    return 'star';
  }

  async onOpen() {
    const container = this.containerEl.children[1];
    container.empty();

    // 直接操作 DOM
    container.createEl('h1', { text: 'My View' });

    const list = container.createEl('ul');
    for (const item of this.getItems()) {
      list.createEl('li', { text: item.name });
    }

    // 添加交互
    const button = container.createEl('button', { text: 'Refresh' });
    button.addEventListener('click', () => this.refresh());
  }

  async onClose() {
    // 清理资源
  }

  // 状态序列化
  getState(): any {
    return { scrollPosition: this.containerEl.scrollTop };
  }

  async setState(state: any, result: ViewStateResult): Promise<void> {
    if (state.scrollPosition) {
      this.containerEl.scrollTop = state.scrollPosition;
    }
  }
}

// 注册视图
this.registerView(MyView.VIEW_TYPE, (leaf) => new MyView(leaf));
```

### 视图交互

```typescript
class MyView extends ItemView {
  private items: Item[] = [];

  async onOpen() {
    this.render();

    // 监听数据变化
    this.registerEvent(
      this.app.vault.on('modify', () => this.refresh())
    );
  }

  private render() {
    const container = this.containerEl.children[1];
    container.empty();

    // 使用 Obsidian 组件
    for (const item of this.items) {
      new Setting(container)
        .setName(item.name)
        .setDesc(item.description)
        .addButton(btn => btn
          .setIcon('trash')
          .onClick(() => this.deleteItem(item)));
    }
  }

  private async refresh() {
    this.items = await this.loadItems();
    this.render();
  }
}
```

### 使用前端框架

```typescript
// 使用 React/Vue/Svelte
class MyReactView extends ItemView {
  private root: Root;

  async onOpen() {
    const container = this.containerEl.children[1];
    this.root = createRoot(container);
    this.root.render(<MyComponent app={this.app} />);
  }

  async onClose() {
    this.root.unmount();
  }
}

// React 组件
function MyComponent({ app }: { app: App }) {
  const [files, setFiles] = useState<TFile[]>([]);

  useEffect(() => {
    setFiles(app.vault.getMarkdownFiles());

    const handler = () => setFiles(app.vault.getMarkdownFiles());
    app.vault.on('create', handler);
    app.vault.on('delete', handler);

    return () => {
      app.vault.off('create', handler);
      app.vault.off('delete', handler);
    };
  }, [app]);

  return (
    <div>
      <h1>Files</h1>
      <ul>
        {files.map(f => <li key={f.path}>{f.name}</li>)}
      </ul>
    </div>
  );
}
```

### Modal

```typescript
class MyModal extends Modal {
  result: string;
  onSubmit: (result: string) => void;

  constructor(app: App, onSubmit: (result: string) => void) {
    super(app);
    this.onSubmit = onSubmit;
  }

  onOpen() {
    const { contentEl } = this;

    contentEl.createEl('h1', { text: 'Enter Value' });

    new Setting(contentEl)
      .setName('Value')
      .addText(text => text
        .onChange(value => this.result = value));

    new Setting(contentEl)
      .addButton(btn => btn
        .setButtonText('Submit')
        .setCta()
        .onClick(() => {
          this.close();
          this.onSubmit(this.result);
        }));
  }

  onClose() {
    const { contentEl } = this;
    contentEl.empty();
  }
}

// 使用
new MyModal(this.app, (result) => {
  console.log('User entered:', result);
}).open();
```

---

## 对比分析

### 视图隔离对比

| 方面 | VSCode (Webview) | Obsidian (ItemView) |
|------|------------------|---------------------|
| **DOM 隔离** | iframe 完全隔离 | 无隔离，直接操作 |
| **CSS 隔离** | 完全隔离 | 共享全局样式 |
| **JS 隔离** | 独立上下文 | 共享全局上下文 |
| **通信方式** | postMessage | 直接调用 |
| **性能** | 有开销 | 无开销 |
| **安全性** | 高 | 低 |

### 开发体验对比

| 方面 | VSCode | Obsidian |
|------|--------|----------|
| **框架支持** | 任意（隔离环境） | 任意（需自行集成） |
| **调试** | 独立 DevTools | 共享 DevTools |
| **热重载** | 需手动处理 | 较容易 |
| **样式管理** | 完全自主 | 需适配主题 |
| **API 访问** | 通过消息 | 直接访问 |

### 代码量对比

**VSCode TreeView**：
```typescript
// 需要：
// 1. package.json 声明
// 2. TreeItem 类
// 3. TreeDataProvider 类
// 4. 注册代码
// 约 50-100 行
```

**Obsidian ItemView**：
```typescript
// 需要：
// 1. ItemView 子类
// 2. 注册代码
// 约 30-50 行
```

---

## 对 Coding Agent Desktop 应用的建议

### 视图类型设计

```typescript
// 视图基类
abstract class BaseView {
  abstract readonly viewType: string;
  abstract readonly displayName: string;
  readonly icon?: string;

  protected container: HTMLElement;
  protected state: any;

  // 生命周期
  abstract onMount(container: HTMLElement): void;
  abstract onUnmount(): void;

  // 可选的生命周期
  onShow?(): void;
  onHide?(): void;
  onResize?(width: number, height: number): void;

  // 状态管理
  getState(): any { return this.state; }
  setState(state: any): void { this.state = state; }
}

// 视图类型
enum ViewCategory {
  /** 编辑器类视图（占据主区域） */
  Editor = 'editor',
  /** 面板类视图（侧边栏、底部） */
  Panel = 'panel',
  /** 弹窗类视图 */
  Modal = 'modal',
  /** 悬浮类视图 */
  Popup = 'popup'
}
```

### 视图注册

```typescript
interface ViewRegistration {
  /** 视图类型 ID */
  viewType: string;
  /** 显示名称 */
  displayName: string;
  /** 图标 */
  icon?: string;
  /** 视图分类 */
  category: ViewCategory;
  /** 视图工厂 */
  factory: () => BaseView;
  /** 默认位置 */
  defaultLocation?: ViewLocation;
  /** 是否可多实例 */
  allowMultiple?: boolean;
  /** 序列化器（用于状态恢复） */
  serializer?: ViewSerializer;
}

// 注册视图
context.registerView({
  viewType: 'chat',
  displayName: 'AI Chat',
  icon: 'message-circle',
  category: ViewCategory.Panel,
  factory: () => new ChatView(),
  defaultLocation: { area: 'assistant' }
});
```

### 聊天视图实现

```typescript
class ChatView extends BaseView {
  viewType = 'chat';
  displayName = 'AI Chat';
  icon = 'message-circle';

  private messages: Message[] = [];
  private root: Root;

  onMount(container: HTMLElement) {
    // 使用 React
    this.root = createRoot(container);
    this.root.render(
      <ChatProvider messages={this.messages} onSend={this.sendMessage}>
        <ChatUI />
      </ChatProvider>
    );
  }

  onUnmount() {
    this.root.unmount();
  }

  private async sendMessage(content: string) {
    // 调用 AI 服务
    const response = await this.context.services.ai.chat(content);
    this.messages.push({ role: 'assistant', content: response });
  }

  // 状态序列化
  getState() {
    return { messages: this.messages };
  }

  setState(state: any) {
    this.messages = state.messages || [];
  }
}
```

### 视图通信

```typescript
// 事件总线方式
class EditorView extends BaseView {
  onMount(container: HTMLElement) {
    // 监听聊天消息
    this.context.eventBus.on('chat:insert-code', (code: string) => {
      this.editor.insertText(code);
    });
  }
}

class ChatView extends BaseView {
  handleCodeBlockClick(code: string) {
    // 发送到编辑器
    this.context.eventBus.emit('chat:insert-code', code);
  }
}

// 或通过共享状态
class AppState {
  selectedCode: string | null = null;
}

class EditorView extends BaseView {
  onMount(container: HTMLElement) {
    // 订阅状态变化
    this.context.state.subscribe('selectedCode', (code) => {
      if (code) this.editor.insertText(code);
    });
  }
}
```

### Webview 封装（可选）

```typescript
// 对于需要完全隔离的场景
class IsolatedView extends BaseView {
  private iframe: HTMLIFrameElement;

  onMount(container: HTMLElement) {
    this.iframe = document.createElement('iframe');
    this.iframe.sandbox.add('allow-scripts');
    this.iframe.src = this.getContentUrl();
    container.appendChild(this.iframe);

    // 消息通信
    window.addEventListener('message', this.handleMessage);
  }

  onUnmount() {
    window.removeEventListener('message', this.handleMessage);
    this.iframe.remove();
  }

  postMessage(data: any) {
    this.iframe.contentWindow?.postMessage(data, '*');
  }

  private handleMessage = (event: MessageEvent) => {
    if (event.source !== this.iframe.contentWindow) return;
    this.onMessage(event.data);
  };

  protected onMessage(data: any) {
    // 处理来自 iframe 的消息
  }
}
```

---

## 关键决策清单

1. **是否需要视图隔离？**
   - 隔离（安全但复杂）
   - 直接 DOM（简单但需信任）

2. **使用什么前端框架？**
   - React / Vue / Svelte
   - 原生 DOM
   - 混合使用

3. **视图状态如何管理？**
   - 组件内状态
   - 全局状态
   - 序列化持久化

4. **视图间如何通信？**
   - 事件总线
   - 共享状态
   - 直接引用

5. **样式如何管理？**
   - 适配宿主主题
   - 完全自定义
   - CSS-in-JS

---

## 参考资料

- [VSCode Webview API](https://code.visualstudio.com/api/extension-guides/webview)
- [VSCode TreeView API](https://code.visualstudio.com/api/extension-guides/tree-view)
- [Obsidian Views](https://docs.obsidian.md/Plugins/User+interface/Views)
