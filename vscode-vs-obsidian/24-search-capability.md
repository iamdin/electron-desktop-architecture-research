# 24. 搜索能力

> **核心问题**：如何实现高效搜索？

---

## 概述

搜索能力决定了：
- 文本搜索的速度和准确性
- 搜索范围（文件、内容、符号）
- 搜索结果的展示方式
- 正则表达式支持程度

---

## VSCode 的搜索系统

### 全局搜索

VSCode 使用 ripgrep 作为后端，提供快速的全文搜索：

```typescript
// 执行搜索
const results = await vscode.workspace.findTextInFiles(
  { pattern: 'TODO' },
  {
    include: '**/*.ts',
    exclude: '**/node_modules/**',
    maxResults: 1000,
    useIgnoreFiles: true,
    useGlobalIgnoreFiles: true,
    followSymlinks: true
  }
);

results.forEach(result => {
  console.log('File:', result.uri.fsPath);
  result.matches.forEach(match => {
    console.log(`  Line ${match.lineNumber}: ${match.preview}`);
  });
});
```

### 文件搜索

```typescript
// 快速打开（文件名搜索）
const files = await vscode.workspace.findFiles(
  '**/*.{ts,js}',  // include
  '**/node_modules/**'  // exclude
);

// 注册文件搜索提供者
vscode.workspace.registerFileSearchProvider('myScheme', {
  provideFileSearchResults(query, options, token) {
    const results: vscode.Uri[] = [];

    // 根据 query.pattern 搜索文件
    for (const file of myFiles) {
      if (file.includes(query.pattern)) {
        results.push(vscode.Uri.parse(`myScheme:/${file}`));
      }
    }

    return results;
  }
});
```

### 符号搜索

```typescript
// 工作区符号搜索
vscode.languages.registerWorkspaceSymbolProvider({
  async provideWorkspaceSymbols(query, token) {
    const symbols: vscode.SymbolInformation[] = [];

    // 搜索所有文件中的符号
    const files = await vscode.workspace.findFiles('**/*.ts');
    for (const file of files) {
      const documentSymbols = await vscode.commands.executeCommand<vscode.DocumentSymbol[]>(
        'vscode.executeDocumentSymbolProvider',
        file
      );

      if (documentSymbols) {
        for (const symbol of documentSymbols) {
          if (symbol.name.toLowerCase().includes(query.toLowerCase())) {
            symbols.push(new vscode.SymbolInformation(
              symbol.name,
              symbol.kind,
              '',
              new vscode.Location(file, symbol.range)
            ));
          }
        }
      }
    }

    return symbols;
  }
});

// 文档符号
vscode.languages.registerDocumentSymbolProvider('typescript', {
  provideDocumentSymbols(document, token) {
    const symbols: vscode.DocumentSymbol[] = [];

    // 解析文档生成符号
    // 通常由语言服务提供

    return symbols;
  }
});
```

### 搜索 API

```typescript
// 注册文本搜索提供者
vscode.workspace.registerTextSearchProvider('myScheme', {
  provideTextSearchResults(query, options, progress, token) {
    // 执行搜索
    for (const file of myFiles) {
      const content = getFileContent(file);
      const matches = findMatches(content, query);

      for (const match of matches) {
        progress.report({
          uri: vscode.Uri.parse(`myScheme:/${file}`),
          ranges: [match.range],
          preview: {
            text: match.lineText,
            matches: [match.range]
          }
        });
      }
    }

    return { limitHit: false };
  }
});
```

### 搜索替换

```typescript
// 批量替换
const edit = new vscode.WorkspaceEdit();

// 搜索并替换
const searchPattern = 'oldFunction';
const replaceWith = 'newFunction';

const files = await vscode.workspace.findFiles('**/*.ts');
for (const file of files) {
  const document = await vscode.workspace.openTextDocument(file);
  const text = document.getText();

  let match;
  const regex = new RegExp(searchPattern, 'g');
  while ((match = regex.exec(text))) {
    const start = document.positionAt(match.index);
    const end = document.positionAt(match.index + match[0].length);
    edit.replace(file, new vscode.Range(start, end), replaceWith);
  }
}

await vscode.workspace.applyEdit(edit);
```

