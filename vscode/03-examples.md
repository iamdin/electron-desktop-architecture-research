# VSCode 扩展示例

> 代码示例与实践用法

## 1. 基础扩展结构

```typescript
import * as vscode from 'vscode';

export function activate(context: vscode.ExtensionContext) {
  console.log('Extension activated');

  // 注册命令、provider 等
  // 所有 Disposable 加入 context.subscriptions
}

export function deactivate() {
  // 可选的异步清理
}
```

---

## 2. 命令注册

### 普通命令

```typescript
const disposable = vscode.commands.registerCommand(
  'myExtension.sayHello',
  () => {
    vscode.window.showInformationMessage('Hello World!');
  }
);
context.subscriptions.push(disposable);
```

### 带参数的命令

```typescript
vscode.commands.registerCommand(
  'myExtension.openFile',
  (uri: vscode.Uri) => {
    vscode.window.showTextDocument(uri);
  }
);

// 调用命令
vscode.commands.executeCommand('myExtension.openFile', fileUri);
```

### 文本编辑器命令

```typescript
vscode.commands.registerTextEditorCommand(
  'myExtension.insertDate',
  (textEditor, edit) => {
    const position = textEditor.selection.active;
    edit.insert(position, new Date().toISOString());
  }
);
```

---

## 3. TreeView 树形视图

### 数据提供者

```typescript
interface MyTreeItem {
  id: string;
  label: string;
  children?: MyTreeItem[];
}

class MyTreeDataProvider implements vscode.TreeDataProvider<MyTreeItem> {
  private _onDidChangeTreeData = new vscode.EventEmitter<MyTreeItem | undefined>();
  readonly onDidChangeTreeData = this._onDidChangeTreeData.event;

  private data: MyTreeItem[] = [
    {
      id: '1',
      label: 'Parent 1',
      children: [
        { id: '1-1', label: 'Child 1-1' },
        { id: '1-2', label: 'Child 1-2' },
      ],
    },
    { id: '2', label: 'Parent 2' },
  ];

  getTreeItem(element: MyTreeItem): vscode.TreeItem {
    const treeItem = new vscode.TreeItem(
      element.label,
      element.children
        ? vscode.TreeItemCollapsibleState.Collapsed
        : vscode.TreeItemCollapsibleState.None
    );

    treeItem.id = element.id;
    treeItem.tooltip = `ID: ${element.id}`;
    treeItem.contextValue = element.children ? 'parent' : 'child';

    // 点击时执行命令
    treeItem.command = {
      command: 'myExtension.selectItem',
      title: 'Select',
      arguments: [element],
    };

    return treeItem;
  }

  getChildren(element?: MyTreeItem): MyTreeItem[] {
    if (!element) {
      return this.data;
    }
    return element.children || [];
  }

  getParent(element: MyTreeItem): MyTreeItem | undefined {
    // 用于 reveal 功能
    for (const item of this.data) {
      if (item.children?.includes(element)) {
        return item;
      }
    }
    return undefined;
  }

  refresh() {
    this._onDidChangeTreeData.fire(undefined);
  }
}
```

### 注册视图

```typescript
const treeDataProvider = new MyTreeDataProvider();

// 方式 1: 简单注册
context.subscriptions.push(
  vscode.window.registerTreeDataProvider('myTreeView', treeDataProvider)
);

// 方式 2: 创建 TreeView（获取更多控制）
const treeView = vscode.window.createTreeView('myTreeView', {
  treeDataProvider,
  showCollapseAll: true,
  canSelectMany: true,
});
context.subscriptions.push(treeView);

// 监听选择变化
treeView.onDidChangeSelection((e) => {
  console.log('Selected:', e.selection);
});

// 显示特定项
treeView.reveal(someItem, { focus: true, select: true });
```

### package.json 配置

