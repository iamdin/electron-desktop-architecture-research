# 05. 扩展点机制

> **核心问题**：插件如何扩展宿主？

---

## 概述

扩展点（Extension Point / Contribution Point）是插件与宿主集成的接口：
- 插件在哪里注入功能
- 如何声明要提供的功能
- 宿主如何发现和加载这些功能

---

## VSCode 的扩展点机制

### 声明式 Contribution Points

VSCode 使用 `package.json` 声明扩展点，宿主在启动时解析：

```json
{
  "name": "my-extension",
  "contributes": {
    "commands": [...],
    "menus": {...},
    "keybindings": [...],
    "views": {...},
    "viewsContainers": {...},
    "configuration": {...},
    "languages": [...],
    "grammars": [...],
    "themes": [...],
    "snippets": [...],
    "debuggers": [...],
    "taskDefinitions": [...],
    "problemMatchers": [...],
    "customEditors": [...],
    "notebookRenderer": [...],
    "walkthroughs": [...],
    "...": "还有更多"
  }
}
```

### 主要扩展点详解

#### 1. Commands（命令）

```json
{
  "contributes": {
    "commands": [
      {
        "command": "myExtension.helloWorld",
        "title": "Hello World",
        "category": "My Extension",
        "icon": {
          "light": "resources/light/icon.svg",
          "dark": "resources/dark/icon.svg"
        }
      }
    ]
  }
}
```

```typescript
// 代码中注册处理器
vscode.commands.registerCommand('myExtension.helloWorld', () => {
  vscode.window.showInformationMessage('Hello World!');
});
```

#### 2. Menus（菜单）

```json
{
  "contributes": {
    "menus": {
      "editor/context": [
        {
          "command": "myExtension.helloWorld",
          "when": "editorTextFocus",
          "group": "navigation"
        }
      ],
      "editor/title": [
        {
          "command": "myExtension.helloWorld",
          "when": "resourceLangId == javascript"
        }
      ],
      "view/item/context": [
        {
          "command": "myExtension.deleteItem",
          "when": "view == myTreeView && viewItem == item"
        }
      ]
    }
  }
}
```

#### 3. Views（视图）

```json
{
  "contributes": {
    "viewsContainers": {
      "activitybar": [
        {
          "id": "myViewContainer",
          "title": "My Container",
          "icon": "resources/icon.svg"
        }
      ]
    },
    "views": {
      "myViewContainer": [
        {
          "id": "myTreeView",
          "name": "My Tree View",
          "icon": "resources/tree.svg",
          "contextualTitle": "My Tree"
        }
      ],
      "explorer": [
        {
          "id": "myExplorerView",
          "name": "My Explorer View"
        }
      ]
    }
  }
}
```

#### 4. Configuration（配置）

```json
{
  "contributes": {
    "configuration": {
      "title": "My Extension",
      "properties": {
        "myExtension.enable": {
          "type": "boolean",
          "default": true,
          "description": "Enable the extension"
        },
        "myExtension.maxItems": {
          "type": "number",
          "default": 10,
          "minimum": 1,
          "maximum": 100,
          "description": "Maximum items to display"
        },
        "myExtension.theme": {
          "type": "string",
          "enum": ["light", "dark", "auto"],
          "default": "auto",
          "description": "Color theme"
        }
      }
    }
  }
}
```

#### 5. Languages（语言支持）

```json
{
  "contributes": {
    "languages": [
      {
        "id": "mylang",
        "aliases": ["My Language", "mylang"],
        "extensions": [".mylang", ".ml"],
        "configuration": "./language-configuration.json"
      }
    ],
    "grammars": [
      {
        "language": "mylang",
        "scopeName": "source.mylang",
        "path": "./syntaxes/mylang.tmLanguage.json"
      }
    ]
  }
}
```

#### 6. Keybindings（快捷键）

```json
{
  "contributes": {
    "keybindings": [
      {
        "command": "myExtension.helloWorld",
        "key": "ctrl+shift+h",
        "mac": "cmd+shift+h",
        "when": "editorTextFocus"
      }
    ]
  }
}
```

### When Clause（条件表达式）

VSCode 使用 when clause 控制何时显示/启用：

```json
{
  "when": "editorTextFocus && resourceLangId == javascript"
}
```

支持的上下文变量：
```
editorTextFocus        - 编辑器获得焦点
editorHasSelection     - 有选中内容
resourceLangId         - 当前文件语言
resourceFilename       - 文件名
resourceExtname        - 扩展名
resourceScheme         - URI scheme
view                   - 当前视图 ID
viewItem               - 树视图项类型
config.xxx             - 配置值
isLinux/isMac/isWindows - 平台
```