### 搜索视图贡献

```json
// package.json
{
  "contributes": {
    "viewsContainers": {
      "activitybar": [
        {
          "id": "mySearch",
          "title": "My Search",
          "icon": "resources/search.svg"
        }
      ]
    },
    "views": {
      "mySearch": [
        {
          "id": "mySearchResults",
          "name": "Search Results"
        }
      ]
    }
  }
}
```

---

## Obsidian 的搜索系统

### 内置搜索

```typescript
// 使用 Obsidian 的搜索 API
export default class MyPlugin extends Plugin {
  onload() {
    this.addCommand({
      id: 'search-notes',
      name: 'Search Notes',
      callback: () => this.searchNotes()
    });
  }

  async searchNotes() {
    const searchTerm = await this.promptForSearch();
    if (!searchTerm) return;

    // 使用全局搜索
    const searchLeaf = this.app.workspace.getLeavesOfType('search')[0];
    if (searchLeaf) {
      const searchView = searchLeaf.view as any;
      searchView.setQuery(searchTerm);
    }
  }
}
```

### 文件搜索

```typescript
// 快速切换器 (Quick Switcher)
export default class MyPlugin extends Plugin {
  onload() {
    // 扩展快速切换器
    this.registerObsidianProtocolHandler('search', (params) => {
      const query = params.query;
      // 处理搜索请求
    });
  }

  // 自定义文件搜索
  searchFiles(query: string): TFile[] {
    const files = this.app.vault.getMarkdownFiles();
    return files.filter(file => {
      return file.basename.toLowerCase().includes(query.toLowerCase());
    });
  }
}
```

### 内容搜索

```typescript
// 搜索文件内容
async searchContent(query: string): Promise<SearchResult[]> {
  const results: SearchResult[] = [];
  const files = this.app.vault.getMarkdownFiles();

  for (const file of files) {
    const content = await this.app.vault.cachedRead(file);
    const lines = content.split('\n');

    lines.forEach((line, index) => {
      if (line.toLowerCase().includes(query.toLowerCase())) {
        results.push({
          file,
          line: index + 1,
          text: line,
          match: this.getMatchContext(line, query)
        });
      }
    });
  }

  return results;
}
```

### 元数据搜索

```typescript
// 搜索标签
function searchByTag(tag: string): TFile[] {
  const files: TFile[] = [];
  const cache = this.app.metadataCache;

  this.app.vault.getMarkdownFiles().forEach(file => {
    const fileCache = cache.getFileCache(file);
    if (fileCache?.tags?.some(t => t.tag === `#${tag}`)) {
      files.push(file);
    }
  });

  return files;
}

// 搜索链接
function searchByLink(targetPath: string): TFile[] {
  const files: TFile[] = [];
  const cache = this.app.metadataCache;

  this.app.vault.getMarkdownFiles().forEach(file => {
    const fileCache = cache.getFileCache(file);
    if (fileCache?.links?.some(link => link.link === targetPath)) {
      files.push(file);
    }
  });

  return files;
}

// Dataview 风格查询
function queryNotes(predicate: (file: TFile, cache: CachedMetadata) => boolean): TFile[] {
  return this.app.vault.getMarkdownFiles().filter(file => {
    const cache = this.app.metadataCache.getFileCache(file);
    return cache && predicate(file, cache);
  });
}
```

### 搜索建议

```typescript
// 扩展搜索建议
class SearchSuggest extends EditorSuggest<SearchResult> {
  constructor(private app: App) {
    super(app);
  }

  onTrigger(cursor: EditorPosition, editor: Editor): EditorSuggestTriggerInfo | null {
    const line = editor.getLine(cursor.line);
    const match = line.match(/search:(\S*)$/);

    if (match) {
      return {
        start: { line: cursor.line, ch: cursor.ch - match[1].length },
        end: cursor,
        query: match[1]
      };
    }
    return null;
  }

