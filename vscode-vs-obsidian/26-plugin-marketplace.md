# 26. 插件市场

> **核心问题**：如何构建插件生态？

---

## 概述

插件市场决定了：
- 插件如何发现和安装
- 插件如何发布和更新
- 如何保证插件质量
- 如何建立开发者生态

---

## VSCode 的插件市场

### Marketplace 架构

VSCode 使用 Azure DevOps 托管的 Marketplace：

```
https://marketplace.visualstudio.com/
├── Extensions
│   ├── Categories (Languages, Themes, Snippets, etc.)
│   ├── Featured
│   ├── Trending
│   └── Search
└── API
    ├── Search
    ├── Details
    ├── Download
    └── Statistics
```

### 发布流程

```bash
# 安装 vsce 工具
npm install -g @vscode/vsce

# 登录（需要 Personal Access Token）
vsce login <publisher>

# 打包
vsce package

# 发布
vsce publish

# 版本更新
vsce publish minor
vsce publish patch
```

### package.json 配置

```json
{
  "name": "my-extension",
  "displayName": "My Extension",
  "description": "Description for marketplace",
  "version": "1.0.0",
  "publisher": "my-publisher",
  "engines": {
    "vscode": "^1.70.0"
  },
  "categories": [
    "Programming Languages",
    "Snippets",
    "Themes"
  ],
  "keywords": [
    "keyword1",
    "keyword2"
  ],
  "icon": "images/icon.png",
  "galleryBanner": {
    "color": "#1e1e1e",
    "theme": "dark"
  },
  "repository": {
    "type": "git",
    "url": "https://github.com/user/repo"
  },
  "bugs": {
    "url": "https://github.com/user/repo/issues"
  },
  "license": "MIT",
  "preview": false,
  "main": "./out/extension.js",
  "contributes": {},
  "activationEvents": []
}
```

### 市场 API

```typescript
// 内置的扩展管理 API（有限）
vscode.extensions.all.forEach(ext => {
  console.log(ext.id, ext.packageJSON.version);
});

// 检查扩展是否安装
const pythonExt = vscode.extensions.getExtension('ms-python.python');
if (pythonExt && !pythonExt.isActive) {
  await pythonExt.activate();
}

// 推荐扩展
vscode.commands.executeCommand('workbench.extensions.action.showRecommendedExtensions');

// 安装扩展（通过命令）
vscode.commands.executeCommand('workbench.extensions.installExtension', 'ext-id');
```

### 工作区推荐

```json
// .vscode/extensions.json
{
  "recommendations": [
    "dbaeumer.vscode-eslint",
    "esbenp.prettier-vscode",
    "ms-python.python"
  ],
  "unwantedRecommendations": [
    "old-extension.deprecated"
  ]
}
```

---

## Obsidian 的插件市场

### 社区插件系统

```typescript
// Obsidian 社区插件通过 GitHub 托管
// 插件列表: https://github.com/obsidianmd/obsidian-releases

// manifest.json
{
  "id": "my-plugin",
  "name": "My Plugin",
  "version": "1.0.0",
  "minAppVersion": "0.15.0",
  "description": "Plugin description",
  "author": "Your Name",
  "authorUrl": "https://github.com/yourname",
  "isDesktopOnly": false
}
```

### 发布流程

```markdown
1. 创建 GitHub Release
2. 包含文件：
   - main.js（编译后的代码）
   - manifest.json
   - styles.css（可选）

3. 提交 PR 到 obsidian-releases 仓库
4. 等待官方审核
5. 合并后自动上线
```

### 插件安装

```typescript
// 用户通过设置页面浏览和安装
// 插件 -> 社区插件 -> 浏览

// 插件也可以手动安装
// 复制到 .obsidian/plugins/my-plugin/
// - main.js
// - manifest.json
// - data.json（可选，设置数据）
```

### 更新检查

