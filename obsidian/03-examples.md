# Obsidian 插件示例

> 代码示例与实践用法

## 1. 基础插件结构

```typescript
import { Plugin } from 'obsidian';

export default class MyPlugin extends Plugin {
  async onload() {
    console.log('Plugin loaded');
    // 注册命令、视图、事件等
  }

  async onunload() {
    console.log('Plugin unloaded');
    // 清理非注册资源（通常不需要）
  }
}
```

---

## 2. 命令注册

### 无条件命令

```typescript
this.addCommand({
  id: 'open-my-view',
  name: 'Open My View',
  callback: () => {
    this.activateView();
  }
});
```

### 条件命令

```typescript
this.addCommand({
  id: 'my-conditional-command',
  name: 'Do Something (when available)',
  checkCallback: (checking) => {
    const view = this.app.workspace.getActiveViewOfType(MyView);
    if (view) {
      if (!checking) {
        view.doSomething();
      }
      return true;  // 命令可用
    }
    return false;   // 命令不可用
  }
});
```

### 编辑器命令

```typescript
this.addCommand({
  id: 'insert-timestamp',
  name: 'Insert Timestamp',
  editorCallback: (editor, ctx) => {
    const cursor = editor.getCursor();
    editor.replaceRange(new Date().toISOString(), cursor);
  }
});
```

---

## 3. 自定义视图

```typescript
import { ItemView, WorkspaceLeaf } from 'obsidian';

export const VIEW_TYPE_MY_VIEW = 'my-view';

export class MyView extends ItemView {
  constructor(leaf: WorkspaceLeaf) {
    super(leaf);
  }

  getViewType(): string {
    return VIEW_TYPE_MY_VIEW;
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
    container.createEl('h1', { text: 'Hello!' });
  }

  async onClose() {
    // 清理资源
  }
}
```

### 注册和激活视图

```typescript
class MyPlugin extends Plugin {
  async onload() {
    this.registerView(VIEW_TYPE_MY_VIEW, (leaf) => new MyView(leaf));

    this.addCommand({
      id: 'open-my-view',
      name: 'Open My View',
      callback: () => this.activateView(),
    });

    this.addRibbonIcon('dice', 'Open My View', () => this.activateView());
  }

  async activateView() {
    const existing = this.app.workspace.getLeavesOfType(VIEW_TYPE_MY_VIEW);
    if (existing.length > 0) {
      this.app.workspace.revealLeaf(existing[0]);
      return;
    }

    const leaf = this.app.workspace.getRightLeaf(false);
    if (leaf) {
      await leaf.setViewState({
        type: VIEW_TYPE_MY_VIEW,
        active: true,
      });
    }
  }

  async onunload() {
    this.app.workspace.detachLeavesOfType(VIEW_TYPE_MY_VIEW);
  }
}
```

---

## 4. 设置标签页

```typescript
interface MyPluginSettings {
  apiKey: string;
  enableFeatureX: boolean;
  maxResults: number;
}

const DEFAULT_SETTINGS: MyPluginSettings = {
  apiKey: '',
  enableFeatureX: true,
  maxResults: 10,
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

    // 文本输入
    new Setting(containerEl)
      .setName('API Key')
      .setDesc('Enter your API key here')
      .addText((text) =>
        text
          .setPlaceholder('sk-...')
          .setValue(this.plugin.settings.apiKey)
          .onChange(async (value) => {
            this.plugin.settings.apiKey = value;
            await this.plugin.saveSettings();
          })
      );

    // 开关
    new Setting(containerEl)
      .setName('Enable Feature X')
      .addToggle((toggle) =>
        toggle
          .setValue(this.plugin.settings.enableFeatureX)
          .onChange(async (value) => {
            this.plugin.settings.enableFeatureX = value;
            await this.plugin.saveSettings();
          })
      );

    // 下拉选择
    new Setting(containerEl)
      .setName('Max Results')
      .addDropdown((dropdown) =>
        dropdown
          .addOption('5', '5')
          .addOption('10', '10')
          .addOption('20', '20')
          .setValue(String(this.plugin.settings.maxResults))
          .onChange(async (value) => {
            this.plugin.settings.maxResults = parseInt(value);
            await this.plugin.saveSettings();
          })
      );
  }
}

// 在插件中使用
class MyPlugin extends Plugin {
  settings: MyPluginSettings;

  async onload() {
    await this.loadSettings();
    this.addSettingTab(new MySettingTab(this.app, this));
  }

  async loadSettings() {
    this.settings = Object.assign({}, DEFAULT_SETTINGS, await this.loadData());
  }

  async saveSettings() {
    await this.saveData(this.settings);
  }
}
```

---

## 5. 事件订阅

### 文件事件

```typescript
// 文件创建
this.registerEvent(
  this.app.vault.on('create', (file: TAbstractFile) => {
    if (file instanceof TFile) {
      console.log('New file:', file.path);
    }
  })
);

// 文件修改
this.registerEvent(
  this.app.vault.on('modify', (file: TAbstractFile) => {
    console.log('Modified:', file.path);
  })
);

// 文件重命名
this.registerEvent(
  this.app.vault.on('rename', (file: TAbstractFile, oldPath: string) => {
    console.log(`Renamed: ${oldPath} → ${file.path}`);
  })
);
```

### 工作区事件

```typescript
// 切换标签页
this.registerEvent(
  this.app.workspace.on('active-leaf-change', (leaf: WorkspaceLeaf | null) => {
    if (leaf) {
      console.log('Active view:', leaf.view.getViewType());
    }
  })
);

// 打开文件
this.registerEvent(
  this.app.workspace.on('file-open', (file: TFile | null) => {
    if (file) {
      console.log('Opened:', file.path);
    }
  })
);

// 编辑器变化
this.registerEvent(
  this.app.workspace.on('editor-change', (editor: Editor, info: MarkdownView) => {
    console.log('Editor changed in:', info.file?.path);
  })
);
```

