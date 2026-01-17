# 23. 终端集成

> **核心问题**：如何集成终端功能？

---

## 概述

终端集成决定了：
- 是否内置终端
- 终端如何与编辑器交互
- 如何支持多终端
- 终端输出如何处理

---

## VSCode 的终端系统

### 内置终端

VSCode 通过 xterm.js 实现完整的终端模拟器：

```typescript
// 创建终端
const terminal = vscode.window.createTerminal({
  name: 'My Terminal',
  shellPath: '/bin/zsh',
  shellArgs: ['-l'],
  cwd: '/path/to/project',
  env: { NODE_ENV: 'development' },
  iconPath: new vscode.ThemeIcon('terminal'),
  color: new vscode.ThemeColor('terminal.ansiGreen')
});

// 显示终端
terminal.show();

// 发送命令
terminal.sendText('npm run dev');
terminal.sendText('echo "Hello"', false); // 不添加换行

// 隐藏终端
terminal.hide();

// 销毁终端
terminal.dispose();
```

### 终端配置

```json
// settings.json
{
  "terminal.integrated.defaultProfile.osx": "zsh",
  "terminal.integrated.profiles.osx": {
    "bash": {
      "path": "bash",
      "args": ["-l"]
    },
    "zsh": {
      "path": "zsh",
      "args": ["-l"]
    },
    "fish": {
      "path": "fish",
      "args": ["-l"]
    }
  },
  "terminal.integrated.fontSize": 14,
  "terminal.integrated.fontFamily": "MesloLGS NF",
  "terminal.integrated.cursorStyle": "line",
  "terminal.integrated.cursorBlinking": true,
  "terminal.integrated.scrollback": 10000
}
```

### 终端 API

```typescript
// 监听终端事件
vscode.window.onDidOpenTerminal(terminal => {
  console.log('Terminal opened:', terminal.name);
});

vscode.window.onDidCloseTerminal(terminal => {
  console.log('Terminal closed:', terminal.name);
});

vscode.window.onDidChangeActiveTerminal(terminal => {
  console.log('Active terminal:', terminal?.name);
});

// 获取所有终端
const terminals = vscode.window.terminals;
terminals.forEach(t => console.log(t.name));

// 活动终端
const activeTerminal = vscode.window.activeTerminal;
```

### 终端链接处理

```typescript
// 注册链接提供者
vscode.window.registerTerminalLinkProvider({
  provideTerminalLinks(context, token) {
    const links: vscode.TerminalLink[] = [];

    // 匹配文件路径
    const fileRegex = /(\/?[\w-]+\/)+[\w.-]+:\d+/g;
    let match;
    while ((match = fileRegex.exec(context.line))) {
      links.push({
        startIndex: match.index,
        length: match[0].length,
        tooltip: 'Open file',
        data: match[0]
      });
    }

    return links;
  },

  handleTerminalLink(link) {
    const [file, line] = link.data.split(':');
    vscode.commands.executeCommand('vscode.open', vscode.Uri.file(file), {
      selection: new vscode.Range(parseInt(line) - 1, 0, parseInt(line) - 1, 0)
    });
  }
});
```

### 伪终端（Pseudoterminal）

```typescript
// 自定义终端实现
class MyPseudoterminal implements vscode.Pseudoterminal {
  private writeEmitter = new vscode.EventEmitter<string>();
  onDidWrite = this.writeEmitter.event;

  private closeEmitter = new vscode.EventEmitter<number | void>();
  onDidClose = this.closeEmitter.event;

  open(initialDimensions: vscode.TerminalDimensions | undefined): void {
    this.writeEmitter.fire('Welcome to My Terminal!\r\n');
    this.writeEmitter.fire('> ');
  }

  close(): void {
    // 清理资源
  }

  handleInput(data: string): void {
    // 处理用户输入
    if (data === '\r') {
      // 回车
      this.writeEmitter.fire('\r\n> ');
    } else if (data === '\x7f') {
      // 退格
      this.writeEmitter.fire('\b \b');
    } else {
      // 回显输入
      this.writeEmitter.fire(data);
    }
  }

  setDimensions(dimensions: vscode.TerminalDimensions): void {
    // 处理尺寸变化
  }
}

// 使用伪终端
const pty = new MyPseudoterminal();
const terminal = vscode.window.createTerminal({
  name: 'Custom Terminal',
  pty
});
terminal.show();
```

### 终端配置文件

