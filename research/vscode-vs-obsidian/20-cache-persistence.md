# 20. 缓存与持久化

> **核心问题**：数据如何持久化？

---

## 概述

缓存与持久化决定了：
- 应用状态如何跨会话保存
- 插件数据如何存储
- 如何优化启动性能
- 如何处理数据迁移

---

## VSCode 的缓存与持久化

### 存储类型

| 类型 | 用途 | 位置 |
|------|------|------|
| **globalState** | 插件全局数据 | 用户目录 |
| **workspaceState** | 工作区数据 | 工作区目录 |
| **secrets** | 敏感数据 | 系统钥匙串 |
| **IndexedDB** | 大量数据 | 浏览器存储 |
| **文件系统** | 配置文件 | .vscode/ |

### ExtensionContext 存储

```typescript
export function activate(context: vscode.ExtensionContext) {
  // 全局状态（跨工作区）
  const count = context.globalState.get<number>('activationCount', 0);
  await context.globalState.update('activationCount', count + 1);

  // 工作区状态
  const lastFile = context.workspaceState.get<string>('lastOpenedFile');
  await context.workspaceState.update('lastOpenedFile', 'example.ts');

  // 密钥存储
  await context.secrets.store('apiKey', 'sk-...');
  const apiKey = await context.secrets.get('apiKey');
  await context.secrets.delete('apiKey');

  // 存储路径
  console.log('Global storage:', context.globalStorageUri.fsPath);
  console.log('Workspace storage:', context.storageUri?.fsPath);
  console.log('Log path:', context.logUri.fsPath);
}
```

### Memento API

```typescript
interface Memento {
  keys(): readonly string[];
  get<T>(key: string): T | undefined;
  get<T>(key: string, defaultValue: T): T;
  update(key: string, value: any): Thenable<void>;
}

// 使用
const state: Memento = context.globalState;

// 获取所有键
const keys = state.keys();

// 类型安全的获取
interface MyData {
  version: number;
  items: string[];
}
const data = state.get<MyData>('myData', { version: 1, items: [] });

// 更新
await state.update('myData', { version: 2, items: ['a', 'b'] });
```

### 文件存储

```typescript
// 使用 globalStorageUri 存储文件
const storagePath = context.globalStorageUri;

// 确保目录存在
await vscode.workspace.fs.createDirectory(storagePath);

// 存储数据
const dataUri = vscode.Uri.joinPath(storagePath, 'data.json');
const data = { version: 1, items: [] };
await vscode.workspace.fs.writeFile(
  dataUri,
  Buffer.from(JSON.stringify(data, null, 2))
);

// 读取数据
const content = await vscode.workspace.fs.readFile(dataUri);
const loaded = JSON.parse(content.toString());
```

---

## Obsidian 的缓存与持久化

### 插件数据存储

```typescript
export default class MyPlugin extends Plugin {
  data: MyPluginData;

  async onload() {
    // 加载数据（从 data.json）
    const saved = await this.loadData();
    this.data = Object.assign({}, DEFAULT_DATA, saved);
  }

  async saveData() {
    // 保存数据（到 data.json）
    await super.saveData(this.data);
  }
}

// 存储位置: .obsidian/plugins/my-plugin/data.json
```

### MetadataCache

```typescript
// Obsidian 内置的元数据缓存
const cache = this.app.metadataCache;

// 获取文件缓存
const fileCache = cache.getFileCache(file);
if (fileCache) {
  // 链接
  const links = fileCache.links || [];
  // 标签
  const tags = fileCache.tags || [];
  // 标题
  const headings = fileCache.headings || [];
  // Front Matter
  const frontmatter = fileCache.frontmatter;
}

// 监听缓存变化
this.registerEvent(
  cache.on('changed', (file, data, cache) => {
    console.log('Cache updated for:', file.path);
  })
);

// 解析后触发
this.registerEvent(
  cache.on('resolved', () => {
    console.log('All caches resolved');
  })
);
```

### LocalStorage

```typescript
// 也可以使用 LocalStorage（跨 Vault）
localStorage.setItem('my-plugin-global', JSON.stringify(data));
const data = JSON.parse(localStorage.getItem('my-plugin-global') || '{}');
```

---

## 对比分析

### 存储对比

| 方面 | VSCode | Obsidian |
|------|--------|----------|
| **插件数据** | Memento API | loadData/saveData |
| **密钥存储** | secrets API | 无内置 |
| **全局/工作区** | 分离 | 单一（Vault） |
| **元数据缓存** | 无内置 | MetadataCache |
| **大数据** | IndexedDB | 文件 |

---

## 对 AI Chat + Editor 应用的建议

### 存储层设计

```typescript
interface IStorageService {
  // 键值存储
  get<T>(key: string): T | undefined;
  set<T>(key: string, value: T): void;
  delete(key: string): void;
  clear(): void;

  // 密钥存储
  getSecret(key: string): Promise<string | undefined>;
  setSecret(key: string, value: string): Promise<void>;
  deleteSecret(key: string): Promise<void>;
}

// 分层存储
interface StorageScopes {
  /** 应用全局 */
  global: IStorageService;
  /** 工作区级 */
  workspace: IStorageService;
  /** 会话级（内存） */
  session: IStorageService;
}
```