```typescript
// Obsidian 定期检查 obsidian-releases 仓库
// 比较本地 manifest.json 和远程版本

// 插件可以检查自己的更新
async checkForUpdates() {
  const response = await fetch(
    `https://raw.githubusercontent.com/user/repo/main/manifest.json`
  );
  const remote = await response.json();

  if (remote.version !== this.manifest.version) {
    new Notice(`Update available: ${remote.version}`);
  }
}
```

---

## 对比分析

### 市场对比

| 方面 | VSCode | Obsidian |
|------|--------|----------|
| **托管方式** | Azure Marketplace | GitHub |
| **审核流程** | 自动化 + 人工 | 人工审核 |
| **发布工具** | vsce CLI | GitHub Release |
| **更新方式** | 自动检查 | 定期检查 |
| **付费插件** | 不支持 | 不支持 |
| **统计数据** | 详细（下载量、评分） | 有限 |
| **搜索功能** | 完善 | 基本 |

---

## 对 AI Chat + Editor 应用的建议

### 市场架构设计

```typescript
interface IMarketplace {
  // 搜索
  search(query: string, options?: SearchOptions): Promise<PluginInfo[]>;

  // 详情
  getDetails(id: string): Promise<PluginDetails>;

  // 安装
  install(id: string, version?: string): Promise<void>;
  uninstall(id: string): Promise<void>;

  // 更新
  checkUpdates(): Promise<PluginUpdate[]>;
  update(id: string): Promise<void>;
  updateAll(): Promise<void>;

  // 事件
  onDidInstall: Event<PluginInfo>;
  onDidUninstall: Event<string>;
  onDidUpdate: Event<PluginInfo>;
}

interface PluginInfo {
  id: string;
  name: string;
  description: string;
  version: string;
  author: string;
  repository: string;
  categories: string[];
  tags: string[];
  downloads: number;
  rating: number;
  icon?: string;
}
```

### 自建市场服务

```typescript
// 后端 API 设计
class MarketplaceService {
  // GET /api/plugins
  async listPlugins(query: PluginQuery): Promise<PluginList> {
    const { search, category, sort, page, limit } = query;

    const plugins = await this.db.plugins
      .find({
        ...(search && { $text: { $search: search } }),
        ...(category && { categories: category }),
        verified: true
      })
      .sort(this.getSortCriteria(sort))
      .skip((page - 1) * limit)
      .limit(limit);

    return {
      plugins,
      total: await this.db.plugins.countDocuments(),
      page,
      limit
    };
  }

  // GET /api/plugins/:id
  async getPlugin(id: string): Promise<PluginDetails> {
    const plugin = await this.db.plugins.findOne({ id });
    if (!plugin) throw new NotFoundError('Plugin not found');

    // 获取版本历史
    const versions = await this.db.versions
      .find({ pluginId: id })
      .sort({ createdAt: -1 });

    return { ...plugin, versions };
  }

  // POST /api/plugins/:id/download
  async getDownloadUrl(id: string, version?: string): Promise<string> {
    const v = version || await this.getLatestVersion(id);

    // 增加下载计数
    await this.db.plugins.updateOne(
      { id },
      { $inc: { downloads: 1 } }
    );

    // 返回签名的下载 URL
    return this.storage.getSignedUrl(`plugins/${id}/${v}/bundle.zip`);
  }

  // POST /api/plugins (发布)
  async publishPlugin(data: PublishRequest, user: User): Promise<PluginInfo> {
    // 验证 manifest
    const manifest = JSON.parse(data.manifest);
    this.validateManifest(manifest);

    // 安全扫描
    await this.securityScanner.scan(data.bundle);

    // 存储
    await this.storage.upload(
      `plugins/${manifest.id}/${manifest.version}/bundle.zip`,
      data.bundle
    );

    // 更新数据库
    const plugin = await this.db.plugins.findOneAndUpdate(
      { id: manifest.id },
      {
        $set: {
          name: manifest.name,
          description: manifest.description,
          author: user.name,
          authorId: user.id,
          latestVersion: manifest.version,
          updatedAt: new Date()
        },
        $setOnInsert: {
          createdAt: new Date(),
          downloads: 0
        }
      },
      { upsert: true, returnDocument: 'after' }
    );

    // 记录版本
    await this.db.versions.insertOne({
      pluginId: manifest.id,
      version: manifest.version,
      changelog: data.changelog,
      createdAt: new Date()
    });

    return plugin;
  }
}
```

### 客户端实现

```typescript
class MarketplaceClient implements IMarketplace {
  constructor(
    private apiUrl: string,
    private pluginService: IPluginService
  ) {}