  getSuggestions(context: EditorSuggestContext): SearchResult[] {
    return this.quickSearch(context.query);
  }

  renderSuggestion(result: SearchResult, el: HTMLElement): void {
    el.createEl('div', { text: result.file.basename });
    el.createEl('small', { text: result.match });
  }

  selectSuggestion(result: SearchResult): void {
    this.app.workspace.openLinkText(result.file.path, '', false);
  }
}
```

---

## 对比分析

### 搜索能力对比

| 方面 | VSCode | Obsidian |
|------|--------|----------|
| **搜索引擎** | ripgrep | 内置 |
| **文件搜索** | 快速，支持 glob | 基本 |
| **内容搜索** | 快速，正则支持 | 基本正则 |
| **符号搜索** | 语言感知 | 仅标题/链接 |
| **替换** | 批量替换 | 单文件替换 |
| **搜索语法** | 标准正则 | 简化语法 |
| **索引** | 无需索引 | MetadataCache |

---

## 对 AI Chat + Editor 应用的建议

### 搜索服务架构

```typescript
interface ISearchService {
  // 文件搜索
  searchFiles(query: string, options?: FileSearchOptions): Promise<FileMatch[]>;

  // 内容搜索
  searchContent(query: string, options?: ContentSearchOptions): Promise<ContentMatch[]>;

  // 符号搜索
  searchSymbols(query: string, options?: SymbolSearchOptions): Promise<SymbolMatch[]>;

  // AI 语义搜索
  searchSemantic(query: string, options?: SemanticSearchOptions): Promise<SemanticMatch[]>;
}

interface FileSearchOptions {
  include?: string[];
  exclude?: string[];
  maxResults?: number;
  fuzzy?: boolean;
}

interface ContentSearchOptions extends FileSearchOptions {
  regex?: boolean;
  caseSensitive?: boolean;
  wholeWord?: boolean;
}
```

### 基于 ripgrep 实现

```typescript
import { spawn } from 'child_process';

class RipgrepSearchService {
  async searchContent(
    pattern: string,
    rootPath: string,
    options: ContentSearchOptions = {}
  ): Promise<ContentMatch[]> {
    const args = this.buildArgs(pattern, options);

    return new Promise((resolve, reject) => {
      const results: ContentMatch[] = [];
      const rg = spawn('rg', args, { cwd: rootPath });

      let stdout = '';
      rg.stdout.on('data', (data) => {
        stdout += data.toString();
      });

      rg.on('close', (code) => {
        if (code === 0 || code === 1) {
          const matches = this.parseOutput(stdout);
          resolve(matches);
        } else {
          reject(new Error(`ripgrep exited with code ${code}`));
        }
      });
    });
  }

  private buildArgs(pattern: string, options: ContentSearchOptions): string[] {
    const args = ['--json'];

    if (options.regex) {
      args.push('-e', pattern);
    } else {
      args.push('-F', pattern); // 固定字符串
    }

    if (!options.caseSensitive) {
      args.push('-i');
    }

    if (options.wholeWord) {
      args.push('-w');
    }

    if (options.include) {
      options.include.forEach(glob => {
        args.push('-g', glob);
      });
    }

    if (options.exclude) {
      options.exclude.forEach(glob => {
        args.push('-g', `!${glob}`);
      });
    }

    if (options.maxResults) {
      args.push('-m', String(options.maxResults));
    }

    args.push('.'); // 搜索当前目录

    return args;
  }

  private parseOutput(output: string): ContentMatch[] {
    const results: ContentMatch[] = [];

    output.split('\n').filter(Boolean).forEach(line => {
      try {
        const data = JSON.parse(line);
        if (data.type === 'match') {
          results.push({
            file: data.data.path.text,
            line: data.data.line_number,
            column: data.data.submatches[0]?.start || 0,
            text: data.data.lines.text,
            preview: this.createPreview(data.data)
          });
        }
      } catch (e) {
        // 忽略解析错误
      }
    });

    return results;
  }
}
```

### AI 语义搜索

```typescript
class SemanticSearchService {
  private embeddings: Map<string, number[]> = new Map();