```json
{
  "contributes": {
    "views": {
      "explorer": [
        {
          "id": "myTreeView",
          "name": "My Tree View",
          "icon": "resources/icon.svg",
          "contextualTitle": "My Extension"
        }
      ]
    },
    "menus": {
      "view/title": [
        {
          "command": "myExtension.refresh",
          "when": "view == myTreeView",
          "group": "navigation"
        }
      ],
      "view/item/context": [
        {
          "command": "myExtension.delete",
          "when": "view == myTreeView && viewItem == child"
        }
      ]
    }
  }
}
```

---

## 4. WebView 面板

### 创建 WebView

```typescript
class MyWebviewPanel {
  public static currentPanel: MyWebviewPanel | undefined;
  private readonly panel: vscode.WebviewPanel;
  private disposables: vscode.Disposable[] = [];

  public static createOrShow(extensionUri: vscode.Uri) {
    const column = vscode.window.activeTextEditor
      ? vscode.window.activeTextEditor.viewColumn
      : undefined;

    if (MyWebviewPanel.currentPanel) {
      MyWebviewPanel.currentPanel.panel.reveal(column);
      return;
    }

    const panel = vscode.window.createWebviewPanel(
      'myWebview',
      'My Webview',
      column || vscode.ViewColumn.One,
      {
        enableScripts: true,
        retainContextWhenHidden: true,
        localResourceRoots: [vscode.Uri.joinPath(extensionUri, 'media')],
      }
    );

    MyWebviewPanel.currentPanel = new MyWebviewPanel(panel, extensionUri);
  }

  private constructor(panel: vscode.WebviewPanel, extensionUri: vscode.Uri) {
    this.panel = panel;
    this.update(extensionUri);

    // 监听面板销毁
    this.panel.onDidDispose(() => this.dispose(), null, this.disposables);

    // 监听消息
    this.panel.webview.onDidReceiveMessage(
      (message) => {
        switch (message.command) {
          case 'alert':
            vscode.window.showInformationMessage(message.text);
            break;
          case 'getData':
            this.panel.webview.postMessage({ command: 'data', data: { foo: 'bar' } });
            break;
        }
      },
      null,
      this.disposables
    );
  }

  private update(extensionUri: vscode.Uri) {
    const webview = this.panel.webview;
    const scriptUri = webview.asWebviewUri(
      vscode.Uri.joinPath(extensionUri, 'media', 'main.js')
    );
    const styleUri = webview.asWebviewUri(
      vscode.Uri.joinPath(extensionUri, 'media', 'style.css')
    );

    this.panel.webview.html = `
      <!DOCTYPE html>
      <html>
      <head>
        <meta charset="UTF-8">
        <meta http-equiv="Content-Security-Policy"
              content="default-src 'none'; style-src ${webview.cspSource}; script-src ${webview.cspSource};">
        <link href="${styleUri}" rel="stylesheet">
      </head>
      <body>
        <h1>My Webview</h1>
        <button id="alertBtn">Show Alert</button>
        <script src="${scriptUri}"></script>
      </body>
      </html>
    `;
  }

  public dispose() {
    MyWebviewPanel.currentPanel = undefined;
    this.panel.dispose();
    while (this.disposables.length) {
      const d = this.disposables.pop();
      if (d) d.dispose();
    }
  }
}
```

### WebView 中的 JavaScript (media/main.js)

```javascript
(function () {
  const vscode = acquireVsCodeApi();

  // 保存状态
  const previousState = vscode.getState();

  document.getElementById('alertBtn').addEventListener('click', () => {
    vscode.postMessage({ command: 'alert', text: 'Hello from WebView!' });
  });

  // 接收消息
  window.addEventListener('message', (event) => {
    const message = event.data;
    switch (message.command) {
      case 'data':
        console.log('Received data:', message.data);
        // 保存状态
        vscode.setState({ lastData: message.data });
        break;
    }
  });
})();
```

---

## 5. 语言特性 Provider

### 自动补全