  async search(query: string, options?: SearchOptions): Promise<PluginInfo[]> {
    const params = new URLSearchParams({
      search: query,
      ...(options?.category && { category: options.category }),
      ...(options?.sort && { sort: options.sort }),
      page: String(options?.page || 1),
      limit: String(options?.limit || 20)
    });

    const response = await fetch(`${this.apiUrl}/plugins?${params}`);
    const data = await response.json();
    return data.plugins;
  }

  async install(id: string, version?: string): Promise<void> {
    // 获取下载地址
    const response = await fetch(`${this.apiUrl}/plugins/${id}/download`, {
      method: 'POST',
      body: JSON.stringify({ version })
    });
    const { url } = await response.json();

    // 下载插件
    const bundle = await this.download(url);

    // 解压并安装
    const pluginPath = path.join(this.pluginDir, id);
    await this.extract(bundle, pluginPath);

    // 加载插件
    await this.pluginService.load(id);

    this._onDidInstall.fire(await this.getDetails(id));
  }

  async checkUpdates(): Promise<PluginUpdate[]> {
    const installed = this.pluginService.getAll();
    const updates: PluginUpdate[] = [];

    // 批量检查
    const response = await fetch(`${this.apiUrl}/plugins/check-updates`, {
      method: 'POST',
      body: JSON.stringify({
        plugins: installed.map(p => ({
          id: p.id,
          version: p.version
        }))
      })
    });

    const data = await response.json();

    for (const update of data.updates) {
      const local = installed.find(p => p.id === update.id);
      if (local && semver.gt(update.version, local.version)) {
        updates.push({
          id: update.id,
          currentVersion: local.version,
          newVersion: update.version,
          changelog: update.changelog
        });
      }
    }

    return updates;
  }
}
```

### 基于 GitHub 的简化方案

```typescript
// 使用 GitHub 作为插件仓库
class GitHubMarketplace {
  private readonly registryUrl = 'https://raw.githubusercontent.com/org/plugin-registry/main';

  async getRegistry(): Promise<PluginRegistry> {
    const response = await fetch(`${this.registryUrl}/registry.json`);
    return response.json();
  }

  async search(query: string): Promise<PluginInfo[]> {
    const registry = await this.getRegistry();

    return registry.plugins.filter(p =>
      p.name.toLowerCase().includes(query.toLowerCase()) ||
      p.description.toLowerCase().includes(query.toLowerCase())
    );
  }

  async install(id: string): Promise<void> {
    const registry = await this.getRegistry();
    const plugin = registry.plugins.find(p => p.id === id);

    if (!plugin) throw new Error('Plugin not found');

    // 从 GitHub Release 下载
    const releaseUrl = `https://github.com/${plugin.repo}/releases/latest/download/bundle.zip`;
    const bundle = await this.download(releaseUrl);

    await this.extract(bundle, path.join(this.pluginDir, id));
  }
}

// registry.json 格式
interface PluginRegistry {
  version: number;
  plugins: Array<{
    id: string;
    name: string;
    description: string;
    repo: string; // GitHub repo: "user/repo"
    version: string;
    minAppVersion: string;
    categories: string[];
  }>;
}
```

### 市场 UI 组件

```tsx
function MarketplacePanel() {
  const [plugins, setPlugins] = useState<PluginInfo[]>([]);
  const [search, setSearch] = useState('');
  const [category, setCategory] = useState<string | null>(null);
  const [loading, setLoading] = useState(false);

  const marketplace = useService(IMarketplace);

  useEffect(() => {
    loadPlugins();
  }, [search, category]);

  const loadPlugins = async () => {
    setLoading(true);
    try {
      const results = await marketplace.search(search, { category });
      setPlugins(results);
    } finally {
      setLoading(false);
    }
  };

  return (
    <div className="marketplace">
      <div className="marketplace-header">
        <input
          type="text"
          value={search}
          onChange={(e) => setSearch(e.target.value)}
          placeholder="Search plugins..."
        />
        <CategoryFilter value={category} onChange={setCategory} />
      </div>

      <div className="plugin-list">
        {loading ? (
          <Loading />
        ) : (
          plugins.map(plugin => (
            <PluginCard key={plugin.id} plugin={plugin} />
          ))
        )}
      </div>
    </div>
  );
}

