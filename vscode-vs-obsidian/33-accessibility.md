# 33. 无障碍

> **核心问题**：如何支持无障碍访问？

---

## 概述

无障碍 (a11y) 包括：屏幕阅读器、键盘导航、高对比度、减少动画。

---

## VSCode - 完善的无障碍支持

### 屏幕阅读器

```typescript
// ARIA 标签
element.setAttribute('role', 'button');
element.setAttribute('aria-label', 'Save file');
element.setAttribute('aria-pressed', 'false');

// 实时区域
const statusElement = document.getElementById('status');
statusElement.setAttribute('role', 'status');
statusElement.setAttribute('aria-live', 'polite');

// VSCode 特有设置
{
  "editor.accessibilitySupport": "on",
  "editor.accessibilityPageSize": 10
}
```

### 键盘导航

```typescript
// 完整键盘支持
// - Tab 焦点循环
// - 快捷键系统
// - 命令面板 (Ctrl+Shift+P)

// 焦点管理
class FocusManager {
  private focusTrap: HTMLElement[] = [];

  trapFocus(container: HTMLElement): void {
    const focusable = container.querySelectorAll(
      'button, [href], input, select, textarea, [tabindex]:not([tabindex="-1"])'
    );
    this.focusTrap = Array.from(focusable) as HTMLElement[];
  }

  handleTab(event: KeyboardEvent, reverse: boolean): void {
    const current = document.activeElement;
    const index = this.focusTrap.indexOf(current as HTMLElement);
    const next = reverse
      ? this.focusTrap[index - 1] || this.focusTrap[this.focusTrap.length - 1]
      : this.focusTrap[index + 1] || this.focusTrap[0];
    next.focus();
    event.preventDefault();
  }
}
```

### 高对比度

```css
/* 高对比度主题 */
@media (prefers-contrast: high) {
  :root {
    --border-color: #ffffff;
    --focus-outline: 2px solid #ffffff;
  }
}

/* VSCode 内置高对比度主题 */
.hc-black {
  --editor-background: #000000;
  --editor-foreground: #ffffff;
  --focus-border: #f38518;
}
```

---

## Obsidian - 基础无障碍

### 现状

```typescript
// 基础键盘支持
// - 命令面板
// - 快捷键

// 部分 ARIA 支持
// 社区主题可提供高对比度
```

### 改进空间

- 屏幕阅读器支持不完整
- 某些 UI 需要鼠标操作

---

## 对比

| 方面 | VSCode | Obsidian |
|------|--------|----------|
| **屏幕阅读器** | 完善 | 基础 |
| **键盘导航** | 完善 | 良好 |
| **高对比度** | 内置主题 | 社区主题 |
| **ARIA** | 完整 | 部分 |
| **减少动画** | 支持 | 有限 |

---

## 对 Coding Agent Desktop 应用的建议

```typescript
// 无障碍工具类
class AccessibilityService {
  // 屏幕阅读器检测
  isScreenReaderActive(): boolean {
    return document.body.classList.contains('screen-reader-mode');
  }

  // 公告（屏幕阅读器会读出）
  announce(message: string, priority: 'polite' | 'assertive' = 'polite'): void {
    const el = document.createElement('div');
    el.setAttribute('role', 'status');
    el.setAttribute('aria-live', priority);
    el.setAttribute('aria-atomic', 'true');
    el.className = 'sr-only';  // 视觉隐藏
    el.textContent = message;

    document.body.appendChild(el);
    setTimeout(() => el.remove(), 1000);
  }

  // 焦点管理
  focusFirst(container: HTMLElement): void {
    const focusable = container.querySelector<HTMLElement>(
      'button, [href], input, select, textarea, [tabindex]:not([tabindex="-1"])'
    );
    focusable?.focus();
  }

  // 跳过链接
  addSkipLinks(): void {
    const skipLink = document.createElement('a');
    skipLink.href = '#main-content';
    skipLink.className = 'skip-link';
    skipLink.textContent = 'Skip to main content';
    document.body.prepend(skipLink);
  }
}

// 组件示例
function AccessibleButton({ label, onClick, pressed }: Props) {
  return (
    <button
      onClick={onClick}
      aria-label={label}
      aria-pressed={pressed}
      tabIndex={0}
    >
      {label}
    </button>
  );
}

// CSS
const styles = `
/* 视觉隐藏但屏幕阅读器可读 */
.sr-only {
  position: absolute;
  width: 1px;
  height: 1px;
  padding: 0;
  margin: -1px;
  overflow: hidden;
  clip: rect(0, 0, 0, 0);
  white-space: nowrap;
  border: 0;
}

/* 跳过链接 */
.skip-link {
  position: absolute;
  top: -40px;
  left: 0;
  z-index: 100;
}
.skip-link:focus {
  top: 0;
}

/* 减少动画 */
@media (prefers-reduced-motion: reduce) {
  * {
    animation-duration: 0.01ms !important;
    transition-duration: 0.01ms !important;
  }
}
`;
```

---

## 参考资料

- [VSCode Accessibility](https://code.visualstudio.com/docs/editor/accessibility)
- [WAI-ARIA](https://www.w3.org/WAI/ARIA/apg/)
- [WCAG Guidelines](https://www.w3.org/WAI/standards-guidelines/wcag/)