  constructor(
    private aiClient: IAIClient,
    private vectorStore: IVectorStore
  ) {}

  async indexFile(file: string, content: string): Promise<void> {
    // 分块
    const chunks = this.splitIntoChunks(content);

    // 生成嵌入
    const embeddings = await this.aiClient.embeddings(chunks);

    // 存储
    for (let i = 0; i < chunks.length; i++) {
      await this.vectorStore.upsert({
        id: `${file}:${i}`,
        vector: embeddings[i],
        metadata: {
          file,
          chunk: i,
          text: chunks[i]
        }
      });
    }
  }

  async search(query: string, options?: SemanticSearchOptions): Promise<SemanticMatch[]> {
    // 生成查询嵌入
    const queryEmbedding = await this.aiClient.embeddings([query]);

    // 向量搜索
    const results = await this.vectorStore.query({
      vector: queryEmbedding[0],
      topK: options?.maxResults || 10,
      filter: options?.filter
    });

    return results.map(r => ({
      file: r.metadata.file,
      text: r.metadata.text,
      score: r.score,
      chunk: r.metadata.chunk
    }));
  }

  private splitIntoChunks(content: string, chunkSize = 500): string[] {
    const chunks: string[] = [];
    const sentences = content.split(/[.!?]\s+/);

    let currentChunk = '';
    for (const sentence of sentences) {
      if (currentChunk.length + sentence.length > chunkSize) {
        if (currentChunk) chunks.push(currentChunk);
        currentChunk = sentence;
      } else {
        currentChunk += (currentChunk ? '. ' : '') + sentence;
      }
    }
    if (currentChunk) chunks.push(currentChunk);

    return chunks;
  }
}
```

### 混合搜索

```typescript
class HybridSearchService implements ISearchService {
  constructor(
    private textSearch: RipgrepSearchService,
    private semanticSearch: SemanticSearchService
  ) {}

  async search(
    query: string,
    options: HybridSearchOptions = {}
  ): Promise<SearchResult[]> {
    // 判断查询类型
    const isSemanticQuery = this.isNaturalLanguageQuery(query);

    if (options.mode === 'text' || !isSemanticQuery) {
      return this.textSearch.searchContent(query, options);
    }

    if (options.mode === 'semantic') {
      return this.semanticSearch.search(query, options);
    }

    // 混合模式：同时执行
    const [textResults, semanticResults] = await Promise.all([
      this.textSearch.searchContent(query, options),
      this.semanticSearch.search(query, options)
    ]);

    // 合并并排序结果
    return this.mergeResults(textResults, semanticResults, options);
  }

  private isNaturalLanguageQuery(query: string): boolean {
    // 简单启发式：包含空格且长度较长
    return query.includes(' ') && query.length > 20;
  }

