# Obsidian API Reference

> 来源：obsidian.d.ts 类型定义

## 1. 核心类层级

```
App (核心应用实例)
├── Vault            # 文件系统抽象
├── Workspace        # UI 布局管理
├── MetadataCache    # 文件元数据缓存
├── FileManager      # 高级文件操作
├── Keymap           # 快捷键管理
└── Scope            # 快捷键作用域

Component (组件基类)
├── Plugin           # 插件基类
├── View             # 视图基类
├── Modal            # 模态框基类
└── Setting          # 设置基类
```

---

## 2. Component 基类

```typescript
abstract class Component {
  // 子组件管理
  addChild<T extends Component>(child: T): T;
  removeChild(child: T): void;

  // 资源注册（自动清理）
  register(cb: () => void): void;
  registerEvent(eventRef: EventRef): void;
  registerDomEvent<K extends keyof WindowEventMap>(
    el: Window,
    type: K,
    callback: (ev: WindowEventMap[K]) => void
  ): void;
  registerInterval(interval: number): number;

  // 生命周期
  load(): void;
  onload(): void;
  unload(): void;
  onunload(): void;
}
```

---

## 3. Plugin 类

```typescript
abstract class Plugin extends Component {
  app: App;
  manifest: PluginManifest;

  // 命令注册
  addCommand(command: Command): Command;

  // UI 扩展
  addRibbonIcon(icon: string, title: string, callback: () => void): HTMLElement;
  addStatusBarItem(): HTMLElement;
  addSettingTab(settingTab: PluginSettingTab): void;

  // 视图注册
  registerView(type: string, viewCreator: ViewCreator): void;

  // 编辑器扩展
  registerEditorExtension(extension: Extension): void;
  registerMarkdownPostProcessor(postProcessor: MarkdownPostProcessor): void;
  registerMarkdownCodeBlockProcessor(
    language: string,
    handler: (source: string, el: HTMLElement, ctx: MarkdownPostProcessorContext) => void
  ): void;

  // 数据持久化
  loadData(): Promise<any>;
  saveData(data: any): Promise<void>;
}
```

---

## 4. Vault 接口

```typescript
interface Vault extends Events {
  adapter: DataAdapter;  // 底层适配器
  configDir: string;     // 配置目录 (.obsidian)

  // 读取方法
  read(file: TFile): Promise<string>;           // 直接读取磁盘
  cachedRead(file: TFile): Promise<string>;     // 从缓存读取
  readBinary(file: TFile): Promise<ArrayBuffer>;

  // 写入方法
  create(path: string, data: string): Promise<TFile>;
  createBinary(path: string, data: ArrayBuffer): Promise<TFile>;
  createFolder(path: string): Promise<void>;
  modify(file: TFile, data: string): Promise<void>;
  modifyBinary(file: TFile, data: ArrayBuffer): Promise<void>;
  append(file: TFile, data: string): Promise<void>;
  process(file: TFile, fn: (data: string) => string): Promise<string>;

  // 文件操作
  delete(file: TAbstractFile, force?: boolean): Promise<void>;
  trash(file: TAbstractFile, system?: boolean): Promise<void>;
  rename(file: TAbstractFile, newPath: string): Promise<void>;
  copy(file: TFile, newPath: string): Promise<TFile>;

  // 文件查询
  getAbstractFileByPath(path: string): TAbstractFile | null;
  getFileByPath(path: string): TFile | null;
  getFolderByPath(path: string): TFolder | null;
  getFiles(): TFile[];
  getMarkdownFiles(): TFile[];
  getAllLoadedFiles(): TAbstractFile[];

  // 事件
  on(name: 'create', callback: (file: TAbstractFile) => void): EventRef;
  on(name: 'modify', callback: (file: TAbstractFile) => void): EventRef;
  on(name: 'delete', callback: (file: TAbstractFile) => void): EventRef;
  on(name: 'rename', callback: (file: TAbstractFile, oldPath: string) => void): EventRef;
}
```

### 文件方法对比

