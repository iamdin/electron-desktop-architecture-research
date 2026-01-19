# 08. UI 布局架构

> **核心问题**：界面如何组织？

---

## 概述

UI 布局架构决定了：
- 界面的整体结构和区域划分
- 用户如何组织工作空间
- 插件如何扩展 UI
- 布局如何持久化和恢复

---

## VSCode 的 UI 布局

### 整体结构

```
┌─────────────────────────────────────────────────────────────────────┐
│                          Title Bar                                   │
├────┬────────────────────────────────────────────────────────────────┤
│    │                                                                 │
│ A  │                         Editor Area                             │
│ c  │  ┌─────────────────┬─────────────────┬─────────────────┐       │
│ t  │  │   Editor Group  │   Editor Group  │   Editor Group  │       │
│ i  │  │   ┌─────────┐   │   ┌─────────┐   │   ┌─────────┐   │       │
│ v  │  │   │  Tab 1  │   │   │  Tab 1  │   │   │  Tab 1  │   │       │
│ i  │  │   │  Tab 2  │   │   └─────────┘   │   └─────────┘   │       │
│ t  │  │   └─────────┘   │                 │                 │       │
│ y  │  │                 │                 │                 │       │
│    │  │   [Editor]      │   [Editor]      │   [Editor]      │       │
│ B  │  │                 │                 │                 │       │
│ a  │  └─────────────────┴─────────────────┴─────────────────┘       │
│ r  ├────────────────────────────────────────────────────────────────┤
│    │                          Panel                                  │
├────┤  ┌──────────┬──────────┬──────────┬──────────┐                 │
│    │  │ Problems │ Output   │ Terminal │ Debug    │                 │
│ S  │  └──────────┴──────────┴──────────┴──────────┘                 │
│ i  ├────────────────────────────────────────────────────────────────┤
│ d  │                                                                 │
│ e  │                       Side Bar                                  │
│ b  │  ┌─────────────────────────────────────────────────────────┐   │
│ a  │  │  Explorer / Search / Source Control / Extensions / ...  │   │
│ r  │  └─────────────────────────────────────────────────────────┘   │
│    │                                                                 │
├────┴────────────────────────────────────────────────────────────────┤
│                          Status Bar                                  │
└─────────────────────────────────────────────────────────────────────┘
```

### 区域说明

| 区域 | 说明 | 插件可扩展 |
|------|------|-----------|
| **Title Bar** | 窗口标题栏 | 有限（Menu Bar） |
| **Activity Bar** | 最左侧图标栏 | ✅ viewsContainers |
| **Side Bar** | Activity Bar 旁的面板 | ✅ views |
| **Editor Area** | 中央编辑区 | ✅ customEditors, webview |
| **Panel** | 底部面板 | ✅ views (panel) |
| **Status Bar** | 底部状态栏 | ✅ statusBarItem |

### Editor Group

```typescript
// 编辑器分组 API
const column1 = vscode.ViewColumn.One;
const column2 = vscode.ViewColumn.Two;

// 在不同分组打开
await vscode.window.showTextDocument(doc, { viewColumn: column1 });
await vscode.window.showTextDocument(doc2, { viewColumn: column2 });

// 分割编辑器
await vscode.commands.executeCommand('workbench.action.splitEditor');
```

### 布局配置

```json
// settings.json
{
  "workbench.sideBar.location": "left",     // left | right
  "workbench.panel.defaultLocation": "bottom", // bottom | right
  "workbench.activityBar.visible": true,
  "workbench.statusBar.visible": true,
  "workbench.editor.showTabs": true,
  "workbench.editor.enablePreview": true
}
```

### 布局持久化

VSCode 自动保存和恢复：
- 打开的编辑器和分组
- 侧边栏状态
- 面板状态
- 窗口大小和位置

```typescript
// 通过 Memento 保存自定义状态
context.workspaceState.update('myLayout', layoutData);
const saved = context.workspaceState.get('myLayout');
```

---

## Obsidian 的 UI 布局

### 整体结构

