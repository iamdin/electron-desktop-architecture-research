# 28. 安全模型

> **核心问题**：如何保证插件安全？

---

## 概述

插件安全涉及：权限控制、代码审核、沙箱隔离、信任机制。

---

## VSCode - 信任 + 审核

### Workspace Trust

```typescript
// 工作区信任检查
if (vscode.workspace.isTrusted) {
  // 执行可能危险的操作
  executeCode();
} else {
  // 受限模式
  vscode.window.showWarningMessage('This feature requires a trusted workspace.');
}

// 监听信任状态变化
vscode.workspace.onDidGrantWorkspaceTrust(() => {
  // 重新启用功能
});
```

### 扩展权限

```json
// package.json
{
  "capabilities": {
    "untrustedWorkspaces": {
      "supported": "limited",
      "description": "Some features disabled in untrusted workspaces"
    },
    "virtualWorkspaces": {
      "supported": false,
      "description": "Requires file system access"
    }
  }
}
```

### 市场审核

- 自动扫描恶意代码
- 检查已知漏洞
- 人工审核（针对可疑扩展）

---

## Obsidian - 最小权限

### 安全机制

```typescript
// 1. 插件只能访问 vault 内的文件
const file = this.app.vault.getAbstractFileByPath('note.md');

// 2. 网络请求需要声明
// manifest.json
{
  "isDesktopOnly": false  // 移动端有更严格限制
}

// 3. 无法访问其他应用数据
// 4. 无法执行系统命令（除非使用 Node.js）
```

### 社区审核

- PR 审核（社区维护者）
- 源码公开（GitHub 托管）
- 用户反馈机制

---

## 对比

| 方面 | VSCode | Obsidian |
|------|--------|----------|
| **沙箱** | 进程隔离 | JS 沙箱 |
| **权限系统** | Workspace Trust | 无细粒度权限 |
| **代码审核** | 自动 + 人工 | 社区 PR |
| **信任机制** | 工作区级别 | 全局开关 |
| **隔离程度** | 中 | 低 |

---

## 安全风险

### 共同风险

```typescript
// 1. 恶意数据收集
// 插件可以读取用户文件并上传

// 2. 代码注入
// 插件可以执行任意 JavaScript

// 3. 供应链攻击
// 依赖包可能被注入恶意代码
```

### VSCode 特有风险

```typescript
// 扩展可以：
// - 访问文件系统
// - 执行终端命令
// - 发起网络请求
// - 调用系统 API
```

---

## 对 Coding Agent Desktop 应用的建议

建议实现基础安全机制：

```typescript
// 权限系统
interface PluginPermissions {
  fileSystem: 'none' | 'read' | 'write';
  network: 'none' | 'localhost' | 'all';
  shell: boolean;
  clipboard: boolean;
}

// 权限检查
class PermissionManager {
  private permissions: Map<string, PluginPermissions> = new Map();

  checkPermission(pluginId: string, permission: keyof PluginPermissions): boolean {
    const perms = this.permissions.get(pluginId);
    if (!perms) return false;
    return !!perms[permission];
  }

  // 代理 API 调用
  createSafeAPI(pluginId: string): PluginAPI {
    return {
      fs: {
        readFile: async (path: string) => {
          if (!this.checkPermission(pluginId, 'fileSystem')) {
            throw new Error('Permission denied: fileSystem');
          }
          // 限制路径范围
          if (!this.isPathAllowed(path)) {
            throw new Error('Access denied: path outside workspace');
          }
          return fs.readFile(path);
        }
      },
      fetch: async (url: string) => {
        if (!this.checkPermission(pluginId, 'network')) {
          throw new Error('Permission denied: network');
        }
        return fetch(url);
      }
    };
  }
}
```

---

## 参考资料

- [VSCode Workspace Trust](https://code.visualstudio.com/docs/editor/workspace-trust)
- [Electron Security](https://www.electronjs.org/docs/latest/tutorial/security)