function PluginCard({ plugin }: { plugin: PluginInfo }) {
  const [installing, setInstalling] = useState(false);
  const marketplace = useService(IMarketplace);
  const pluginService = useService(IPluginService);

  const isInstalled = pluginService.isInstalled(plugin.id);

  const handleInstall = async () => {
    setInstalling(true);
    try {
      await marketplace.install(plugin.id);
    } finally {
      setInstalling(false);
    }
  };

  return (
    <div className="plugin-card">
      <img src={plugin.icon} alt={plugin.name} className="plugin-icon" />
      <div className="plugin-info">
        <h3>{plugin.name}</h3>
        <p className="author">by {plugin.author}</p>
        <p className="description">{plugin.description}</p>
        <div className="stats">
          <span>⬇️ {plugin.downloads}</span>
          <span>⭐ {plugin.rating}</span>
        </div>
      </div>
      <div className="plugin-actions">
        {isInstalled ? (
          <button disabled>Installed</button>
        ) : (
          <button onClick={handleInstall} disabled={installing}>
            {installing ? 'Installing...' : 'Install'}
          </button>
        )}
      </div>
    </div>
  );
}
```

### 插件审核流程

```typescript
class PluginReviewService {
  async submitForReview(pluginId: string, version: string): Promise<ReviewRequest> {
    // 创建审核请求
    const request = await this.db.reviews.insertOne({
      pluginId,
      version,
      status: 'pending',
      submittedAt: new Date()
    });

    // 自动化检查
    const autoChecks = await this.runAutoChecks(pluginId, version);

    if (autoChecks.passed) {
      // 标记为待人工审核
      await this.db.reviews.updateOne(
        { _id: request.insertedId },
        { $set: { status: 'auto_approved', autoChecks } }
      );
    } else {
      // 自动拒绝
      await this.db.reviews.updateOne(
        { _id: request.insertedId },
        { $set: { status: 'rejected', autoChecks, reason: autoChecks.issues } }
      );
    }

    return request;
  }

  private async runAutoChecks(pluginId: string, version: string): Promise<AutoCheckResult> {
    const issues: string[] = [];

    // 安全扫描
    const securityResult = await this.securityScanner.scan(pluginId, version);
    if (!securityResult.passed) {
      issues.push(...securityResult.vulnerabilities);
    }

    // 清单验证
    const manifest = await this.getManifest(pluginId, version);
    if (!this.validateManifest(manifest)) {
      issues.push('Invalid manifest');
    }

    // 代码质量
    const codeResult = await this.codeAnalyzer.analyze(pluginId, version);
    if (codeResult.issues.length > 0) {
      issues.push(...codeResult.issues);
    }

    return {
      passed: issues.length === 0,
      issues
    };
  }
}
```

---

## 关键决策清单

1. **自建还是使用现有平台？**
   - 自建：完全控制
   - GitHub：简单，成本低

2. **审核流程？**
   - 自动化：速度快
   - 人工：质量高

3. **付费插件？**
   - 支持：需要支付系统
   - 不支持：简化运营

4. **更新机制？**
   - 自动更新
   - 手动确认

---

## 参考资料

- [VSCode Marketplace](https://marketplace.visualstudio.com/)
- [Publishing Extensions](https://code.visualstudio.com/api/working-with-extensions/publishing-extension)
- [Obsidian Plugin Releases](https://github.com/obsidianmd/obsidian-releases)
