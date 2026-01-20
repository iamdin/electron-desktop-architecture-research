# 22. 调试能力

> **核心问题**：如何支持代码调试？

---

## 概述

调试能力决定了：
- 是否支持断点调试
- 如何集成调试协议
- 调试 UI 如何设计
- 调试状态如何管理

---

## VSCode 的调试系统

### Debug Adapter Protocol (DAP)

VSCode 定义了 DAP 协议，类似于 LSP：

```typescript
// DAP 适配器实现
import {
  LoggingDebugSession,
  InitializedEvent,
  StoppedEvent,
  BreakpointEvent,
  OutputEvent,
  Thread,
  StackFrame,
  Scope,
  Source,
  Handles
} from '@vscode/debugadapter';

class MyDebugSession extends LoggingDebugSession {
  private variableHandles = new Handles<string>();

  protected initializeRequest(
    response: DebugProtocol.InitializeResponse,
    args: DebugProtocol.InitializeRequestArguments
  ): void {
    response.body = {
      supportsConfigurationDoneRequest: true,
      supportsEvaluateForHovers: true,
      supportsStepBack: false,
      supportsSetVariable: true,
      supportsRestartRequest: true,
      supportsBreakpointLocationsRequest: true
    };

    this.sendResponse(response);
    this.sendEvent(new InitializedEvent());
  }

  protected async launchRequest(
    response: DebugProtocol.LaunchResponse,
    args: LaunchRequestArguments
  ): Promise<void> {
    // 启动调试目标
    await this.startDebugging(args.program, args.args);
    this.sendResponse(response);
  }

  protected setBreakPointsRequest(
    response: DebugProtocol.SetBreakpointsResponse,
    args: DebugProtocol.SetBreakpointsArguments
  ): void {
    const path = args.source.path!;
    const clientLines = args.lines || [];

    const breakpoints = clientLines.map(line => {
      const bp = this.createBreakpoint(path, line);
      return {
        verified: bp.verified,
        line: bp.line,
        id: bp.id
      };
    });

    response.body = { breakpoints };
    this.sendResponse(response);
  }

  protected threadsRequest(
    response: DebugProtocol.ThreadsResponse
  ): void {
    response.body = {
      threads: [
        new Thread(1, 'Main Thread'),
        new Thread(2, 'Worker Thread')
      ]
    };
    this.sendResponse(response);
  }

  protected stackTraceRequest(
    response: DebugProtocol.StackTraceResponse,
    args: DebugProtocol.StackTraceArguments
  ): void {
    const stack = this.getStackFrames(args.threadId);

    response.body = {
      stackFrames: stack.map((frame, index) => new StackFrame(
        index,
        frame.name,
        new Source(frame.file, frame.path),
        frame.line,
        frame.column
      )),
      totalFrames: stack.length
    };
    this.sendResponse(response);
  }

  protected scopesRequest(
    response: DebugProtocol.ScopesResponse,
    args: DebugProtocol.ScopesArguments
  ): void {
    response.body = {
      scopes: [
        new Scope('Local', this.variableHandles.create('local'), false),
        new Scope('Global', this.variableHandles.create('global'), true)
      ]
    };
    this.sendResponse(response);
  }

  protected variablesRequest(
    response: DebugProtocol.VariablesResponse,
    args: DebugProtocol.VariablesArguments
  ): void {
    const scope = this.variableHandles.get(args.variablesReference);
    const variables = this.getVariables(scope);

    response.body = { variables };
    this.sendResponse(response);
  }

  protected continueRequest(
    response: DebugProtocol.ContinueResponse,
    args: DebugProtocol.ContinueArguments
  ): void {
    this.continue();
    this.sendResponse(response);
  }

  protected nextRequest(
    response: DebugProtocol.NextResponse,
    args: DebugProtocol.NextArguments
  ): void {
    this.stepOver();
    this.sendResponse(response);
  }

  // 发送停止事件
  private onBreakpointHit(threadId: number, reason: string): void {
    this.sendEvent(new StoppedEvent(reason, threadId));
  }
}
```

