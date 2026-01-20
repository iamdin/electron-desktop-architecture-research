# 35. Obsidian 特色功能

> **核心问题**：Obsidian 有哪些独特功能？

---

## 概述

Obsidian 作为知识管理工具，有许多面向笔记和思维管理的特色功能。

---

## 双向链接

### 核心概念

```markdown
<!-- 创建链接 -->
这是一个 [[笔记名称]] 的链接。
这是带别名的 [[笔记名称|显示文本]] 链接。

<!-- 嵌入内容 -->
![[另一个笔记]]
![[图片.png]]

<!-- 块引用 -->
[[笔记#标题]]
[[笔记#^block-id]]
```

### 实现原理

```typescript
// 链接解析
class LinkResolver {
  private linkIndex: Map<string, Set<string>> = new Map();  // 被链接 -> 链接源

  parseLinks(content: string): string[] {
    const regex = /\[\[([^\]|]+)(?:\|[^\]]+)?\]\]/g;
    const links: string[] = [];
    let match;
    while ((match = regex.exec(content)) !== null) {
      links.push(match[1]);
    }
    return links;
  }

  buildBacklinks(file: string, content: string): void {
    const links = this.parseLinks(content);
    for (const link of links) {
      if (!this.linkIndex.has(link)) {
        this.linkIndex.set(link, new Set());
      }
      this.linkIndex.get(link)!.add(file);
    }
  }

  getBacklinks(file: string): string[] {
    return Array.from(this.linkIndex.get(file) || []);
  }
}
```

---

## Graph View

### 可视化知识图谱

```typescript
// 图形数据结构
interface GraphData {
  nodes: Array<{
    id: string;
    label: string;
    group?: string;  // 文件夹/标签
  }>;
  edges: Array<{
    source: string;
    target: string;
  }>;
}

// 使用 D3.js 或类似库渲染
class GraphView {
  private simulation: d3.Simulation<GraphNode, GraphEdge>;

  render(data: GraphData): void {
    this.simulation = d3.forceSimulation(data.nodes)
      .force('link', d3.forceLink(data.edges).id(d => d.id))
      .force('charge', d3.forceManyBody().strength(-100))
      .force('center', d3.forceCenter(width / 2, height / 2));
  }
}
```

---

## Canvas

### 无限画布

```json
// .canvas 文件格式
{
  "nodes": [
    {
      "id": "node1",
      "type": "text",
      "text": "想法 1",
      "x": 0, "y": 0,
      "width": 200, "height": 100
    },
    {
      "id": "node2",
      "type": "file",
      "file": "notes/idea.md",
      "x": 300, "y": 0,
      "width": 200, "height": 200
    }
  ],
  "edges": [
    {
      "id": "edge1",
      "fromNode": "node1",
      "toNode": "node2",
      "fromSide": "right",
      "toSide": "left"
    }
  ]
}
```

---

## 本地优先

### 数据所有权

```typescript
// 所有数据都是本地 Markdown 文件
// - 无需账户
// - 无云依赖
// - 可用任何编辑器打开

class LocalFirstVault {
  private basePath: string;

  async readNote(name: string): Promise<string> {
    return fs.readFile(path.join(this.basePath, `${name}.md`), 'utf-8');
  }

  async writeNote(name: string, content: string): Promise<void> {
    await fs.writeFile(path.join(this.basePath, `${name}.md`), content);
  }

  // 同步是可选的，通过第三方服务
  // - iCloud
  // - Dropbox
  // - Git
  // - Obsidian Sync (官方付费)
}
```

---

## 标签系统

```markdown
<!-- 行内标签 -->
这是一个 #标签 的示例。
支持嵌套 #父标签/子标签。

<!-- YAML frontmatter -->
---
tags:
  - 项目/工作
  - 重要
---
```

```typescript
// 标签索引
class TagIndex {
  private tagToFiles: Map<string, Set<string>> = new Map();

  indexFile(file: string, content: string): void {
    // 解析行内标签
    const inlineTags = content.match(/#[\w/]+/g) || [];

    // 解析 frontmatter 标签
    const frontmatter = this.parseFrontmatter(content);
    const fmTags = frontmatter?.tags || [];

    const allTags = [...inlineTags.map(t => t.slice(1)), ...fmTags];

    for (const tag of allTags) {
      if (!this.tagToFiles.has(tag)) {
        this.tagToFiles.set(tag, new Set());
      }
      this.tagToFiles.get(tag)!.add(file);
    }
  }

  getFilesWithTag(tag: string): string[] {
    return Array.from(this.tagToFiles.get(tag) || []);
  }
}
```

---

## Daily Notes

```typescript
// 每日笔记模板
const template = `# {{date:YYYY-MM-DD}}

## 今日任务
- [ ]

## 笔记

## 回顾
`;

class DailyNotes {
  async openToday(): Promise<void> {
    const date = moment().format('YYYY-MM-DD');
    const path = `Daily Notes/${date}.md`;

    if (!await this.exists(path)) {
      await this.createFromTemplate(path, template);
    }

    await this.openFile(path);
  }
}
```

---

## 对 Coding Agent Desktop 应用的启示

| 功能 | 是否适用 | 说明 |
|------|----------|------|
| 双向链接 | ✅ | 知识管理核心 |
| Graph View | ⚠️ | 可视化对话/知识 |
| Canvas | ⚠️ | 思维导图场景 |
| 本地优先 | ✅ | 隐私和离线支持 |
| 标签系统 | ✅ | 组织对话/笔记 |
| Daily Notes | ⚠️ | 可作为对话日志 |

---

## 参考资料

- [Obsidian Help](https://help.obsidian.md/)
- [Obsidian API](https://docs.obsidian.md/)