```typescript
// 注册终端配置
vscode.window.registerTerminalProfileProvider('myExtension.customShell', {
  provideTerminalProfile(token) {
    return {
      options: {
        name: 'Custom Shell',
        shellPath: '/usr/local/bin/custom-shell',
        shellArgs: ['--config', '~/.custom-shell-rc'],
        env: {
          CUSTOM_VAR: 'value'
        }
      }
    };
  }
});
```

---

## Obsidian 的终端能力

### 无内置终端

Obsidian 不提供内置终端，主要通过第三方插件：

```typescript
// 第三方终端插件示例
export default class TerminalPlugin extends Plugin {
  private terminalView: TerminalView | null = null;

  onload() {
    this.registerView(
      'terminal-view',
      (leaf) => (this.terminalView = new TerminalView(leaf))
    );

    this.addCommand({
      id: 'open-terminal',
      name: 'Open Terminal',
      callback: () => this.openTerminal()
    });
  }

  async openTerminal() {
    const leaf = this.app.workspace.getRightLeaf(false);
    if (leaf) {
      await leaf.setViewState({
        type: 'terminal-view',
        active: true
      });
    }
  }
}

class TerminalView extends ItemView {
  private terminal: Terminal | null = null;

  constructor(leaf: WorkspaceLeaf) {
    super(leaf);
  }

  getViewType(): string {
    return 'terminal-view';
  }

  getDisplayText(): string {
    return 'Terminal';
  }

  async onOpen() {
    const container = this.containerEl.children[1];
    container.empty();

    // 使用 xterm.js
    const { Terminal } = await import('xterm');
    const { FitAddon } = await import('xterm-addon-fit');

    this.terminal = new Terminal({
      fontFamily: 'monospace',
      fontSize: 14,
      theme: {
        background: '#1e1e1e',
        foreground: '#d4d4d4'
      }
    });

    const fitAddon = new FitAddon();
    this.terminal.loadAddon(fitAddon);

    this.terminal.open(container as HTMLElement);
    fitAddon.fit();

    // 连接到 shell（需要 Electron 支持）
    this.connectShell();
  }

  private connectShell() {
    // 通过 Electron 的 node-pty 连接
    // 这需要插件能访问 Node.js API
  }

  async onClose() {
    this.terminal?.dispose();
  }
}
```

### Shell 命令执行

```typescript
// 执行 shell 命令（通过 Obsidian 的有限 API）
export default class ShellPlugin extends Plugin {
  onload() {
    this.addCommand({
      id: 'run-shell-command',
      name: 'Run Shell Command',
      callback: async () => {
        const command = await this.promptForCommand();
        if (command) {
          await this.executeCommand(command);
        }
      }
    });
  }

  async executeCommand(command: string): Promise<string> {
    // 使用 Node.js child_process
    const { exec } = require('child_process');

    return new Promise((resolve, reject) => {
      exec(command, {
        cwd: this.app.vault.adapter.basePath
      }, (error: Error, stdout: string, stderr: string) => {
        if (error) {
          reject(error);
        } else {
          resolve(stdout);
        }
      });
    });
  }
}
```

---

## 对比分析

### 终端能力对比

| 方面 | VSCode | Obsidian |
|------|--------|----------|
| **内置终端** | 有 (xterm.js) | 无 |
| **多终端** | 支持 | 需插件 |
| **终端配置** | 丰富 | 无 |
| **链接检测** | 支持 | 需自行实现 |
| **伪终端** | 支持 | 无 |
| **Shell 集成** | 深度集成 | 有限 |

---

## 对 AI Chat + Editor 应用的建议

### 终端需求评估

对于 AI Chat + Editor，终端可能是重要功能：

```typescript
// 终端服务接口
interface ITerminalService {
  // 创建终端
  create(options: TerminalOptions): ITerminal;

  // 获取所有终端
  getAll(): ITerminal[];

  // 获取活动终端
  getActive(): ITerminal | undefined;

  // 事件
  onDidCreate: Event<ITerminal>;
  onDidClose: Event<ITerminal>;
  onDidChangeActive: Event<ITerminal | undefined>;
}

interface ITerminal {
  id: string;
  name: string;

  // 操作
  sendText(text: string): void;
  show(): void;
  hide(): void;
  dispose(): void;

  // 状态
  isActive: boolean;

  // 事件
  onDidOutput: Event<string>;
  onDidExit: Event<number>;
}
```

### 基于 xterm.js 实现