支持的操作符：
```
==, !=                 - 等于/不等于
=~                     - 正则匹配
&&, ||, !              - 逻辑运算
in                     - 包含关系
```

### Activation Events（激活事件）

控制插件何时被激活：

```json
{
  "activationEvents": [
    "onLanguage:javascript",           // 打开特定语言文件
    "onCommand:myExtension.hello",     // 执行特定命令
    "onView:myTreeView",               // 视图可见
    "onUri",                           // URI 激活
    "onFileSystem:sftp",               // 文件系统 scheme
    "onDebug",                         // 调试启动
    "onStartupFinished",               // 启动完成后
    "workspaceContains:**/.myconfig",  // 工作区包含特定文件
    "*"                                // 立即激活（不推荐）
  ]
}
```

---

## Obsidian 的扩展点机制

### 命令式注册

Obsidian 使用代码注册，而非声明式：

```typescript
export default class MyPlugin extends Plugin {
  async onload() {
    // 注册命令
    this.addCommand({
      id: 'my-command',
      name: 'My Command',
      callback: () => { /* ... */ }
    });

    // 注册视图
    this.registerView('my-view', (leaf) => new MyView(leaf));

    // 注册设置页
    this.addSettingTab(new MySettingTab(this.app, this));

    // 注册编辑器扩展
    this.registerEditorExtension([myExtension]);

    // 注册 Markdown 后处理器
    this.registerMarkdownPostProcessor((el, ctx) => { /* ... */ });

    // 注册事件
    this.registerEvent(this.app.vault.on('create', () => { /* ... */ }));

    // 注册 DOM 事件
    this.registerDomEvent(document, 'click', () => { /* ... */ });

    // 注册协议处理
    this.registerObsidianProtocolHandler('my-action', (params) => { /* ... */ });

    // 注册文件扩展名
    this.registerExtensions(['myext'], 'markdown');
  }
}
```

### 主要扩展点详解

#### 1. Commands（命令）

```typescript
this.addCommand({
  id: 'insert-template',
  name: 'Insert Template',
  icon: 'document',  // Lucide 图标名
  hotkeys: [{ modifiers: ['Mod', 'Shift'], key: 't' }],
  editorCallback: (editor: Editor, view: MarkdownView) => {
    editor.replaceSelection('Template content');
  },
  // 或者使用 checkCallback 控制是否可用
  checkCallback: (checking: boolean) => {
    const view = this.app.workspace.getActiveViewOfType(MarkdownView);
    if (view) {
      if (!checking) {
        // 执行命令
      }
      return true;  // 命令可用
    }
    return false;  // 命令不可用
  }
});
```

#### 2. Views（视图）

```typescript
// 定义视图
class MyView extends ItemView {
  getViewType(): string {
    return 'my-view';
  }

  getDisplayText(): string {
    return 'My View';
  }

  getIcon(): string {
    return 'dice';
  }

  async onOpen() {
    const container = this.containerEl.children[1];
    container.empty();
    container.createEl('h1', { text: 'My View' });
  }

  async onClose() {
    // 清理
  }
}

// 注册视图
this.registerView('my-view', (leaf) => new MyView(leaf));

// 添加 Ribbon 图标打开视图
this.addRibbonIcon('dice', 'Open My View', () => {
  this.activateView();
});

// 激活视图
async activateView() {
  const { workspace } = this.app;
  let leaf = workspace.getLeavesOfType('my-view')[0];
  if (!leaf) {
    leaf = workspace.getRightLeaf(false);
    await leaf.setViewState({ type: 'my-view', active: true });
  }
  workspace.revealLeaf(leaf);
}
```

#### 3. Settings（设置）

```typescript
interface MyPluginSettings {
  mySetting: string;
  enableFeature: boolean;
}

const DEFAULT_SETTINGS: MyPluginSettings = {
  mySetting: 'default',
  enableFeature: true
};

class MySettingTab extends PluginSettingTab {
  plugin: MyPlugin;

  constructor(app: App, plugin: MyPlugin) {
    super(app, plugin);
    this.plugin = plugin;
  }

  display(): void {
    const { containerEl } = this;
    containerEl.empty();

    new Setting(containerEl)
      .setName('My Setting')
      .setDesc('Description of my setting')
      .addText(text => text
        .setPlaceholder('Enter value')
        .setValue(this.plugin.settings.mySetting)
        .onChange(async (value) => {
          this.plugin.settings.mySetting = value;
          await this.plugin.saveSettings();
        }));

    new Setting(containerEl)
      .setName('Enable Feature')
      .setDesc('Toggle this feature')
      .addToggle(toggle => toggle
        .setValue(this.plugin.settings.enableFeature)
        .onChange(async (value) => {
          this.plugin.settings.enableFeature = value;
          await this.plugin.saveSettings();
        }));
  }
}
```

