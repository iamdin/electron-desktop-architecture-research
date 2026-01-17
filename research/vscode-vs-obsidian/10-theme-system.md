# 10. 主题系统

> **核心问题**：主题如何实现？

---

## 概述

主题系统决定了：
- 应用的视觉定制能力
- 插件如何适配主题
- 用户如何切换和自定义主题
- 暗色/亮色模式如何处理

---

## VSCode 的主题系统

### 主题类型

| 类型 | 说明 |
|------|------|
| **Color Theme** | 编辑器和 UI 颜色 |
| **File Icon Theme** | 文件图标 |
| **Product Icon Theme** | UI 图标 |

### Color Theme 结构

```json
// themes/my-theme.json
{
  "name": "My Theme",
  "type": "dark",  // dark | light | hc (高对比度)
  "colors": {
    // UI 颜色
    "editor.background": "#1e1e1e",
    "editor.foreground": "#d4d4d4",
    "activityBar.background": "#333333",
    "sideBar.background": "#252526",
    "statusBar.background": "#007acc",
    // ... 600+ 颜色变量
  },
  "tokenColors": [
    // 语法高亮颜色（TextMate 格式）
    {
      "scope": "comment",
      "settings": {
        "foreground": "#6A9955"
      }
    },
    {
      "scope": ["string", "string.quoted"],
      "settings": {
        "foreground": "#CE9178"
      }
    },
    {
      "scope": "keyword",
      "settings": {
        "foreground": "#569CD6"
      }
    }
  ],
  "semanticHighlighting": true,
  "semanticTokenColors": {
    "variable.readonly": "#4FC1FF"
  }
}
```

### package.json 声明

```json
{
  "contributes": {
    "themes": [
      {
        "label": "My Dark Theme",
        "uiTheme": "vs-dark",
        "path": "./themes/my-dark-theme.json"
      },
      {
        "label": "My Light Theme",
        "uiTheme": "vs",
        "path": "./themes/my-light-theme.json"
      }
    ]
  }
}
```

### 颜色变量分类

```typescript
// 编辑器相关
'editor.background'
'editor.foreground'
'editor.lineHighlightBackground'
'editor.selectionBackground'
'editorCursor.foreground'
'editorLineNumber.foreground'

// 工作台相关
'activityBar.background'
'sideBar.background'
'statusBar.background'
'titleBar.activeBackground'
'panel.background'

// 组件相关
'button.background'
'input.background'
'dropdown.background'
'list.activeSelectionBackground'

// 状态相关
'errorForeground'
'warningForeground'
'textLink.foreground'
```

### 在代码中使用主题色

```typescript
// Webview 中获取主题色
function getWebviewContent(): string {
  return `
    <style>
      body {
        color: var(--vscode-editor-foreground);
        background: var(--vscode-editor-background);
      }
      button {
        background: var(--vscode-button-background);
        color: var(--vscode-button-foreground);
      }
      a {
        color: var(--vscode-textLink-foreground);
      }
    </style>
  `;
}

// 扩展代码中获取颜色
const config = vscode.workspace.getConfiguration();
const editorBackground = config.get('workbench.colorCustomizations')['editor.background'];
```

---

## Obsidian 的主题系统

### 主题结构

Obsidian 主题是纯 CSS 文件：

```css
/* .obsidian/themes/my-theme.css */

.theme-dark {
  --background-primary: #1e1e1e;
  --background-secondary: #252526;
  --text-normal: #d4d4d4;
  --text-muted: #6A9955;
  --text-accent: #569CD6;
  --interactive-accent: #007acc;
}

.theme-light {
  --background-primary: #ffffff;
  --background-secondary: #f3f3f3;
  --text-normal: #1e1e1e;
  --text-muted: #6A9955;
  --text-accent: #0066cc;
  --interactive-accent: #007acc;
}

/* 组件样式 */
.workspace-leaf {
  background: var(--background-primary);
}

.markdown-preview-view {
  color: var(--text-normal);
}

/* 自定义样式 */
.my-plugin-container {
  background: var(--background-secondary);
  border: 1px solid var(--background-modifier-border);
}
```

### CSS 变量

```css
/* 主要变量（约 200+） */

/* 背景 */
--background-primary: ...;
--background-secondary: ...;
--background-modifier-border: ...;
--background-modifier-hover: ...;

/* 文字 */
--text-normal: ...;
--text-muted: ...;
--text-faint: ...;
--text-accent: ...;

/* 交互 */
--interactive-normal: ...;
--interactive-hover: ...;
--interactive-accent: ...;
--interactive-accent-hover: ...;

/* 编辑器 */
--code-normal: ...;
--code-comment: ...;
--code-function: ...;
--code-keyword: ...;
```

### CSS 片段

用户可以添加自定义 CSS 片段：

```css
/* .obsidian/snippets/my-customization.css */

/* 自定义标题样式 */
.markdown-preview-view h1 {
  color: var(--text-accent);
  border-bottom: 2px solid var(--interactive-accent);
}

/* 自定义代码块 */
.markdown-preview-view pre {
  background: var(--background-secondary);
  border-radius: 8px;
}
```

### 插件适配主题