### 调试配置

```json
// launch.json
{
  "version": "0.2.0",
  "configurations": [
    {
      "type": "node",
      "request": "launch",
      "name": "Debug Program",
      "program": "${workspaceFolder}/src/index.ts",
      "preLaunchTask": "npm: build",
      "outFiles": ["${workspaceFolder}/dist/**/*.js"],
      "sourceMaps": true,
      "env": {
        "NODE_ENV": "development"
      }
    },
    {
      "type": "node",
      "request": "attach",
      "name": "Attach to Process",
      "port": 9229,
      "restart": true
    }
  ],
  "compounds": [
    {
      "name": "Full Stack",
      "configurations": ["Debug Server", "Debug Client"]
    }
  ]
}
```

### 插件注册调试器

```typescript
// package.json
{
  "contributes": {
    "debuggers": [
      {
        "type": "myDebugger",
        "label": "My Debugger",
        "program": "./out/debugAdapter.js",
        "runtime": "node",
        "configurationAttributes": {
          "launch": {
            "required": ["program"],
            "properties": {
              "program": {
                "type": "string",
                "description": "Path to the program to debug"
              },
              "stopOnEntry": {
                "type": "boolean",
                "default": true
              }
            }
          }
        },
        "initialConfigurations": [
          {
            "type": "myDebugger",
            "request": "launch",
            "name": "Debug",
            "program": "${file}"
          }
        ]
      }
    ],
    "breakpoints": [
      { "language": "javascript" },
      { "language": "typescript" }
    ]
  }
}

// 调试 API 使用
vscode.debug.registerDebugAdapterDescriptorFactory('myDebugger', {
  createDebugAdapterDescriptor(session, executable) {
    return new vscode.DebugAdapterInlineImplementation(new MyDebugSession());
  }
});

// 监听调试事件
vscode.debug.onDidStartDebugSession(session => {
  console.log('Debug started:', session.name);
});

vscode.debug.onDidTerminateDebugSession(session => {
  console.log('Debug ended:', session.name);
});

// 断点变化
vscode.debug.onDidChangeBreakpoints(event => {
  console.log('Added:', event.added);
  console.log('Removed:', event.removed);
  console.log('Changed:', event.changed);
});
```

### 调试 UI

```typescript
// Debug Toolbar 自定义
vscode.debug.registerDebugAdapterTrackerFactory('*', {
  createDebugAdapterTracker(session) {
    return {
      onWillReceiveMessage: (message) => {
        console.log('>>> ', JSON.stringify(message));
      },
      onDidSendMessage: (message) => {
        console.log('<<< ', JSON.stringify(message));
      }
    };
  }
});

// 自定义调试视图
const debugView = vscode.window.createTreeView('myDebugView', {
  treeDataProvider: new DebugTreeProvider()
});

// 调试控制台输出
vscode.debug.activeDebugConsole?.appendLine('Debug message');
```

---

## Obsidian 的调试能力

### 无内置调试支持

Obsidian 不提供代码调试功能，主要关注笔记编辑：

```typescript
// Obsidian 插件中调试自己的代码
export default class MyPlugin extends Plugin {
  onload() {
    // 开发者工具调试
    console.log('Plugin loaded');

    // 使用 debugger 语句
    if (process.env.NODE_ENV === 'development') {
      debugger;
    }
  }
}
```

### 第三方调试插件

```typescript
// 可以创建简单的 REPL/执行环境
export default class CodeRunnerPlugin extends Plugin {
  onload() {
    this.addCommand({
      id: 'run-code-block',
      name: 'Run Code Block',
      editorCallback: async (editor) => {
        const selection = editor.getSelection();
        if (selection) {
          const result = await this.executeCode(selection);
          new Notice(`Result: ${result}`);
        }
      }
    });
  }

  async executeCode(code: string): Promise<string> {
    try {
      // 简单的 eval（生产环境需要沙箱）
      const result = eval(code);
      return String(result);
    } catch (error) {
      return `Error: ${error.message}`;
    }
  }
}
```