#### 4. Editor Extensions（编辑器扩展）

```typescript
import { EditorView, ViewPlugin, Decoration, DecorationSet } from '@codemirror/view';

// CodeMirror 6 扩展
const myExtension = ViewPlugin.fromClass(
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
      // 构建装饰
      return Decoration.none;
    }
  },
  { decorations: v => v.decorations }
);

// 注册
this.registerEditorExtension([myExtension]);
```

#### 5. Markdown Post Processor（Markdown 后处理）

```typescript
// 处理渲染后的 Markdown
this.registerMarkdownPostProcessor((el, ctx) => {
  // 查找代码块
  const codeBlocks = el.querySelectorAll('pre > code');
  for (const codeBlock of codeBlocks) {
    if (codeBlock.className.includes('language-chart')) {
      // 替换为图表
      const chartEl = el.createDiv({ cls: 'my-chart' });
      renderChart(chartEl, codeBlock.textContent);
      codeBlock.parentElement?.replaceWith(chartEl);
    }
  }
});

// 代码块处理器
this.registerMarkdownCodeBlockProcessor('chart', (source, el, ctx) => {
  renderChart(el, source);
});
```

#### 6. Event Handlers（事件处理）

```typescript
// Vault 事件
this.registerEvent(this.app.vault.on('create', (file) => {}));
this.registerEvent(this.app.vault.on('modify', (file) => {}));
this.registerEvent(this.app.vault.on('delete', (file) => {}));
this.registerEvent(this.app.vault.on('rename', (file, oldPath) => {}));

// Workspace 事件
this.registerEvent(this.app.workspace.on('file-open', (file) => {}));
this.registerEvent(this.app.workspace.on('active-leaf-change', (leaf) => {}));
this.registerEvent(this.app.workspace.on('layout-change', () => {}));

// MetadataCache 事件
this.registerEvent(this.app.metadataCache.on('changed', (file) => {}));
this.registerEvent(this.app.metadataCache.on('resolved', () => {}));
```

---

## 对比分析

### 扩展点对比表

| 扩展点类型 | VSCode | Obsidian |
|-----------|--------|----------|
| **命令** | package.json + registerCommand | addCommand() |
| **菜单** | package.json menus | registerEvent + Menu |
| **视图** | package.json views + TreeDataProvider | registerView + ItemView |
| **设置** | package.json configuration | addSettingTab + PluginSettingTab |
| **快捷键** | package.json keybindings | command hotkeys |
| **语言** | package.json languages/grammars | N/A |
| **主题** | package.json themes | CSS 片段 |
| **代码片段** | package.json snippets | N/A |
| **编辑器** | languages.register* | registerEditorExtension |
| **Markdown** | N/A | registerMarkdownPostProcessor |

### 声明式 vs 命令式

| 方面 | 声明式 (VSCode) | 命令式 (Obsidian) |
|------|----------------|-------------------|
| **发现时机** | 启动时扫描 | 运行时注册 |
| **性能** | 延迟加载支持好 | 需要立即执行 |
| **灵活性** | 受限（JSON 表达能力） | 完全灵活 |
| **类型安全** | JSON Schema 验证 | TypeScript 检查 |
| **工具支持** | IDE 补全、验证 | 代码补全 |
| **动态性** | 静态声明 | 可动态注册/注销 |

### 条件控制对比

**VSCode when clause**：
```json
{
  "when": "editorTextFocus && resourceLangId == javascript && config.myExt.enabled"
}
```

**Obsidian checkCallback**：
```typescript
this.addCommand({
  id: 'my-command',
  name: 'My Command',
  checkCallback: (checking) => {
    const view = this.app.workspace.getActiveViewOfType(MarkdownView);
    const isEnabled = this.settings.enabled;
    if (view && isEnabled) {
      if (!checking) {
        // 执行
      }
      return true;
    }
    return false;
  }
});
```

---

## 对 Coding Agent Desktop 应用的建议

### 混合方案：声明 + 注册

```typescript
// manifest.json - 静态声明
{
  "name": "my-plugin",
  "contributes": {
    // 静态的、不常变的
    "commands": [
      {
        "id": "my-plugin.hello",
        "title": "Hello",
        "category": "My Plugin"
      }
    ],
    "configuration": {
      "properties": {
        "myPlugin.enabled": {
          "type": "boolean",
          "default": true
        }
      }
    },
    // 激活条件
    "activationEvents": [
      "onCommand:my-plugin.hello"
    ]
  }
}
```