```typescript
const completionProvider = vscode.languages.registerCompletionItemProvider(
  'javascript', // 或 { scheme: 'file', language: 'javascript' }
  {
    provideCompletionItems(
      document: vscode.TextDocument,
      position: vscode.Position
    ): vscode.CompletionItem[] {
      const linePrefix = document
        .lineAt(position)
        .text.slice(0, position.character);

      if (!linePrefix.endsWith('console.')) {
        return [];
      }

      const logItem = new vscode.CompletionItem(
        'log',
        vscode.CompletionItemKind.Method
      );
      logItem.insertText = new vscode.SnippetString('log($1)$0');
      logItem.documentation = new vscode.MarkdownString('Log to console');

      const errorItem = new vscode.CompletionItem(
        'error',
        vscode.CompletionItemKind.Method
      );
      errorItem.insertText = new vscode.SnippetString('error($1)$0');

      return [logItem, errorItem];
    },
  },
  '.' // 触发字符
);
context.subscriptions.push(completionProvider);
```

### 悬停提示

```typescript
const hoverProvider = vscode.languages.registerHoverProvider('javascript', {
  provideHover(
    document: vscode.TextDocument,
    position: vscode.Position
  ): vscode.Hover | undefined {
    const range = document.getWordRangeAtPosition(position);
    const word = document.getText(range);

    if (word === 'console') {
      return new vscode.Hover([
        new vscode.MarkdownString('**Console API**'),
        new vscode.MarkdownString('用于调试输出'),
      ]);
    }
    return undefined;
  },
});
context.subscriptions.push(hoverProvider);
```

### 代码操作（快速修复）

```typescript
const codeActionProvider = vscode.languages.registerCodeActionsProvider(
  'javascript',
  {
    provideCodeActions(
      document: vscode.TextDocument,
      range: vscode.Range,
      context: vscode.CodeActionContext
    ): vscode.CodeAction[] {
      // 只处理诊断信息
      if (context.diagnostics.length === 0) {
        return [];
      }

      const actions: vscode.CodeAction[] = [];

      for (const diagnostic of context.diagnostics) {
        if (diagnostic.code === 'unused-import') {
          const fix = new vscode.CodeAction(
            'Remove unused import',
            vscode.CodeActionKind.QuickFix
          );

          fix.edit = new vscode.WorkspaceEdit();
          fix.edit.delete(document.uri, diagnostic.range);
          fix.diagnostics = [diagnostic];
          fix.isPreferred = true;

          actions.push(fix);
        }
      }

      return actions;
    },
  },
  {
    providedCodeActionKinds: [vscode.CodeActionKind.QuickFix],
  }
);
context.subscriptions.push(codeActionProvider);
```

### 定义跳转

```typescript
const definitionProvider = vscode.languages.registerDefinitionProvider(
  'javascript',
  {
    provideDefinition(
      document: vscode.TextDocument,
      position: vscode.Position
    ): vscode.Definition | undefined {
      const range = document.getWordRangeAtPosition(position);
      const word = document.getText(range);

      // 查找定义位置
      const text = document.getText();
      const definitionRegex = new RegExp(`(function|const|let|var)\\s+${word}\\b`);
      const match = text.match(definitionRegex);

      if (match && match.index !== undefined) {
        const pos = document.positionAt(match.index);
        return new vscode.Location(document.uri, pos);
      }

      return undefined;
    },
  }
);
context.subscriptions.push(definitionProvider);
```

---

## 6. 诊断信息