---

## 6. 文件操作

### 读写文件

```typescript
// 读取文件
const file = this.app.vault.getFileByPath('notes/todo.md');
if (file) {
  const content = await this.app.vault.read(file);
  // 或使用缓存（性能更好）
  const cachedContent = await this.app.vault.cachedRead(file);
}

// 修改文件
await this.app.vault.modify(file, newContent);

// 原子操作：读取→处理→写入
await this.app.vault.process(file, (content) => {
  return content.replace(/old/g, 'new');
});

// 追加内容
await this.app.vault.append(file, '\n\nNew content');

// 创建文件
const newFile = await this.app.vault.create('path/to/new.md', 'content');
```

### 获取文件元数据

```typescript
const file = this.app.vault.getFileByPath('notes/todo.md');
if (file) {
  const cache = this.app.metadataCache.getFileCache(file);
  if (cache) {
    console.log('Frontmatter:', cache.frontmatter);
    console.log('Headings:', cache.headings);
    console.log('Links:', cache.links);
    console.log('Tags:', cache.tags);
  }
}

// 解析链接
const linkedFile = this.app.metadataCache.getFirstLinkpathDest(
  'other-note',
  'notes/current.md'
);
```

---

## 7. UI 组件

### Modal（模态框）

```typescript
import { App, Modal, Setting } from 'obsidian';

class MyModal extends Modal {
  result: string;
  onSubmit: (result: string) => void;

  constructor(app: App, onSubmit: (result: string) => void) {
    super(app);
    this.onSubmit = onSubmit;
  }

  onOpen() {
    const { contentEl } = this;
    contentEl.createEl('h1', { text: 'Enter a value' });

    new Setting(contentEl)
      .setName('Value')
      .addText((text) =>
        text.onChange((value) => {
          this.result = value;
        })
      );

    new Setting(contentEl)
      .addButton((btn) =>
        btn
          .setButtonText('Submit')
          .setCta()
          .onClick(() => {
            this.close();
            this.onSubmit(this.result);
          })
      );
  }

  onClose() {
    const { contentEl } = this;
    contentEl.empty();
  }
}

// 使用
new MyModal(this.app, (result) => {
  console.log('User entered:', result);
}).open();
```

### Notice（通知）

```typescript
import { Notice } from 'obsidian';

// 简单通知
new Notice('Operation completed!');

// 自定义时长（毫秒）
new Notice('This will stay longer', 10000);

// 带 HTML
const notice = new Notice('');
notice.noticeEl.createEl('strong', { text: 'Success!' });
notice.noticeEl.createEl('br');
notice.noticeEl.createEl('span', { text: 'File saved.' });

// 手动隐藏
notice.hide();
```

### Ribbon 和 StatusBar

```typescript
// Ribbon（左侧边栏图标）
const ribbonIcon = this.addRibbonIcon('dice', 'My Plugin', (evt) => {
  new Notice('Ribbon clicked!');
});
ribbonIcon.addClass('my-ribbon-class');

// StatusBar（底部状态栏，仅桌面版）
const statusBar = this.addStatusBarItem();
statusBar.setText('Ready');
```

---

## 8. 编辑器扩展

### Markdown 后处理器

```typescript
// 处理所有渲染后的内容
this.registerMarkdownPostProcessor((el, ctx) => {
  el.querySelectorAll('a.external-link').forEach((link) => {
    const icon = document.createElement('span');
    icon.textContent = ' ↗';
    link.appendChild(icon);
  });
});

// 处理特定代码块
this.registerMarkdownCodeBlockProcessor('chart', (source, el, ctx) => {
  try {
    const config = JSON.parse(source);
    el.createEl('div', { cls: 'chart-container' });
    // 渲染图表...
  } catch (e) {
    el.createEl('pre', { text: 'Invalid JSON: ' + e.message });
  }
});
```

### 编辑器建议（自动补全）

```typescript
import { EditorSuggest, EditorPosition, Editor } from 'obsidian';

class MyEditorSuggest extends EditorSuggest<Suggestion> {
  onTrigger(cursor: EditorPosition, editor: Editor) {
    const line = editor.getLine(cursor.line);
    const match = line.slice(0, cursor.ch).match(/:(\w*)$/);

    if (match) {
      return {
        start: { line: cursor.line, ch: cursor.ch - match[0].length },
        end: cursor,
        query: match[1],
      };
    }
    return null;
  }

  getSuggestions(ctx) {
    return [
      { text: ':smile:', description: '😊' },
      { text: ':heart:', description: '❤️' },
    ].filter((e) => e.text.includes(ctx.query));
  }

  renderSuggestion(suggestion, el) {
    el.createSpan({ text: suggestion.description + ' ' });
    el.createSpan({ text: suggestion.text });
  }

  selectSuggestion(suggestion) {
    if (this.context) {
      this.context.editor.replaceRange(
        suggestion.description,
        this.context.start,
        this.context.end
      );
    }
  }
}

// 注册
this.registerEditorSuggest(new MyEditorSuggest(this.app));
```

---

## 9. 定时器和 DOM 事件

```typescript
// 定时器（自动清理）
this.registerInterval(
  window.setInterval(() => {
    console.log('Tick');
  }, 1000)
);

// DOM 事件（自动清理）
this.registerDomEvent(document, 'click', (evt) => {
  console.log('Clicked');
});
```

---

## 参考

- [Obsidian Sample Plugin](https://github.com/obsidianmd/obsidian-sample-plugin)
- [Obsidian Developer Docs](https://docs.obsidian.md/)
