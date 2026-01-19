# VSCode Extension API Reference

> 来源：vscode.d.ts 类型定义及官方文档

## 1. 核心 API 层级

```
vscode (命名空间入口)
├── commands          # 命令系统
├── window            # 窗口/UI 管理
├── workspace         # 工作区管理
├── languages         # 语言特性
├── debug             # 调试系统
├── extensions        # 扩展管理
├── env               # 环境信息
├── tasks             # 任务系统
├── scm               # 源代码管理
├── authentication    # 认证系统
├── notebooks         # Notebook 支持
├── tests             # 测试 API
├── comments          # 评论 API
└── l10n              # 国际化
```

---

## 2. ExtensionContext 核心对象

```typescript
interface ExtensionContext {
  // 订阅管理（自动清理）
  subscriptions: { dispose(): any }[];

  // 工作区状态存储（工作区范围）
  workspaceState: Memento;
  // 全局状态存储（跨工作区）
  globalState: Memento & { setKeysForSync(keys: readonly string[]): void };

  // 秘钥存储（系统凭证管理器）
  secrets: SecretStorage;

  // 扩展路径
  extensionUri: Uri;
  extensionPath: string;

  // 存储路径
  storageUri: Uri | undefined;        // 工作区存储
  globalStorageUri: Uri;              // 全局存储
  logUri: Uri;                        // 日志目录

  // 扩展模式
  extensionMode: ExtensionMode;       // Production | Development | Test

  // 环境变量（用于子进程）
  environmentVariableCollection: GlobalEnvironmentVariableCollection;
}
```

---

## 3. Commands API

```typescript
namespace commands {
  // 注册命令
  function registerCommand(
    command: string,
    callback: (...args: any[]) => any,
    thisArg?: any
  ): Disposable;

  // 注册文本编辑器命令
  function registerTextEditorCommand(
    command: string,
    callback: (editor: TextEditor, edit: TextEditorEdit, ...args: any[]) => void,
    thisArg?: any
  ): Disposable;

  // 执行命令
  function executeCommand<T = unknown>(
    command: string,
    ...rest: any[]
  ): Thenable<T>;

  // 获取所有命令
  function getCommands(filterInternal?: boolean): Thenable<string[]>;
}
```

---

## 4. Window API

```typescript
namespace window {
  // 当前编辑器
  activeTextEditor: TextEditor | undefined;
  visibleTextEditors: readonly TextEditor[];

  // 终端
  activeTerminal: Terminal | undefined;
  terminals: readonly Terminal[];

  // 通知
  function showInformationMessage(message: string, ...items: string[]): Thenable<string | undefined>;
  function showWarningMessage(message: string, ...items: string[]): Thenable<string | undefined>;
  function showErrorMessage(message: string, ...items: string[]): Thenable<string | undefined>;

  // 输入
  function showInputBox(options?: InputBoxOptions): Thenable<string | undefined>;
  function showQuickPick(items: string[], options?: QuickPickOptions): Thenable<string | undefined>;
  function showQuickPick<T extends QuickPickItem>(
    items: T[],
    options?: QuickPickOptions
  ): Thenable<T | undefined>;

  // 高级选择器
  function createQuickPick<T extends QuickPickItem>(): QuickPick<T>;
  function createInputBox(): InputBox;

  // 进度
  function withProgress<R>(
    options: ProgressOptions,
    task: (progress: Progress<{message?: string; increment?: number}>, token: CancellationToken) => Thenable<R>
  ): Thenable<R>;

  // 终端
  function createTerminal(options?: TerminalOptions): Terminal;

  // 输出
  function createOutputChannel(name: string, options?: { log: true }): LogOutputChannel;
  function createOutputChannel(name: string, languageId?: string): OutputChannel;

  // WebView
  function createWebviewPanel(
    viewType: string,
    title: string,
    showOptions: ViewColumn | { viewColumn: ViewColumn; preserveFocus?: boolean },
    options?: WebviewPanelOptions & WebviewOptions
  ): WebviewPanel;

  // 视图注册
  function registerTreeDataProvider<T>(viewId: string, treeDataProvider: TreeDataProvider<T>): Disposable;
  function createTreeView<T>(viewId: string, options: TreeViewOptions<T>): TreeView<T>;
  function registerWebviewViewProvider(
    viewId: string,
    provider: WebviewViewProvider,
    options?: { webviewOptions?: { retainContextWhenHidden?: boolean } }
  ): Disposable;

  // 编辑器操作
  function showTextDocument(
    document: TextDocument,
    column?: ViewColumn,
    preserveFocus?: boolean
  ): Thenable<TextEditor>;
  function createTextEditorDecorationType(options: DecorationRenderOptions): TextEditorDecorationType;

  // 状态栏
  function createStatusBarItem(
    id: string,
    alignment?: StatusBarAlignment,
    priority?: number
  ): StatusBarItem;

  // 事件
  onDidChangeActiveTextEditor: Event<TextEditor | undefined>;
  onDidChangeVisibleTextEditors: Event<readonly TextEditor[]>;
  onDidChangeTextEditorSelection: Event<TextEditorSelectionChangeEvent>;
  onDidChangeActiveTerminal: Event<Terminal | undefined>;
}
```

