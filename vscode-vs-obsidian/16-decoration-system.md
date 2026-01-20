# 16. 装饰器系统

> **核心问题**：编辑器装饰如何实现？

---

## 概述

装饰器系统用于在编辑器中添加视觉效果：
- 语法高亮
- 错误/警告标记
- 搜索结果高亮
- AI 建议显示
- 代码覆盖率

---

## VSCode 的装饰系统

### 装饰类型

```typescript
// 创建装饰类型
const errorDecorationType = vscode.window.createTextEditorDecorationType({
  // 背景色
  backgroundColor: 'rgba(255, 0, 0, 0.3)',
  // 边框
  border: '1px solid red',
  borderRadius: '2px',
  // 文字样式
  color: '#ff6b6b',
  fontWeight: 'bold',
  // 下划线
  textDecoration: 'underline wavy red',
  // 边距装饰
  gutterIconPath: '/path/to/icon.svg',
  gutterIconSize: '80%',
  // 概览标尺
  overviewRulerColor: 'red',
  overviewRulerLane: vscode.OverviewRulerLane.Right,
  // 整行高亮
  isWholeLine: false
});

const warningDecorationType = vscode.window.createTextEditorDecorationType({
  backgroundColor: 'rgba(255, 255, 0, 0.2)',
  textDecoration: 'underline wavy yellow'
});
```

### 应用装饰

```typescript
const editor = vscode.window.activeTextEditor;
if (editor) {
  // 定义装饰范围
  const decorations: vscode.DecorationOptions[] = [
    {
      range: new vscode.Range(0, 0, 0, 10),
      hoverMessage: 'Error: Variable not defined',
      renderOptions: {
        after: {
          contentText: ' ← Error here',
          color: 'red'
        }
      }
    },
    {
      range: new vscode.Range(2, 5, 2, 15),
      hoverMessage: 'Warning: Unused variable'
    }
  ];

  // 应用装饰
  editor.setDecorations(errorDecorationType, decorations);
}
```

### 动态更新

```typescript
class DecorationManager {
  private decorationType: vscode.TextEditorDecorationType;
  private activeEditor: vscode.TextEditor | undefined;

  constructor() {
    this.decorationType = vscode.window.createTextEditorDecorationType({
      backgroundColor: 'rgba(100, 100, 255, 0.2)'
    });

    // 监听编辑器变化
    vscode.window.onDidChangeActiveTextEditor(editor => {
      this.activeEditor = editor;
      this.updateDecorations();
    });

    // 监听内容变化
    vscode.workspace.onDidChangeTextDocument(event => {
      if (this.activeEditor && event.document === this.activeEditor.document) {
        this.updateDecorations();
      }
    });
  }

  private updateDecorations() {
    if (!this.activeEditor) return;

    const text = this.activeEditor.document.getText();
    const decorations: vscode.DecorationOptions[] = [];

    // 高亮所有 TODO
    const todoRegex = /TODO:.*$/gm;
    let match;
    while ((match = todoRegex.exec(text))) {
      const startPos = this.activeEditor.document.positionAt(match.index);
      const endPos = this.activeEditor.document.positionAt(match.index + match[0].length);
      decorations.push({ range: new vscode.Range(startPos, endPos) });
    }

    this.activeEditor.setDecorations(this.decorationType, decorations);
  }

  dispose() {
    this.decorationType.dispose();
  }
}
```

### 内联装饰（After/Before）

```typescript
const inlineHintType = vscode.window.createTextEditorDecorationType({
  after: {
    margin: '0 0 0 1em',
    textDecoration: 'none; opacity: 0.5;'
  }
});

// 类型提示风格
editor.setDecorations(inlineHintType, [
  {
    range: new vscode.Range(5, 20, 5, 20),
    renderOptions: {
      after: {
        contentText: ': string',
        color: '#888'
      }
    }
  }
]);
```

---

## CodeMirror 6 的装饰系统

### 装饰类型

