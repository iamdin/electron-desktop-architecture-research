# 14. 编辑器引擎

> **核心问题**：用什么编辑器引擎？

---

## 概述

编辑器引擎是文本编辑应用的核心，决定了：
- 基础编辑能力
- 性能上限
- 扩展方式
- 开发成本

---

## 主流编辑器引擎对比

| 引擎 | 使用者 | 特点 |
|------|--------|------|
| **Monaco** | VSCode | 功能完整，重量级 |
| **CodeMirror 6** | Obsidian, Replit | 模块化，轻量 |
| **ProseMirror** | Notion, Confluence | 富文本，schema 驱动 |
| **Ace** | Cloud9, 旧版 | 成熟稳定 |
| **Lexical** | Facebook | React 优先 |

---

## Monaco Editor (VSCode)

### 特点

- 从 VSCode 提取的独立编辑器
- 功能完整：语法高亮、智能提示、多光标等
- TypeScript 原生支持
- 体积较大（~2MB）

### 基本使用

```typescript
import * as monaco from 'monaco-editor';

// 创建编辑器
const editor = monaco.editor.create(document.getElementById('container'), {
  value: 'function hello() {\n\treturn "world";\n}',
  language: 'javascript',
  theme: 'vs-dark',
  automaticLayout: true,
  minimap: { enabled: true },
  fontSize: 14,
  lineNumbers: 'on',
  wordWrap: 'on'
});

// 获取/设置内容
const content = editor.getValue();
editor.setValue('new content');

// 监听变化
editor.onDidChangeModelContent((event) => {
  console.log('Content changed:', event.changes);
});
```

### 语言支持

```typescript
// 注册语言
monaco.languages.register({ id: 'myLang' });

// 语法高亮（Monarch）
monaco.languages.setMonarchTokensProvider('myLang', {
  tokenizer: {
    root: [
      [/[a-zA-Z_]\w*/, 'identifier'],
      [/[0-9]+/, 'number'],
      [/".*?"/, 'string'],
      [/\/\/.*$/, 'comment'],
    ]
  }
});

// 补全
monaco.languages.registerCompletionItemProvider('myLang', {
  provideCompletionItems: (model, position) => {
    return {
      suggestions: [
        {
          label: 'hello',
          kind: monaco.languages.CompletionItemKind.Function,
          insertText: 'hello()',
          documentation: 'A hello function'
        }
      ]
    };
  }
});
```

### 装饰器

```typescript
// 行装饰
const decorations = editor.deltaDecorations([], [
  {
    range: new monaco.Range(1, 1, 1, 1),
    options: {
      isWholeLine: true,
      className: 'myLineDecoration',
      glyphMarginClassName: 'myGlyphMargin'
    }
  }
]);

// 内联装饰
const inlineDecorations = editor.deltaDecorations([], [
  {
    range: new monaco.Range(1, 5, 1, 10),
    options: {
      inlineClassName: 'myInlineDecoration',
      hoverMessage: { value: 'Hover message' }
    }
  }
]);
```

---

## CodeMirror 6 (Obsidian)

### 特点

- 模块化架构
- 状态驱动（不可变）
- 轻量（核心 ~50KB）
- 高度可扩展

### 基本使用

```typescript
import { EditorState } from '@codemirror/state';
import { EditorView, keymap, lineNumbers } from '@codemirror/view';
import { defaultKeymap } from '@codemirror/commands';
import { javascript } from '@codemirror/lang-javascript';

// 创建状态
const state = EditorState.create({
  doc: 'function hello() {\n  return "world";\n}',
  extensions: [
    lineNumbers(),
    keymap.of(defaultKeymap),
    javascript(),
  ]
});

// 创建视图
const view = new EditorView({
  state,
  parent: document.getElementById('container')
});

// 获取内容
const content = view.state.doc.toString();

// 修改内容（不可变，需要 dispatch）
view.dispatch({
  changes: { from: 0, to: view.state.doc.length, insert: 'new content' }
});
```

### 扩展系统

```typescript
import { Extension } from '@codemirror/state';
import { ViewPlugin, Decoration, DecorationSet } from '@codemirror/view';

// 自定义扩展
const myExtension: Extension = ViewPlugin.fromClass(
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
      // 构建装饰...
      return Decoration.set(widgets);
    }
  },
  { decorations: v => v.decorations }
);
```

### 状态管理

```typescript
import { StateField, StateEffect } from '@codemirror/state';

// 定义 Effect
const addHighlight = StateEffect.define<{ from: number; to: number }>();

// 定义 State Field
const highlightField = StateField.define<DecorationSet>({
  create() {
    return Decoration.none;
  },
  update(highlights, tr) {
    highlights = highlights.map(tr.changes);
    for (const effect of tr.effects) {
      if (effect.is(addHighlight)) {
        highlights = highlights.update({
          add: [highlightMark.range(effect.value.from, effect.value.to)]
        });
      }
    }
    return highlights;
  },
  provide: f => EditorView.decorations.from(f)
});

// 使用
view.dispatch({
  effects: addHighlight.of({ from: 0, to: 10 })
});
```

---

## 对比分析

### 编辑器引擎对比

