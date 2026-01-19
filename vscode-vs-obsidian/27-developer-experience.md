# 27. 开发者体验

> **核心问题**：如何提供良好的插件开发体验？

---

## 概述

开发者体验决定了：
- 插件开发的学习曲线
- 开发调试的便利性
- 文档和工具的完善度
- 社区生态的活跃度

---

## VSCode 的开发者体验

### 脚手架工具

```bash
# 使用 Yeoman 生成器
npm install -g yo generator-code
yo code

# 交互式选择
? What type of extension do you want to create?
  > New Extension (TypeScript)
    New Extension (JavaScript)
    New Color Theme
    New Language Support
    New Code Snippets
    New Keymap
    New Extension Pack
    New Language Pack

# 生成的项目结构
my-extension/
├── .vscode/
│   ├── launch.json      # 调试配置
│   ├── tasks.json       # 构建任务
│   └── extensions.json  # 推荐扩展
├── src/
│   └── extension.ts     # 入口文件
├── package.json
├── tsconfig.json
├── .vscodeignore        # 打包忽略文件
└── README.md
```

### 调试支持

```json
// .vscode/launch.json
{
  "version": "0.2.0",
  "configurations": [
    {
      "name": "Run Extension",
      "type": "extensionHost",
      "request": "launch",
      "args": [
        "--extensionDevelopmentPath=${workspaceFolder}"
      ],
      "outFiles": [
        "${workspaceFolder}/out/**/*.js"
      ],
      "preLaunchTask": "npm: watch"
    },
    {
      "name": "Extension Tests",
      "type": "extensionHost",
      "request": "launch",
      "args": [
        "--extensionDevelopmentPath=${workspaceFolder}",
        "--extensionTestsPath=${workspaceFolder}/out/test/suite/index"
      ],
      "outFiles": [
        "${workspaceFolder}/out/test/**/*.js"
      ],
      "preLaunchTask": "npm: watch"
    }
  ]
}
```

### 类型定义

```typescript
// @types/vscode 提供完整类型
import * as vscode from 'vscode';

// 完整的智能提示
vscode.window.showInformationMessage('Hello');

// 接口定义清晰
interface TextDocument {
  readonly uri: Uri;
  readonly fileName: string;
  readonly languageId: string;
  readonly version: number;
  readonly isDirty: boolean;
  getText(range?: Range): string;
  // ...
}
```

### 测试框架

```typescript
// 使用 Mocha + @vscode/test-electron
import * as vscode from 'vscode';
import * as assert from 'assert';

suite('Extension Test Suite', () => {
  vscode.window.showInformationMessage('Start all tests.');

  test('Sample test', async () => {
    // 等待扩展激活
    const ext = vscode.extensions.getExtension('my.extension');
    await ext?.activate();

    // 执行命令
    await vscode.commands.executeCommand('my.command');

    // 断言
    assert.strictEqual(1 + 1, 2);
  });

  test('Document test', async () => {
    // 打开文档
    const doc = await vscode.workspace.openTextDocument({
      content: 'Hello World',
      language: 'plaintext'
    });

    const editor = await vscode.window.showTextDocument(doc);

    // 编辑文档
    await editor.edit(editBuilder => {
      editBuilder.insert(new vscode.Position(0, 0), 'Start: ');
    });

    assert.strictEqual(doc.getText(), 'Start: Hello World');
  });
});
```

### 文档资源

```
https://code.visualstudio.com/api
├── Get Started
│   ├── Your First Extension
│   ├── Extension Anatomy
│   └── Wrapping Up
├── Extension Capabilities
│   ├── Common Capabilities
│   ├── Theming
│   ├── Extending Workbench
│   └── ...
├── Extension Guides
│   ├── Command
│   ├── Color Theme
│   ├── Language Extensions
│   └── ...
├── Language Extensions
│   ├── Syntax Highlighting
│   ├── Semantic Highlighting
│   └── Language Server
├── Testing
├── Publishing
└── API Reference
```

---

## Obsidian 的开发者体验

### 示例插件模板

```bash
# 克隆官方示例
git clone https://github.com/obsidianmd/obsidian-sample-plugin

# 项目结构
obsidian-sample-plugin/
├── src/
│   └── main.ts
├── manifest.json
├── package.json
├── tsconfig.json
├── esbuild.config.mjs
└── versions.json
```

### 开发流程

