# 15. 编辑器扩展

> **核心问题**：编辑器功能如何扩展？

---

## 概述

编辑器扩展决定了：
- 如何添加语法高亮
- 如何实现代码补全
- 如何添加代码诊断
- 如何实现悬停提示

---

## VSCode 的编辑器扩展

### Language Server Protocol (LSP)

VSCode 使用 LSP 分离语言服务：

```
┌─────────────────┐         ┌─────────────────┐
│  VSCode Editor  │  JSON   │ Language Server │
│                 │◄───────►│   (独立进程)    │
│  - 编辑器 UI    │  RPC    │  - 语法分析     │
│  - 装饰渲染     │         │  - 补全计算     │
│                 │         │  - 诊断生成     │
└─────────────────┘         └─────────────────┘
```

### 语法高亮

```json
// package.json
{
  "contributes": {
    "languages": [{
      "id": "mylang",
      "extensions": [".ml"]
    }],
    "grammars": [{
      "language": "mylang",
      "scopeName": "source.mylang",
      "path": "./syntaxes/mylang.tmLanguage.json"
    }]
  }
}
```

```json
// syntaxes/mylang.tmLanguage.json (TextMate 格式)
{
  "scopeName": "source.mylang",
  "patterns": [
    {
      "name": "comment.line.mylang",
      "match": "//.*$"
    },
    {
      "name": "string.quoted.double.mylang",
      "begin": "\"",
      "end": "\""
    },
    {
      "name": "keyword.control.mylang",
      "match": "\\b(if|else|while|return)\\b"
    }
  ]
}
```

### 代码补全

```typescript
vscode.languages.registerCompletionItemProvider('mylang', {
  provideCompletionItems(document, position, token, context) {
    const items: vscode.CompletionItem[] = [];

    // 关键字补全
    items.push({
      label: 'function',
      kind: vscode.CompletionItemKind.Keyword,
      insertText: 'function ${1:name}(${2:params}) {\n\t$0\n}',
      insertTextRules: vscode.CompletionItemInsertTextRule.InsertAsSnippet,
      documentation: 'Define a function'
    });

    // 上下文相关补全
    const linePrefix = document.lineAt(position).text.substr(0, position.character);
    if (linePrefix.endsWith('console.')) {
      items.push(
        { label: 'log', kind: vscode.CompletionItemKind.Method },
        { label: 'error', kind: vscode.CompletionItemKind.Method },
        { label: 'warn', kind: vscode.CompletionItemKind.Method }
      );
    }

    return items;
  },

  resolveCompletionItem(item, token) {
    // 延迟加载详细信息
    item.documentation = new vscode.MarkdownString(`**${item.label}**\n\nDetailed documentation...`);
    return item;
  }
});
```

### 代码诊断

```typescript
const diagnosticCollection = vscode.languages.createDiagnosticCollection('mylang');

// 分析文档并生成诊断
function analyzeDocument(document: vscode.TextDocument) {
  const diagnostics: vscode.Diagnostic[] = [];
  const text = document.getText();

  // 示例：检测未使用的变量
  const unusedVarRegex = /let\s+(\w+)\s*=/g;
  let match;
  while ((match = unusedVarRegex.exec(text)) !== null) {
    const varName = match[1];
    if (!text.includes(varName, match.index + match[0].length)) {
      const range = new vscode.Range(
        document.positionAt(match.index),
        document.positionAt(match.index + match[0].length)
      );
      diagnostics.push({
        range,
        message: `Variable '${varName}' is declared but never used`,
        severity: vscode.DiagnosticSeverity.Warning,
        source: 'mylang'
      });
    }
  }

  diagnosticCollection.set(document.uri, diagnostics);
}

// 监听文档变化
vscode.workspace.onDidChangeTextDocument(e => {
  analyzeDocument(e.document);
});
```

### 悬停提示

