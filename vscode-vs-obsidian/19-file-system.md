# 19. 文件系统

> **核心问题**：文件系统如何抽象？

---

## 概述

文件系统抽象决定了：
- 如何统一访问本地/远程文件
- 如何监听文件变化
- 如何处理虚拟文件
- 如何管理工作区

---

## VSCode 的文件系统

### FileSystemProvider

VSCode 通过 FileSystemProvider 抽象不同的文件系统：

```typescript
interface FileSystemProvider {
  // 元数据
  stat(uri: Uri): FileStat | Thenable<FileStat>;

  // 读写
  readFile(uri: Uri): Uint8Array | Thenable<Uint8Array>;
  writeFile(uri: Uri, content: Uint8Array, options: { create: boolean; overwrite: boolean }): void | Thenable<void>;

  // 目录操作
  readDirectory(uri: Uri): [string, FileType][] | Thenable<[string, FileType][]>;
  createDirectory(uri: Uri): void | Thenable<void>;
  delete(uri: Uri, options: { recursive: boolean }): void | Thenable<void>;
  rename(oldUri: Uri, newUri: Uri, options: { overwrite: boolean }): void | Thenable<void>;

  // 文件监听
  watch(uri: Uri, options: { recursive: boolean; excludes: string[] }): Disposable;
  onDidChangeFile: Event<FileChangeEvent[]>;
}
```

### 注册自定义文件系统

```typescript
// 注册虚拟文件系统
class MemoryFileSystemProvider implements vscode.FileSystemProvider {
  private files = new Map<string, Uint8Array>();
  private _emitter = new vscode.EventEmitter<vscode.FileChangeEvent[]>();
  readonly onDidChangeFile = this._emitter.event;

  stat(uri: vscode.Uri): vscode.FileStat {
    const content = this.files.get(uri.path);
    if (content) {
      return {
        type: vscode.FileType.File,
        ctime: Date.now(),
        mtime: Date.now(),
        size: content.length
      };
    }
    throw vscode.FileSystemError.FileNotFound(uri);
  }

  readFile(uri: vscode.Uri): Uint8Array {
    const content = this.files.get(uri.path);
    if (content) return content;
    throw vscode.FileSystemError.FileNotFound(uri);
  }

  writeFile(uri: vscode.Uri, content: Uint8Array): void {
    this.files.set(uri.path, content);
    this._emitter.fire([{ type: vscode.FileChangeType.Changed, uri }]);
  }

  // ... 其他方法
}

// 注册
vscode.workspace.registerFileSystemProvider('memfs', new MemoryFileSystemProvider());

// 使用
const uri = vscode.Uri.parse('memfs:/example.txt');
await vscode.workspace.fs.writeFile(uri, Buffer.from('Hello'));
```

### workspace.fs API

```typescript
// 统一的文件系统 API
const fs = vscode.workspace.fs;

// 读取文件
const content = await fs.readFile(uri);
const text = new TextDecoder().decode(content);

// 写入文件
const data = new TextEncoder().encode('Hello');
await fs.writeFile(uri, data);

// 目录操作
const entries = await fs.readDirectory(folderUri);
for (const [name, type] of entries) {
  console.log(name, type === vscode.FileType.Directory ? 'dir' : 'file');
}

// 元数据
const stat = await fs.stat(uri);
console.log('Size:', stat.size, 'Modified:', stat.mtime);

// 复制/删除
await fs.copy(srcUri, destUri, { overwrite: true });
await fs.delete(uri, { recursive: true });
```

### 文件监听

```typescript
// 创建文件监听器
const watcher = vscode.workspace.createFileSystemWatcher('**/*.ts');

watcher.onDidCreate(uri => console.log('Created:', uri.fsPath));
watcher.onDidChange(uri => console.log('Changed:', uri.fsPath));
watcher.onDidDelete(uri => console.log('Deleted:', uri.fsPath));

// 全局文件变化
vscode.workspace.onDidChangeTextDocument(event => {
  console.log('Document changed:', event.document.uri.fsPath);
});
```

---

## Obsidian 的文件系统

### Vault API

```typescript
// Vault 是 Obsidian 的文件系统抽象
interface Vault {
  // 读取
  read(file: TFile): Promise<string>;
  cachedRead(file: TFile): Promise<string>;
  readBinary(file: TFile): Promise<ArrayBuffer>;

  // 写入
  create(path: string, data: string): Promise<TFile>;
  createBinary(path: string, data: ArrayBuffer): Promise<TFile>;
  modify(file: TFile, data: string): Promise<void>;
  modifyBinary(file: TFile, data: ArrayBuffer): Promise<void>;
  append(file: TFile, data: string): Promise<void>;

  // 操作
  delete(file: TAbstractFile, force?: boolean): Promise<void>;
  rename(file: TAbstractFile, newPath: string): Promise<void>;
  copy(file: TFile, newPath: string): Promise<TFile>;

  // 查询
  getAbstractFileByPath(path: string): TAbstractFile | null;
  getMarkdownFiles(): TFile[];
  getAllLoadedFiles(): TAbstractFile[];
  getRoot(): TFolder;
}
```