```typescript
// plugin.ts - 动态注册
export default class MyPlugin implements Plugin {
  async onActivate(context: PluginContext) {
    // 注册命令处理器
    context.registerCommand('my-plugin.hello', () => {
      // ...
    });

    // 动态注册视图（基于配置）
    if (this.settings.showView) {
      context.registerView('my-view', () => new MyView());
    }

    // 动态注册菜单项
    context.registerMenuItem({
      location: 'chat/message',
      label: 'Copy to Editor',
      condition: (ctx) => ctx.message.role === 'assistant',
      action: (ctx) => this.copyToEditor(ctx.message)
    });
  }
}
```

### 推荐的扩展点

```typescript
// 适合 Coding Agent Desktop 的扩展点

interface ContributionPoints {
  // 1. 命令
  commands: CommandContribution[];

  // 2. 菜单位置
  menus: {
    'chat/input': MenuContribution[];      // 聊天输入框
    'chat/message': MenuContribution[];    // 消息上下文
    'editor/context': MenuContribution[];  // 编辑器右键
    'sidebar/header': MenuContribution[];  // 侧边栏头部
  };

  // 3. 视图
  views: {
    'sidebar': ViewContribution[];         // 侧边栏视图
    'panel': ViewContribution[];           // 底部面板
    'editor': ViewContribution[];          // 编辑器标签页
  };

  // 4. AI 相关
  ai: {
    'prompts': PromptContribution[];       // 预设提示词
    'models': ModelContribution[];         // 自定义模型
    'tools': ToolContribution[];           // AI 工具/函数
  };

  // 5. 编辑器
  editor: {
    'completions': CompletionContribution[];  // 补全
    'actions': CodeActionContribution[];      // 代码操作
    'decorations': DecorationContribution[];  // 装饰
  };

  // 6. 配置
  configuration: ConfigurationContribution;

  // 7. 快捷键
  keybindings: KeybindingContribution[];
}
```

### 激活策略

```typescript
interface ActivationEvents {
  // 启动相关
  'onStartup': void;                    // 应用启动
  'onStartupFinished': void;            // 启动完成

  // 命令相关
  'onCommand': string;                  // 执行命令时

  // 视图相关
  'onView': string;                     // 视图可见时

  // 文件相关
  'onFileOpen': string;                 // 打开特定类型文件
  'workspaceContains': string;          // 工作区包含文件

  // AI 相关
  'onChat': void;                       // 发起聊天时
  'onChatMessage': string;              // 特定类型消息

  // 编辑器相关
  'onLanguage': string;                 // 特定语言文件
}

// manifest.json
{
  "activationEvents": [
    "onCommand:my-plugin.activate",
    "onChat",
    "onLanguage:markdown"
  ]
}
```

### 扩展点注册 API

```typescript
// context.ts
interface PluginContext {
  // 命令
  registerCommand(id: string, handler: CommandHandler): Disposable;

  // 菜单
  registerMenuItem(item: MenuItem): Disposable;

  // 视图
  registerView(id: string, factory: ViewFactory): Disposable;

  // AI 工具
  registerAITool(tool: AITool): Disposable;

  // 编辑器扩展
  registerCompletionProvider(provider: CompletionProvider): Disposable;
  registerCodeActionProvider(provider: CodeActionProvider): Disposable;

  // 事件
  onEvent<T>(event: string, handler: (data: T) => void): Disposable;

  // 配置
  getConfiguration<T>(section: string): T;
  onConfigurationChanged(handler: (e: ConfigChangeEvent) => void): Disposable;
}
```

---

## 关键决策清单

1. **声明式还是命令式？**
   - 静态信息用声明式（manifest.json）
   - 动态行为用命令式（代码注册）

2. **需要哪些扩展点？**
   - 根据产品需求确定
   - 开始时可以少，逐步扩展

3. **激活策略是什么？**
   - 立即激活 vs 按需激活
   - 影响启动性能

4. **条件控制如何实现？**
   - when clause（表达式）
   - checkCallback（函数）
   - 混合使用

5. **扩展点如何版本化？**
   - 新增扩展点不影响旧插件
   - 废弃扩展点要有迁移路径

---

## 参考资料

- [VSCode Contribution Points](https://code.visualstudio.com/api/references/contribution-points)
- [VSCode Activation Events](https://code.visualstudio.com/api/references/activation-events)
- [VSCode When Clause Contexts](https://code.visualstudio.com/api/references/when-clause-contexts)
- [Obsidian Plugin API](https://docs.obsidian.md/Plugins/Getting+started/Build+a+plugin)
