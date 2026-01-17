# 31. 多平台支持

> **核心问题**：如何支持多平台？

---

## 概述

多平台支持包括：Windows、macOS、Linux，以及 Web 和移动端。

---

## VSCode - 全平台覆盖

### 桌面平台

```typescript
// 平台检测
import { platform } from 'os';

const isWindows = platform() === 'win32';
const isMacOS = platform() === 'darwin';
const isLinux = platform() === 'linux';

// 平台特定代码
if (isMacOS) {
  // macOS 特有逻辑
  app.dock.setIcon(icon);
}

// 路径处理
import { sep, join } from 'path';
const configPath = join(homedir(), '.vscode', 'settings.json');
```

### Web 版本

```typescript
// VSCode for Web 架构
// - 完整 UI 在浏览器运行
// - 后端服务提供文件系统/终端

// 虚拟文件系统
vscode.workspace.registerFileSystemProvider('vscode-remote', {
  readFile(uri): Promise<Uint8Array> {
    return fetch(`/api/file?path=${uri.path}`).then(r => r.arrayBuffer());
  },
  writeFile(uri, content): Promise<void> {
    return fetch(`/api/file?path=${uri.path}`, {
      method: 'PUT',
      body: content
    });
  }
});
```

### 构建配置

```json
// electron-builder.json
{
  "appId": "com.microsoft.vscode",
  "mac": {
    "target": ["dmg", "zip"],
    "category": "public.app-category.developer-tools"
  },
  "win": {
    "target": ["nsis", "zip"]
  },
  "linux": {
    "target": ["deb", "rpm", "AppImage"]
  }
}
```

---

## Obsidian - 桌面 + 移动

### 桌面平台

```typescript
// 平台 API
const platform = Platform.isMacOS ? 'mac' :
                 Platform.isWin ? 'win' : 'linux';

// 使用 Electron API
if (!Platform.isMobileApp) {
  const { shell } = require('electron');
  shell.openExternal(url);
}
```

### 移动平台

```typescript
// 移动端特有 API
if (Platform.isMobileApp) {
  // 1. 触摸事件
  this.registerDomEvent(el, 'touchstart', handler);

  // 2. 键盘适配
  Platform.isIosApp && this.adjustForKeyboard();

  // 3. 文件访问受限
  // 只能访问 vault 目录
}
```

### 平台差异处理

```typescript
class CrossPlatformService {
  getConfigPath(): string {
    if (Platform.isMobileApp) {
      return this.app.vault.configDir;
    }
    // 桌面端
    const basePath = process.platform === 'win32'
      ? process.env.APPDATA
      : process.platform === 'darwin'
        ? `${process.env.HOME}/Library/Application Support`
        : `${process.env.HOME}/.config`;
    return `${basePath}/obsidian`;
  }
}
```

---

## 对比

| 方面 | VSCode | Obsidian |
|------|--------|----------|
| **Windows** | ✅ | ✅ |
| **macOS** | ✅ | ✅ |
| **Linux** | ✅ | ✅ |
| **Web** | ✅ | ❌ |
| **iOS** | ❌ | ✅ |
| **Android** | ❌ | ✅ |

---

## 对 AI Chat + Editor 应用的建议

```typescript
// 平台抽象层
interface PlatformService {
  readonly isDesktop: boolean;
  readonly isMobile: boolean;
  readonly isWeb: boolean;
  readonly os: 'windows' | 'macos' | 'linux' | 'ios' | 'android' | 'web';

  openExternal(url: string): Promise<void>;
  getAppDataPath(): string;
  showNotification(title: string, body: string): void;
}

// Electron 实现
class ElectronPlatformService implements PlatformService {
  readonly isDesktop = true;
  readonly isMobile = false;
  readonly isWeb = false;
  readonly os = process.platform === 'win32' ? 'windows' :
                process.platform === 'darwin' ? 'macos' : 'linux';

  async openExternal(url: string): Promise<void> {
    const { shell } = require('electron');
    await shell.openExternal(url);
  }

  getAppDataPath(): string {
    const { app } = require('electron');
    return app.getPath('userData');
  }

  showNotification(title: string, body: string): void {
    new Notification({ title, body }).show();
  }
}

// Web 实现
class WebPlatformService implements PlatformService {
  readonly isDesktop = false;
  readonly isMobile = false;
  readonly isWeb = true;
  readonly os = 'web';

  async openExternal(url: string): Promise<void> {
    window.open(url, '_blank');
  }

  getAppDataPath(): string {
    return '/virtual/appdata';  // IndexedDB backed
  }

  showNotification(title: string, body: string): void {
    if (Notification.permission === 'granted') {
      new Notification(title, { body });
    }
  }
}
```

---

## 参考资料

- [Electron Multi-Platform](https://www.electronjs.org/docs/latest/tutorial/support)
- [VSCode Web](https://github.com/nicksenger/vscode-web)