| 方面 | Monaco | CodeMirror 6 |
|------|--------|--------------|
| **体积** | ~2MB | ~50KB (核心) |
| **架构** | 面向对象 | 函数式/不可变 |
| **状态管理** | 内部可变 | 不可变状态 |
| **扩展方式** | API 调用 | 组合扩展 |
| **语法高亮** | Monarch / TextMate | Lezer 解析器 |
| **性能** | 优秀 | 优秀 |
| **移动支持** | 一般 | 好 |
| **学习曲线** | 中等 | 陡峭 |
| **TypeScript** | 内置支持 | 需要扩展 |

### 选择建议

| 场景 | 推荐 |
|------|------|
| **IDE 类应用** | Monaco |
| **笔记/知识管理** | CodeMirror 6 |
| **富文本编辑** | ProseMirror |
| **轻量嵌入** | CodeMirror 6 |
| **需要 LSP** | Monaco |

---

## 对 AI Chat + Editor 应用的建议

### 编辑器选择

对于 AI Chat + Editor 应用，推荐 **CodeMirror 6**：

1. **轻量**：体积小，加载快
2. **模块化**：按需加载功能
3. **状态驱动**：易于与 React/Vue 集成
4. **扩展性**：AI 功能容易集成

### 封装层设计

```typescript
// 编辑器抽象接口
interface IEditor {
  // 内容操作
  getValue(): string;
  setValue(value: string): void;
  getSelection(): string;
  replaceSelection(text: string): void;
  insertText(text: string, position?: Position): void;

  // 光标操作
  getCursor(): Position;
  setCursor(position: Position): void;
  setSelection(from: Position, to: Position): void;

  // 装饰
  addDecoration(range: Range, options: DecorationOptions): string;
  removeDecoration(id: string): void;

  // 事件
  onDidChangeContent: Event<ContentChangeEvent>;
  onDidChangeCursor: Event<CursorChangeEvent>;
  onDidChangeSelection: Event<SelectionChangeEvent>;

  // 视图
  focus(): void;
  blur(): void;
  scrollToPosition(position: Position): void;

  // 生命周期
  dispose(): void;
}
```

### CodeMirror 6 封装

```typescript
class CodeMirrorEditor implements IEditor {
  private view: EditorView;
  private contentChangeEmitter = new EventEmitter<ContentChangeEvent>();

  constructor(container: HTMLElement, options: EditorOptions) {
    const state = EditorState.create({
      doc: options.initialValue || '',
      extensions: this.buildExtensions(options)
    });

    this.view = new EditorView({
      state,
      parent: container
    });
  }

  private buildExtensions(options: EditorOptions): Extension[] {
    return [
      // 基础
      lineNumbers(),
      highlightActiveLine(),
      highlightSelectionMatches(),

      // 语言
      options.language ? this.getLanguageExtension(options.language) : [],

      // 主题
      options.theme === 'dark' ? oneDark : [],

      // AI 相关扩展
      this.aiSuggestExtension(),
      this.aiDecorationsExtension(),

      // 变更监听
      EditorView.updateListener.of(update => {
        if (update.docChanged) {
          this.contentChangeEmitter.emit({
            changes: update.changes
          });
        }
      })
    ];
  }

  getValue(): string {
    return this.view.state.doc.toString();
  }

  setValue(value: string): void {
    this.view.dispatch({
      changes: {
        from: 0,
        to: this.view.state.doc.length,
        insert: value
      }
    });
  }

  insertText(text: string, position?: Position): void {
    const pos = position
      ? this.positionToOffset(position)
      : this.view.state.selection.main.head;

    this.view.dispatch({
      changes: { from: pos, insert: text }
    });
  }

  // AI 建议扩展
  private aiSuggestExtension(): Extension {
    return ViewPlugin.fromClass(
      class {
        // AI 内联建议实现
      }
    );
  }
}
```

### AI 功能集成

```typescript
// AI 内联补全
const aiCompletionExtension = EditorView.updateListener.of(async (update) => {
  if (update.docChanged) {
    const cursor = update.state.selection.main.head;
    const context = getCodeContext(update.state, cursor);

    // 请求 AI 补全
    const suggestion = await aiService.getCompletion(context);

    if (suggestion) {
      // 显示内联建议
      showInlineSuggestion(update.view, cursor, suggestion);
    }
  }
});

// AI 代码装饰（如错误提示）
const aiDiagnosticsExtension = StateField.define<DecorationSet>({
  create() {
    return Decoration.none;
  },
  update(decorations, tr) {
    // 根据 AI 分析结果更新装饰
    for (const effect of tr.effects) {
      if (effect.is(setDiagnostics)) {
        decorations = buildDiagnosticDecorations(effect.value);
      }
    }
    return decorations;
  }
});
```

---

## 关键决策清单

1. **选择哪个编辑器引擎？**
   - Monaco：功能完整但重
   - CodeMirror 6：轻量灵活

2. **是否需要语言服务？**
   - LSP 支持（Monaco 更好）
   - 简单语法高亮（都可以）

3. **如何与 AI 集成？**
   - 内联补全
   - 代码解释
   - 错误诊断

4. **状态如何管理？**
   - 编辑器内部状态
   - 外部状态同步

---

## 参考资料

- [Monaco Editor](https://microsoft.github.io/monaco-editor/)
- [CodeMirror 6](https://codemirror.net/6/)
- [ProseMirror](https://prosemirror.net/)
- [Lezer Parser](https://lezer.codemirror.net/)