### 文件操作示例

```typescript
export default class MyPlugin extends Plugin {
  async onload() {
    const vault = this.app.vault;

    // 获取文件
    const file = vault.getAbstractFileByPath('notes/example.md');
    if (file instanceof TFile) {
      // 读取内容
      const content = await vault.read(file);
      console.log('Content:', content);

      // 修改内容
      await vault.modify(file, content + '\n\n## New Section');
    }

    // 创建文件
    const newFile = await vault.create('notes/new-note.md', '# New Note\n');

    // 遍历所有 Markdown 文件
    const files = vault.getMarkdownFiles();
    for (const f of files) {
      console.log(f.path);
    }
  }
}
```

### 文件事件

```typescript
this.registerEvent(
  this.app.vault.on('create', (file) => {
    console.log('Created:', file.path);
  })
);

this.registerEvent(
  this.app.vault.on('modify', (file) => {
    console.log('Modified:', file.path);
  })
);

this.registerEvent(
  this.app.vault.on('delete', (file) => {
    console.log('Deleted:', file.path);
  })
);

this.registerEvent(
  this.app.vault.on('rename', (file, oldPath) => {
    console.log('Renamed:', oldPath, '->', file.path);
  })
);
```

---

## 对比分析

### 文件系统对比

| 方面 | VSCode | Obsidian |
|------|--------|----------|
| **抽象层** | FileSystemProvider | Vault |
| **虚拟文件系统** | 支持（自定义 scheme） | 不支持 |
| **远程文件** | 支持（Remote） | 不支持 |
| **文件类型** | 任意 | 主要 Markdown |
| **二进制文件** | 完整支持 | 有限支持 |
| **事件** | onDidChangeFile | vault.on |

---

## 对 Coding Agent Desktop 应用的建议

### 文件系统抽象

```typescript
interface IFileSystem {
  // 读取
  readFile(path: string): Promise<string>;
  readBinary(path: string): Promise<ArrayBuffer>;

  // 写入
  writeFile(path: string, content: string): Promise<void>;
  writeBinary(path: string, content: ArrayBuffer): Promise<void>;

  // 操作
  createDirectory(path: string): Promise<void>;
  delete(path: string, recursive?: boolean): Promise<void>;
  rename(oldPath: string, newPath: string): Promise<void>;
  copy(srcPath: string, destPath: string): Promise<void>;

  // 查询
  exists(path: string): Promise<boolean>;
  stat(path: string): Promise<FileStat>;
  readDirectory(path: string): Promise<DirectoryEntry[]>;

  // 监听
  watch(pattern: string, callback: (event: FileEvent) => void): Disposable;
}

interface FileStat {
  type: 'file' | 'directory' | 'symlink';
  size: number;
  createdAt: number;
  modifiedAt: number;
}

interface FileEvent {
  type: 'create' | 'change' | 'delete';
  path: string;
}
```

### 工作区管理

```typescript
interface IWorkspace {
  /** 根目录 */
  readonly rootPath: string;

  /** 文件系统 */
  readonly fs: IFileSystem;

  /** 打开的文件 */
  readonly openFiles: OpenFile[];

  /** 活动文件 */
  readonly activeFile: OpenFile | null;

  // 操作
  openFile(path: string): Promise<OpenFile>;
  closeFile(path: string): Promise<void>;
  saveFile(path: string): Promise<void>;
  saveAllFiles(): Promise<void>;

  // 事件
  onDidOpenFile: Event<OpenFile>;
  onDidCloseFile: Event<string>;
  onDidChangeActiveFile: Event<OpenFile | null>;
}
```

### 会话/项目管理

```typescript
interface IProjectManager {
  /** 当前项目 */
  currentProject: Project | null;

  /** 最近项目 */
  recentProjects: Project[];

  // 操作
  openProject(path: string): Promise<Project>;
  createProject(path: string, template?: string): Promise<Project>;
  closeProject(): Promise<void>;

  // 事件
  onDidChangeProject: Event<Project | null>;
}

interface Project {
  name: string;
  path: string;
  settings: ProjectSettings;
  chatSessions: ChatSession[];
}
```

---

## 关键决策清单

1. **是否需要虚拟文件系统？**
   - 支持：可以集成远程、内存文件
   - 不支持：简单

2. **工作区模型？**
   - 单根目录
   - 多根目录

3. **如何处理大文件？**
   - 流式读取
   - 分块处理

4. **如何同步文件变化？**
   - 文件监听
   - 定期扫描

---

## 参考资料

- [VSCode FileSystem API](https://code.visualstudio.com/api/references/vscode-api#FileSystem)
- [VSCode FileSystemProvider](https://code.visualstudio.com/api/extension-guides/virtual-documents)
- [Obsidian Vault](https://docs.obsidian.md/Reference/TypeScript+API/Vault)
