# 21. 语言服务

> **核心问题**：如何提供智能代码功能？

---

## 概述

语言服务决定了：
- 代码补全如何实现
- 语法错误如何检测
- 跳转定义如何工作
- 重构功能如何提供

---

## VSCode 的语言服务

### Language Server Protocol (LSP)

VSCode 是 LSP 的发起者和主要推动者：

```typescript
// LSP 服务端示例
import {
  createConnection,
  TextDocuments,
  ProposedFeatures,
  InitializeParams,
  CompletionItem,
  CompletionItemKind,
  TextDocumentPositionParams,
  TextDocumentSyncKind
} from 'vscode-languageserver/node';
import { TextDocument } from 'vscode-languageserver-textdocument';

const connection = createConnection(ProposedFeatures.all);
const documents = new TextDocuments(TextDocument);

connection.onInitialize((params: InitializeParams) => {
  return {
    capabilities: {
      textDocumentSync: TextDocumentSyncKind.Incremental,
      completionProvider: {
        resolveProvider: true,
        triggerCharacters: ['.', ':', '<']
      },
      hoverProvider: true,
      definitionProvider: true,
      referencesProvider: true,
      documentSymbolProvider: true,
      workspaceSymbolProvider: true,
      codeActionProvider: true,
      renameProvider: true
    }
  };
});

// 代码补全
connection.onCompletion((params: TextDocumentPositionParams): CompletionItem[] => {
  return [
    {
      label: 'console',
      kind: CompletionItemKind.Module,
      data: 1
    },
    {
      label: 'log',
      kind: CompletionItemKind.Function,
      detail: 'Log to console',
      documentation: 'Outputs a message to the console'
    }
  ];
});

// 悬停信息
connection.onHover((params) => {
  const document = documents.get(params.textDocument.uri);
  if (!document) return null;

  return {
    contents: {
      kind: 'markdown',
      value: '```typescript\nfunction log(message: string): void\n```\nLogs a message to the console.'
    }
  };
});

// 跳转定义
connection.onDefinition((params) => {
  return {
    uri: params.textDocument.uri,
    range: {
      start: { line: 0, character: 0 },
      end: { line: 0, character: 10 }
    }
  };
});

documents.listen(connection);
connection.listen();
```

### 客户端集成

```typescript
// VSCode 插件客户端
import * as vscode from 'vscode';
import {
  LanguageClient,
  LanguageClientOptions,
  ServerOptions,
  TransportKind
} from 'vscode-languageclient/node';

let client: LanguageClient;

export function activate(context: vscode.ExtensionContext) {
  const serverModule = context.asAbsolutePath('server/out/server.js');

  const serverOptions: ServerOptions = {
    run: { module: serverModule, transport: TransportKind.ipc },
    debug: {
      module: serverModule,
      transport: TransportKind.ipc,
      options: { execArgv: ['--nolazy', '--inspect=6009'] }
    }
  };

  const clientOptions: LanguageClientOptions = {
    documentSelector: [{ scheme: 'file', language: 'myLanguage' }],
    synchronize: {
      fileEvents: vscode.workspace.createFileSystemWatcher('**/*.mylang')
    }
  };

  client = new LanguageClient(
    'myLanguageServer',
    'My Language Server',
    serverOptions,
    clientOptions
  );

  client.start();
}

export function deactivate(): Thenable<void> | undefined {
  return client?.stop();
}
```

### 内置语言 API

```typescript
// 不使用 LSP，直接注册语言功能
vscode.languages.registerCompletionItemProvider('javascript', {
  provideCompletionItems(document, position, token, context) {
    const linePrefix = document.lineAt(position).text.slice(0, position.character);
    if (!linePrefix.endsWith('console.')) {
      return undefined;
    }

    return [
      new vscode.CompletionItem('log', vscode.CompletionItemKind.Method),
      new vscode.CompletionItem('warn', vscode.CompletionItemKind.Method),
      new vscode.CompletionItem('error', vscode.CompletionItemKind.Method)
    ];
  }
});

// 诊断
const diagnosticCollection = vscode.languages.createDiagnosticCollection('myLinter');

vscode.workspace.onDidChangeTextDocument(event => {
  const document = event.document;
  const diagnostics: vscode.Diagnostic[] = [];

  // 简单的 lint 规则
  const text = document.getText();
  const regex = /console\.log/g;
  let match;
  while ((match = regex.exec(text))) {
    const startPos = document.positionAt(match.index);
    const endPos = document.positionAt(match.index + match[0].length);
    diagnostics.push(new vscode.Diagnostic(
      new vscode.Range(startPos, endPos),
      'Avoid console.log in production',
      vscode.DiagnosticSeverity.Warning
    ));
  }

  diagnosticCollection.set(document.uri, diagnostics);
});
```