  private mergeResults(
    textResults: ContentMatch[],
    semanticResults: SemanticMatch[],
    options: HybridSearchOptions
  ): SearchResult[] {
    const merged = new Map<string, SearchResult>();

    // 文本匹配权重
    textResults.forEach((r, i) => {
      const key = `${r.file}:${r.line}`;
      merged.set(key, {
        ...r,
        score: 1 - (i / textResults.length) * 0.5
      });
    });

    // 语义匹配权重
    semanticResults.forEach(r => {
      const key = r.file;
      const existing = merged.get(key);
      if (existing) {
        existing.score = (existing.score + r.score) / 2;
      } else {
        merged.set(key, {
          file: r.file,
          text: r.text,
          score: r.score * 0.8 // 稍微降低纯语义结果
        });
      }
    });

    return Array.from(merged.values())
      .sort((a, b) => b.score - a.score)
      .slice(0, options.maxResults || 50);
  }
}
```

### 搜索 UI 组件

```tsx
function SearchPanel() {
  const [query, setQuery] = useState('');
  const [results, setResults] = useState<SearchResult[]>([]);
  const [mode, setMode] = useState<'text' | 'semantic' | 'hybrid'>('hybrid');
  const [loading, setLoading] = useState(false);

  const searchService = useService(ISearchService);

  const handleSearch = async () => {
    if (!query.trim()) return;

    setLoading(true);
    try {
      const results = await searchService.search(query, { mode });
      setResults(results);
    } finally {
      setLoading(false);
    }
  };

  return (
    <div className="search-panel">
      <div className="search-header">
        <input
          type="text"
          value={query}
          onChange={(e) => setQuery(e.target.value)}
          onKeyDown={(e) => e.key === 'Enter' && handleSearch()}
          placeholder="Search files and content..."
        />
        <select value={mode} onChange={(e) => setMode(e.target.value as any)}>
          <option value="text">Text Search</option>
          <option value="semantic">AI Search</option>
          <option value="hybrid">Hybrid</option>
        </select>
      </div>

      <div className="search-results">
        {loading ? (
          <div className="loading">Searching...</div>
        ) : (
          results.map((result, index) => (
            <SearchResultItem
              key={`${result.file}:${index}`}
              result={result}
              query={query}
            />
          ))
        )}
      </div>
    </div>
  );
}

function SearchResultItem({ result, query }: { result: SearchResult; query: string }) {
  const highlightMatch = (text: string) => {
    const parts = text.split(new RegExp(`(${query})`, 'gi'));
    return parts.map((part, i) =>
      part.toLowerCase() === query.toLowerCase()
        ? <mark key={i}>{part}</mark>
        : part
    );
  };

  return (
    <div className="search-result-item" onClick={() => openFile(result.file, result.line)}>
      <div className="result-file">{result.file}</div>
      {result.line && <div className="result-line">Line {result.line}</div>}
      <div className="result-preview">{highlightMatch(result.text)}</div>
      {result.score && (
        <div className="result-score">
          Score: {(result.score * 100).toFixed(0)}%
        </div>
      )}
    </div>
  );
}
```

### 搜索替换

```typescript
class SearchReplaceService {
  constructor(private searchService: ISearchService) {}

  async replaceAll(
    searchPattern: string,
    replaceWith: string,
    options: ReplaceOptions = {}
  ): Promise<ReplaceResult> {
    const results = await this.searchService.searchContent(searchPattern, options);

    // 按文件分组
    const fileGroups = this.groupByFile(results);

    let totalReplacements = 0;
    const affectedFiles: string[] = [];

    for (const [file, matches] of Object.entries(fileGroups)) {
      if (options.preview) {
        // 预览模式，不实际修改
        continue;
      }

      const content = await this.readFile(file);
      let newContent = content;

      // 从后往前替换，避免位置偏移
      const sortedMatches = matches.sort((a, b) => b.line - a.line);
      for (const match of sortedMatches) {
        newContent = this.replaceMatch(newContent, match, replaceWith);
        totalReplacements++;
      }

      await this.writeFile(file, newContent);
      affectedFiles.push(file);
    }

    return {
      totalReplacements,
      affectedFiles,
      preview: options.preview ? this.generatePreview(fileGroups, replaceWith) : undefined
    };
  }
}
```

---

## 关键决策清单

1. **搜索引擎选择？**
   - ripgrep：快速、成熟
   - 自研：可定制

2. **是否需要索引？**
   - 实时搜索：无需索引
   - 大量文件：需要索引

3. **是否需要语义搜索？**
   - 是：需要向量数据库
   - 否：传统文本搜索

4. **搜索范围？**
   - 仅文本
   - 符号/引用
   - 语义理解

---

## 参考资料

- [ripgrep](https://github.com/BurntSushi/ripgrep)
- [VSCode Search](https://code.visualstudio.com/docs/editor/codebasics#_search-across-files)
- [Vector Search](https://www.pinecone.io/learn/what-is-similarity-search/)