```typescript
const diagnosticCollection = vscode.languages.createDiagnosticCollection('myLinter');
context.subscriptions.push(diagnosticCollection);

function updateDiagnostics(document: vscode.TextDocument) {
  if (document.languageId !== 'javascript') {
    return;
  }

  const diagnostics: vscode.Diagnostic[] = [];
  const text = document.getText();

  // 示例：检查 TODO 注释
  const todoRegex = /\/\/\s*TODO:/g;
  let match;

  while ((match = todoRegex.exec(text)) !== null) {
    const startPos = document.positionAt(match.index);
    const endPos = document.positionAt(match.index + match[0].length);
    const range = new vscode.Range(startPos, endPos);

    const diagnostic = new vscode.Diagnostic(
      range,
      'TODO comment found',
      vscode.DiagnosticSeverity.Information
    );
    diagnostic.code = 'todo-found';
    diagnostic.source = 'myLinter';

    diagnostics.push(diagnostic);
  }

  diagnosticCollection.set(document.uri, diagnostics);
}

// 监听文档变化
context.subscriptions.push(
  vscode.workspace.onDidChangeTextDocument((e) => updateDiagnostics(e.document)),
  vscode.workspace.onDidOpenTextDocument(updateDiagnostics)
);

// 初始检查
vscode.workspace.textDocuments.forEach(updateDiagnostics);
```

---

## 7. 配置管理

### 读取配置

```typescript
const config = vscode.workspace.getConfiguration('myExtension');
const enabled = config.get<boolean>('enabled', true);
const maxItems = config.get<number>('maxItems', 10);
const items = config.get<string[]>('items', []);
```

### 更新配置

```typescript
// 更新用户级配置
await config.update('enabled', false, vscode.ConfigurationTarget.Global);

// 更新工作区配置
await config.update('maxItems', 20, vscode.ConfigurationTarget.Workspace);

// 更新文件夹配置
await config.update('items', ['a', 'b'], vscode.ConfigurationTarget.WorkspaceFolder);
```

### 监听配置变化

```typescript
context.subscriptions.push(
  vscode.workspace.onDidChangeConfiguration((e) => {
    if (e.affectsConfiguration('myExtension.enabled')) {
      const newValue = vscode.workspace
        .getConfiguration('myExtension')
        .get<boolean>('enabled');
      console.log('enabled changed to:', newValue);
    }
  })
);
```

### package.json 配置声明

```json
{
  "contributes": {
    "configuration": {
      "title": "My Extension",
      "properties": {
        "myExtension.enabled": {
          "type": "boolean",
          "default": true,
          "description": "Enable the extension"
        },
        "myExtension.maxItems": {
          "type": "number",
          "default": 10,
          "minimum": 1,
          "maximum": 100,
          "description": "Maximum number of items"
        },
        "myExtension.items": {
          "type": "array",
          "items": { "type": "string" },
          "default": [],
          "description": "List of items"
        },
        "myExtension.mode": {
          "type": "string",
          "enum": ["fast", "balanced", "thorough"],
          "default": "balanced",
          "enumDescriptions": [
            "Fast mode with less accuracy",
            "Balanced mode",
            "Thorough mode with best accuracy"
          ]
        }
      }
    }
  }
}
```

---

## 8. 状态存储

### 工作区状态

```typescript
// 存储（工作区范围）
await context.workspaceState.update('lastOpenedFile', '/path/to/file');

// 读取
const lastFile = context.workspaceState.get<string>('lastOpenedFile');
```

### 全局状态

```typescript
// 存储（跨工作区）
await context.globalState.update('totalOpens', 42);

// 读取
const count = context.globalState.get<number>('totalOpens', 0);

// 同步到其他设备（Settings Sync）
context.globalState.setKeysForSync(['totalOpens']);
```

### 秘钥存储

```typescript
// 存储敏感信息（使用系统凭证管理器）
await context.secrets.store('apiKey', 'sk-xxx');

// 读取
const apiKey = await context.secrets.get('apiKey');

// 删除
await context.secrets.delete('apiKey');

// 监听变化
context.subscriptions.push(
  context.secrets.onDidChange((e) => {
    if (e.key === 'apiKey') {
      console.log('API key changed');
    }
  })
);
```

---

## 9. 文件系统操作