---

## 5. Workspace API

```typescript
namespace workspace {
  // 工作区信息
  workspaceFolders: readonly WorkspaceFolder[] | undefined;
  name: string | undefined;
  workspaceFile: Uri | undefined;

  // 文件系统
  fs: FileSystem;

  // 配置
  function getConfiguration(
    section?: string,
    scope?: ConfigurationScope | null
  ): WorkspaceConfiguration;

  // 文件操作
  function openTextDocument(uri: Uri): Thenable<TextDocument>;
  function openTextDocument(fileName: string): Thenable<TextDocument>;
  function openTextDocument(options?: { language?: string; content?: string }): Thenable<TextDocument>;

  // 编辑
  function applyEdit(edit: WorkspaceEdit): Thenable<boolean>;

  // 文件查找
  function findFiles(
    include: GlobPattern,
    exclude?: GlobPattern | null,
    maxResults?: number,
    token?: CancellationToken
  ): Thenable<Uri[]>;

  // 文件系统监视器
  function createFileSystemWatcher(
    globPattern: GlobPattern,
    ignoreCreateEvents?: boolean,
    ignoreChangeEvents?: boolean,
    ignoreDeleteEvents?: boolean
  ): FileSystemWatcher;

  // 保存
  function saveAll(includeUntitled?: boolean): Thenable<boolean>;

  // 事件
  onDidOpenTextDocument: Event<TextDocument>;
  onDidCloseTextDocument: Event<TextDocument>;
  onDidChangeTextDocument: Event<TextDocumentChangeEvent>;
  onDidSaveTextDocument: Event<TextDocument>;
  onDidChangeWorkspaceFolders: Event<WorkspaceFoldersChangeEvent>;
  onDidChangeConfiguration: Event<ConfigurationChangeEvent>;
  onWillSaveTextDocument: Event<TextDocumentWillSaveEvent>;
}
```

---

## 6. Languages API

```typescript
namespace languages {
  // 语言特性提供者注册
  function registerCompletionItemProvider(
    selector: DocumentSelector,
    provider: CompletionItemProvider,
    ...triggerCharacters: string[]
  ): Disposable;

  function registerHoverProvider(
    selector: DocumentSelector,
    provider: HoverProvider
  ): Disposable;

  function registerDefinitionProvider(
    selector: DocumentSelector,
    provider: DefinitionProvider
  ): Disposable;

  function registerReferenceProvider(
    selector: DocumentSelector,
    provider: ReferenceProvider
  ): Disposable;

  function registerDocumentSymbolProvider(
    selector: DocumentSelector,
    provider: DocumentSymbolProvider,
    metaData?: DocumentSymbolProviderMetadata
  ): Disposable;

  function registerCodeActionsProvider(
    selector: DocumentSelector,
    provider: CodeActionProvider,
    metadata?: CodeActionProviderMetadata
  ): Disposable;

  function registerCodeLensProvider(
    selector: DocumentSelector,
    provider: CodeLensProvider
  ): Disposable;

  function registerDocumentFormattingEditProvider(
    selector: DocumentSelector,
    provider: DocumentFormattingEditProvider
  ): Disposable;

  function registerRenameProvider(
    selector: DocumentSelector,
    provider: RenameProvider
  ): Disposable;

  function registerSignatureHelpProvider(
    selector: DocumentSelector,
    provider: SignatureHelpProvider,
    ...triggerCharacters: string[]
  ): Disposable;

  function registerInlayHintsProvider(
    selector: DocumentSelector,
    provider: InlayHintsProvider
  ): Disposable;

  // 诊断
  function createDiagnosticCollection(name?: string): DiagnosticCollection;

  // 语言相关
  function getLanguages(): Thenable<string[]>;
  function setLanguageConfiguration(
    language: string,
    configuration: LanguageConfiguration
  ): Disposable;

  // 事件
  onDidChangeDiagnostics: Event<DiagnosticChangeEvent>;
}
```