---

## 对比分析

### 调试能力对比

| 方面 | VSCode | Obsidian |
|------|--------|----------|
| **调试协议** | DAP 标准 | 无 |
| **断点支持** | 完整（条件、日志等） | 无 |
| **调试 UI** | 完整（变量、调用栈） | 无 |
| **多语言** | 通过 DAP 扩展 | 无 |
| **目标场景** | 代码开发 | 笔记编辑 |

---

## 对 Coding Agent Desktop 应用的建议

### 调试需求评估

对于 Coding Agent Desktop 应用，完整的调试支持可能不是核心需求，但可以考虑：

```typescript
// 1. 简化的代码执行
interface ICodeRunner {
  execute(code: string, language: string): Promise<ExecutionResult>;
  cancel(): void;
}

interface ExecutionResult {
  output: string;
  error?: string;
  duration: number;
}

// 2. AI 辅助调试
interface IAIDebugger {
  explainError(error: Error, context: CodeContext): Promise<string>;
  suggestFix(error: Error, code: string): Promise<CodeFix[]>;
}
```

### 轻量级代码执行

```typescript
class SimpleCodeRunner implements ICodeRunner {
  private workers: Map<string, Worker> = new Map();

  async execute(code: string, language: string): Promise<ExecutionResult> {
    const start = Date.now();

    try {
      const output = await this.runInSandbox(code, language);
      return {
        output,
        duration: Date.now() - start
      };
    } catch (error) {
      return {
        output: '',
        error: error.message,
        duration: Date.now() - start
      };
    }
  }

  private async runInSandbox(code: string, language: string): Promise<string> {
    // JavaScript/TypeScript 可以用 Web Worker
    if (language === 'javascript' || language === 'typescript') {
      return this.runJavaScript(code);
    }

    // 其他语言可能需要后端服务
    return this.runRemote(code, language);
  }

  private runJavaScript(code: string): Promise<string> {
    return new Promise((resolve, reject) => {
      const workerCode = `
        self.onmessage = function(e) {
          try {
            const result = eval(e.data);
            self.postMessage({ success: true, result: String(result) });
          } catch (error) {
            self.postMessage({ success: false, error: error.message });
          }
        };
      `;

      const blob = new Blob([workerCode], { type: 'application/javascript' });
      const worker = new Worker(URL.createObjectURL(blob));

      const timeout = setTimeout(() => {
        worker.terminate();
        reject(new Error('Execution timeout'));
      }, 5000);

      worker.onmessage = (e) => {
        clearTimeout(timeout);
        worker.terminate();
        if (e.data.success) {
          resolve(e.data.result);
        } else {
          reject(new Error(e.data.error));
        }
      };

      worker.postMessage(code);
    });
  }
}
```

### AI 错误分析

```typescript
class AIErrorAnalyzer {
  constructor(private aiClient: IAIClient) {}

  async analyzeError(
    error: Error,
    code: string,
    language: string
  ): Promise<ErrorAnalysis> {
    const prompt = `
Analyze this ${language} error and suggest fixes:

Error: ${error.message}
Stack: ${error.stack}

Code:
\`\`\`${language}
${code}
\`\`\`

Provide:
1. Error explanation
2. Root cause
3. Suggested fixes with code
`;

    const response = await this.aiClient.chat([
      { role: 'user', content: prompt }
    ]);

    return this.parseAnalysis(response);
  }

  async debugStep(
    code: string,
    input: string,
    language: string
  ): Promise<StepByStepExecution> {
    const prompt = `
Execute this ${language} code step by step, showing the state at each step:

\`\`\`${language}
${code}
\`\`\`

Input: ${input}

For each step, show:
- Line being executed
- Variable values
- Output (if any)
`;

    const response = await this.aiClient.chat([
      { role: 'user', content: prompt }
    ]);

    return this.parseSteps(response);
  }
}
```

### 可选的 DAP 支持

```typescript
// 如果需要完整调试，可以集成 DAP
class DebugManager {
  private sessions: Map<string, DebugSession> = new Map();