### 读写文件

```typescript
const uri = vscode.Uri.file('/path/to/file.txt');

// 读取
const content = await vscode.workspace.fs.readFile(uri);
const text = Buffer.from(content).toString('utf8');

// 写入
const newContent = Buffer.from('Hello World', 'utf8');
await vscode.workspace.fs.writeFile(uri, newContent);

// 删除
await vscode.workspace.fs.delete(uri);

// 创建目录
await vscode.workspace.fs.createDirectory(vscode.Uri.file('/path/to/dir'));

// 复制
await vscode.workspace.fs.copy(srcUri, destUri);

// 重命名/移动
await vscode.workspace.fs.rename(oldUri, newUri);
```

### 文件查找

```typescript
// 查找文件
const files = await vscode.workspace.findFiles(
  '**/*.ts',           // include pattern
  '**/node_modules/**' // exclude pattern
);

// 限制数量
const limitedFiles = await vscode.workspace.findFiles('**/*.js', null, 10);
```

### 文件监视

```typescript
const watcher = vscode.workspace.createFileSystemWatcher('**/*.json');

context.subscriptions.push(
  watcher,
  watcher.onDidCreate((uri) => console.log('Created:', uri.fsPath)),
  watcher.onDidChange((uri) => console.log('Changed:', uri.fsPath)),
  watcher.onDidDelete((uri) => console.log('Deleted:', uri.fsPath))
);
```

---

## 10. 输出和日志

### 输出通道

```typescript
const outputChannel = vscode.window.createOutputChannel('My Extension');
context.subscriptions.push(outputChannel);

outputChannel.appendLine('Starting...');
outputChannel.appendLine(`Processing: ${fileName}`);
outputChannel.show(); // 显示输出面板
```

### 日志输出通道

```typescript
const logChannel = vscode.window.createOutputChannel('My Extension', { log: true });
context.subscriptions.push(logChannel);

logChannel.trace('Trace message');
logChannel.debug('Debug message');
logChannel.info('Info message');
logChannel.warn('Warning message');
logChannel.error('Error message');
```

---

## 11. 进度显示

### 通知区域进度

```typescript
await vscode.window.withProgress(
  {
    location: vscode.ProgressLocation.Notification,
    title: 'Processing files',
    cancellable: true,
  },
  async (progress, token) => {
    token.onCancellationRequested(() => {
      console.log('User cancelled');
    });

    for (let i = 0; i < 10; i++) {
      if (token.isCancellationRequested) {
        break;
      }

      progress.report({
        increment: 10,
        message: `File ${i + 1}/10`,
      });

      await new Promise((r) => setTimeout(r, 500));
    }
  }
);
```

### 状态栏进度

```typescript
await vscode.window.withProgress(
  {
    location: vscode.ProgressLocation.Window,
    title: 'Loading...',
  },
  async () => {
    await doSomething();
  }
);
```

---

## 12. 状态栏

```typescript
const statusBarItem = vscode.window.createStatusBarItem(
  'myExtension.status',
  vscode.StatusBarAlignment.Right,
  100
);

statusBarItem.text = '$(sync~spin) Processing';
statusBarItem.tooltip = 'Click to see details';
statusBarItem.command = 'myExtension.showDetails';
statusBarItem.backgroundColor = new vscode.ThemeColor(
  'statusBarItem.warningBackground'
);
statusBarItem.show();

context.subscriptions.push(statusBarItem);

// 更新状态
function updateStatus(count: number) {
  statusBarItem.text = `$(check) ${count} items`;
}
```

---

## 13. 用户输入

### 输入框

```typescript
const name = await vscode.window.showInputBox({
  prompt: 'Enter your name',
  placeHolder: 'John Doe',
  value: 'Default Value',
  validateInput: (value) => {
    if (value.length < 2) {
      return 'Name must be at least 2 characters';
    }
    return null;
  },
});
```

### 快速选择