### 实现

```typescript
class LocalStorageService implements IStorageService {
  constructor(private prefix: string) {}

  get<T>(key: string): T | undefined {
    const value = localStorage.getItem(`${this.prefix}:${key}`);
    return value ? JSON.parse(value) : undefined;
  }

  set<T>(key: string, value: T): void {
    localStorage.setItem(`${this.prefix}:${key}`, JSON.stringify(value));
  }

  delete(key: string): void {
    localStorage.removeItem(`${this.prefix}:${key}`);
  }

  clear(): void {
    const keysToRemove: string[] = [];
    for (let i = 0; i < localStorage.length; i++) {
      const key = localStorage.key(i);
      if (key?.startsWith(this.prefix)) {
        keysToRemove.push(key);
      }
    }
    keysToRemove.forEach(key => localStorage.removeItem(key));
  }
}

// IndexedDB 大数据存储
class IndexedDBStorage {
  private db: IDBDatabase | null = null;

  async init(dbName: string): Promise<void> {
    return new Promise((resolve, reject) => {
      const request = indexedDB.open(dbName, 1);
      request.onerror = () => reject(request.error);
      request.onsuccess = () => {
        this.db = request.result;
        resolve();
      };
      request.onupgradeneeded = (event) => {
        const db = (event.target as IDBOpenDBRequest).result;
        db.createObjectStore('data', { keyPath: 'id' });
      };
    });
  }

  async get<T>(id: string): Promise<T | undefined> {
    return new Promise((resolve, reject) => {
      const tx = this.db!.transaction('data', 'readonly');
      const store = tx.objectStore('data');
      const request = store.get(id);
      request.onerror = () => reject(request.error);
      request.onsuccess = () => resolve(request.result?.value);
    });
  }

  async set<T>(id: string, value: T): Promise<void> {
    return new Promise((resolve, reject) => {
      const tx = this.db!.transaction('data', 'readwrite');
      const store = tx.objectStore('data');
      const request = store.put({ id, value });
      request.onerror = () => reject(request.error);
      request.onsuccess = () => resolve();
    });
  }
}
```

### 聊天历史存储

```typescript
interface ChatHistoryStorage {
  // 会话
  getSessions(): Promise<ChatSession[]>;
  getSession(id: string): Promise<ChatSession | undefined>;
  saveSession(session: ChatSession): Promise<void>;
  deleteSession(id: string): Promise<void>;

  // 消息
  getMessages(sessionId: string): Promise<Message[]>;
  addMessage(sessionId: string, message: Message): Promise<void>;
  deleteMessage(sessionId: string, messageId: string): Promise<void>;
}

class ChatHistoryService implements ChatHistoryStorage {
  constructor(private storage: IndexedDBStorage) {}

  async saveSession(session: ChatSession): Promise<void> {
    await this.storage.set(`session:${session.id}`, session);

    // 更新索引
    const index = await this.storage.get<string[]>('session-index') || [];
    if (!index.includes(session.id)) {
      index.push(session.id);
      await this.storage.set('session-index', index);
    }
  }

  async getSessions(): Promise<ChatSession[]> {
    const index = await this.storage.get<string[]>('session-index') || [];
    const sessions = await Promise.all(
      index.map(id => this.storage.get<ChatSession>(`session:${id}`))
    );
    return sessions.filter((s): s is ChatSession => s !== undefined);
  }
}
```

### 数据迁移

```typescript
interface MigrationStrategy {
  version: number;
  migrate(data: any): any;
}

class DataMigrator {
  private migrations: MigrationStrategy[] = [];

  addMigration(migration: MigrationStrategy): void {
    this.migrations.push(migration);
    this.migrations.sort((a, b) => a.version - b.version);
  }

  migrate(data: any, fromVersion: number, toVersion: number): any {
    let current = data;
    for (const migration of this.migrations) {
      if (migration.version > fromVersion && migration.version <= toVersion) {
        current = migration.migrate(current);
      }
    }
    return current;
  }
}

// 使用
const migrator = new DataMigrator();

migrator.addMigration({
  version: 2,
  migrate: (data) => ({
    ...data,
    newField: 'default'
  })
});

migrator.addMigration({
  version: 3,
  migrate: (data) => ({
    ...data,
    renamedField: data.oldField,
    oldField: undefined
  })
});
```

---

## 关键决策清单

1. **使用什么存储？**
   - LocalStorage：小数据
   - IndexedDB：大数据
   - 文件系统：配置

2. **如何处理敏感数据？**
   - 系统钥匙串
   - 加密存储

3. **如何处理数据迁移？**
   - 版本号
   - 迁移脚本

4. **缓存策略？**
   - LRU
   - TTL
   - 按需加载

---

## 参考资料

- [VSCode ExtensionContext](https://code.visualstudio.com/api/references/vscode-api#ExtensionContext)
- [IndexedDB API](https://developer.mozilla.org/en-US/docs/Web/API/IndexedDB_API)
- [Electron safeStorage](https://www.electronjs.org/docs/latest/api/safe-storage)