```
┌─────────────────────────────────────────────────────────────────────┐
│                          Title Bar                                   │
├──────┬──────────────────────────────────────────────────────┬───────┤
│      │                                                      │       │
│  R   │                    Workspace                         │  R    │
│  i   │  ┌────────────────────────────────────────────────┐  │  i    │
│  b   │  │                                                │  │  g    │
│  b   │  │              Split (horizontal/vertical)       │  │  h    │
│  o   │  │  ┌─────────────────┐  ┌─────────────────┐     │  │  t    │
│  n   │  │  │   Leaf (Tab)    │  │   Leaf (Tab)    │     │  │       │
│      │  │  │   ┌─────────┐   │  │   ┌─────────┐   │     │  │  S    │
│      │  │  │   │ View    │   │  │   │ View    │   │     │  │  i    │
│  L   │  │  │   └─────────┘   │  │   └─────────┘   │     │  │  d    │
│  e   │  │  └─────────────────┘  └─────────────────┘     │  │  e    │
│  f   │  │                                                │  │  b    │
│  t   │  └────────────────────────────────────────────────┘  │  a    │
│      │                                                      │  r    │
│  S   ├──────────────────────────────────────────────────────┤       │
│  i   │                     Left Sidebar                     │       │
│  d   │  ┌────────────────────────────────────────────────┐  │       │
│  e   │  │  File Explorer / Search / Bookmarks / ...      │  │       │
│  b   │  └────────────────────────────────────────────────┘  │       │
│  a   │                                                      │       │
│  r   │                                                      │       │
├──────┴──────────────────────────────────────────────────────┴───────┤
│                          Status Bar                                  │
└─────────────────────────────────────────────────────────────────────┘
```

### 核心概念

#### Workspace

```typescript
// Workspace 是布局的根容器
const workspace = this.app.workspace;

// 获取当前活动视图
const activeView = workspace.getActiveViewOfType(MarkdownView);

// 获取所有叶子节点
const leaves = workspace.getLeavesOfType('markdown');
```

#### Leaf（叶子节点）

```typescript
// Leaf 是可以显示 View 的容器
interface WorkspaceLeaf {
  view: View;
  getViewState(): ViewState;
  setViewState(state: ViewState): Promise<void>;
  detach(): void;
}

// 获取或创建 Leaf
const leaf = workspace.getLeaf(false);  // 复用现有
const newLeaf = workspace.getLeaf(true); // 创建新的
const rightLeaf = workspace.getRightLeaf(false); // 右侧边栏
```

#### Split（分割）

```typescript
// 分割当前叶子
workspace.splitActiveLeaf('horizontal'); // 水平分割
workspace.splitActiveLeaf('vertical');   // 垂直分割

// 或者指定方向
const split = workspace.createLeafBySplit(existingLeaf, 'horizontal');
```

### 布局类型

```typescript
// Workspace 布局树
interface WorkspaceLayout {
  main: WorkspaceItem;      // 主区域
  left: WorkspaceItem;       // 左侧边栏
  right: WorkspaceItem;      // 右侧边栏
}

// WorkspaceItem 可以是：
// - WorkspaceLeaf（叶子节点，包含 View）
// - WorkspaceSplit（分割节点，包含子节点）
// - WorkspaceTabs（标签组，包含多个叶子）
```

### 布局持久化

```typescript
// 保存布局
const layout = workspace.getLayout();
localStorage.setItem('my-layout', JSON.stringify(layout));

// 恢复布局
const saved = localStorage.getItem('my-layout');
if (saved) {
  workspace.changeLayout(JSON.parse(saved));
}
```

### 自定义视图

```typescript
// 创建自定义视图
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
    container.createEl('h1', { text: 'Hello' });
  }
}

// 注册视图
this.registerView('my-view', (leaf) => new MyView(leaf));

// 激活视图
async activateView() {
  const { workspace } = this.app;

  let leaf = workspace.getLeavesOfType('my-view')[0];
  if (!leaf) {
    // 在右侧边栏创建
    leaf = workspace.getRightLeaf(false);
    await leaf.setViewState({
      type: 'my-view',
      active: true
    });
  }
  workspace.revealLeaf(leaf);
}
```

---

## 对比分析

### 布局模型对比

