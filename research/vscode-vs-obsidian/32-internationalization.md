# 32. 国际化

> **核心问题**：如何支持多语言？

---

## 概述

国际化 (i18n) 包括：文本翻译、日期/数字格式、RTL 支持。

---

## VSCode - 完整 i18n 系统

### 架构

```typescript
// 本地化字符串定义
// package.nls.json (默认语言)
{
  "command.hello": "Hello World"
}

// package.nls.zh-cn.json (中文)
{
  "command.hello": "你好世界"
}

// 在 package.json 中引用
{
  "contributes": {
    "commands": [{
      "command": "extension.hello",
      "title": "%command.hello%"
    }]
  }
}
```

### 运行时本地化

```typescript
import * as nls from 'vscode-nls';

// 初始化
const localize = nls.loadMessageBundle();

// 使用
const message = localize('key', 'Default message');

// 带参数
const greeting = localize('greeting', 'Hello {0}!', userName);
```

### 语言包扩展

```json
// 语言包 package.json
{
  "contributes": {
    "localizations": [{
      "languageId": "zh-cn",
      "languageName": "Chinese Simplified",
      "localizedLanguageName": "简体中文",
      "translations": [{
        "id": "vscode",
        "path": "./translations/main.i18n.json"
      }]
    }]
  }
}
```

---

## Obsidian - 基础 i18n

### 内置支持

```typescript
// Obsidian 内置翻译
const text = this.app.vault.getName(); // 自动本地化

// 插件翻译需自行实现
class MyPlugin extends Plugin {
  private locale: Record<string, string>;

  async onload() {
    const lang = moment.locale(); // 'zh-cn', 'en', etc.
    this.locale = await this.loadLocale(lang);
  }

  t(key: string): string {
    return this.locale[key] || key;
  }
}
```

### 日期本地化

```typescript
// Obsidian 使用 moment.js
import { moment } from 'obsidian';

// 自动使用系统语言
const date = moment().format('LL'); // "2024年1月15日"

// 设置语言
moment.locale('zh-cn');
```

---

## 对比

| 方面 | VSCode | Obsidian |
|------|--------|----------|
| **翻译系统** | 完善 | 基础 |
| **语言包** | 扩展机制 | 内置 |
| **插件翻译** | 标准 API | 自行实现 |
| **RTL** | 支持 | 有限 |
| **语言数量** | 40+ | 30+ |

---

## 对 AI Chat + Editor 应用的建议

```typescript
// 简单 i18n 系统
class I18n {
  private translations: Map<string, Record<string, string>> = new Map();
  private currentLocale: string = 'en';

  async loadLocale(locale: string): Promise<void> {
    if (!this.translations.has(locale)) {
      const response = await fetch(`/locales/${locale}.json`);
      this.translations.set(locale, await response.json());
    }
    this.currentLocale = locale;
  }

  t(key: string, params?: Record<string, string>): string {
    const translations = this.translations.get(this.currentLocale);
    let text = translations?.[key] || key;

    // 替换参数
    if (params) {
      for (const [k, v] of Object.entries(params)) {
        text = text.replace(`{${k}}`, v);
      }
    }

    return text;
  }

  // 获取系统语言
  getSystemLocale(): string {
    return navigator.language.toLowerCase();
  }
}

// 使用
const i18n = new I18n();
await i18n.loadLocale('zh-cn');

console.log(i18n.t('greeting', { name: 'Alice' }));
// "你好，Alice！"

// React 集成
function useTranslation() {
  const i18n = useContext(I18nContext);
  return { t: i18n.t.bind(i18n) };
}

function MyComponent() {
  const { t } = useTranslation();
  return <button>{t('button.save')}</button>;
}
```

---

## 参考资料

- [VSCode Localization](https://code.visualstudio.com/api/references/extension-manifest#localization)
- [vscode-nls](https://github.com/microsoft/vscode-nls)
