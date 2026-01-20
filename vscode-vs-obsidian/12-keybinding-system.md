# 12. 快捷键系统

> **核心问题**：快捷键如何绑定？

---

## 概述

快捷键系统决定了：
- 如何定义默认快捷键
- 用户如何自定义快捷键
- 快捷键冲突如何处理
- 上下文条件如何判断

---

## VSCode 的快捷键系统

### 声明式绑定

```json
// package.json
{
  "contributes": {
    "keybindings": [
      {
        "command": "myExtension.helloWorld",
        "key": "ctrl+shift+h",
        "mac": "cmd+shift+h",
        "when": "editorTextFocus"
      },
      {
        "command": "myExtension.openPanel",
        "key": "ctrl+shift+p",
        "when": "!panelFocus"
      }
    ]
  }
}
```

### 键序列（Chord）

```json
{
  "keybindings": [
    {
      "command": "myExtension.action1",
      "key": "ctrl+k ctrl+1"
    },
    {
      "command": "myExtension.action2",
      "key": "ctrl+k ctrl+2"
    }
  ]
}
```

### When Clause

```json
{
  "when": "editorTextFocus && !editorReadonly && resourceLangId == javascript"
}
```

常用上下文变量：

```
editorTextFocus         - 编辑器获得焦点
editorHasSelection      - 有选中内容
editorReadonly          - 只读模式
resourceLangId          - 文件语言
resourceExtname         - 文件扩展名
panelFocus              - 面板获得焦点
sideBarFocus            - 侧边栏获得焦点
inQuickOpen             - 快速打开面板
suggestWidgetVisible    - 建议列表可见
terminalFocus           - 终端获得焦点
```

### 用户自定义

```json
// keybindings.json
[
  {
    "key": "ctrl+shift+n",
    "command": "myExtension.newFile",
    "when": "explorerViewletFocus"
  },
  {
    "key": "ctrl+shift+h",
    "command": "-myExtension.helloWorld"  // 移除绑定
  }
]
```

### 代码中监听

```typescript
// 通常不需要直接监听按键
// 而是通过命令系统

// 如果需要自定义处理
vscode.commands.registerCommand('type', (args) => {
  // 拦截输入
  const char = args.text;
  if (char === '{') {
    // 自动补全 }
    vscode.commands.executeCommand('default:type', { text: '{}' });
    vscode.commands.executeCommand('cursorLeft');
    return;
  }
  vscode.commands.executeCommand('default:type', args);
});
```

---

## Obsidian 的快捷键系统

### 命令绑定

```typescript
this.addCommand({
  id: 'my-command',
  name: 'My Command',
  hotkeys: [
    { modifiers: ['Mod', 'Shift'], key: 'h' }
  ],
  callback: () => {}
});
```

### 修饰键

```typescript
type Modifier = 'Mod' | 'Ctrl' | 'Meta' | 'Shift' | 'Alt';

// Mod 会自动映射：
// - macOS: Cmd
// - Windows/Linux: Ctrl
```

### 用户自定义

用户通过设置 > 快捷键 界面配置，存储在：

```json
// .obsidian/hotkeys.json
{
  "my-plugin:my-command": [
    { "modifiers": ["Mod", "Shift"], "key": "H" }
  ]
}
```

### 监听键盘事件

```typescript
// 在特定视图中监听
class MyView extends ItemView {
  onOpen() {
    this.containerEl.addEventListener('keydown', (evt) => {
      if (evt.key === 'Escape') {
        this.close();
      }
    });
  }
}

// 全局监听（不推荐）
this.registerDomEvent(document, 'keydown', (evt) => {
  // 处理
});
```

---

## 对比分析

### 快捷键系统对比

| 方面 | VSCode | Obsidian |
|------|--------|----------|
| **定义方式** | package.json + keybindings.json | addCommand hotkeys |
| **键序列** | 支持（ctrl+k ctrl+c） | 不支持 |
| **条件** | when clause（声明式） | 无（通过 checkCallback） |
| **平台差异** | key + mac 字段 | Mod 自动映射 |
| **冲突提示** | 有 | 无 |
| **用户配置** | keybindings.json | 设置界面 |

### 代码示例对比

**定义快捷键**：

```typescript
// VSCode (package.json)
{
  "keybindings": [{
    "command": "myExt.action",
    "key": "ctrl+shift+a",
    "mac": "cmd+shift+a",
    "when": "editorFocus"
  }]
}

// Obsidian
this.addCommand({
  id: 'action',
  name: 'Action',
  hotkeys: [{ modifiers: ['Mod', 'Shift'], key: 'a' }],
  callback: () => {}
});
```

