# 11. 命令系统

> **核心问题**：命令如何注册和执行？

---

## 概述

命令系统是用户与应用交互的核心：
- 通过命令面板快速访问功能
- 通过快捷键执行命令
- 通过菜单触发命令
- 插件通过命令暴露功能

---

## VSCode 的命令系统

### 命令注册

```json
// package.json
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
        },
        "enablement": "editorTextFocus"
      }
    ]
  }
}
```

```typescript
// 注册命令处理器
const disposable = vscode.commands.registerCommand(
  'myExtension.helloWorld',
  async (arg1: string, arg2: number) => {
    vscode.window.showInformationMessage(`Hello: ${arg1}, ${arg2}`);
    return 'result';  // 可以返回值
  }
);
context.subscriptions.push(disposable);
```

### 执行命令

```typescript
// 执行命令
await vscode.commands.executeCommand('myExtension.helloWorld', 'arg1', 42);

// 执行内置命令
await vscode.commands.executeCommand('vscode.open', uri);
await vscode.commands.executeCommand('editor.action.formatDocument');
await vscode.commands.executeCommand('workbench.action.showCommands');

// 获取返回值
const result = await vscode.commands.executeCommand<string>('myExtension.getData');
```

### 命令分类

```typescript
// 内置命令类别
'editor.*'       // 编辑器相关
'workbench.*'    // 工作台相关
'vscode.*'       // VSCode 核心
'debug.*'        // 调试相关
'git.*'          // Git 相关

// 扩展命令约定
'publisherId.extensionName.commandName'
```

### 带参数的命令

```typescript
// 定义命令参数
vscode.commands.registerCommand('myExtension.openFile', async (uri: vscode.Uri) => {
  if (uri) {
    await vscode.window.showTextDocument(uri);
  } else {
    // 无参数时提示用户选择
    const uris = await vscode.window.showOpenDialog();
    if (uris?.[0]) {
      await vscode.window.showTextDocument(uris[0]);
    }
  }
});

// 在菜单中使用（自动传参）
// package.json
{
  "menus": {
    "explorer/context": [{
      "command": "myExtension.openFile",
      "when": "resourceScheme == file"
    }]
  }
}
```

### TextEditor 命令

```typescript
// 只在编辑器可用时执行
vscode.commands.registerTextEditorCommand(
  'myExtension.insertDate',
  (textEditor, edit) => {
    const position = textEditor.selection.active;
    edit.insert(position, new Date().toISOString());
  }
);
```

---

## Obsidian 的命令系统

### 命令注册

```typescript
export default class MyPlugin extends Plugin {
  onload() {
    // 基础命令
    this.addCommand({
      id: 'my-command',
      name: 'My Command',
      callback: () => {
        new Notice('Command executed!');
      }
    });

    // 编辑器命令
    this.addCommand({
      id: 'insert-date',
      name: 'Insert Current Date',
      editorCallback: (editor: Editor, view: MarkdownView) => {
        editor.replaceSelection(new Date().toISOString());
      }
    });

    // 条件命令
    this.addCommand({
      id: 'conditional-command',
      name: 'Conditional Command',
      checkCallback: (checking: boolean) => {
        const view = this.app.workspace.getActiveViewOfType(MarkdownView);
        if (view) {
          if (!checking) {
            // 执行命令
            new Notice('Executed in Markdown view!');
          }
          return true;  // 命令可用
        }
        return false;  // 命令不可用
      }
    });

    // 带快捷键的命令
    this.addCommand({
      id: 'with-hotkey',
      name: 'Command With Hotkey',
      hotkeys: [{ modifiers: ['Mod', 'Shift'], key: 'p' }],
      callback: () => {}
    });
  }
}
```

### 执行命令

```typescript
// 通过 ID 执行
this.app.commands.executeCommandById('my-plugin:my-command');

// 获取所有命令
const commands = this.app.commands.commands;
for (const [id, command] of Object.entries(commands)) {
  console.log(id, command.name);
}
```

### 命令类型

```typescript
interface Command {
  id: string;
  name: string;
  icon?: string;

  // 三种回调类型（互斥）
  callback?: () => any;
  editorCallback?: (editor: Editor, view: MarkdownView) => any;
  checkCallback?: (checking: boolean) => boolean | void;

  // 默认快捷键
  hotkeys?: Hotkey[];
}

interface Hotkey {
  modifiers: Modifier[];  // 'Mod' | 'Ctrl' | 'Meta' | 'Shift' | 'Alt'
  key: string;
}
```

---

## 对比分析

### 命令系统对比

| 方面 | VSCode | Obsidian |
|------|--------|----------|
| **注册方式** | package.json + registerCommand | addCommand() |
| **命令 ID** | `publisher.extension.command` | `plugin-id:command-id` |
| **参数传递** | 支持任意参数 | 不支持（通过状态传递） |
| **返回值** | 支持 | 不支持 |
| **条件可用** | enablement（声明式） | checkCallback（代码） |
| **分类** | category 字段 | 无 |
| **图标** | 支持 | 支持（icon 字段） |