```typescript
// main.ts
import { App, Plugin, PluginSettingTab, Setting } from 'obsidian';

interface MyPluginSettings {
  mySetting: string;
}

const DEFAULT_SETTINGS: MyPluginSettings = {
  mySetting: 'default'
};

export default class MyPlugin extends Plugin {
  settings: MyPluginSettings;

  async onload() {
    await this.loadSettings();

    // 添加命令
    this.addCommand({
      id: 'open-sample-modal',
      name: 'Open Sample Modal',
      callback: () => {
        new SampleModal(this.app).open();
      }
    });

    // 添加设置页
    this.addSettingTab(new SampleSettingTab(this.app, this));
  }

  onunload() {
    // 清理
  }

  async loadSettings() {
    this.settings = Object.assign({}, DEFAULT_SETTINGS, await this.loadData());
  }

  async saveSettings() {
    await this.saveData(this.settings);
  }
}
```

### 调试方式

```javascript
// 使用 console.log
console.log('Debug:', this.app);

// 开发者工具
// Cmd/Ctrl + Shift + I 打开

// 热重载（需要手动）
// 1. 修改代码
// 2. npm run dev（监听模式）
// 3. Cmd/Ctrl + R 刷新 Obsidian

// 或使用 Hot Reload 插件
```

### 类型定义

```typescript
// obsidian.d.ts（由 Obsidian 提供）
// 但文档有限，很多需要查看源码或社区

import { App, TFile, Plugin } from 'obsidian';

// 类型注释
declare module 'obsidian' {
  interface App {
    plugins: {
      enabledPlugins: Set<string>;
      plugins: Record<string, Plugin>;
    };
    // 其他内部 API
  }
}
```

### 社区资源

```
官方资源:
- https://docs.obsidian.md/Plugins
- GitHub: obsidian-sample-plugin
- Discord: Obsidian Members Group

社区资源:
- https://marcus.se.net/obsidian-plugin-docs/
- obsidian-typings（增强类型）
- 各种示例插件
```

---

## 对比分析

### 开发者体验对比

| 方面 | VSCode | Obsidian |
|------|--------|----------|
| **脚手架** | 官方 Yeoman 生成器 | 示例项目克隆 |
| **调试** | 专用调试配置 | 开发者工具 + 日志 |
| **热重载** | Extension Host 支持 | 需手动刷新 |
| **类型定义** | 完整 @types/vscode | 有限 obsidian.d.ts |
| **文档** | 详尽官方文档 | 基础文档 + 社区补充 |
| **测试** | 官方测试框架 | 无官方方案 |
| **示例代码** | 丰富 | 有限 |

---

## 对 AI Chat + Editor 应用的建议

### CLI 脚手架工具

```typescript
// create-plugin CLI
import { Command } from 'commander';
import inquirer from 'inquirer';
import fs from 'fs-extra';
import path from 'path';

const program = new Command();

program
  .name('create-plugin')
  .description('Create a new plugin project')
  .argument('[name]', 'Plugin name')
  .option('-t, --template <template>', 'Template to use', 'basic')
  .action(async (name, options) => {
    // 交互式收集信息
    const answers = await inquirer.prompt([
      {
        type: 'input',
        name: 'name',
        message: 'Plugin name:',
        default: name || 'my-plugin'
      },
      {
        type: 'list',
        name: 'template',
        message: 'Choose a template:',
        choices: [
          { name: 'Basic - Simple plugin', value: 'basic' },
          { name: 'With Settings - Plugin with settings page', value: 'settings' },
          { name: 'AI Integration - Plugin with AI features', value: 'ai' },
          { name: 'Editor Extension - Editor enhancements', value: 'editor' }
        ],
        default: options.template
      },
      {
        type: 'input',
        name: 'description',
        message: 'Description:'
      },
      {
        type: 'input',
        name: 'author',
        message: 'Author:'
      }
    ]);

    // 生成项目
    await generateProject(answers);

    console.log(`
✨ Plugin created successfully!

Next steps:
  cd ${answers.name}
  npm install
  npm run dev