---

## Obsidian 的语言服务

### 无内置 LSP 支持

Obsidian 主要面向 Markdown 编辑，没有内置的 LSP 支持：

```typescript
// Obsidian 通过 CodeMirror 6 扩展实现类似功能
import { autocompletion, CompletionContext, CompletionResult } from '@codemirror/autocomplete';

// 自定义补全源
function myCompletions(context: CompletionContext): CompletionResult | null {
  const word = context.matchBefore(/\w*/);
  if (!word || (word.from === word.to && !context.explicit)) {
    return null;
  }

  return {
    from: word.from,
    options: [
      { label: '[[', detail: 'Internal link' },
      { label: '#', detail: 'Tag' },
      { label: '```', detail: 'Code block' }
    ]
  };
}

// 注册到编辑器
export default class MyPlugin extends Plugin {
  onload() {
    this.registerEditorExtension([
      autocompletion({
        override: [myCompletions]
      })
    ]);
  }
}
```

### 链接补全

```typescript
// Obsidian 内置的链接建议
export default class MyPlugin extends Plugin {
  onload() {
    // 通过 EditorSuggest 提供补全
    this.registerEditorSuggest(new LinkSuggest(this.app));
  }
}

class LinkSuggest extends EditorSuggest<TFile> {
  constructor(private app: App) {
    super(app);
  }

  onTrigger(cursor: EditorPosition, editor: Editor, file: TFile | null): EditorSuggestTriggerInfo | null {
    const line = editor.getLine(cursor.line);
    const subString = line.substring(0, cursor.ch);

    // 检测 [[ 触发
    const match = subString.match(/\[\[([^\]]*)?$/);
    if (match) {
      return {
        start: { line: cursor.line, ch: match.index! },
        end: cursor,
        query: match[1] || ''
      };
    }
    return null;
  }

  getSuggestions(context: EditorSuggestContext): TFile[] {
    const files = this.app.vault.getMarkdownFiles();
    return files.filter(file =>
      file.basename.toLowerCase().includes(context.query.toLowerCase())
    );
  }

  renderSuggestion(file: TFile, el: HTMLElement): void {
    el.createEl('div', { text: file.basename });
    el.createEl('small', { text: file.path, cls: 'suggestion-path' });
  }

  selectSuggestion(file: TFile, evt: MouseEvent | KeyboardEvent): void {
    const { editor, start, end } = this.context!;
    editor.replaceRange(`[[${file.basename}]]`, start, end);
  }
}
```

### Linter 插件模式

```typescript
// 第三方 linter 插件实现
export default class LinterPlugin extends Plugin {
  private diagnostics: Map<string, Diagnostic[]> = new Map();

  async onload() {
    // 监听文件变化
    this.registerEvent(
      this.app.workspace.on('editor-change', (editor, info) => {
        this.lint(editor, info.file);
      })
    );
  }

  private async lint(editor: Editor, file: TFile | null) {
    if (!file) return;

    const content = editor.getValue();
    const diagnostics: Diagnostic[] = [];

    // 检查规则
    this.checkHeadingIncrement(content, diagnostics);
    this.checkEmptyLinks(content, diagnostics);
    this.checkTrailingSpaces(content, diagnostics);

    // 通过装饰显示
    this.showDiagnostics(editor, diagnostics);
  }