```typescript
vscode.languages.registerHoverProvider('mylang', {
  provideHover(document, position, token) {
    const range = document.getWordRangeAtPosition(position);
    const word = document.getText(range);

    // 查找定义
    const definition = findDefinition(word);
    if (definition) {
      return new vscode.Hover([
        `**${definition.name}**: ${definition.type}`,
        new vscode.MarkdownString(`\`\`\`mylang\n${definition.signature}\n\`\`\``)
      ]);
    }
  }
});
```

### Code Action（代码操作）

```typescript
vscode.languages.registerCodeActionsProvider('mylang', {
  provideCodeActions(document, range, context, token) {
    const actions: vscode.CodeAction[] = [];

    // 针对诊断提供修复
    for (const diagnostic of context.diagnostics) {
      if (diagnostic.message.includes('never used')) {
        const fix = new vscode.CodeAction(
          'Remove unused variable',
          vscode.CodeActionKind.QuickFix
        );
        fix.edit = new vscode.WorkspaceEdit();
        fix.edit.delete(document.uri, diagnostic.range);
        fix.diagnostics = [diagnostic];
        actions.push(fix);
      }
    }

    return actions;
  }
});
```

---

## Obsidian 的编辑器扩展

Obsidian 使用 CodeMirror 6，扩展方式不同。

### 语法高亮

```typescript
import { HighlightStyle, syntaxHighlighting } from '@codemirror/language';
import { tags } from '@lezer/highlight';

// 定义高亮样式
const myHighlightStyle = HighlightStyle.define([
  { tag: tags.keyword, color: '#569CD6' },
  { tag: tags.string, color: '#CE9178' },
  { tag: tags.comment, color: '#6A9955' },
  { tag: tags.function(tags.variableName), color: '#DCDCAA' },
]);

// 注册扩展
this.registerEditorExtension([
  syntaxHighlighting(myHighlightStyle)
]);
```

### 代码补全

```typescript
import { EditorSuggest, EditorSuggestContext, EditorSuggestTriggerInfo } from 'obsidian';

class MyEditorSuggest extends EditorSuggest<string> {
  onTrigger(cursor: EditorPosition, editor: Editor, file: TFile): EditorSuggestTriggerInfo | null {
    const line = editor.getLine(cursor.line);
    const match = line.slice(0, cursor.ch).match(/::(\w*)$/);

    if (match) {
      return {
        start: { line: cursor.line, ch: cursor.ch - match[0].length },
        end: cursor,
        query: match[1]
      };
    }
    return null;
  }

  getSuggestions(context: EditorSuggestContext): string[] {
    const query = context.query.toLowerCase();
    return this.getAllSuggestions()
      .filter(s => s.toLowerCase().includes(query));
  }

  renderSuggestion(suggestion: string, el: HTMLElement): void {
    el.createEl('span', { text: suggestion });
  }

  selectSuggestion(suggestion: string, evt: MouseEvent | KeyboardEvent): void {
    const { editor, start, end } = this.context;
    editor.replaceRange(suggestion, start, end);
  }
}

// 注册
this.registerEditorSuggest(new MyEditorSuggest(this.app));
```

### CodeMirror 6 扩展

```typescript
import { ViewPlugin, DecorationSet, Decoration } from '@codemirror/view';
import { syntaxTree } from '@codemirror/language';

// 自定义装饰扩展
const highlightExtension = ViewPlugin.fromClass(
  class {
    decorations: DecorationSet;

    constructor(view: EditorView) {
      this.decorations = this.buildDecorations(view);
    }

    update(update: ViewUpdate) {
      if (update.docChanged || update.viewportChanged) {
        this.decorations = this.buildDecorations(update.view);
      }
    }

    buildDecorations(view: EditorView): DecorationSet {
      const widgets: Range<Decoration>[] = [];

      for (const { from, to } of view.visibleRanges) {
        syntaxTree(view.state).iterate({
          from, to,
          enter: (node) => {
            if (node.name === 'URL') {
              widgets.push(
                linkDecoration.range(node.from, node.to)
              );
            }
          }
        });
      }

      return Decoration.set(widgets);
    }
  },
  { decorations: v => v.decorations }
);

// 注册
this.registerEditorExtension([highlightExtension]);
```

### Markdown 后处理

```typescript
// 处理渲染后的 Markdown
this.registerMarkdownPostProcessor((el, ctx) => {
  // 找到代码块
  const codeBlocks = el.querySelectorAll('pre > code');

  codeBlocks.forEach(block => {
    if (block.classList.contains('language-chart')) {
      const source = block.textContent;
      const chartEl = document.createElement('div');
      renderChart(chartEl, source);
      block.parentElement?.replaceWith(chartEl);
    }
  });
});

