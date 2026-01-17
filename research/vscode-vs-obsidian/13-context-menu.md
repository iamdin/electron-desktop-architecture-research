# 13. 上下文菜单

> **核心问题**：右键菜单如何扩展？

---

## 概述

上下文菜单（右键菜单）是重要的交互方式：
- 提供与当前元素相关的操作
- 减少用户记忆负担
- 插件如何添加菜单项
- 菜单如何按上下文显示/隐藏

---

## VSCode 的上下文菜单

### 声明式菜单

```json
// package.json
{
  "contributes": {
    "menus": {
      "editor/context": [
        {
          "command": "myExtension.formatSelection",
          "when": "editorHasSelection",
          "group": "1_modification"
        }
      ],
      "explorer/context": [
        {
          "command": "myExtension.openInTerminal",
          "when": "explorerResourceIsFolder",
          "group": "navigation"
        }
      ],
      "view/item/context": [
        {
          "command": "myExtension.deleteItem",
          "when": "view == myTreeView && viewItem == deletable",
          "group": "inline"
        }
      ]
    }
  }
}
```

### 菜单位置

| 位置 | 说明 |
|------|------|
| `editor/context` | 编辑器右键菜单 |
| `editor/title` | 编辑器标题栏 |
| `editor/title/context` | 编辑器标签右键 |
| `explorer/context` | 资源管理器右键 |
| `view/title` | 视图标题栏 |
| `view/item/context` | 视图项右键 |
| `scm/title` | 源代码管理标题 |
| `scm/resourceState/context` | SCM 资源右键 |
| `terminal/context` | 终端右键 |
| `commandPalette` | 命令面板 |

### 分组排序

```json
{
  "menus": {
    "editor/context": [
      { "command": "cmd1", "group": "navigation" },
      { "command": "cmd2", "group": "1_modification" },
      { "command": "cmd3", "group": "1_modification@1" },
      { "command": "cmd4", "group": "1_modification@2" },
      { "command": "cmd5", "group": "z_commands" }
    ]
  }
}
```

分组顺序：
- `navigation` - 最顶部
- `1_modification` - 修改操作
- `9_cutcopypaste` - 剪切复制粘贴
- `z_commands` - 最底部

`@` 后的数字控制组内排序。

### 子菜单

```json
{
  "contributes": {
    "submenus": [
      {
        "id": "myExtension.submenu",
        "label": "My Extension"
      }
    ],
    "menus": {
      "editor/context": [
        {
          "submenu": "myExtension.submenu",
          "group": "myExtension"
        }
      ],
      "myExtension.submenu": [
        { "command": "myExtension.action1" },
        { "command": "myExtension.action2" },
        { "command": "myExtension.action3" }
      ]
    }
  }
}
```

### 内联操作

```json
{
  "menus": {
    "view/item/context": [
      {
        "command": "myExtension.editItem",
        "when": "view == myTreeView",
        "group": "inline"
      }
    ]
  }
}
```

`group: "inline"` 会在视图项旁边显示图标按钮。

---

## Obsidian 的上下文菜单

### 代码注册

```typescript
export default class MyPlugin extends Plugin {
  onload() {
    // 文件菜单
    this.registerEvent(
      this.app.workspace.on('file-menu', (menu, file) => {
        menu.addItem((item) => {
          item
            .setTitle('My Action')
            .setIcon('star')
            .onClick(() => {
              new Notice(`Action on ${file.path}`);
            });
        });
      })
    );

    // 编辑器菜单
    this.registerEvent(
      this.app.workspace.on('editor-menu', (menu, editor, view) => {
        menu.addItem((item) => {
          item
            .setTitle('Format Selection')
            .setIcon('wand')
            .onClick(() => {
              const selection = editor.getSelection();
              editor.replaceSelection(selection.toUpperCase());
            });
        });
      })
    );
  }
}
```

### Menu API

```typescript
// 创建菜单
const menu = new Menu();

menu.addItem((item) => {
  item
    .setTitle('Item 1')
    .setIcon('file')
    .onClick(() => {});
});

menu.addSeparator();

menu.addItem((item) => {
  item
    .setTitle('Item 2')
    .setIcon('folder')
    .setDisabled(true);  // 禁用
});

// 显示菜单
menu.showAtPosition({ x: event.clientX, y: event.clientY });
// 或
menu.showAtMouseEvent(event);
```

### 条件显示

```typescript
this.registerEvent(
  this.app.workspace.on('file-menu', (menu, file) => {
    // 只对 Markdown 文件显示
    if (file instanceof TFile && file.extension === 'md') {
      menu.addItem((item) => {
        item
          .setTitle('Markdown Action')
          .onClick(() => {});
      });
    }
  })
);
```

---

## 对比分析

### 上下文菜单对比

| 方面 | VSCode | Obsidian |
|------|--------|----------|
| **注册方式** | 声明式 (package.json) | 代码注册 (registerEvent) |
| **条件显示** | when clause | 代码判断 |
| **分组排序** | group 字段 | addSeparator |
| **子菜单** | 支持 | 手动实现 |
| **图标** | 支持 | 支持 |
| **内联操作** | 支持 | 不支持 |

---

## 对 AI Chat + Editor 应用的建议