| 方法 | 用途 | 何时使用 |
|------|------|---------|
| `read(file)` | 直接从磁盘读取 | 需要最新内容 |
| `cachedRead(file)` | 从内存缓存读取 | 频繁读取（性能更好） |
| `modify(file, data)` | 完全覆盖内容 | 整体替换 |
| `append(file, data)` | 追加到末尾 | 日志、追加笔记 |
| `process(file, fn)` | 读取→处理→写入（原子） | 基于当前内容修改 |

---

## 5. Workspace 接口

```typescript
interface Workspace extends Events {
  // 布局结构
  activeLeaf: WorkspaceLeaf | null;
  leftSplit: WorkspaceSidedock;
  rightSplit: WorkspaceSidedock;
  rootSplit: WorkspaceSplit;

  // Leaf 操作
  getLeaf(newLeaf?: boolean | 'tab' | 'split' | 'window'): WorkspaceLeaf;
  getLeaf(newLeaf: 'split', direction?: 'horizontal' | 'vertical'): WorkspaceLeaf;
  getLeftLeaf(split: boolean): WorkspaceLeaf;
  getRightLeaf(split: boolean): WorkspaceLeaf;
  splitActiveLeaf(): WorkspaceLeaf;
  createLeafInParent(parent: WorkspaceParent, index?: number): WorkspaceLeaf;

  // 视图查询
  getActiveViewOfType<T extends View>(type: Constructor<T>): T | null;
  getLeavesOfType(viewType: string): WorkspaceLeaf[];
  iterateAllLeaves(callback: (leaf: WorkspaceLeaf) => void): void;

  // 布局操作
  getLayout(): any;
  changeLayout(layout: any): Promise<void>;
  revealLeaf(leaf: WorkspaceLeaf): void;
  detachLeavesOfType(viewType: string): void;

  // 事件
  on(name: 'active-leaf-change', callback: (leaf: WorkspaceLeaf | null) => void): EventRef;
  on(name: 'layout-change', callback: () => void): EventRef;
  on(name: 'file-open', callback: (file: TFile | null) => void): EventRef;
  on(name: 'editor-change', callback: (editor: Editor, info: MarkdownView) => void): EventRef;
  on(name: 'resize', callback: () => void): EventRef;
}
```

### 布局层级

```
Workspace
├── WorkspaceSplit (root)
│   ├── WorkspaceSplit (左侧边栏)
│   │   └── WorkspaceTabs
│   │       └── WorkspaceLeaf
│   ├── WorkspaceSplit (主编辑区)
│   │   └── WorkspaceTabs
│   │       └── WorkspaceLeaf
│   └── WorkspaceSplit (右侧边栏)
│       └── WorkspaceTabs
│           └── WorkspaceLeaf
```

---

## 6. MetadataCache 接口

```typescript
interface MetadataCache extends Events {
  // 获取缓存
  getFileCache(file: TFile): CachedMetadata | null;
  getCache(path: string): CachedMetadata | null;

  // 链接解析
  getFirstLinkpathDest(linkpath: string, sourcePath: string): TFile | null;
  resolvedLinks: Record<string, Record<string, number>>;
  unresolvedLinks: Record<string, Record<string, number>>;

  // 事件
  on(name: 'changed', callback: (file: TFile, data: string, cache: CachedMetadata) => void): EventRef;
  on(name: 'resolve', callback: (file: TFile) => void): EventRef;
  on(name: 'resolved', callback: () => void): EventRef;
}

interface CachedMetadata {
  frontmatter?: FrontMatterCache;
  headings?: HeadingCache[];
  links?: LinkCache[];
  embeds?: EmbedCache[];
  tags?: TagCache[];
  sections?: SectionCache[];
  listItems?: ListItemCache[];
  blocks?: Record<string, BlockCache>;
}
```

---

## 7. FileManager 接口

