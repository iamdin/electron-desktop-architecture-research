# 17. 状态管理

> **核心问题**：状态如何管理？

---

## 概述

状态管理决定了：
- 数据如何在应用中流动
- 组件如何响应状态变化
- 状态如何同步（多窗口/多进程）
- 状态如何持久化

---

## VSCode 的状态管理

### 服务分散管理

VSCode 没有集中式状态管理，而是通过服务分散管理：

```typescript
// 各服务管理自己的状态
class EditorService {
  private readonly _onDidActiveEditorChange = new Emitter<IEditorIdentifier>();
  readonly onDidActiveEditorChange = this._onDidActiveEditorChange.event;

  private _activeEditor: IEditorInput | undefined;

  get activeEditor(): IEditorInput | undefined {
    return this._activeEditor;
  }

  set activeEditor(editor: IEditorInput | undefined) {
    if (this._activeEditor !== editor) {
      this._activeEditor = editor;
      this._onDidActiveEditorChange.fire(editor);
    }
  }
}
```

### Event Emitter 模式

```typescript
// 定义事件
import { Emitter, Event } from 'vs/base/common/event';

class MyService {
  private readonly _onDidChange = new Emitter<ChangeEvent>();
  readonly onDidChange: Event<ChangeEvent> = this._onDidChange.event;

  private _state: MyState = { value: 0 };

  get state(): MyState {
    return this._state;
  }

  updateState(newValue: number): void {
    this._state = { value: newValue };
    this._onDidChange.fire({ type: 'valueChanged', value: newValue });
  }

  dispose(): void {
    this._onDidChange.dispose();
  }
}

// 使用
const service = instantiationService.get(IMyService);
service.onDidChange(event => {
  console.log('State changed:', event);
});
```

### 上下文键（Context Key）

```typescript
// 设置上下文
const myContextKey = new RawContextKey<boolean>('myExtension.isActive', false);
myContextKey.bindTo(contextKeyService).set(true);

// 在 when clause 中使用
{
  "when": "myExtension.isActive"
}

// 代码中检查
const isActive = contextKeyService.getContextKeyValue('myExtension.isActive');
```

### 插件状态存储

```typescript
export function activate(context: vscode.ExtensionContext) {
  // 全局状态（跨工作区）
  const globalCount = context.globalState.get<number>('count', 0);
  context.globalState.update('count', globalCount + 1);

  // 工作区状态
  const workspaceData = context.workspaceState.get<MyData>('data');
  context.workspaceState.update('data', newData);

  // 密钥存储
  await context.secrets.store('apiKey', 'secret');
  const key = await context.secrets.get('apiKey');
}
```

---

## Obsidian 的状态管理

### 集中式 App 对象

```typescript
// App 是全局状态的入口
interface App {
  vault: Vault;           // 文件状态
  workspace: Workspace;   // 布局状态
  metadataCache: MetadataCache; // 缓存状态
  plugins: PluginManifest; // 插件状态
  // ...
}

// 插件中访问
export default class MyPlugin extends Plugin {
  onload() {
    // 直接访问全局状态
    const files = this.app.vault.getMarkdownFiles();
    const activeView = this.app.workspace.getActiveViewOfType(MarkdownView);
  }
}
```

### EventRef 事件系统

```typescript
// 订阅事件
this.registerEvent(
  this.app.vault.on('create', (file) => {
    console.log('File created:', file.path);
  })
);

this.registerEvent(
  this.app.workspace.on('active-leaf-change', (leaf) => {
    console.log('Active leaf:', leaf);
  })
);

this.registerEvent(
  this.app.metadataCache.on('changed', (file) => {
    console.log('Metadata changed:', file.path);
  })
);
```

### 插件数据存储

```typescript
interface MyPluginData {
  setting1: string;
  setting2: number;
  history: string[];
}

const DEFAULT_DATA: MyPluginData = {
  setting1: 'default',
  setting2: 42,
  history: []
};

export default class MyPlugin extends Plugin {
  data: MyPluginData;

  async onload() {
    // 加载数据
    this.data = Object.assign({}, DEFAULT_DATA, await this.loadData());
  }

  async saveData() {
    // 保存到 data.json
    await super.saveData(this.data);
  }
}

// 存储位置: .obsidian/plugins/my-plugin/data.json
```

---

## 对比分析

### 状态管理对比

| 方面 | VSCode | Obsidian |
|------|--------|----------|
| **架构** | 分散式（服务） | 集中式（App） |
| **事件系统** | EventEmitter | EventRef |
| **状态变更** | 服务方法 | 直接修改 + 事件 |
| **可观察性** | 明确的事件契约 | 隐式事件 |
| **类型安全** | 强类型 | 一般 |
| **持久化** | globalState/workspaceState | loadData/saveData |

---

## 对 AI Chat + Editor 应用的建议

### 推荐架构：服务 + 响应式状态

```typescript
// 状态定义
interface AppState {
  chat: ChatState;
  editor: EditorState;
  workspace: WorkspaceState;
  settings: SettingsState;
}

interface ChatState {
  sessions: ChatSession[];
  activeSessionId: string | null;
  isLoading: boolean;
}

interface EditorState {
  openFiles: OpenFile[];
  activeFileId: string | null;
  unsavedChanges: Set<string>;
}
```