```typescript
import { Terminal } from 'xterm';
import { FitAddon } from 'xterm-addon-fit';
import { WebLinksAddon } from 'xterm-addon-web-links';
import { SearchAddon } from 'xterm-addon-search';
import * as pty from 'node-pty';

class TerminalInstance implements ITerminal {
  private terminal: Terminal;
  private ptyProcess: pty.IPty;
  private fitAddon: FitAddon;

  private _onDidOutput = new Emitter<string>();
  readonly onDidOutput = this._onDidOutput.event;

  private _onDidExit = new Emitter<number>();
  readonly onDidExit = this._onDidExit.event;

  constructor(
    readonly id: string,
    readonly name: string,
    private container: HTMLElement,
    private options: TerminalOptions
  ) {
    this.terminal = new Terminal({
      fontFamily: options.fontFamily || 'Menlo, Monaco, monospace',
      fontSize: options.fontSize || 14,
      theme: this.getTheme(options.theme),
      cursorBlink: true,
      cursorStyle: 'bar',
      scrollback: 10000
    });

    // 加载插件
    this.fitAddon = new FitAddon();
    this.terminal.loadAddon(this.fitAddon);
    this.terminal.loadAddon(new WebLinksAddon());
    this.terminal.loadAddon(new SearchAddon());

    // 挂载到 DOM
    this.terminal.open(container);
    this.fitAddon.fit();

    // 创建 PTY 进程
    this.ptyProcess = pty.spawn(
      options.shell || this.getDefaultShell(),
      options.args || [],
      {
        name: 'xterm-256color',
        cols: this.terminal.cols,
        rows: this.terminal.rows,
        cwd: options.cwd || process.env.HOME,
        env: { ...process.env, ...options.env }
      }
    );

    // 连接终端和 PTY
    this.terminal.onData(data => {
      this.ptyProcess.write(data);
    });

    this.ptyProcess.onData(data => {
      this.terminal.write(data);
      this._onDidOutput.fire(data);
    });

    this.ptyProcess.onExit(({ exitCode }) => {
      this._onDidExit.fire(exitCode);
    });

    // 监听尺寸变化
    this.terminal.onResize(({ cols, rows }) => {
      this.ptyProcess.resize(cols, rows);
    });
  }

  private getDefaultShell(): string {
    if (process.platform === 'win32') {
      return process.env.COMSPEC || 'cmd.exe';
    }
    return process.env.SHELL || '/bin/bash';
  }

  private getTheme(name?: string) {
    // 主题配置
    const themes = {
      dark: {
        background: '#1e1e1e',
        foreground: '#d4d4d4',
        cursor: '#d4d4d4',
        selection: '#264f78'
      },
      light: {
        background: '#ffffff',
        foreground: '#333333',
        cursor: '#333333',
        selection: '#add6ff'
      }
    };
    return themes[name || 'dark'];
  }

  sendText(text: string): void {
    this.ptyProcess.write(text);
  }

  show(): void {
    this.container.style.display = 'block';
    this.fitAddon.fit();
  }

  hide(): void {
    this.container.style.display = 'none';
  }

  resize(): void {
    this.fitAddon.fit();
  }

  dispose(): void {
    this.ptyProcess.kill();
    this.terminal.dispose();
  }
}
```

### AI 与终端集成

```typescript
class AITerminalAssistant {
  constructor(
    private aiClient: IAIClient,
    private terminal: ITerminal
  ) {}

  // 解释命令
  async explainCommand(command: string): Promise<string> {
    const response = await this.aiClient.chat([
      {
        role: 'user',
        content: `Explain this shell command concisely:\n\n${command}`
      }
    ]);
    return response.content;
  }

  // 生成命令
  async generateCommand(description: string): Promise<string> {
    const response = await this.aiClient.chat([
      {
        role: 'system',
        content: 'You are a shell command expert. Generate commands for the user\'s request. Only output the command, no explanation.'
      },
      {
        role: 'user',
        content: description
      }
    ]);
    return response.content.trim();
  }

  // 修复错误
  async fixError(command: string, error: string): Promise<string> {
    const response = await this.aiClient.chat([
      {
        role: 'user',
        content: `This command failed:\n\n${command}\n\nError:\n${error}\n\nProvide the corrected command.`
      }
    ]);
    return response.content.trim();
  }

  // 自动补全
  async suggestCompletion(partialCommand: string): Promise<string[]> {
    const response = await this.aiClient.chat([
      {
        role: 'user',
        content: `Complete this shell command (provide 3 options):\n\n${partialCommand}`
      }
    ]);
    return response.content.split('\n').filter(Boolean);
  }
}
```

### 终端 UI 组件