---

## 7. Debug API

```typescript
namespace debug {
  // 当前调试会话
  activeDebugSession: DebugSession | undefined;
  activeDebugConsole: DebugConsole;
  breakpoints: readonly Breakpoint[];

  // 调试控制
  function startDebugging(
    folder: WorkspaceFolder | undefined,
    nameOrConfiguration: string | DebugConfiguration,
    parentSessionOrOptions?: DebugSession | DebugSessionOptions
  ): Thenable<boolean>;

  function stopDebugging(session?: DebugSession): Thenable<void>;

  // 断点
  function addBreakpoints(breakpoints: readonly Breakpoint[]): void;
  function removeBreakpoints(breakpoints: readonly Breakpoint[]): void;

  // 注册调试适配器
  function registerDebugAdapterDescriptorFactory(
    debugType: string,
    factory: DebugAdapterDescriptorFactory
  ): Disposable;

  function registerDebugConfigurationProvider(
    debugType: string,
    provider: DebugConfigurationProvider,
    triggerKind?: DebugConfigurationProviderTriggerKind
  ): Disposable;

  // 事件
  onDidStartDebugSession: Event<DebugSession>;
  onDidTerminateDebugSession: Event<DebugSession>;
  onDidChangeActiveDebugSession: Event<DebugSession | undefined>;
  onDidChangeBreakpoints: Event<BreakpointsChangeEvent>;
}
```

---

## 8. Extensions API

```typescript
namespace extensions {
  // 获取扩展
  function getExtension<T = any>(extensionId: string): Extension<T> | undefined;

  // 所有扩展
  all: readonly Extension<any>[];

  // 事件
  onDidChange: Event<void>;
}

interface Extension<T> {
  id: string;
  extensionUri: Uri;
  extensionPath: string;
  isActive: boolean;
  packageJSON: any;
  extensionKind: ExtensionKind;
  exports: T;

  // 激活扩展
  activate(): Thenable<T>;
}
```

---

## 9. TreeDataProvider 接口

```typescript
interface TreeDataProvider<T> {
  // 获取元素数据
  getTreeItem(element: T): TreeItem | Thenable<TreeItem>;

  // 获取子元素
  getChildren(element?: T): ProviderResult<T[]>;

  // 获取父元素（可选，用于 reveal）
  getParent?(element: T): ProviderResult<T>;

  // 解析 TreeItem（可选，懒加载详情）
  resolveTreeItem?(
    item: TreeItem,
    element: T,
    token: CancellationToken
  ): ProviderResult<TreeItem>;

  // 数据变化事件
  onDidChangeTreeData?: Event<T | T[] | undefined | null | void>;
}

class TreeItem {
  label: string | TreeItemLabel;
  id?: string;
  iconPath?: string | Uri | { light: string | Uri; dark: string | Uri } | ThemeIcon;
  description?: string | boolean;
  tooltip?: string | MarkdownString | undefined;
  command?: Command;
  collapsibleState?: TreeItemCollapsibleState;
  contextValue?: string;
  accessibilityInformation?: AccessibilityInformation;
  checkboxState?: TreeItemCheckboxState;
}
```

---

## 10. WebView API

```typescript
interface WebviewPanel {
  viewType: string;
  title: string;
  iconPath?: Uri | { light: Uri; dark: Uri };
  webview: Webview;
  options: WebviewPanelOptions;
  viewColumn: ViewColumn | undefined;
  active: boolean;
  visible: boolean;

  reveal(viewColumn?: ViewColumn, preserveFocus?: boolean): void;
  dispose(): void;

  onDidChangeViewState: Event<WebviewPanelOnDidChangeViewStateEvent>;
  onDidDispose: Event<void>;
}

interface Webview {
  options: WebviewOptions;
  html: string;
  cspSource: string;

  asWebviewUri(localResource: Uri): Uri;
  postMessage(message: any): Thenable<boolean>;
  onDidReceiveMessage: Event<any>;
}
```

---

## 11. TextDocument 接口

```typescript
interface TextDocument {
  uri: Uri;
  fileName: string;
  isUntitled: boolean;
  languageId: string;
  version: number;
  isDirty: boolean;
  isClosed: boolean;
  eol: EndOfLine;
  lineCount: number;

  // 文本访问
  getText(range?: Range): string;
  lineAt(line: number): TextLine;
  lineAt(position: Position): TextLine;
  offsetAt(position: Position): number;
  positionAt(offset: number): Position;
  getWordRangeAtPosition(position: Position, regex?: RegExp): Range | undefined;
  validateRange(range: Range): Range;
  validatePosition(position: Position): Position;

  // 保存
  save(): Thenable<boolean>;
}
```

