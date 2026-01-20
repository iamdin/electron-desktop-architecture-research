# 34. VSCode 特色功能

> **核心问题**：VSCode 有哪些独特功能？

---

## 概述

VSCode 作为代码编辑器，有许多面向开发者的特色功能。

---

## Remote Development

### 架构

```
┌─────────────────┐          ┌─────────────────────────────┐
│  VSCode Client  │◄────────►│      Remote Host            │
│  (本地 UI)      │   SSH    │  - VSCode Server            │
└─────────────────┘          │  - Extensions               │
                             │  - File System              │
                             │  - Terminals                │
                             └─────────────────────────────┘
```

### 使用方式

```typescript
// Remote 扩展 API
const remoteAuthority = vscode.env.remoteName;
if (remoteAuthority === 'ssh-remote') {
  // 运行在远程主机
}

// 文件 URI
// vscode-remote://ssh-remote+hostname/path/to/file.ts
```

### 支持的远程类型

- SSH Remote
- WSL (Windows Subsystem for Linux)
- Dev Containers (Docker)
- GitHub Codespaces
- Tunnel

---

## Dev Containers

```json
// .devcontainer/devcontainer.json
{
  "name": "Node.js",
  "image": "mcr.microsoft.com/devcontainers/javascript-node:18",
  "features": {
    "ghcr.io/devcontainers/features/git:1": {}
  },
  "customizations": {
    "vscode": {
      "extensions": ["dbaeumer.vscode-eslint"],
      "settings": { "editor.formatOnSave": true }
    }
  },
  "forwardPorts": [3000],
  "postCreateCommand": "npm install"
}
```

---

## Multi-Root Workspaces

```json
// workspace.code-workspace
{
  "folders": [
    { "path": "frontend" },
    { "path": "backend" },
    { "path": "../shared-lib" }
  ],
  "settings": {
    "editor.tabSize": 2
  },
  "extensions": {
    "recommendations": ["esbenp.prettier-vscode"]
  }
}
```

```typescript
// 多根目录 API
const folders = vscode.workspace.workspaceFolders;
// [{ uri: frontend, name: 'frontend' }, { uri: backend, name: 'backend' }]

// 查找文件时指定 folder
vscode.workspace.findFiles('**/*.ts', null, 100, folder);
```

---

## IntelliSense / AI 代码补全

```typescript
// 补全 Provider
vscode.languages.registerCompletionItemProvider('typescript', {
  provideCompletionItems(document, position) {
    return [
      {
        label: 'console.log',
        kind: vscode.CompletionItemKind.Snippet,
        insertText: new vscode.SnippetString('console.log($1);$0'),
        documentation: 'Log to console'
      }
    ];
  }
});

// GitHub Copilot 集成
// 使用 Inline Completion API
vscode.languages.registerInlineCompletionItemProvider('*', {
  async provideInlineCompletionItems(document, position) {
    const suggestion = await copilot.getSuggestion(document, position);
    return [{ insertText: suggestion }];
  }
});
```

---

## 任务系统

```json
// tasks.json
{
  "version": "2.0.0",
  "tasks": [
    {
      "label": "Build",
      "type": "npm",
      "script": "build",
      "group": { "kind": "build", "isDefault": true },
      "problemMatcher": ["$tsc"]
    },
    {
      "label": "Test",
      "type": "shell",
      "command": "npm test",
      "group": "test"
    }
  ]
}
```

```typescript
// 任务 API
const task = new vscode.Task(
  { type: 'custom', task: 'build' },
  vscode.TaskScope.Workspace,
  'Build Project',
  'custom',
  new vscode.ShellExecution('npm run build')
);

vscode.tasks.executeTask(task);
```

---

## 对 Coding Agent Desktop 应用的启示

| 功能 | 是否适用 | 说明 |
|------|----------|------|
| Remote Development | ⚠️ | 如需远程编辑可考虑 |
| Dev Containers | ❌ | 开发环境管理，不适用 |
| Multi-Root | ✅ | 多项目管理有用 |
| IntelliSense | ✅ | AI 补全核心功能 |
| 任务系统 | ⚠️ | 简化版本可能有用 |

---

## 参考资料

- [VSCode Remote Development](https://code.visualstudio.com/docs/remote/remote-overview)
- [Dev Containers](https://containers.dev/)
- [VSCode Tasks](https://code.visualstudio.com/docs/editor/tasks)
