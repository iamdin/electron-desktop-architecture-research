# 18. 配置系统

> **核心问题**：配置如何存储和访问？

---

## 概述

配置系统决定了：
- 用户如何定制应用
- 插件如何定义设置
- 配置如何持久化
- 配置 UI 如何生成

---

## VSCode 的配置系统

### 声明式配置

```json
// package.json
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
          "enumDescriptions": [
            "Light theme",
            "Dark theme",
            "Follow system"
          ],
          "default": "auto"
        },
        "myExtension.excludePatterns": {
          "type": "array",
          "items": { "type": "string" },
          "default": ["**/node_modules/**"],
          "description": "Glob patterns to exclude"
        },
        "myExtension.customMapping": {
          "type": "object",
          "additionalProperties": { "type": "string" },
          "default": {},
          "description": "Custom key-value mapping"
        }
      }
    }
  }
}
```

### 配置作用域

```json
{
  "myExtension.setting": {
    "type": "string",
    "scope": "resource",  // window | resource | language-overridable
    "default": "value"
  }
}
```

| 作用域 | 说明 |
|--------|------|
| `application` | 全局应用级 |
| `machine` | 机器级 |
| `window` | 窗口级 |
| `resource` | 文件夹/工作区级 |
| `language-overridable` | 可按语言覆盖 |

### 读取配置

```typescript
// 获取配置
const config = vscode.workspace.getConfiguration('myExtension');
const enable = config.get<boolean>('enable', true);
const maxItems = config.get<number>('maxItems', 10);

// 带资源 URI
const resourceConfig = vscode.workspace.getConfiguration('myExtension', uri);

// 监听变化
vscode.workspace.onDidChangeConfiguration(event => {
  if (event.affectsConfiguration('myExtension.enable')) {
    const newValue = config.get<boolean>('enable');
    console.log('Enable changed:', newValue);
  }
});
```

### 修改配置

```typescript
// 更新配置
await config.update('maxItems', 20, vscode.ConfigurationTarget.Global);
await config.update('maxItems', 30, vscode.ConfigurationTarget.Workspace);

// 检查配置
const inspection = config.inspect('maxItems');
// {
//   key: 'myExtension.maxItems',
//   defaultValue: 10,
//   globalValue: 20,
//   workspaceValue: 30,
//   workspaceFolderValue: undefined
// }
```

### 配置文件

```json
// settings.json (用户/工作区)
{
  "myExtension.enable": true,
  "myExtension.maxItems": 20,
  "[javascript]": {
    "myExtension.theme": "dark"
  }
}
```

---

## Obsidian 的配置系统

### 插件设置页

```typescript
interface MyPluginSettings {
  enableFeature: boolean;
  maxItems: number;
  theme: 'light' | 'dark' | 'auto';
}

const DEFAULT_SETTINGS: MyPluginSettings = {
  enableFeature: true,
  maxItems: 10,
  theme: 'auto'
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

    containerEl.createEl('h2', { text: 'My Plugin Settings' });

    // 开关
    new Setting(containerEl)
      .setName('Enable Feature')
      .setDesc('Toggle this feature on or off')
      .addToggle(toggle => toggle
        .setValue(this.plugin.settings.enableFeature)
        .onChange(async (value) => {
          this.plugin.settings.enableFeature = value;
          await this.plugin.saveSettings();
        }));

    // 数字
    new Setting(containerEl)
      .setName('Max Items')
      .setDesc('Maximum number of items to display')
      .addSlider(slider => slider
        .setLimits(1, 100, 1)
        .setValue(this.plugin.settings.maxItems)
        .setDynamicTooltip()
        .onChange(async (value) => {
          this.plugin.settings.maxItems = value;
          await this.plugin.saveSettings();
        }));

    // 下拉选择
    new Setting(containerEl)
      .setName('Theme')
      .setDesc('Select theme preference')
      .addDropdown(dropdown => dropdown
        .addOption('light', 'Light')
        .addOption('dark', 'Dark')
        .addOption('auto', 'Auto')
        .setValue(this.plugin.settings.theme)
        .onChange(async (value) => {
          this.plugin.settings.theme = value as 'light' | 'dark' | 'auto';
          await this.plugin.saveSettings();
        }));

    // 文本输入
    new Setting(containerEl)
      .setName('API Key')
      .setDesc('Enter your API key')
      .addText(text => text
        .setPlaceholder('sk-...')
        .setValue(this.plugin.settings.apiKey || '')
        .onChange(async (value) => {
          this.plugin.settings.apiKey = value;
          await this.plugin.saveSettings();
        }));
  }
}
```

### 注册设置页

```typescript
export default class MyPlugin extends Plugin {
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

### 存储位置

```
.obsidian/plugins/my-plugin/data.json
```

---

## 对比分析

### 配置系统对比

| 方面 | VSCode | Obsidian |
|------|--------|----------|
| **定义方式** | JSON Schema | 代码构建 UI |
| **UI 生成** | 自动生成 | 手动编写 |
| **作用域** | 多级（用户/工作区/文件夹） | 单一 |
| **验证** | JSON Schema 验证 | 无内置验证 |
| **存储格式** | JSON (settings.json) | JSON (data.json) |
| **变更通知** | onDidChangeConfiguration | 无内置 |

---

## 对 Coding Agent Desktop 应用的建议

### 配置定义

```typescript
// 配置 Schema
interface SettingsSchema {
  general: {
    theme: 'light' | 'dark' | 'system';
    language: string;
    fontSize: number;
  };
  ai: {
    provider: 'openai' | 'anthropic' | 'custom';
    model: string;
    apiKey: string;
    temperature: number;
    maxTokens: number;
    streamResponse: boolean;
  };
  editor: {
    tabSize: number;
    wordWrap: boolean;
    lineNumbers: boolean;
    minimap: boolean;
  };
  chat: {
    saveHistory: boolean;
    maxHistoryItems: number;
    showTimestamps: boolean;
  };
}