```typescript
import { Decoration, DecorationSet, EditorView, ViewPlugin, WidgetType } from '@codemirror/view';

// Mark 装饰（样式）
const highlightMark = Decoration.mark({
  class: 'cm-highlight',
  attributes: { 'data-type': 'highlight' }
});

// Line 装饰（整行）
const lineHighlight = Decoration.line({
  class: 'cm-active-line'
});

// Widget 装饰（插入 DOM）
class PlaceholderWidget extends WidgetType {
  constructor(readonly text: string) { super(); }

  toDOM() {
    const span = document.createElement('span');
    span.className = 'cm-placeholder';
    span.textContent = this.text;
    return span;
  }

  ignoreEvent() { return false; }
}

const placeholderDeco = Decoration.widget({
  widget: new PlaceholderWidget('Type here...'),
  side: 1
});
```

### 状态驱动装饰

```typescript
import { StateField, StateEffect } from '@codemirror/state';

// 定义 Effect
const addHighlight = StateEffect.define<{ from: number; to: number }>();
const clearHighlights = StateEffect.define();

// 定义 StateField
const highlightField = StateField.define<DecorationSet>({
  create() {
    return Decoration.none;
  },

  update(decorations, tr) {
    // 映射位置变化
    decorations = decorations.map(tr.changes);

    for (const effect of tr.effects) {
      if (effect.is(addHighlight)) {
        decorations = decorations.update({
          add: [highlightMark.range(effect.value.from, effect.value.to)]
        });
      } else if (effect.is(clearHighlights)) {
        decorations = Decoration.none;
      }
    }

    return decorations;
  },

  // 提供装饰到视图
  provide: f => EditorView.decorations.from(f)
});

// 使用
view.dispatch({
  effects: addHighlight.of({ from: 10, to: 20 })
});
```

### ViewPlugin 装饰

```typescript
const searchHighlighter = ViewPlugin.fromClass(
  class {
    decorations: DecorationSet;

    constructor(view: EditorView) {
      this.decorations = this.highlight(view);
    }

    update(update: ViewUpdate) {
      if (update.docChanged || update.viewportChanged || this.searchTermChanged(update)) {
        this.decorations = this.highlight(update.view);
      }
    }

    highlight(view: EditorView): DecorationSet {
      const searchTerm = getSearchTerm();  // 从某处获取搜索词
      if (!searchTerm) return Decoration.none;

      const decorations: Range<Decoration>[] = [];
      const doc = view.state.doc.toString();

      let index = 0;
      while ((index = doc.indexOf(searchTerm, index)) !== -1) {
        decorations.push(
          Decoration.mark({ class: 'cm-search-match' })
            .range(index, index + searchTerm.length)
        );
        index += searchTerm.length;
      }

      return Decoration.set(decorations);
    }
  },
  { decorations: v => v.decorations }
);
```

### Obsidian 中的使用

```typescript
// 在 Obsidian 插件中注册 CM6 扩展
export default class MyPlugin extends Plugin {
  onload() {
    this.registerEditorExtension([
      highlightField,
      searchHighlighter,
      // 其他扩展
    ]);
  }
}

// 通过 Editor 接口操作
this.registerEvent(
  this.app.workspace.on('editor-change', (editor) => {
    // 获取 EditorView
    // @ts-ignore
    const view = editor.cm as EditorView;

    // 添加装饰
    view.dispatch({
      effects: addHighlight.of({ from: 0, to: 10 })
    });
  })
);
```

---

## 对比分析

### 装饰系统对比

| 方面 | VSCode | CodeMirror 6 |
|------|--------|--------------|
| **装饰类型** | mark, line, gutter, overview | mark, line, widget |
| **状态管理** | 外部管理 | StateField 驱动 |
| **更新方式** | setDecorations 重置 | 增量更新 |
| **性能** | 重置全部 | 只更新变化 |
| **交互装饰** | renderOptions.after | WidgetType |
| **样式定义** | 创建时定义 | Decoration + CSS |

---

## 对 Coding Agent Desktop 应用的建议