Documentation: https://docs.yourapp.com/plugins
`);
  });

async function generateProject(config: PluginConfig) {
  const templateDir = path.join(__dirname, 'templates', config.template);
  const targetDir = path.join(process.cwd(), config.name);

  // 复制模板
  await fs.copy(templateDir, targetDir);

  // 替换占位符
  await replaceInFiles(targetDir, {
    '{{PLUGIN_NAME}}': config.name,
    '{{PLUGIN_ID}}': config.name.toLowerCase().replace(/\s+/g, '-'),
    '{{DESCRIPTION}}': config.description,
    '{{AUTHOR}}': config.author
  });
}
```

### 开发服务器

```typescript
// 开发服务器，支持热重载
import chokidar from 'chokidar';
import esbuild from 'esbuild';
import WebSocket from 'ws';

class DevServer {
  private wss: WebSocket.Server;
  private buildContext: esbuild.BuildContext;

  async start(options: DevServerOptions) {
    // 启动 WebSocket 服务器（用于热重载通知）
    this.wss = new WebSocket.Server({ port: 35729 });

    // 启动 esbuild 监听模式
    this.buildContext = await esbuild.context({
      entryPoints: ['src/main.ts'],
      bundle: true,
      outfile: 'dist/main.js',
      platform: 'node',
      external: ['obsidian', 'electron'],
      sourcemap: 'inline',
      plugins: [
        {
          name: 'reload-notifier',
          setup(build) {
            build.onEnd(() => {
              // 通知客户端重载
              this.notifyReload();
            });
          }
        }
      ]
    });

    await this.buildContext.watch();

    console.log('Dev server started');
    console.log('Watching for changes...');
  }

  private notifyReload() {
    this.wss.clients.forEach(client => {
      if (client.readyState === WebSocket.OPEN) {
        client.send(JSON.stringify({ type: 'reload' }));
      }
    });
  }
}

// 在应用中注入热重载客户端
const hotReloadClient = `
  const ws = new WebSocket('ws://localhost:35729');
  ws.onmessage = (event) => {
    const data = JSON.parse(event.data);
    if (data.type === 'reload') {
      // 重载插件
      pluginService.reload('dev-plugin');
    }
  };
`;
```

### API 文档生成

```typescript
// 使用 TypeDoc 生成 API 文档
// typedoc.json
{
  "entryPoints": ["src/api/index.ts"],
  "out": "docs/api",
  "theme": "default",
  "name": "Plugin API",
  "includeVersion": true,
  "excludePrivate": true,
  "excludeProtected": true,
  "readme": "docs/api-intro.md"
}

// 带有详细注释的 API
/**
 * 注册一个新命令
 *
 * @param command - 命令配置
 * @returns 用于取消注册的 Disposable
 *
 * @example
 * ```typescript
 * const dispose = plugin.registerCommand({
 *   id: 'my-command',
 *   name: 'My Command',
 *   callback: () => {
 *     console.log('Command executed!');
 *   }
 * });
 * ```
 */
registerCommand(command: CommandConfig): Disposable;
```

### 测试框架

```typescript
// 内置测试工具
import { PluginTester } from '@yourapp/testing';

describe('MyPlugin', () => {
  let tester: PluginTester;

  beforeEach(async () => {
    tester = new PluginTester();
    await tester.setup();
  });

  afterEach(async () => {
    await tester.teardown();
  });

  it('should register command', async () => {
    const plugin = await tester.loadPlugin('./dist/main.js');

    expect(tester.hasCommand('my-command')).toBe(true);
  });

  it('should respond to command', async () => {
    const plugin = await tester.loadPlugin('./dist/main.js');

    await tester.executeCommand('my-command');

    expect(tester.getNotifications()).toContain('Command executed!');
  });

  it('should save settings', async () => {
    const plugin = await tester.loadPlugin('./dist/main.js');

    await plugin.saveSettings({ theme: 'dark' });

    const settings = await plugin.loadSettings();
    expect(settings.theme).toBe('dark');
  });
});

// PluginTester 实现
class PluginTester {
  private app: MockApp;
  private plugins: Map<string, Plugin> = new Map();

  async setup() {
    this.app = new MockApp();
    // 初始化虚拟文件系统
    // 初始化虚拟工作区
  }

  async loadPlugin(path: string): Promise<Plugin> {
    const module = require(path);
    const PluginClass = module.default;

    const plugin = new PluginClass(this.app, this.createManifest());
    await plugin.onload();

    this.plugins.set(plugin.manifest.id, plugin);
    return plugin;
  }

  hasCommand(id: string): boolean {
    return this.app.commands.has(id);
  }

  async executeCommand(id: string, ...args: any[]): Promise<void> {
    const command = this.app.commands.get(id);
    if (command) {
      await command.callback(...args);
    }
  }

  getNotifications(): string[] {
    return this.app.notifications;
  }
}
```