const DEFAULT_SETTINGS: SettingsSchema = {
  general: {
    theme: 'system',
    language: 'en',
    fontSize: 14
  },
  ai: {
    provider: 'openai',
    model: 'gpt-4',
    apiKey: '',
    temperature: 0.7,
    maxTokens: 4096,
    streamResponse: true
  },
  editor: {
    tabSize: 2,
    wordWrap: true,
    lineNumbers: true,
    minimap: false
  },
  chat: {
    saveHistory: true,
    maxHistoryItems: 100,
    showTimestamps: true
  }
};
```

### 配置服务

```typescript
class SettingsService {
  private settings: SettingsSchema;
  private listeners: Set<(settings: SettingsSchema) => void> = new Set();

  constructor(private storage: StorageService) {
    this.settings = this.load();
  }

  private load(): SettingsSchema {
    const saved = this.storage.get<SettingsSchema>('settings');
    return this.merge(DEFAULT_SETTINGS, saved || {});
  }

  private merge(defaults: any, saved: any): any {
    const result = { ...defaults };
    for (const key in saved) {
      if (typeof defaults[key] === 'object' && !Array.isArray(defaults[key])) {
        result[key] = this.merge(defaults[key], saved[key]);
      } else if (saved[key] !== undefined) {
        result[key] = saved[key];
      }
    }
    return result;
  }

  get<T>(path: string): T {
    return path.split('.').reduce((obj, key) => obj?.[key], this.settings) as T;
  }

  set<T>(path: string, value: T): void {
    const keys = path.split('.');
    const lastKey = keys.pop()!;
    const target = keys.reduce((obj, key) => obj[key], this.settings as any);
    target[lastKey] = value;

    this.save();
    this.notify();
  }

  private save(): void {
    this.storage.set('settings', this.settings);
  }

  private notify(): void {
    this.listeners.forEach(fn => fn(this.settings));
  }

  onChange(listener: (settings: SettingsSchema) => void): Disposable {
    this.listeners.add(listener);
    return { dispose: () => this.listeners.delete(listener) };
  }
}
```

### 设置 UI 组件

```tsx
// React 设置页
function SettingsPage() {
  const settings = useSettings();
  const updateSetting = useSettingUpdater();

  return (
    <div className="settings-page">
      <SettingsSection title="General">
        <SettingItem
          label="Theme"
          description="Choose your preferred theme"
        >
          <Select
            value={settings.general.theme}
            onChange={(value) => updateSetting('general.theme', value)}
            options={[
              { value: 'light', label: 'Light' },
              { value: 'dark', label: 'Dark' },
              { value: 'system', label: 'System' }
            ]}
          />
        </SettingItem>

        <SettingItem
          label="Font Size"
          description="Editor font size in pixels"
        >
          <Slider
            value={settings.general.fontSize}
            onChange={(value) => updateSetting('general.fontSize', value)}
            min={10}
            max={24}
            step={1}
          />
        </SettingItem>
      </SettingsSection>

      <SettingsSection title="AI">
        <SettingItem
          label="API Key"
          description="Your AI provider API key"
        >
          <PasswordInput
            value={settings.ai.apiKey}
            onChange={(value) => updateSetting('ai.apiKey', value)}
            placeholder="sk-..."
          />
        </SettingItem>

        <SettingItem
          label="Temperature"
          description="Creativity level (0-1)"
        >
          <Slider
            value={settings.ai.temperature}
            onChange={(value) => updateSetting('ai.temperature', value)}
            min={0}
            max={1}
            step={0.1}
          />
        </SettingItem>
      </SettingsSection>
    </div>
  );
}
```

### 插件配置扩展

```typescript
// 插件声明配置
// manifest.json
{
  "configuration": {
    "title": "My Plugin",
    "properties": {
      "myPlugin.enabled": {
        "type": "boolean",
        "default": true,
        "description": "Enable the plugin"
      }
    }
  }
}

// 插件读取配置
class MyPlugin extends BasePlugin {
  async onActivate() {
    const enabled = this.context.settings.get<boolean>('myPlugin.enabled');

    this.context.settings.onChange((settings) => {
      // 响应配置变化
    });
  }
}
```

---

## 关键决策清单

1. **声明式还是代码式？**
   - 声明式：自动生成 UI，易于验证
   - 代码式：灵活，但需要手动构建

2. **支持多少作用域？**
   - 单一作用域：简单
   - 多级作用域：灵活但复杂

3. **如何验证配置？**
   - JSON Schema
   - 运行时验证
   - TypeScript 类型

4. **如何处理敏感配置？**
   - 加密存储
   - 系统钥匙串
   - 环境变量

---

## 参考资料

- [VSCode Configuration](https://code.visualstudio.com/api/references/contribution-points#contributes.configuration)
- [JSON Schema](https://json-schema.org/)
- [Obsidian Settings](https://docs.obsidian.md/Plugins/User+interface/Settings)