### 统一装饰接口

```typescript
interface Decoration {
  /** 范围 */
  range: Range;
  /** 装饰类型 */
  type: DecorationType;
  /** 样式 */
  style?: DecorationStyle;
  /** 悬停内容 */
  hoverContent?: string | MarkdownString;
  /** Widget（交互装饰） */
  widget?: DecorationWidget;
}

type DecorationType = 'highlight' | 'error' | 'warning' | 'info' | 'ai-suggestion' | 'custom';

interface DecorationStyle {
  backgroundColor?: string;
  color?: string;
  border?: string;
  textDecoration?: string;
  opacity?: number;
}

interface DecorationWidget {
  render(): HTMLElement;
  destroy?(): void;
}
```

### AI 建议装饰

```typescript
// AI 内联建议
class AISuggestionWidget implements DecorationWidget {
  constructor(
    private suggestion: string,
    private onAccept: () => void,
    private onReject: () => void
  ) {}

  render(): HTMLElement {
    const container = document.createElement('span');
    container.className = 'ai-suggestion';
    container.textContent = this.suggestion;

    // Tab 接受
    container.tabIndex = 0;
    container.addEventListener('keydown', (e) => {
      if (e.key === 'Tab') {
        e.preventDefault();
        this.onAccept();
      } else if (e.key === 'Escape') {
        this.onReject();
      }
    });

    return container;
  }
}

// 使用
class AIInlineSuggestion {
  private currentSuggestion: Decoration | null = null;

  async showSuggestion(editor: IEditor, position: Position) {
    // 获取 AI 建议
    const suggestion = await this.aiService.getInlineSuggestion({
      document: editor.getDocument(),
      position
    });

    if (suggestion) {
      this.currentSuggestion = editor.addDecoration({
        range: { start: position, end: position },
        type: 'ai-suggestion',
        style: { opacity: 0.5 },
        widget: new AISuggestionWidget(
          suggestion.text,
          () => this.acceptSuggestion(editor, position, suggestion.text),
          () => this.dismissSuggestion(editor)
        )
      });
    }
  }

  private acceptSuggestion(editor: IEditor, position: Position, text: string) {
    editor.insertText(text, position);
    this.dismissSuggestion(editor);
  }

  private dismissSuggestion(editor: IEditor) {
    if (this.currentSuggestion) {
      editor.removeDecoration(this.currentSuggestion.id);
      this.currentSuggestion = null;
    }
  }
}
```

### 错误/警告装饰

```typescript
class DiagnosticDecorations {
  private decorations: Map<string, string> = new Map();  // uri -> decorationId

  updateDiagnostics(editor: IEditor, diagnostics: Diagnostic[]) {
    // 清除旧装饰
    const existingId = this.decorations.get(editor.uri);
    if (existingId) {
      editor.removeDecoration(existingId);
    }

    // 添加新装饰
    const decos = diagnostics.map(d => ({
      range: d.range,
      type: d.severity === 'error' ? 'error' : 'warning',
      style: {
        textDecoration: `underline wavy ${d.severity === 'error' ? 'red' : 'yellow'}`
      },
      hoverContent: d.message
    }));

    const id = editor.addDecorations(decos);
    this.decorations.set(editor.uri, id);
  }
}
```

---

## 关键决策清单

1. **装饰如何存储？**
   - 外部管理（简单）
   - StateField（性能好）

2. **更新策略？**
   - 全量更新（简单）
   - 增量更新（性能好）

3. **交互装饰如何实现？**
   - Widget
   - 事件委托

4. **AI 建议如何显示？**
   - 内联 ghost text
   - 弹窗
   - 侧边提示

---

## 参考资料

- [VSCode Decorations](https://code.visualstudio.com/api/references/vscode-api#TextEditorDecorationType)
- [CodeMirror 6 Decorations](https://codemirror.net/docs/ref/#view.Decoration)
- [GitHub Copilot 交互设计](https://github.blog/2022-11-09-how-github-copilot-helps-developers-write-better-code/)