### 菜单系统设计

```typescript
interface MenuItem {
  /** ID */
  id: string;
  /** 标题 */
  title: string;
  /** 图标 */
  icon?: string;
  /** 快捷键提示 */
  keybinding?: string;
  /** 点击处理 */
  onClick?: (context: MenuContext) => void;
  /** 子菜单 */
  submenu?: MenuItem[];
  /** 是否禁用 */
  disabled?: boolean | ((context: MenuContext) => boolean);
  /** 是否可见 */
  visible?: boolean | ((context: MenuContext) => boolean);
}

interface MenuContext {
  /** 触发位置 */
  target: HTMLElement;
  /** 相关数据 */
  data?: any;
  /** 当前视图 */
  view?: string;
  /** 选中内容 */
  selection?: string;
}
```

### 菜单位置定义

```typescript
enum MenuLocation {
  // 聊天相关
  ChatMessage = 'chat/message',
  ChatInput = 'chat/input',
  ChatHistory = 'chat/history',

  // 编辑器相关
  EditorContext = 'editor/context',
  EditorSelection = 'editor/selection',
  EditorGutter = 'editor/gutter',

  // 文件相关
  FileTree = 'file/tree',
  FileTab = 'file/tab',

  // 通用
  Sidebar = 'sidebar',
  StatusBar = 'statusbar',
}
```

### 菜单注册

```typescript
// 插件注册菜单项
context.registerMenuItem({
  location: MenuLocation.ChatMessage,
  item: {
    id: 'copy-code',
    title: 'Copy Code',
    icon: 'copy',
    visible: (ctx) => ctx.data?.hasCodeBlock,
    onClick: (ctx) => {
      navigator.clipboard.writeText(ctx.data.code);
    }
  }
});

// 注册子菜单
context.registerMenuItem({
  location: MenuLocation.EditorSelection,
  item: {
    id: 'ai-actions',
    title: 'AI Actions',
    icon: 'sparkles',
    submenu: [
      { id: 'explain', title: 'Explain', onClick: (ctx) => aiService.explain(ctx.selection) },
      { id: 'refactor', title: 'Refactor', onClick: (ctx) => aiService.refactor(ctx.selection) },
      { id: 'test', title: 'Generate Tests', onClick: (ctx) => aiService.generateTests(ctx.selection) },
    ]
  }
});
```

### 菜单渲染

```typescript
class ContextMenu {
  private items: MenuItem[] = [];
  private element: HTMLElement;

  show(position: { x: number; y: number }, context: MenuContext) {
    this.element = document.createElement('div');
    this.element.className = 'context-menu';

    for (const item of this.getVisibleItems(context)) {
      const itemEl = this.renderItem(item, context);
      this.element.appendChild(itemEl);
    }

    this.element.style.left = `${position.x}px`;
    this.element.style.top = `${position.y}px`;
    document.body.appendChild(this.element);

    // 点击外部关闭
    document.addEventListener('click', this.handleClickOutside);
  }

  private getVisibleItems(context: MenuContext): MenuItem[] {
    return this.items.filter(item => {
      if (typeof item.visible === 'function') {
        return item.visible(context);
      }
      return item.visible !== false;
    });
  }

  hide() {
    this.element?.remove();
    document.removeEventListener('click', this.handleClickOutside);
  }
}
```

### AI 消息菜单示例

```typescript
// 聊天消息右键菜单
const chatMessageMenuItems: MenuItem[] = [
  {
    id: 'copy',
    title: 'Copy Message',
    icon: 'copy',
    onClick: (ctx) => navigator.clipboard.writeText(ctx.data.content)
  },
  {
    id: 'copy-code',
    title: 'Copy Code Block',
    icon: 'code',
    visible: (ctx) => ctx.data.hasCodeBlock,
    onClick: (ctx) => navigator.clipboard.writeText(ctx.data.codeBlocks[0])
  },
  { id: 'separator', title: '-' },
  {
    id: 'insert-to-editor',
    title: 'Insert to Editor',
    icon: 'file-plus',
    visible: (ctx) => ctx.data.role === 'assistant',
    onClick: (ctx) => editorService.insertText(ctx.data.content)
  },
  {
    id: 'regenerate',
    title: 'Regenerate',
    icon: 'refresh',
    visible: (ctx) => ctx.data.role === 'assistant',
    onClick: (ctx) => chatService.regenerate(ctx.data.id)
  },
  { id: 'separator2', title: '-' },
  {
    id: 'delete',
    title: 'Delete Message',
    icon: 'trash',
    onClick: (ctx) => chatService.deleteMessage(ctx.data.id)
  }
];
```

---

## 关键决策清单

1. **声明式还是代码式？**
   - 声明式：静态配置，易于管理
   - 代码式：动态灵活，但分散

2. **菜单项如何分组？**
   - 内置分组规则
   - 自定义分组

3. **子菜单层级？**
   - 限制层级（避免太深）
   - 无限制

4. **样式如何定义？**
   - 适配主题
   - 自定义样式

---

## 参考资料

- [VSCode Menus](https://code.visualstudio.com/api/references/contribution-points#contributes.menus)
- [Obsidian Menu](https://docs.obsidian.md/Reference/TypeScript+API/Menu)