### 代码示例对比

**注册命令**：

```typescript
// VSCode
vscode.commands.registerCommand('myExt.hello', (name: string) => {
  return `Hello, ${name}`;
});

// Obsidian
this.addCommand({
  id: 'hello',
  name: 'Say Hello',
  callback: () => {
    new Notice('Hello!');
  }
});
```

**执行命令**：

```typescript
// VSCode
const result = await vscode.commands.executeCommand('myExt.hello', 'World');

// Obsidian
this.app.commands.executeCommandById('my-plugin:hello');
```

---

## 对 AI Chat + Editor 应用的建议

### 命令系统设计

```typescript
interface Command {
  /** 命令 ID */
  id: string;
  /** 显示名称 */
  name: string;
  /** 分类 */
  category?: string;
  /** 图标 */
  icon?: string;
  /** 描述 */
  description?: string;
  /** 默认快捷键 */
  keybinding?: Keybinding;
  /** 命令处理器 */
  handler: CommandHandler;
  /** 是否可用 */
  when?: CommandCondition;
}

type CommandHandler = (...args: any[]) => any | Promise<any>;
type CommandCondition = string | ((context: CommandContext) => boolean);

interface CommandContext {
  activeView: string | null;
  hasSelection: boolean;
  hasActiveEditor: boolean;
  hasActiveChat: boolean;
  // ...
}
```

### 命令注册 API

```typescript
interface CommandRegistry {
  /** 注册命令 */
  register(command: Command): Disposable;

  /** 批量注册 */
  registerMany(commands: Command[]): Disposable;

  /** 执行命令 */
  execute<T = any>(id: string, ...args: any[]): Promise<T>;

  /** 获取所有命令 */
  getCommands(): Command[];

  /** 获取可用命令 */
  getAvailableCommands(context: CommandContext): Command[];

  /** 监听命令执行 */
  onDidExecuteCommand: Event<{ id: string; args: any[] }>;
}
```

### AI 相关命令

```typescript
// 注册 AI 命令
context.commands.registerMany([
  {
    id: 'ai.sendMessage',
    name: 'Send Message',
    category: 'AI',
    keybinding: { key: 'Enter', modifiers: ['Mod'] },
    when: 'chatInputFocused',
    handler: () => chatService.sendCurrentMessage()
  },
  {
    id: 'ai.newChat',
    name: 'New Chat',
    category: 'AI',
    keybinding: { key: 'n', modifiers: ['Mod', 'Shift'] },
    handler: () => chatService.createNewChat()
  },
  {
    id: 'ai.insertCode',
    name: 'Insert Code to Editor',
    category: 'AI',
    when: 'chatHasCodeBlock && hasActiveEditor',
    handler: (code: string) => editorService.insertCode(code)
  },
  {
    id: 'ai.explainCode',
    name: 'Explain Selected Code',
    category: 'AI',
    keybinding: { key: 'e', modifiers: ['Mod', 'Shift'] },
    when: 'editorHasSelection',
    handler: async () => {
      const code = editorService.getSelection();
      await chatService.sendMessage(`Explain this code:\n\`\`\`\n${code}\n\`\`\``);
    }
  }
]);
```

### 命令面板

```typescript
class CommandPalette {
  private isOpen = false;
  private searchQuery = '';
  private filteredCommands: Command[] = [];

  open() {
    this.isOpen = true;
    this.searchQuery = '';
    this.updateFilteredCommands();
  }

  close() {
    this.isOpen = false;
  }

  setQuery(query: string) {
    this.searchQuery = query;
    this.updateFilteredCommands();
  }

  private updateFilteredCommands() {
    const context = this.getContext();
    const available = this.commandRegistry.getAvailableCommands(context);

    this.filteredCommands = available.filter(cmd =>
      this.matchesQuery(cmd, this.searchQuery)
    );
  }

  private matchesQuery(command: Command, query: string): boolean {
    const searchText = `${command.category || ''} ${command.name}`.toLowerCase();
    return fuzzyMatch(searchText, query.toLowerCase());
  }

  async executeSelected(command: Command) {
    this.close();
    await this.commandRegistry.execute(command.id);
  }
}
```

---

## 关键决策清单

1. **命令 ID 格式？**
   - `plugin:command`（Obsidian 风格）
   - `plugin.command`（VSCode 风格）

2. **是否支持参数？**
   - 支持（更灵活）
   - 不支持（更简单）

3. **条件如何表达？**
   - 声明式（when clause）
   - 代码式（checkCallback）
   - 混合

4. **命令面板功能？**
   - 基础搜索
   - 模糊匹配
   - 最近使用
   - 分类筛选

---

## 参考资料

- [VSCode Commands](https://code.visualstudio.com/api/extension-guides/command)
- [VSCode Built-in Commands](https://code.visualstudio.com/api/references/commands)
- [Obsidian Commands](https://docs.obsidian.md/Plugins/User+interface/Commands)