// 代码块处理器
this.registerMarkdownCodeBlockProcessor('mermaid', (source, el, ctx) => {
  mermaid.render('mermaid-svg', source).then(result => {
    el.innerHTML = result.svg;
  });
});
```

---

## 对比分析

### 扩展方式对比

| 功能 | VSCode | Obsidian |
|------|--------|----------|
| **语法高亮** | TextMate / Semantic | Lezer + HighlightStyle |
| **代码补全** | CompletionItemProvider | EditorSuggest |
| **诊断** | DiagnosticCollection | CM6 Linter |
| **悬停** | HoverProvider | CM6 Tooltip |
| **代码操作** | CodeActionProvider | 无内置 |
| **后处理** | 无（Webview） | MarkdownPostProcessor |

---

## 对 Coding Agent Desktop 应用的建议

### 统一扩展接口

```typescript
interface EditorExtension {
  /** 扩展 ID */
  id: string;

  /** 语法高亮 */
  syntaxHighlight?: SyntaxHighlightConfig;

  /** 补全提供者 */
  completionProvider?: CompletionProvider;

  /** 诊断提供者 */
  diagnosticProvider?: DiagnosticProvider;

  /** 悬停提供者 */
  hoverProvider?: HoverProvider;

  /** 代码操作提供者 */
  codeActionProvider?: CodeActionProvider;
}
```

### AI 补全集成

```typescript
class AICompletionProvider implements CompletionProvider {
  async provideCompletions(
    document: TextDocument,
    position: Position,
    context: CompletionContext
  ): Promise<CompletionItem[]> {
    // 获取上下文
    const prefix = document.getText({
      start: { line: position.line - 10, character: 0 },
      end: position
    });

    // 调用 AI 服务
    const suggestions = await this.aiService.getCompletions({
      prefix,
      language: document.languageId,
      maxSuggestions: 5
    });

    return suggestions.map(s => ({
      label: s.text.split('\n')[0],
      insertText: s.text,
      kind: CompletionKind.Snippet,
      documentation: 'AI generated suggestion',
      sortText: '0'  // 优先显示
    }));
  }
}
```

### AI 诊断集成

```typescript
class AIDiagnosticProvider implements DiagnosticProvider {
  async provideDiagnostics(document: TextDocument): Promise<Diagnostic[]> {
    const code = document.getText();

    // AI 分析代码
    const analysis = await this.aiService.analyzeCode({
      code,
      language: document.languageId
    });

    return analysis.issues.map(issue => ({
      range: issue.range,
      message: issue.message,
      severity: issue.severity,
      source: 'AI',
      data: {
        suggestion: issue.fix  // 存储修复建议
      }
    }));
  }
}

// 配合 Code Action
class AICodeActionProvider implements CodeActionProvider {
  async provideCodeActions(
    document: TextDocument,
    range: Range,
    context: CodeActionContext
  ): Promise<CodeAction[]> {
    const actions: CodeAction[] = [];

    for (const diagnostic of context.diagnostics) {
      if (diagnostic.source === 'AI' && diagnostic.data?.suggestion) {
        actions.push({
          title: `AI Fix: ${diagnostic.message}`,
          kind: CodeActionKind.QuickFix,
          edit: {
            changes: [{
              range: diagnostic.range,
              newText: diagnostic.data.suggestion
            }]
          },
          diagnostics: [diagnostic]
        });
      }
    }

    return actions;
  }
}
```

---

## 关键决策清单

1. **是否使用 LSP？**
   - LSP：标准化，可复用
   - 内置：简单，性能好

2. **语法高亮方案？**
   - TextMate：成熟，资源丰富
   - Lezer：现代，性能好

3. **AI 如何集成？**
   - 补全
   - 诊断
   - 代码操作

4. **实时还是按需？**
   - 实时分析（性能压力）
   - 手动触发（用户体验）

---

## 参考资料

- [VSCode Language Extensions](https://code.visualstudio.com/api/language-extensions/overview)
- [Language Server Protocol](https://microsoft.github.io/language-server-protocol/)
- [CodeMirror 6 Extensions](https://codemirror.net/docs/extensions/)
- [Lezer Parser](https://lezer.codemirror.net/)
