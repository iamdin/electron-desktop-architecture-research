# 29. 启动性能

> **核心问题**：如何优化启动速度？

---

## 概述

启动性能直接影响用户体验，包括：冷启动、热启动、首次渲染。

---

## VSCode - 分层启动

### 启动阶段

```
┌─────────────────────────────────────────────────────────────┐
│                      VSCode 启动流程                         │
├─────────────────────────────────────────────────────────────┤
│  1. Electron 启动 (~200ms)                                  │
│  2. Main Process 初始化 (~100ms)                            │
│  3. Renderer 创建 (~150ms)                                  │
│  4. Workbench 加载 (~300ms)                                 │
│  5. 扩展激活 (懒加载)                                        │
└─────────────────────────────────────────────────────────────┘
```

### 优化策略

```typescript
// 1. 代码拆分 - AMD 模块按需加载
define(['vs/base/common/lifecycle'], function(lifecycle) {
  // 模块代码
});

// 2. 扩展懒激活
// package.json
{
  "activationEvents": [
    "onLanguage:javascript",     // 打开 JS 文件时激活
    "onCommand:extension.cmd",   // 执行命令时激活
    "workspaceContains:**/*.py"  // 工作区包含文件时激活
  ]
}

// 3. 缓存
// - 扩展 manifest 缓存
// - 工作区状态缓存
// - 编辑器布局缓存
```

### 性能追踪

```typescript
// 内置性能追踪
// 开发者工具 > Performance
// 或使用 --prof 启动参数

// 扩展启动时间
// 帮助 > 开发者 > 显示正在运行的扩展
```

---

## Obsidian - 轻量启动

### 启动特点

```
┌─────────────────────────────────────────────────────────────┐
│                     Obsidian 启动流程                        │
├─────────────────────────────────────────────────────────────┤
│  1. Electron 启动 (~200ms)                                  │
│  2. 核心加载 (~100ms)                                       │
│  3. Vault 索引 (取决于文件数量)                              │
│  4. 插件加载 (~50-200ms)                                    │
└─────────────────────────────────────────────────────────────┘
```

### 优化措施

```typescript
// 1. 核心精简
// Obsidian 核心功能较少，启动快

// 2. 插件延迟加载
// 部分插件支持延迟加载

// 3. 索引优化
// 使用增量索引
```

---

## 对比

| 方面 | VSCode | Obsidian |
|------|--------|----------|
| **冷启动** | ~2-3s | ~1-2s |
| **热启动** | ~1s | ~0.5s |
| **扩展影响** | 可控 (懒加载) | 较大 |
| **缓存机制** | 完善 | 基础 |

---

## 对 Coding Agent Desktop 应用的建议

```typescript
// 分阶段启动
class AppBootstrap {
  async boot(): Promise<void> {
    // 阶段 1: 显示 UI 骨架 (~100ms)
    await this.showSkeleton();

    // 阶段 2: 加载核心功能 (~200ms)
    await this.loadCore();

    // 阶段 3: 恢复工作区状态
    await this.restoreWorkspace();

    // 阶段 4: 后台加载插件
    this.loadPluginsAsync();
  }

  private async showSkeleton(): Promise<void> {
    // 立即显示基础 UI
    document.getElementById('app')!.innerHTML = `
      <div class="skeleton">
        <div class="sidebar-skeleton"></div>
        <div class="editor-skeleton"></div>
      </div>
    `;
  }

  private loadPluginsAsync(): void {
    // 不阻塞主流程
    requestIdleCallback(async () => {
      for (const plugin of this.plugins) {
        await plugin.load();
        // 让出主线程
        await new Promise(r => setTimeout(r, 0));
      }
    });
  }
}

// 启动性能监控
class StartupProfiler {
  private marks: Map<string, number> = new Map();

  mark(name: string): void {
    this.marks.set(name, performance.now());
  }

  report(): void {
    console.table(
      Array.from(this.marks.entries()).map(([name, time]) => ({
        phase: name,
        time: `${time.toFixed(2)}ms`
      }))
    );
  }
}
```

---

## 参考资料

- [VSCode Performance](https://github.com/microsoft/vscode/wiki/Performance-Issues)
- [Electron Performance](https://www.electronjs.org/docs/latest/tutorial/performance)