### 状态存储

```typescript
import { create } from 'zustand';
import { subscribeWithSelector } from 'zustand/middleware';

// 聊天状态
interface ChatStore {
  sessions: ChatSession[];
  activeSessionId: string | null;
  isLoading: boolean;

  // Actions
  createSession: () => string;
  deleteSession: (id: string) => void;
  setActiveSession: (id: string) => void;
  addMessage: (sessionId: string, message: Message) => void;
  setLoading: (loading: boolean) => void;
}

const useChatStore = create<ChatStore>()(
  subscribeWithSelector((set, get) => ({
    sessions: [],
    activeSessionId: null,
    isLoading: false,

    createSession: () => {
      const id = generateId();
      const session: ChatSession = {
        id,
        title: 'New Chat',
        messages: [],
        createdAt: Date.now()
      };
      set(state => ({
        sessions: [...state.sessions, session],
        activeSessionId: id
      }));
      return id;
    },

    deleteSession: (id) => {
      set(state => ({
        sessions: state.sessions.filter(s => s.id !== id),
        activeSessionId: state.activeSessionId === id
          ? state.sessions[0]?.id ?? null
          : state.activeSessionId
      }));
    },

    addMessage: (sessionId, message) => {
      set(state => ({
        sessions: state.sessions.map(s =>
          s.id === sessionId
            ? { ...s, messages: [...s.messages, message] }
            : s
        )
      }));
    },

    setLoading: (loading) => set({ isLoading: loading })
  }))
);
```

### 事件系统

```typescript
class EventBus {
  private emitters = new Map<string, Set<Function>>();

  on<T>(event: string, handler: (data: T) => void): Disposable {
    if (!this.emitters.has(event)) {
      this.emitters.set(event, new Set());
    }
    this.emitters.get(event)!.add(handler);

    return {
      dispose: () => {
        this.emitters.get(event)?.delete(handler);
      }
    };
  }

  emit<T>(event: string, data: T): void {
    this.emitters.get(event)?.forEach(handler => handler(data));
  }

  once<T>(event: string, handler: (data: T) => void): Disposable {
    const disposable = this.on<T>(event, (data) => {
      disposable.dispose();
      handler(data);
    });
    return disposable;
  }
}

// 预定义事件
const events = {
  'chat:message-sent': null as unknown as (message: Message) => void,
  'chat:response-received': null as unknown as (response: Message) => void,
  'editor:file-opened': null as unknown as (file: OpenFile) => void,
  'editor:content-changed': null as unknown as (change: ContentChange) => void,
};

type EventMap = typeof events;
type EventName = keyof EventMap;
```

### 持久化

```typescript
// 持久化中间件
const persistMiddleware = (config) => (set, get, api) =>
  config(
    (...args) => {
      set(...args);
      // 自动持久化
      saveToStorage(get());
    },
    get,
    api
  );

// 或使用 zustand/persist
import { persist } from 'zustand/middleware';

const useSettingsStore = create<SettingsStore>()(
  persist(
    (set) => ({
      theme: 'system',
      fontSize: 14,
      // ...
    }),
    {
      name: 'app-settings',
      storage: createJSONStorage(() => localStorage)
    }
  )
);
```

### 跨进程同步

```typescript
// 主进程
class StateSync {
  private state: AppState;

  constructor() {
    // 监听渲染进程状态变化
    ipcMain.on('state:update', (event, patch) => {
      this.state = applyPatch(this.state, patch);
      // 广播给所有窗口
      BrowserWindow.getAllWindows().forEach(win => {
        if (win.webContents.id !== event.sender.id) {
          win.webContents.send('state:sync', this.state);
        }
      });
    });
  }
}

// 渲染进程
const useSyncedStore = create((set) => {
  // 初始化
  ipcRenderer.invoke('state:get').then(state => set(state));

  // 监听同步
  ipcRenderer.on('state:sync', (_, state) => set(state));

  return {
    // 状态更新时通知主进程
    updateState: (patch) => {
      set(state => applyPatch(state, patch));
      ipcRenderer.send('state:update', patch);
    }
  };
});
```

---

## 关键决策清单

1. **集中式还是分散式？**
   - 集中式：易于调试，但可能臃肿
   - 分散式：模块化，但协调复杂

2. **使用什么状态库？**
   - Zustand：轻量，React 友好
   - Redux：生态完善，但样板多
   - MobX：响应式，但学习成本高

3. **如何处理跨窗口/进程？**
   - 主进程作为状态源
   - 事件广播同步

4. **如何持久化？**
   - localStorage（小数据）
   - IndexedDB（大数据）
   - 文件系统（配置）

---

## 参考资料

- [Zustand](https://zustand-demo.pmnd.rs/)
- [VSCode Event System](https://github.com/microsoft/vscode/blob/main/src/vs/base/common/event.ts)
- [Electron IPC](https://www.electronjs.org/docs/latest/tutorial/ipc)