| 方面 | VSCode | Obsidian |
|------|--------|----------|
| **模型** | 固定区域 | 灵活树状 |
| **编辑区** | Editor Group（最多 3 列推荐） | Split + Tabs（任意分割） |
| **侧边栏** | 固定左右，通过 Activity Bar 切换 | 左右可折叠，任意组织 |
| **面板** | 固定底部/右侧 | 无专门面板概念 |
| **标签页** | Editor Tabs（单行滚动） | Leaf Tabs（可拖拽） |

### 灵活性对比

| 能力 | VSCode | Obsidian |
|------|--------|----------|
| **任意分割** | 仅 Editor 区域 | 任意区域 |
| **拖拽重排** | 有限制 | 完全自由 |
| **布局保存** | 自动 | 自动 + 手动 |
| **多布局** | 无（靠窗口） | 无内置 |
| **浮动窗口** | 不支持 | 支持（弹出窗口） |

### 代码示例对比

#### 创建自定义面板

**VSCode**：
```typescript
// 1. 声明视图位置
// package.json
{
  "contributes": {
    "views": {
      "explorer": [{
        "id": "myView",
        "name": "My View"
      }]
    }
  }
}

// 2. 实现 TreeDataProvider
class MyTreeProvider implements vscode.TreeDataProvider<MyItem> {
  getTreeItem(element: MyItem): vscode.TreeItem {
    return new vscode.TreeItem(element.label);
  }
  getChildren(): MyItem[] {
    return [{ label: 'Item 1' }, { label: 'Item 2' }];
  }
}

// 3. 注册
vscode.window.registerTreeDataProvider('myView', new MyTreeProvider());
```

**Obsidian**：
```typescript
// 1. 实现 ItemView
class MyView extends ItemView {
  getViewType() { return 'my-view'; }
  getDisplayText() { return 'My View'; }

  async onOpen() {
    this.contentEl.createEl('ul', {}, ul => {
      ul.createEl('li', { text: 'Item 1' });
      ul.createEl('li', { text: 'Item 2' });
    });
  }
}

// 2. 注册
this.registerView('my-view', (leaf) => new MyView(leaf));

// 3. 在左侧边栏打开
this.addRibbonIcon('list', 'Open My View', () => {
  const leaf = this.app.workspace.getLeftLeaf(false);
  leaf.setViewState({ type: 'my-view' });
});
```

---

## 对 AI Chat + Editor 应用的建议

### 推荐的布局结构

```
┌─────────────────────────────────────────────────────────────────────┐
│                          Title Bar                                   │
├──────┬──────────────────────────────────────────────────────┬───────┤
│      │                                                      │       │
│  N   │                    Main Area                         │  A    │
│  a   │  ┌────────────────────┬─────────────────────────┐   │  s    │
│  v   │  │                    │                         │   │  s    │
│  i   │  │    Editor Panel    │      Chat Panel         │   │  i    │
│  g   │  │                    │                         │   │  s    │
│  a   │  │  - Code Editor     │  - Message List         │   │  t    │
│  t   │  │  - Markdown        │  - Input Area           │   │  a    │
│  i   │  │  - Preview         │  - Context              │   │  n    │
│  o   │  │                    │                         │   │  t    │
│  n   │  │                    │                         │   │       │
│      │  └────────────────────┴─────────────────────────┘   │  P    │
│  B   │                                                      │  a    │
│  a   ├──────────────────────────────────────────────────────┤  n    │
│  r   │                   Bottom Panel                       │  e    │
│      │  ┌─────────┬─────────┬─────────┬─────────┐          │  l    │
│      │  │Terminal │ Output  │ Problems│  ...    │          │       │
│      │  └─────────┴─────────┴─────────┴─────────┘          │       │
├──────┴──────────────────────────────────────────────────────┴───────┤
│                          Status Bar                                  │
└─────────────────────────────────────────────────────────────────────┘
```

### 布局系统设计

```typescript
// 布局区域定义
type LayoutArea = 'navigation' | 'main' | 'assistant' | 'panel' | 'status';

interface LayoutConfig {
  /** 导航栏位置 */
  navigation: {
    position: 'left' | 'top';
    collapsed: boolean;
  };

  /** 主区域分割 */
  main: {
    splits: SplitConfig[];
    activeSplit: string;
  };

  /** 助手面板 */
  assistant: {
    visible: boolean;
    position: 'right' | 'bottom';
    width: number;
  };

  /** 底部面板 */
  panel: {
    visible: boolean;
    height: number;
    activeTab: string;
  };
}

// 分割配置
interface SplitConfig {
  id: string;
  direction: 'horizontal' | 'vertical';
  ratio: number;
  children: (SplitConfig | LeafConfig)[];
}

interface LeafConfig {
  id: string;
  viewType: string;
  state: any;
}
```