  private checkHeadingIncrement(content: string, diagnostics: Diagnostic[]) {
    const lines = content.split('\n');
    let prevLevel = 0;

    lines.forEach((line, index) => {
      const match = line.match(/^(#{1,6})\s/);
      if (match) {
        const level = match[1].length;
        if (prevLevel > 0 && level > prevLevel + 1) {
          diagnostics.push({
            line: index,
            message: `Heading level jumped from ${prevLevel} to ${level}`,
            severity: 'warning'
          });
        }
        prevLevel = level;
      }
    });
  }
}
```

---

## 对比分析

### 语言服务对比

| 方面 | VSCode | Obsidian |
|------|--------|----------|
| **协议标准** | LSP（行业标准） | 无标准 |
| **补全机制** | LSP / Provider API | EditorSuggest / CM6 |
| **诊断** | Diagnostic API | 需自行实现 |
| **跳转定义** | 内置支持 | 需自行实现 |
| **重构** | CodeAction | 无内置支持 |
| **目标场景** | 通用编程语言 | Markdown 编辑 |

---

## 对 AI Chat + Editor 应用的建议

### 语言服务架构

```typescript
interface ILanguageService {
  // 补全
  provideCompletions(
    document: IDocument,
    position: Position,
    context: CompletionContext
  ): Promise<CompletionItem[]>;

  // 悬停
  provideHover(
    document: IDocument,
    position: Position
  ): Promise<Hover | null>;

  // 诊断
  provideDiagnostics(
    document: IDocument
  ): Promise<Diagnostic[]>;

  // 定义跳转
  provideDefinition(
    document: IDocument,
    position: Position
  ): Promise<Location[]>;

  // 代码动作
  provideCodeActions(
    document: IDocument,
    range: Range,
    context: CodeActionContext
  ): Promise<CodeAction[]>;
}

// 语言服务注册
interface ILanguageServiceRegistry {
  register(languageId: string, service: ILanguageService): Disposable;
  get(languageId: string): ILanguageService | undefined;
}
```

### AI 增强的语言服务

```typescript
class AILanguageService implements ILanguageService {
  constructor(
    private aiClient: IAIClient,
    private baseService?: ILanguageService
  ) {}

  async provideCompletions(
    document: IDocument,
    position: Position,
    context: CompletionContext
  ): Promise<CompletionItem[]> {
    // 获取基础补全
    const baseCompletions = await this.baseService?.provideCompletions(
      document, position, context
    ) || [];

    // AI 补全（更长的代码片段）
    if (context.triggerKind === 'Invoke' || this.shouldTriggerAI(context)) {
      const aiCompletions = await this.getAICompletions(document, position);
      return [...aiCompletions, ...baseCompletions];
    }

    return baseCompletions;
  }

  private async getAICompletions(
    document: IDocument,
    position: Position
  ): Promise<CompletionItem[]> {
    const prefix = document.getText({
      start: { line: Math.max(0, position.line - 50), character: 0 },
      end: position
    });

    const suffix = document.getText({
      start: position,
      end: { line: position.line + 10, character: 0 }
    });

    const completion = await this.aiClient.complete({
      prefix,
      suffix,
      language: document.languageId,
      maxTokens: 150
    });

    if (completion) {
      return [{
        label: completion.slice(0, 50) + '...',
        insertText: completion,
        kind: 'ai-suggestion',
        detail: 'AI Suggestion',
        sortText: '0' // 优先显示
      }];
    }

    return [];
  }

  async provideCodeActions(
    document: IDocument,
    range: Range,
    context: CodeActionContext
  ): Promise<CodeAction[]> {
    const actions: CodeAction[] = [];

    // AI 代码解释
    actions.push({
      title: 'Explain this code',
      kind: 'ai.explain',
      command: {
        command: 'ai.explainCode',
        arguments: [document.uri, range]
      }
    });

    // AI 代码重构
    actions.push({
      title: 'Refactor with AI',
      kind: 'ai.refactor',
      command: {
        command: 'ai.refactorCode',
        arguments: [document.uri, range]
      }
    });

    // AI 修复建议
    if (context.diagnostics.length > 0) {
      actions.push({
        title: 'Fix with AI',
        kind: 'ai.quickfix',
        command: {
          command: 'ai.fixCode',
          arguments: [document.uri, range, context.diagnostics]
        }
      });
    }

    return actions;
  }
}
```

### LSP 集成（可选）

```typescript
// 支持标准 LSP 服务端
class LSPClientManager {
  private clients: Map<string, LanguageClient> = new Map();

  async startServer(
    languageId: string,
    config: LSPServerConfig
  ): Promise<void> {
    const client = new LanguageClient({
      serverPath: config.path,
      args: config.args,
      transport: config.transport || 'stdio'
    });

    await client.start();
    this.clients.set(languageId, client);
  }

  async stopServer(languageId: string): Promise<void> {
    const client = this.clients.get(languageId);
    if (client) {
      await client.stop();
      this.clients.delete(languageId);
    }
  }

  getClient(languageId: string): LanguageClient | undefined {
    return this.clients.get(languageId);
  }
}

// 简化的 LSP 客户端
class LanguageClient {
  private process: ChildProcess | null = null;
  private connection: MessageConnection | null = null;

  async start(): Promise<void> {
    this.process = spawn(this.config.serverPath, this.config.args);

    const reader = new StreamMessageReader(this.process.stdout!);
    const writer = new StreamMessageWriter(this.process.stdin!);

    this.connection = createMessageConnection(reader, writer);
    this.connection.listen();

    // 初始化
    const initResult = await this.connection.sendRequest('initialize', {
      processId: process.pid,
      capabilities: this.getClientCapabilities(),
      rootUri: this.workspaceRoot
    });

    await this.connection.sendNotification('initialized', {});
  }

  async completion(uri: string, position: Position): Promise<CompletionItem[]> {
    if (!this.connection) return [];

    const result = await this.connection.sendRequest('textDocument/completion', {
      textDocument: { uri },
      position
    });

    return Array.isArray(result) ? result : result?.items || [];
  }
}
```

### Markdown 特化服务

```typescript
// Markdown 专用语言服务
class MarkdownLanguageService implements ILanguageService {
  async provideCompletions(
    document: IDocument,
    position: Position,
    context: CompletionContext
  ): Promise<CompletionItem[]> {
    const line = document.getLine(position.line);
    const prefix = line.slice(0, position.character);

    // 链接补全 [[
    if (prefix.endsWith('[[')) {
      return this.getLinkCompletions(document);
    }

    // 标签补全 #
    if (prefix.endsWith('#')) {
      return this.getTagCompletions(document);
    }

    // 代码块语言
    if (prefix.match(/^```\w*$/)) {
      return this.getLanguageCompletions();
    }

    // Emoji :
    if (prefix.endsWith(':')) {
      return this.getEmojiCompletions();
    }

    return [];
  }

  async provideDiagnostics(document: IDocument): Promise<Diagnostic[]> {
    const diagnostics: Diagnostic[] = [];
    const content = document.getText();

    // 检查死链接
    const linkRegex = /\[\[([^\]]+)\]\]/g;
    let match;
    while ((match = linkRegex.exec(content))) {
      const linkTarget = match[1];
      if (!this.fileExists(linkTarget)) {
        const pos = document.positionAt(match.index);
        diagnostics.push({
          range: {
            start: pos,
            end: { line: pos.line, character: pos.character + match[0].length }
          },
          message: `Link target "${linkTarget}" not found`,
          severity: 'warning'
        });
      }
    }

    return diagnostics;
  }
}
```

---

## 关键决策清单

1. **是否支持 LSP？**
   - 支持：可复用现有语言服务器
   - 不支持：简化架构，专注核心功能

2. **AI 补全如何集成？**
   - 作为独立补全源
   - 增强现有补全

3. **诊断如何显示？**
   - 编辑器装饰
   - Problems 面板
   - 内联提示

4. **性能策略？**
   - 防抖
   - 增量更新
   - 缓存

---

## 参考资料

- [Language Server Protocol](https://microsoft.github.io/language-server-protocol/)
- [VSCode Language Extensions](https://code.visualstudio.com/api/language-extensions/overview)
- [CodeMirror Autocomplete](https://codemirror.net/docs/ref/#autocomplete)