```typescript
// 简单选择
const choice = await vscode.window.showQuickPick(['Option 1', 'Option 2', 'Option 3'], {
  placeHolder: 'Select an option',
});

// 带详情的选择
interface MyQuickPickItem extends vscode.QuickPickItem {
  id: string;
}

const items: MyQuickPickItem[] = [
  { id: '1', label: 'Item 1', description: 'First item', detail: 'More details here' },
  { id: '2', label: 'Item 2', description: 'Second item', picked: true },
];

const selected = await vscode.window.showQuickPick(items, {
  placeHolder: 'Select an item',
  matchOnDescription: true,
  matchOnDetail: true,
});

if (selected) {
  console.log('Selected ID:', selected.id);
}
```

### 高级 QuickPick

```typescript
const quickPick = vscode.window.createQuickPick<MyQuickPickItem>();
quickPick.placeholder = 'Search items...';
quickPick.items = [];
quickPick.busy = true;

quickPick.onDidChangeValue(async (value) => {
  quickPick.busy = true;
  const results = await searchItems(value);
  quickPick.items = results;
  quickPick.busy = false;
});

quickPick.onDidChangeSelection((items) => {
  if (items.length > 0) {
    console.log('Selected:', items[0].id);
    quickPick.hide();
  }
});

quickPick.onDidHide(() => quickPick.dispose());
quickPick.show();
```

---

## 14. 终端操作

```typescript
// 创建终端
const terminal = vscode.window.createTerminal({
  name: 'My Terminal',
  cwd: '/path/to/dir',
  env: { MY_VAR: 'value' },
});

// 发送命令
terminal.sendText('npm install');
terminal.sendText('npm run build');

// 显示终端
terminal.show();

// 监听终端
context.subscriptions.push(
  vscode.window.onDidOpenTerminal((t) => console.log('Opened:', t.name)),
  vscode.window.onDidCloseTerminal((t) => console.log('Closed:', t.name)),
  vscode.window.onDidChangeActiveTerminal((t) =>
    console.log('Active:', t?.name)
  )
);
```

---

## 15. 编辑器装饰

```typescript
// 创建装饰类型
const decorationType = vscode.window.createTextEditorDecorationType({
  backgroundColor: 'rgba(255,255,0,0.3)',
  border: '1px solid yellow',
  after: {
    contentText: ' ← Here',
    color: 'gray',
    fontStyle: 'italic',
  },
});

// 应用装饰
function updateDecorations(editor: vscode.TextEditor) {
  const text = editor.document.getText();
  const regex = /TODO:/g;
  const decorations: vscode.DecorationOptions[] = [];
  let match;

  while ((match = regex.exec(text))) {
    const startPos = editor.document.positionAt(match.index);
    const endPos = editor.document.positionAt(match.index + match[0].length);
    const range = new vscode.Range(startPos, endPos);

    decorations.push({
      range,
      hoverMessage: new vscode.MarkdownString('**TODO** item found'),
    });
  }

  editor.setDecorations(decorationType, decorations);
}

// 监听编辑器变化
context.subscriptions.push(
  vscode.window.onDidChangeActiveTextEditor((editor) => {
    if (editor) updateDecorations(editor);
  }),
  vscode.workspace.onDidChangeTextDocument((e) => {
    const editor = vscode.window.activeTextEditor;
    if (editor && e.document === editor.document) {
      updateDecorations(editor);
    }
  })
);

// 初始化
if (vscode.window.activeTextEditor) {
  updateDecorations(vscode.window.activeTextEditor);
}

// 清理
context.subscriptions.push(decorationType);
```

---

## 参考

- [VSCode Extension Samples](https://github.com/microsoft/vscode-extension-samples)
- [VSCode API Documentation](https://code.visualstudio.com/api/references/vscode-api)
- [VSCode Extension Guides](https://code.visualstudio.com/api/extension-guides/overview)