```typescript
// 使用 CSS 变量
class MyView extends ItemView {
  async onOpen() {
    this.containerEl.style.setProperty('--my-color', 'var(--text-accent)');

    // 或直接使用变量
    const container = this.containerEl.createDiv({ cls: 'my-container' });
  }
}

// CSS
.my-container {
  background: var(--background-primary);
  color: var(--text-normal);
  border: 1px solid var(--background-modifier-border);
}

.my-container:hover {
  background: var(--background-modifier-hover);
}
```

---

## 对比分析

### 主题系统对比

| 方面 | VSCode | Obsidian |
|------|--------|----------|
| **格式** | JSON (颜色) + TextMate (语法) | CSS |
| **变量数量** | 600+ | 200+ |
| **语法高亮** | TextMate / Semantic Tokens | CSS (CodeMirror) |
| **图标主题** | 支持 | 不支持 |
| **继承** | 支持（从基础主题继承） | 不支持 |
| **动态切换** | 需重载 | 立即生效 |

### 开发体验对比

| 方面 | VSCode | Obsidian |
|------|--------|----------|
| **学习曲线** | 陡峭（变量多） | 平缓 |
| **调试** | 需要重载 | 实时预览 |
| **工具支持** | Theme Color 文档 | 社区工具 |
| **灵活性** | 中（JSON 限制） | 高（CSS 完全自由） |

---

## 对 AI Chat + Editor 应用的建议

### 主题系统设计

```typescript
// 主题配置
interface ThemeConfig {
  name: string;
  type: 'light' | 'dark';
  colors: ThemeColors;
}

interface ThemeColors {
  // 基础色
  background: {
    primary: string;
    secondary: string;
    tertiary: string;
  };
  foreground: {
    primary: string;
    secondary: string;
    muted: string;
  };
  accent: {
    primary: string;
    secondary: string;
  };
  border: {
    default: string;
    focus: string;
  };

  // 语义色
  semantic: {
    success: string;
    warning: string;
    error: string;
    info: string;
  };

  // AI 相关
  ai: {
    userMessage: string;
    assistantMessage: string;
    systemMessage: string;
    codeBlock: string;
  };

  // 编辑器
  editor: {
    background: string;
    lineNumber: string;
    selection: string;
    cursor: string;
  };
}
```

### CSS 变量生成

```typescript
class ThemeManager {
  private currentTheme: ThemeConfig;

  applyTheme(theme: ThemeConfig) {
    this.currentTheme = theme;
    const root = document.documentElement;

    // 生成 CSS 变量
    root.style.setProperty('--bg-primary', theme.colors.background.primary);
    root.style.setProperty('--bg-secondary', theme.colors.background.secondary);
    root.style.setProperty('--fg-primary', theme.colors.foreground.primary);
    root.style.setProperty('--accent-primary', theme.colors.accent.primary);

    // AI 相关
    root.style.setProperty('--ai-user-msg', theme.colors.ai.userMessage);
    root.style.setProperty('--ai-assistant-msg', theme.colors.ai.assistantMessage);

    // 设置类名
    root.classList.remove('theme-light', 'theme-dark');
    root.classList.add(`theme-${theme.type}`);

    this.emit('themeChanged', theme);
  }
}
```

### 使用示例

```css
/* 全局样式 */
:root {
  /* 由 ThemeManager 设置 */
}

.app-container {
  background: var(--bg-primary);
  color: var(--fg-primary);
}

/* 聊天消息 */
.chat-message.user {
  background: var(--ai-user-msg);
}

.chat-message.assistant {
  background: var(--ai-assistant-msg);
}

/* 暗色模式特殊处理 */
.theme-dark .code-block {
  filter: brightness(0.9);
}
```

### 插件主题适配

```typescript
// 插件应使用 CSS 变量
class MyPluginView extends BaseView {
  onMount(container: HTMLElement) {
    container.innerHTML = `
      <div class="my-plugin">
        <h1 style="color: var(--fg-primary)">Title</h1>
        <p style="color: var(--fg-secondary)">Content</p>
      </div>
    `;
  }
}

// 或提供 CSS 文件
// my-plugin.css
.my-plugin {
  background: var(--bg-secondary);
  border: 1px solid var(--border-default);
  border-radius: 8px;
  padding: 16px;
}

.my-plugin h1 {
  color: var(--fg-primary);
  margin-bottom: 8px;
}
```

---

## 关键决策清单

1. **主题格式？**
   - JSON（结构化，易验证）
   - CSS（灵活，易调试）
   - 混合（JSON 定义变量，CSS 补充）

2. **变量命名规范？**
   - 语义化（primary, secondary）
   - 功能化（editor.background）
   - 混合使用

3. **暗色/亮色模式？**
   - 跟随系统
   - 用户手动选择
   - 定时切换

4. **插件如何适配主题？**
   - 强制使用 CSS 变量
   - 提供主题 API
   - 文档规范

---

## 参考资料

- [VSCode Color Theme](https://code.visualstudio.com/api/extension-guides/color-theme)
- [VSCode Theme Color Reference](https://code.visualstudio.com/api/references/theme-color)
- [Obsidian CSS Variables](https://docs.obsidian.md/Reference/CSS+variables)