---

## 12. TextEditor 接口

```typescript
interface TextEditor {
  document: TextDocument;
  selection: Selection;
  selections: readonly Selection[];
  visibleRanges: readonly Range[];
  options: TextEditorOptions;
  viewColumn: ViewColumn | undefined;

  // 编辑
  edit(
    callback: (editBuilder: TextEditorEdit) => void,
    options?: { undoStopBefore: boolean; undoStopAfter: boolean }
  ): Thenable<boolean>;

  // 插入代码片段
  insertSnippet(
    snippet: SnippetString,
    location?: Position | Range | readonly Position[] | readonly Range[],
    options?: { undoStopBefore: boolean; undoStopAfter: boolean }
  ): Thenable<boolean>;

  // 装饰
  setDecorations(
    decorationType: TextEditorDecorationType,
    rangesOrOptions: readonly Range[] | readonly DecorationOptions[]
  ): void;

  // 滚动
  revealRange(range: Range, revealType?: TextEditorRevealType): void;

  // 显示
  show(column?: ViewColumn): void;
  hide(): void;
}

interface TextEditorEdit {
  replace(location: Position | Range | Selection, value: string): void;
  insert(location: Position, value: string): void;
  delete(location: Range | Selection): void;
  setEndOfLine(endOfLine: EndOfLine): void;
}
```

---

## 13. Disposable 模式

```typescript
class Disposable {
  constructor(callOnDispose: () => any);
  dispose(): any;

  // 组合多个 Disposable
  static from(...disposableLikes: { dispose: () => any }[]): Disposable;
}

// 使用模式
class MyClass {
  private disposables: Disposable[] = [];

  activate(context: ExtensionContext) {
    // 方式 1: 直接加入 subscriptions
    context.subscriptions.push(
      commands.registerCommand('my.command', () => {})
    );

    // 方式 2: 自己管理
    this.disposables.push(
      commands.registerCommand('another.command', () => {})
    );
  }

  dispose() {
    this.disposables.forEach(d => d.dispose());
    this.disposables = [];
  }
}
```

---

## 14. Event 模式

```typescript
interface Event<T> {
  // 订阅事件
  (listener: (e: T) => any, thisArgs?: any, disposables?: Disposable[]): Disposable;
}

// 创建自定义事件
class EventEmitter<T> {
  event: Event<T>;
  fire(data: T): void;
  dispose(): void;
}

// 使用示例
class MyDataProvider implements TreeDataProvider<MyItem> {
  private _onDidChangeTreeData = new EventEmitter<MyItem | undefined>();
  readonly onDidChangeTreeData = this._onDidChangeTreeData.event;

  refresh(item?: MyItem) {
    this._onDidChangeTreeData.fire(item);
  }
}
```

---

## 15. 命名规范

### API 命名空间

| 命名空间 | 职责 |
|---------|------|
| `commands` | 命令注册和执行 |
| `window` | UI/窗口操作 |
| `workspace` | 工作区/文件操作 |
| `languages` | 语言特性 |
| `debug` | 调试系统 |
| `extensions` | 扩展管理 |
| `env` | 环境信息 |
| `tasks` | 任务系统 |

### 方法命名

| 前缀 | 动作 | 返回值 |
|------|------|--------|
| `register*` | 注册扩展点 | Disposable |
| `create*` | 创建对象 | 新对象 |
| `show*` | 显示 UI | Thenable |
| `open*` | 打开资源 | Thenable |
| `get*` | 获取数据 | 数据/undefined |
| `execute*` | 执行操作 | Thenable |

### 事件命名

```typescript
// 窗口事件
onDidChangeActiveTextEditor
onDidChangeVisibleTextEditors
onDidChangeTextEditorSelection

// 工作区事件
onDidOpenTextDocument
onDidCloseTextDocument
onDidChangeTextDocument
onDidSaveTextDocument
onDidChangeConfiguration

// 调试事件
onDidStartDebugSession
onDidTerminateDebugSession
```

---

## 参考

- [VSCode API Documentation](https://code.visualstudio.com/api/references/vscode-api)
- [VSCode Extension Guides](https://code.visualstudio.com/api/extension-guides/overview)
- [vscode.d.ts](https://github.com/microsoft/vscode/blob/main/src/vscode-dts/vscode.d.ts)