```tsx
function TerminalPanel() {
  const [terminals, setTerminals] = useState<ITerminal[]>([]);
  const [activeId, setActiveId] = useState<string | null>(null);
  const terminalService = useService(ITerminalService);

  const createTerminal = () => {
    const terminal = terminalService.create({
      name: `Terminal ${terminals.length + 1}`
    });
    setTerminals([...terminals, terminal]);
    setActiveId(terminal.id);
  };

  const closeTerminal = (id: string) => {
    const terminal = terminals.find(t => t.id === id);
    terminal?.dispose();
    setTerminals(terminals.filter(t => t.id !== id));
    if (activeId === id) {
      setActiveId(terminals[0]?.id || null);
    }
  };

  return (
    <div className="terminal-panel">
      <div className="terminal-tabs">
        {terminals.map(t => (
          <div
            key={t.id}
            className={`terminal-tab ${t.id === activeId ? 'active' : ''}`}
            onClick={() => setActiveId(t.id)}
          >
            <span>{t.name}</span>
            <button onClick={(e) => {
              e.stopPropagation();
              closeTerminal(t.id);
            }}>×</button>
          </div>
        ))}
        <button className="new-terminal" onClick={createTerminal}>+</button>
      </div>

      <div className="terminal-content">
        {terminals.map(t => (
          <TerminalView
            key={t.id}
            terminal={t}
            visible={t.id === activeId}
          />
        ))}
      </div>
    </div>
  );
}

function TerminalView({ terminal, visible }: { terminal: ITerminal; visible: boolean }) {
  const containerRef = useRef<HTMLDivElement>(null);

  useEffect(() => {
    if (containerRef.current) {
      // 终端已经在 TerminalInstance 中初始化
    }
  }, []);

  useEffect(() => {
    if (visible) {
      terminal.show();
    } else {
      terminal.hide();
    }
  }, [visible]);

  return (
    <div
      ref={containerRef}
      className={`terminal-view ${visible ? 'visible' : 'hidden'}`}
    />
  );
}
```

### Shell 集成

```typescript
// Shell 集成脚本（bash/zsh）
const shellIntegrationScript = `
# 命令开始标记
__prompt_command_start() {
  printf '\\033]633;A\\007'
}

# 命令结束标记
__prompt_command_end() {
  printf '\\033]633;B;%s\\007' "$?"
}

# 工作目录变化
__prompt_cwd() {
  printf '\\033]633;P;Cwd=%s\\007' "$PWD"
}

PROMPT_COMMAND="__prompt_command_end; __prompt_cwd; __prompt_command_start"
`;

class ShellIntegration {
  private commands: Map<number, CommandInfo> = new Map();
  private currentCommand: CommandInfo | null = null;

  constructor(private terminal: ITerminal) {
    terminal.onDidOutput(data => {
      this.parseOutput(data);
    });
  }

  private parseOutput(data: string): void {
    // 解析 OSC 633 序列
    const oscRegex = /\x1b\]633;([A-Z]);?([^\x07]*)\x07/g;
    let match;

    while ((match = oscRegex.exec(data))) {
      const [_, code, params] = match;

      switch (code) {
        case 'A': // 命令开始
          this.currentCommand = {
            startTime: Date.now(),
            output: ''
          };
          break;

        case 'B': // 命令结束
          if (this.currentCommand) {
            this.currentCommand.exitCode = parseInt(params);
            this.currentCommand.duration = Date.now() - this.currentCommand.startTime;
            this.onCommandComplete(this.currentCommand);
          }
          break;

        case 'P': // 属性
          const [key, value] = params.split('=');
          if (key === 'Cwd') {
            this.onCwdChange(value);
          }
          break;
      }
    }
  }

  private onCommandComplete(command: CommandInfo): void {
    // 命令完成后可以触发 AI 分析
    if (command.exitCode !== 0) {
      // 命令失败，可以提供修复建议
      this.suggestFix(command);
    }
  }

  private onCwdChange(cwd: string): void {
    // 同步工作目录到应用状态
  }
}
```

---

## 关键决策清单

1. **是否需要内置终端？**
   - 是：开发者工具
   - 否：纯编辑器

2. **使用什么终端库？**
   - xterm.js：成熟稳定
   - hterm：Chrome 终端

3. **PTY 如何集成？**
   - node-pty：完整 PTY
   - child_process：简单命令

4. **AI 如何与终端交互？**
   - 命令生成
   - 错误分析
   - 输出解释

---

## 参考资料

- [xterm.js](https://xtermjs.org/)
- [node-pty](https://github.com/microsoft/node-pty)
- [VSCode Terminal](https://code.visualstudio.com/docs/terminal/basics)
- [Shell Integration](https://code.visualstudio.com/docs/terminal/shell-integration)