### 调试配置

```json
// .vscode/launch.json
{
  "version": "0.2.0",
  "configurations": [
    {
      "name": "Debug Plugin",
      "type": "node",
      "request": "launch",
      "runtimeExecutable": "${workspaceFolder}/node_modules/.bin/electron",
      "args": [
        "${workspaceFolder}/../app/main.js",
        "--plugin-dev=${workspaceFolder}"
      ],
      "outFiles": ["${workspaceFolder}/dist/**/*.js"],
      "sourceMaps": true,
      "preLaunchTask": "npm: watch"
    },
    {
      "name": "Run Tests",
      "type": "node",
      "request": "launch",
      "program": "${workspaceFolder}/node_modules/jest/bin/jest",
      "args": ["--runInBand"],
      "console": "integratedTerminal"
    }
  ]
}
```

### 文档网站

```typescript
// 使用 Docusaurus 或 VitePress
// docusaurus.config.js
module.exports = {
  title: 'Plugin Development',
  tagline: 'Build amazing plugins',
  url: 'https://docs.yourapp.com',
  baseUrl: '/plugins/',
  onBrokenLinks: 'throw',
  favicon: 'img/favicon.ico',
  themeConfig: {
    navbar: {
      title: 'Plugin Docs',
      items: [
        { to: '/getting-started', label: 'Getting Started' },
        { to: '/api', label: 'API Reference' },
        { to: '/guides', label: 'Guides' },
        { to: '/examples', label: 'Examples' }
      ]
    },
    sidebar: {
      docs: [
        {
          type: 'category',
          label: 'Getting Started',
          items: ['introduction', 'quick-start', 'project-structure']
        },
        {
          type: 'category',
          label: 'Core Concepts',
          items: ['plugin-lifecycle', 'commands', 'settings', 'events']
        },
        {
          type: 'category',
          label: 'API Reference',
          items: ['plugin-api', 'editor-api', 'workspace-api', 'ai-api']
        }
      ]
    }
  }
};
```

### 开发者门户

```tsx
// 开发者控制台
function DeveloperPortal() {
  return (
    <div className="dev-portal">
      <header>
        <h1>Plugin Developer Portal</h1>
        <nav>
          <Link to="/docs">Documentation</Link>
          <Link to="/api">API Reference</Link>
          <Link to="/playground">Playground</Link>
          <Link to="/publish">Publish</Link>
        </nav>
      </header>

      <main>
        <section className="quick-start">
          <h2>Quick Start</h2>
          <CodeBlock language="bash">
            {`npx create-plugin my-plugin
cd my-plugin
npm install
npm run dev`}
          </CodeBlock>
        </section>

        <section className="featured-guides">
          <h2>Featured Guides</h2>
          <GuideCard title="Your First Plugin" />
          <GuideCard title="Working with the Editor" />
          <GuideCard title="AI Integration" />
        </section>

        <section className="playground">
          <h2>Interactive Playground</h2>
          <PluginPlayground />
        </section>
      </main>
    </div>
  );
}

// 在线 Playground
function PluginPlayground() {
  const [code, setCode] = useState(defaultPluginCode);
  const [output, setOutput] = useState('');

  const runPlugin = async () => {
    const sandbox = createSandbox();
    const result = await sandbox.run(code);
    setOutput(result);
  };

  return (
    <div className="playground">
      <CodeEditor value={code} onChange={setCode} />
      <button onClick={runPlugin}>Run</button>
      <OutputPanel output={output} />
    </div>
  );
}
```

---

## 关键决策清单

1. **脚手架工具？**
   - CLI 工具
   - 模板仓库

2. **调试支持程度？**
   - 专用调试配置
   - 日志 + DevTools

3. **文档形式？**
   - 静态网站
   - 交互式文档

4. **测试支持？**
   - 提供测试框架
   - 依赖社区方案

---

## 参考资料

- [VSCode Extension API](https://code.visualstudio.com/api)
- [Yeoman](https://yeoman.io/)
- [Obsidian Plugin Docs](https://docs.obsidian.md/Plugins)
- [Docusaurus](https://docusaurus.io/)