### 布局管理 API

```typescript
interface LayoutManager {
  // 获取当前布局
  getLayout(): LayoutConfig;

  // 设置布局
  setLayout(config: LayoutConfig): void;

  // 保存布局为预设
  saveLayoutPreset(name: string): void;

  // 加载布局预设
  loadLayoutPreset(name: string): void;

  // 重置为默认布局
  resetLayout(): void;

  // 切换区域可见性
  toggleArea(area: LayoutArea): void;

  // 事件
  onLayoutChanged: Event<LayoutConfig>;
}
```

### 视图系统设计

```typescript
// 视图基类
abstract class BaseView {
  abstract readonly viewType: string;
  abstract readonly displayName: string;
  abstract readonly icon?: string;

  /** 视图容器 */
  protected container: HTMLElement;

  /** 视图状态 */
  protected state: any;

  /** 初始化 */
  abstract onMount(container: HTMLElement): void;

  /** 销毁 */
  abstract onUnmount(): void;

  /** 获取状态（用于持久化） */
  getState(): any {
    return this.state;
  }

  /** 恢复状态 */
  setState(state: any): void {
    this.state = state;
  }
}

// 编辑器视图
class EditorView extends BaseView {
  viewType = 'editor';
  displayName = 'Editor';
  icon = 'file-code';

  onMount(container: HTMLElement) {
    // 初始化编辑器
  }
}

// 聊天视图
class ChatView extends BaseView {
  viewType = 'chat';
  displayName = 'Chat';
  icon = 'message-circle';

  onMount(container: HTMLElement) {
    // 初始化聊天界面
  }
}
```

### 插件视图扩展

```typescript
// 插件注册自定义视图
class MyPlugin extends BasePlugin {
  async onActivate() {
    // 注册视图
    this.context.registerView({
      viewType: 'my-custom-view',
      displayName: 'My View',
      icon: 'star',
      factory: () => new MyCustomView()
    });

    // 注册视图位置
    this.context.registerViewPosition({
      viewType: 'my-custom-view',
      defaultPosition: 'assistant', // navigation | main | assistant | panel
      allowedPositions: ['assistant', 'panel']
    });
  }
}
```

### 响应式布局

```typescript
// 响应式断点
const breakpoints = {
  mobile: 640,
  tablet: 1024,
  desktop: 1280
};

// 自适应布局配置
const responsiveLayouts: Record<string, LayoutConfig> = {
  mobile: {
    navigation: { position: 'top', collapsed: true },
    assistant: { visible: false, position: 'bottom' },
    // ...
  },
  tablet: {
    navigation: { position: 'left', collapsed: true },
    assistant: { visible: true, position: 'bottom' },
    // ...
  },
  desktop: {
    navigation: { position: 'left', collapsed: false },
    assistant: { visible: true, position: 'right' },
    // ...
  }
};
```

---

## 关键决策清单

1. **固定布局还是灵活布局？**
   - 固定：简单、一致，但限制用户自由
   - 灵活：自由、强大，但实现复杂

2. **编辑器和聊天的关系？**
   - 并排显示（推荐）
   - 标签切换
   - 浮动窗口

3. **是否支持多窗口？**
   - 单窗口（简单）
   - 多窗口（复杂但灵活）

4. **布局如何持久化？**
   - 自动保存当前状态
   - 支持布局预设

5. **响应式如何处理？**
   - 固定断点
   - 用户自定义

---

## 参考资料

- [VSCode Workbench](https://code.visualstudio.com/api/ux-guidelines/overview)
- [Obsidian Workspace](https://docs.obsidian.md/Reference/TypeScript+API/Workspace)
- [React DnD](https://react-dnd.github.io/react-dnd/about) - 拖拽库
- [Golden Layout](http://golden-layout.com/) - 布局管理库
