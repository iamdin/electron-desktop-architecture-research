# 30. 运行时性能

> **核心问题**：如何保证运行时流畅？

---

## 概述

运行时性能包括：渲染性能、内存管理、CPU 使用、响应延迟。

---

## VSCode - 分进程优化

### 渲染优化

```typescript
// 1. 虚拟化列表 - 只渲染可见项
class VirtualizedList {
  private visibleRange: { start: number; end: number };

  render(): void {
    const items = this.data.slice(
      this.visibleRange.start,
      this.visibleRange.end
    );
    // 只渲染可见项
  }
}

// 2. 增量 DOM 更新
// Monaco Editor 使用行级别增量渲染

// 3. requestAnimationFrame 批量更新
class BatchUpdater {
  private pending = false;
  private updates: (() => void)[] = [];

  schedule(update: () => void): void {
    this.updates.push(update);
    if (!this.pending) {
      this.pending = true;
      requestAnimationFrame(() => {
        this.flush();
      });
    }
  }

  private flush(): void {
    const updates = this.updates;
    this.updates = [];
    this.pending = false;
    updates.forEach(u => u());
  }
}
```

### 内存管理

```typescript
// 1. 对象池
class ObjectPool<T> {
  private pool: T[] = [];

  acquire(): T {
    return this.pool.pop() || this.create();
  }

  release(obj: T): void {
    this.reset(obj);
    this.pool.push(obj);
  }
}

// 2. 弱引用缓存
const cache = new WeakMap<object, CachedData>();

// 3. 大文件分块
class LargeFileHandler {
  async* readChunks(file: string, chunkSize = 64 * 1024): AsyncIterable<string> {
    const stream = fs.createReadStream(file, { highWaterMark: chunkSize });
    for await (const chunk of stream) {
      yield chunk.toString();
    }
  }
}
```

### Worker 分流

```typescript
// 语法高亮在 Worker 中处理
const tokenizationWorker = new Worker('tokenizer.js');

tokenizationWorker.postMessage({
  type: 'tokenize',
  content: document.getText(),
  language: 'typescript'
});

tokenizationWorker.onmessage = (e) => {
  applyTokens(e.data.tokens);
};
```

---

## Obsidian - 单进程优化

### 优化策略

```typescript
// 1. 防抖/节流
function debounce<T extends (...args: any[]) => any>(
  fn: T,
  delay: number
): T {
  let timer: NodeJS.Timeout;
  return ((...args) => {
    clearTimeout(timer);
    timer = setTimeout(() => fn(...args), delay);
  }) as T;
}

// 搜索防抖
const debouncedSearch = debounce(this.search.bind(this), 300);

// 2. 异步渲染
async function renderMarkdown(content: string): Promise<void> {
  const chunks = splitIntoChunks(content, 1000);

  for (const chunk of chunks) {
    await renderChunk(chunk);
    // 让出主线程
    await new Promise(r => requestAnimationFrame(r));
  }
}

// 3. 缓存解析结果
class MarkdownCache {
  private cache = new Map<string, ParsedContent>();

  get(content: string): ParsedContent {
    const hash = this.hash(content);
    if (!this.cache.has(hash)) {
      this.cache.set(hash, this.parse(content));
    }
    return this.cache.get(hash)!;
  }
}
```

---

## 对比

| 方面 | VSCode | Obsidian |
|------|--------|----------|
| **进程模型** | 多进程 | 单进程 |
| **渲染策略** | 虚拟化 | 按需渲染 |
| **Worker** | 广泛使用 | 较少使用 |
| **内存占用** | 较高 | 较低 |
| **大文件** | 支持好 | 一般 |

---

## 性能指标

```typescript
// 关键指标
interface PerformanceMetrics {
  // 输入延迟 (目标 < 16ms)
  inputLatency: number;

  // 帧率 (目标 60fps)
  fps: number;

  // 内存使用
  memoryUsage: {
    heapUsed: number;
    heapTotal: number;
  };

  // 长任务 (> 50ms)
  longTasks: number;
}
```

---

## 对 Coding Agent Desktop 应用的建议

```typescript
// 性能监控系统
class PerformanceMonitor {
  private observer: PerformanceObserver;

  constructor() {
    // 监控长任务
    this.observer = new PerformanceObserver((list) => {
      for (const entry of list.getEntries()) {
        if (entry.duration > 50) {
          console.warn(`Long task: ${entry.duration}ms`);
        }
      }
    });
    this.observer.observe({ entryTypes: ['longtask'] });
  }

  // 测量操作耗时
  async measure<T>(name: string, fn: () => Promise<T>): Promise<T> {
    const start = performance.now();
    try {
      return await fn();
    } finally {
      const duration = performance.now() - start;
      if (duration > 100) {
        console.warn(`${name} took ${duration}ms`);
      }
    }
  }
}

// 渲染优化
class OptimizedRenderer {
  // 虚拟滚动
  renderVirtualList<T>(
    items: T[],
    itemHeight: number,
    viewportHeight: number,
    scrollTop: number,
    renderItem: (item: T) => HTMLElement
  ): HTMLElement[] {
    const startIndex = Math.floor(scrollTop / itemHeight);
    const endIndex = Math.min(
      items.length,
      startIndex + Math.ceil(viewportHeight / itemHeight) + 1
    );

    return items.slice(startIndex, endIndex).map(renderItem);
  }

  // 时间切片
  async* timeSlice<T>(
    items: T[],
    process: (item: T) => void,
    deadline = 16
  ): AsyncIterable<void> {
    let start = performance.now();

    for (const item of items) {
      process(item);

      if (performance.now() - start > deadline) {
        yield;
        await new Promise(r => requestAnimationFrame(r));
        start = performance.now();
      }
    }
  }
}
```

---

## 参考资料

- [Web Performance](https://web.dev/performance/)
- [Chrome DevTools Performance](https://developer.chrome.com/docs/devtools/performance/)