  async startSession(config: DebugConfig): Promise<DebugSession> {
    const session = new DebugSession(config);

    // 启动 DAP 适配器
    const adapter = await this.createAdapter(config.type);
    session.connect(adapter);

    // 初始化
    await session.initialize();
    await session.launch(config);

    this.sessions.set(session.id, session);
    return session;
  }

  private async createAdapter(type: string): Promise<DebugAdapter> {
    // 查找已注册的调试器
    const debuggerConfig = this.getDebuggerConfig(type);

    if (debuggerConfig.runtime === 'node') {
      // 启动 Node.js 调试适配器
      return new NodeDebugAdapter(debuggerConfig.program);
    }

    // 其他类型...
    throw new Error(`Unknown debugger type: ${type}`);
  }
}

class DebugSession extends EventEmitter {
  private connection: DAProtocol | null = null;
  private breakpoints: Map<string, Breakpoint[]> = new Map();

  async setBreakpoint(file: string, line: number): Promise<Breakpoint> {
    const response = await this.connection!.sendRequest('setBreakpoints', {
      source: { path: file },
      breakpoints: [{ line }]
    });

    const bp = response.breakpoints[0];
    return {
      id: bp.id,
      verified: bp.verified,
      line: bp.line
    };
  }

  async continue(): Promise<void> {
    await this.connection!.sendRequest('continue', {
      threadId: this.currentThreadId
    });
  }

  async stepOver(): Promise<void> {
    await this.connection!.sendRequest('next', {
      threadId: this.currentThreadId
    });
  }

  async stepInto(): Promise<void> {
    await this.connection!.sendRequest('stepIn', {
      threadId: this.currentThreadId
    });
  }

  async getVariables(): Promise<Variable[]> {
    const scopes = await this.connection!.sendRequest('scopes', {
      frameId: this.currentFrameId
    });

    const variables: Variable[] = [];
    for (const scope of scopes.scopes) {
      const vars = await this.connection!.sendRequest('variables', {
        variablesReference: scope.variablesReference
      });
      variables.push(...vars.variables);
    }

    return variables;
  }
}
```

### 调试 UI 组件

```tsx
// 简化的调试面板
function DebugPanel({ session }: { session: DebugSession }) {
  const [state, setState] = useState<DebugState>('idle');
  const [variables, setVariables] = useState<Variable[]>([]);
  const [callStack, setCallStack] = useState<StackFrame[]>([]);

  useEffect(() => {
    session.on('stopped', async () => {
      setState('paused');
      setVariables(await session.getVariables());
      setCallStack(await session.getCallStack());
    });

    session.on('continued', () => {
      setState('running');
    });

    session.on('terminated', () => {
      setState('idle');
    });
  }, [session]);

  return (
    <div className="debug-panel">
      <DebugToolbar
        state={state}
        onContinue={() => session.continue()}
        onStepOver={() => session.stepOver()}
        onStepInto={() => session.stepInto()}
        onStop={() => session.stop()}
      />

      <div className="debug-views">
        <VariablesView variables={variables} />
        <CallStackView frames={callStack} />
        <BreakpointsView breakpoints={session.breakpoints} />
      </div>
    </div>
  );
}
```

---

## 关键决策清单

1. **是否需要完整调试？**
   - 是：集成 DAP
   - 否：简单代码执行即可

2. **支持哪些语言？**
   - JavaScript/TypeScript：Web Worker
   - 其他语言：后端服务

3. **AI 如何参与调试？**
   - 错误解释
   - 修复建议
   - 步进模拟

4. **调试 UI 复杂度？**
   - 简单：输出面板
   - 完整：变量/调用栈/断点

---

## 参考资料

- [Debug Adapter Protocol](https://microsoft.github.io/debug-adapter-protocol/)
- [VSCode Debugging](https://code.visualstudio.com/api/extension-guides/debugger-extension)
- [Chrome DevTools Protocol](https://chromedevtools.github.io/devtools-protocol/)