---

## 对 Coding Agent Desktop 应用的建议

### 快捷键系统设计

```typescript
interface Keybinding {
  /** 按键 */
  key: string;
  /** 修饰键 */
  modifiers?: Modifier[];
  /** 触发的命令 */
  command: string;
  /** 命令参数 */
  args?: any[];
  /** 生效条件 */
  when?: string;
}

type Modifier = 'Ctrl' | 'Cmd' | 'Alt' | 'Shift' | 'Mod';

interface KeybindingService {
  /** 注册快捷键 */
  register(binding: Keybinding): Disposable;

  /** 批量注册 */
  registerMany(bindings: Keybinding[]): Disposable;

  /** 获取命令的快捷键 */
  getKeybinding(commandId: string): Keybinding | undefined;

  /** 获取所有快捷键 */
  getAllKeybindings(): Keybinding[];

  /** 检测冲突 */
  getConflicts(binding: Keybinding): Keybinding[];

  /** 用户配置 */
  setUserKeybinding(commandId: string, binding: Keybinding | null): void;
}
```

### 上下文系统

```typescript
interface KeybindingContext {
  /** 设置上下文值 */
  set(key: string, value: any): void;

  /** 获取上下文值 */
  get(key: string): any;

  /** 评估条件表达式 */
  evaluate(when: string): boolean;
}

// 使用
context.set('editorFocus', true);
context.set('chatInputFocus', false);
context.set('hasSelection', editor.hasSelection());

// 条件表达式
const binding: Keybinding = {
  key: 'Enter',
  modifiers: ['Mod'],
  command: 'ai.sendMessage',
  when: 'chatInputFocus && !isComposing'
};
```

### 默认快捷键

```typescript
const defaultKeybindings: Keybinding[] = [
  // AI 相关
  { key: 'Enter', modifiers: ['Mod'], command: 'ai.sendMessage', when: 'chatInputFocus' },
  { key: 'n', modifiers: ['Mod', 'Shift'], command: 'ai.newChat' },
  { key: 'l', modifiers: ['Mod'], command: 'ai.clearChat', when: 'chatFocus' },

  // 编辑器相关
  { key: 's', modifiers: ['Mod'], command: 'editor.save' },
  { key: 'z', modifiers: ['Mod'], command: 'editor.undo' },
  { key: 'z', modifiers: ['Mod', 'Shift'], command: 'editor.redo' },

  // 通用
  { key: 'p', modifiers: ['Mod', 'Shift'], command: 'commandPalette.open' },
  { key: ',', modifiers: ['Mod'], command: 'settings.open' },
  { key: 'Escape', command: 'modal.close', when: 'modalOpen' },
];
```

### 快捷键显示

```typescript
function formatKeybinding(binding: Keybinding): string {
  const isMac = navigator.platform.includes('Mac');
  const parts: string[] = [];

  for (const mod of binding.modifiers || []) {
    switch (mod) {
      case 'Mod':
        parts.push(isMac ? '⌘' : 'Ctrl');
        break;
      case 'Ctrl':
        parts.push(isMac ? '⌃' : 'Ctrl');
        break;
      case 'Cmd':
        parts.push('⌘');
        break;
      case 'Alt':
        parts.push(isMac ? '⌥' : 'Alt');
        break;
      case 'Shift':
        parts.push(isMac ? '⇧' : 'Shift');
        break;
    }
  }

  parts.push(binding.key.toUpperCase());
  return parts.join(isMac ? '' : '+');
}

// formatKeybinding({ key: 's', modifiers: ['Mod'] })
// macOS: "⌘S"
// Windows: "Ctrl+S"
```

---

## 关键决策清单

1. **是否支持键序列？**
   - 支持（更多快捷键空间）
   - 不支持（更简单）

2. **条件如何表达？**
   - 声明式 when clause
   - 代码判断

3. **如何处理冲突？**
   - 优先级规则
   - 警告用户
   - 自动解决

4. **用户配置存储？**
   - JSON 文件
   - 设置界面

---

## 参考资料

- [VSCode Key Bindings](https://code.visualstudio.com/docs/getstarted/keybindings)
- [VSCode When Clause Contexts](https://code.visualstudio.com/api/references/when-clause-contexts)
- [Obsidian Hotkeys](https://docs.obsidian.md/Plugins/User+interface/Commands)