```typescript
interface FileManager {
  // 带链接更新的文件操作
  renameFile(file: TAbstractFile, newPath: string): Promise<void>;
  trashFile(file: TAbstractFile): Promise<void>;

  // 链接生成
  generateMarkdownLink(
    file: TFile,
    sourcePath: string,
    subpath?: string,
    alias?: string
  ): string;

  // Frontmatter 操作
  processFrontMatter(
    file: TFile,
    fn: (frontmatter: any) => void,
    options?: DataWriteOptions
  ): Promise<void>;
}
```

**与 Vault 的区别：**
- `Vault.rename()` — 只重命名文件
- `FileManager.renameFile()` — 重命名并更新所有引用链接

---

## 8. View 基类

```typescript
abstract class View extends Component {
  app: App;
  leaf: WorkspaceLeaf;
  containerEl: HTMLElement;

  // 视图信息（必须实现）
  abstract getViewType(): string;
  abstract getDisplayText(): string;
  getIcon(): string;

  // 生命周期
  onOpen(): Promise<void>;
  onClose(): Promise<void>;

  // 状态管理
  getState(): any;
  setState(state: any, result: ViewStateResult): Promise<void>;
}
```

### 视图类型层级

```
View (抽象基类)
├── ItemView (带内容区的视图)
│   ├── MarkdownView (Markdown 编辑/预览)
│   ├── FileView (文件视图基类)
│   └── 自定义视图...
├── EditableFileView (可编辑文件视图)
└── TextFileView (纯文本视图)
```

---

## 9. Command 接口

```typescript
interface Command {
  id: string;              // 唯一标识符
  name: string;            // 命令面板显示名称
  icon?: string;           // Lucide 图标名
  mobileOnly?: boolean;    // 仅移动端
  hotkeys?: Hotkey[];      // 默认快捷键

  // 四种回调类型（按优先级）
  checkCallback?: (checking: boolean) => boolean | void;
  callback?: () => any;
  editorCheckCallback?: (
    checking: boolean,
    editor: Editor,
    ctx: MarkdownView | MarkdownFileInfo
  ) => boolean | void;
  editorCallback?: (
    editor: Editor,
    ctx: MarkdownView | MarkdownFileInfo
  ) => any;
}
```

---

## 10. Events 基类

```typescript
class Events {
  on(name: string, callback: (...data: any) => any, ctx?: any): EventRef;
  off(name: string, callback: (...data: any) => any): void;
  offref(ref: EventRef): void;
  trigger(name: string, ...data: any[]): void;
  tryTrigger(evt: EventRef, args: any[]): void;
}
```

---

## 11. 命名规范

### 类前缀

| 前缀 | 含义 | 示例 |
|------|------|------|
| 无前缀 | 公开核心类 | `Plugin`, `View`, `Modal` |
| `T` | 文件系统类型 | `TFile`, `TFolder`, `TAbstractFile` |
| `Workspace` | 布局组件 | `WorkspaceLeaf`, `WorkspaceSplit` |
| `Cached` | 缓存数据结构 | `CachedMetadata` |
| `Markdown` | Markdown 相关 | `MarkdownView`, `MarkdownRenderer` |

### 方法命名

| 前缀 | 动作 | 返回值 |
|------|------|--------|
| `add*` | 添加 UI 元素 | HTMLElement |
| `register*` | 注册扩展点 | void / EventRef |
| `get*` | 获取数据 | 数据 / null |
| `create*` | 创建新对象 | 新对象 |
| `on*` | 生命周期钩子 | void / Promise |
| `load*` | 加载数据 | Promise |
| `save*` | 保存数据 | Promise |

### 事件命名

```typescript
// Vault 事件
'create' | 'modify' | 'delete' | 'rename'

// Workspace 事件
'active-leaf-change' | 'layout-change' | 'file-open' | 'editor-change' | 'resize'

// MetadataCache 事件
'changed' | 'resolve' | 'resolved'
```

---

## 参考

- [obsidian.d.ts](https://github.com/obsidianmd/obsidian-api/blob/master/obsidian.d.ts)
- [Obsidian Developer Docs](https://docs.obsidian.md/)
- [Obsidian Typings](https://fevol.github.io/obsidian-typings/)
