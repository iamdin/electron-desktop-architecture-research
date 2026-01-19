# 25. 版本控制

> **核心问题**：如何集成版本控制？

---

## 概述

版本控制集成决定了：
- Git 操作如何执行
- 变更如何可视化
- 如何处理合并冲突
- 如何支持协作工作流

---

## VSCode 的版本控制

### Source Control API

VSCode 通过抽象的 Source Control API 支持多种版本控制系统：

```typescript
// 注册 Source Control Provider
const gitProvider = vscode.scm.createSourceControl('git', 'Git');

// 创建资源组
const stagedChanges = gitProvider.createResourceGroup('staged', 'Staged Changes');
const workingChanges = gitProvider.createResourceGroup('changes', 'Changes');

// 设置资源
stagedChanges.resourceStates = [
  {
    resourceUri: vscode.Uri.file('/path/to/file.ts'),
    decorations: {
      strikeThrough: false,
      faded: false,
      tooltip: 'Modified'
    },
    command: {
      command: 'vscode.diff',
      title: 'Show Changes',
      arguments: [
        vscode.Uri.file('/path/to/file.ts~HEAD'),
        vscode.Uri.file('/path/to/file.ts'),
        'file.ts (HEAD vs Working)'
      ]
    }
  }
];

// 设置输入框
gitProvider.inputBox.placeholder = 'Commit message';
gitProvider.inputBox.value = '';

// 快速 Diff 装饰
gitProvider.quickDiffProvider = {
  provideOriginalResource(uri: vscode.Uri): vscode.Uri | null {
    return uri.with({ scheme: 'git', query: 'HEAD' });
  }
};

// 接受输入（提交）
gitProvider.acceptInputCommand = {
  command: 'git.commit',
  title: 'Commit'
};

// 计数徽章
gitProvider.count = 5;
```

### Git 扩展 API

```typescript
// 获取 Git 扩展
const gitExtension = vscode.extensions.getExtension<GitExtension>('vscode.git');
const git = gitExtension?.exports.getAPI(1);

if (git) {
  // 获取仓库
  const repos = git.repositories;

  for (const repo of repos) {
    // 获取状态
    const head = repo.state.HEAD;
    console.log('Branch:', head?.name);
    console.log('Commit:', head?.commit);

    // 获取变更
    const changes = repo.state.workingTreeChanges;
    changes.forEach(change => {
      console.log('Changed:', change.uri.fsPath, change.status);
    });

    // 获取远程
    const remotes = repo.state.remotes;
    remotes.forEach(remote => {
      console.log('Remote:', remote.name, remote.fetchUrl);
    });

    // 执行 Git 命令
    await repo.checkout('main');
    await repo.pull();
    await repo.push();

    // 暂存
    await repo.add([vscode.Uri.file('/path/to/file.ts')]);

    // 提交
    await repo.commit('Commit message');

    // 创建分支
    await repo.createBranch('feature/new', true);
  }

  // 监听变化
  git.onDidChangeState(() => {
    console.log('Git state changed');
  });
}
```

### Diff 编辑器

```typescript
// 打开 Diff 视图
vscode.commands.executeCommand('vscode.diff',
  vscode.Uri.file('/path/old.ts'),
  vscode.Uri.file('/path/new.ts'),
  'Compare Files'
);

// 注册 Diff 内容提供者
vscode.workspace.registerTextDocumentContentProvider('git', {
  provideTextDocumentContent(uri: vscode.Uri): string {
    // 返回指定版本的内容
    const ref = uri.query; // HEAD, commit hash, etc.
    return getGitFileContent(uri.fsPath, ref);
  }
});
```

### 编辑器装饰（Gutter）

```typescript
// 行级别变更装饰
const addedDecoration = vscode.window.createTextEditorDecorationType({
  gutterIconPath: '/path/to/added.svg',
  gutterIconSize: 'contain',
  overviewRulerColor: 'green',
  overviewRulerLane: vscode.OverviewRulerLane.Left
});

const deletedDecoration = vscode.window.createTextEditorDecorationType({
  gutterIconPath: '/path/to/deleted.svg',
  overviewRulerColor: 'red'
});

// 应用装饰
editor.setDecorations(addedDecoration, addedRanges);
editor.setDecorations(deletedDecoration, deletedRanges);
```

---

## Obsidian 的版本控制

### 无内置 Git 支持

Obsidian 不内置版本控制，主要通过第三方插件：

```typescript
// Obsidian Git 插件示例
import simpleGit, { SimpleGit } from 'simple-git';

export default class ObsidianGit extends Plugin {
  private git: SimpleGit;

  async onload() {
    const basePath = (this.app.vault.adapter as any).basePath;
    this.git = simpleGit(basePath);

    // 检查是否是 Git 仓库
    const isRepo = await this.git.checkIsRepo();
    if (!isRepo) {
      new Notice('Not a git repository');
      return;
    }

    // 添加命令
    this.addCommand({
      id: 'commit',
      name: 'Commit all changes',
      callback: () => this.commitAll()
    });

    this.addCommand({
      id: 'push',
      name: 'Push changes',
      callback: () => this.push()
    });

    this.addCommand({
      id: 'pull',
      name: 'Pull changes',
      callback: () => this.pull()
    });

    // 自动备份
    if (this.settings.autoBackup) {
      this.registerInterval(
        window.setInterval(() => this.autoBackup(), this.settings.autoBackupInterval)
      );
    }
  }

  async commitAll() {
    try {
      await this.git.add('.');
      await this.git.commit(this.getCommitMessage());
      new Notice('Committed successfully');
    } catch (error) {
      new Notice(`Commit failed: ${error.message}`);
    }
  }

  async push() {
    try {
      await this.git.push();
      new Notice('Pushed successfully');
    } catch (error) {
      new Notice(`Push failed: ${error.message}`);
    }
  }

  async pull() {
    try {
      await this.git.pull();
      new Notice('Pulled successfully');
    } catch (error) {
      new Notice(`Pull failed: ${error.message}`);
    }
  }

  private getCommitMessage(): string {
    const date = new Date().toISOString();
    return `Vault backup: ${date}`;
  }
}
```

### 文件历史

```typescript
// 查看文件历史
class FileHistoryView extends ItemView {
  private file: TFile;
  private history: GitLogEntry[] = [];

  async loadHistory() {
    const git = simpleGit(this.getVaultPath());
    const log = await git.log({
      file: this.file.path,
      maxCount: 50
    });

    this.history = log.all.map(entry => ({
      hash: entry.hash,
      date: entry.date,
      message: entry.message,
      author: entry.author_name
    }));

    this.render();
  }

  render() {
    const container = this.containerEl.children[1];
    container.empty();

    this.history.forEach(entry => {
      const el = container.createEl('div', { cls: 'history-entry' });
      el.createEl('span', { text: entry.hash.slice(0, 7), cls: 'hash' });
      el.createEl('span', { text: entry.message, cls: 'message' });
      el.createEl('span', { text: entry.date, cls: 'date' });

      el.addEventListener('click', () => this.showVersion(entry.hash));
    });
  }

  async showVersion(hash: string) {
    const git = simpleGit(this.getVaultPath());
    const content = await git.show([`${hash}:${this.file.path}`]);

    // 显示历史版本
    const leaf = this.app.workspace.getLeaf('split');
    await leaf.openFile(this.file);
    // 设置为只读并显示历史内容
  }
}
```

---

## 对比分析

### 版本控制对比

| 方面 | VSCode | Obsidian |
|------|--------|----------|
| **内置支持** | 完整 Git 集成 | 无 |
| **API** | Source Control API | 需插件 |
| **可视化** | Diff 编辑器、Gutter | 需插件 |
| **合并冲突** | 三向合并 | 无 |
| **Git 扩展** | 官方扩展 | 社区插件 |
| **操作方式** | GUI + 命令 | 主要命令 |

---

## 对 AI Chat + Editor 应用的建议

### Git 服务架构

```typescript
interface IGitService {
  // 仓库操作
  init(path: string): Promise<void>;
  clone(url: string, path: string): Promise<void>;
  open(path: string): Promise<IGitRepository>;

  // 仓库发现
  discover(path: string): Promise<IGitRepository | null>;
}

interface IGitRepository {
  // 状态
  getStatus(): Promise<GitStatus>;
  getBranches(): Promise<GitBranch[]>;
  getCurrentBranch(): Promise<GitBranch | null>;
  getRemotes(): Promise<GitRemote[]>;
  getLog(options?: GitLogOptions): Promise<GitLogEntry[]>;

  // 工作区操作
  stage(paths: string[]): Promise<void>;
  unstage(paths: string[]): Promise<void>;
  discard(paths: string[]): Promise<void>;
  commit(message: string, options?: CommitOptions): Promise<string>;

  // 分支操作
  checkout(ref: string): Promise<void>;
  createBranch(name: string, startPoint?: string): Promise<void>;
  deleteBranch(name: string, force?: boolean): Promise<void>;
  merge(branch: string): Promise<MergeResult>;

  // 远程操作
  fetch(remote?: string): Promise<void>;
  pull(remote?: string, branch?: string): Promise<void>;
  push(remote?: string, branch?: string, options?: PushOptions): Promise<void>;

  // Diff
  diff(ref?: string): Promise<GitDiff[]>;
  diffFile(path: string, ref?: string): Promise<string>;

  // 事件
  onDidChange: Event<void>;
}
```

### 基于 simple-git 实现

```typescript
import simpleGit, { SimpleGit, StatusResult } from 'simple-git';

class GitRepository implements IGitRepository {
  private git: SimpleGit;
  private _onDidChange = new Emitter<void>();
  readonly onDidChange = this._onDidChange.event;

  constructor(private path: string) {
    this.git = simpleGit(path);
    this.watchRepository();
  }

  private watchRepository() {
    // 监听 .git 目录变化
    const watcher = fs.watch(path.join(this.path, '.git'), { recursive: true });
    watcher.on('change', () => {
      this._onDidChange.fire();
    });
  }

  async getStatus(): Promise<GitStatus> {
    const status = await this.git.status();

    return {
      branch: status.current || '',
      ahead: status.ahead,
      behind: status.behind,
      staged: status.staged.map(f => this.mapFileStatus(f, 'staged')),
      unstaged: status.modified.map(f => this.mapFileStatus(f, 'modified')),
      untracked: status.not_added.map(f => this.mapFileStatus(f, 'untracked'))
    };
  }

  async stage(paths: string[]): Promise<void> {
    await this.git.add(paths);
    this._onDidChange.fire();
  }

  async commit(message: string, options?: CommitOptions): Promise<string> {
    const result = await this.git.commit(message, undefined, {
      '--amend': options?.amend,
      '--no-verify': options?.noVerify
    });
    this._onDidChange.fire();
    return result.commit;
  }

  async diff(ref = 'HEAD'): Promise<GitDiff[]> {
    const summary = await this.git.diffSummary([ref]);

    return summary.files.map(file => ({
      path: file.file,
      insertions: file.insertions,
      deletions: file.deletions,
      binary: file.binary
    }));
  }

  async diffFile(filePath: string, ref = 'HEAD'): Promise<string> {
    return this.git.diff([ref, '--', filePath]);
  }

  async getLog(options?: GitLogOptions): Promise<GitLogEntry[]> {
    const log = await this.git.log({
      maxCount: options?.maxCount || 100,
      file: options?.file,
      from: options?.from,
      to: options?.to
    });

    return log.all.map(entry => ({
      hash: entry.hash,
      message: entry.message,
      author: entry.author_name,
      email: entry.author_email,
      date: new Date(entry.date)
    }));
  }

  async merge(branch: string): Promise<MergeResult> {
    try {
      await this.git.merge([branch]);
      return { success: true };
    } catch (error) {
      // 检查合并冲突
      const status = await this.git.status();
      if (status.conflicted.length > 0) {
        return {
          success: false,
          conflicts: status.conflicted
        };
      }
      throw error;
    }
  }
}
```

### 合并冲突解决

```typescript
interface ConflictMarker {
  start: number;
  separator: number;
  end: number;
  ours: string;
  theirs: string;
}

class ConflictResolver {
  parseConflicts(content: string): ConflictMarker[] {
    const conflicts: ConflictMarker[] = [];
    const lines = content.split('\n');

    let current: Partial<ConflictMarker> | null = null;

    for (let i = 0; i < lines.length; i++) {
      const line = lines[i];

      if (line.startsWith('<<<<<<<')) {
        current = { start: i, ours: '' };
      } else if (line.startsWith('=======') && current) {
        current.separator = i;
      } else if (line.startsWith('>>>>>>>') && current) {
        current.end = i;
        // 提取内容
        current.ours = lines.slice(current.start! + 1, current.separator).join('\n');
        current.theirs = lines.slice(current.separator! + 1, current.end).join('\n');
        conflicts.push(current as ConflictMarker);
        current = null;
      }
    }

    return conflicts;
  }

  resolveConflict(
    content: string,
    conflict: ConflictMarker,
    resolution: 'ours' | 'theirs' | 'both' | string
  ): string {
    const lines = content.split('\n');
    const before = lines.slice(0, conflict.start);
    const after = lines.slice(conflict.end + 1);

    let resolved: string;
    switch (resolution) {
      case 'ours':
        resolved = conflict.ours;
        break;
      case 'theirs':
        resolved = conflict.theirs;
        break;
      case 'both':
        resolved = conflict.ours + '\n' + conflict.theirs;
        break;
      default:
        resolved = resolution;
    }

    return [...before, resolved, ...after].join('\n');
  }
}
```

### AI 辅助功能

```typescript
class AIGitAssistant {
  constructor(private aiClient: IAIClient) {}

  // 生成提交信息
  async generateCommitMessage(diff: string): Promise<string> {
    const response = await this.aiClient.chat([
      {
        role: 'system',
        content: 'Generate a concise git commit message following conventional commits format. Only output the message, no explanation.'
      },
      {
        role: 'user',
        content: `Generate a commit message for these changes:\n\n${diff}`
      }
    ]);
    return response.content.trim();
  }

  // 解释变更
  async explainChanges(diff: string): Promise<string> {
    const response = await this.aiClient.chat([
      {
        role: 'user',
        content: `Explain these code changes in plain language:\n\n${diff}`
      }
    ]);
    return response.content;
  }

  // 解决冲突建议
  async suggestConflictResolution(
    ours: string,
    theirs: string,
    context: string
  ): Promise<string> {
    const response = await this.aiClient.chat([
      {
        role: 'user',
        content: `Help resolve this merge conflict.

Context (surrounding code):
${context}

Our version:
${ours}

Their version:
${theirs}

Provide the best merged version.`
      }
    ]);
    return response.content;
  }

  // 代码审查
  async reviewChanges(diff: string): Promise<ReviewComment[]> {
    const response = await this.aiClient.chat([
      {
        role: 'system',
        content: 'Review the code changes and provide feedback as JSON array: [{line: number, comment: string, severity: "info"|"warning"|"error"}]'
      },
      {
        role: 'user',
        content: diff
      }
    ]);

    try {
      return JSON.parse(response.content);
    } catch {
      return [];
    }
  }
}
```

### Git UI 组件

```tsx
function SourceControlPanel() {
  const [status, setStatus] = useState<GitStatus | null>(null);
  const [commitMessage, setCommitMessage] = useState('');
  const gitService = useService(IGitService);
  const aiAssistant = useService(AIGitAssistant);

  useEffect(() => {
    loadStatus();
    gitService.onDidChange(() => loadStatus());
  }, []);

  const loadStatus = async () => {
    const repo = await gitService.discover(workspacePath);
    if (repo) {
      setStatus(await repo.getStatus());
    }
  };

  const handleCommit = async () => {
    const repo = await gitService.discover(workspacePath);
    if (repo && commitMessage) {
      await repo.commit(commitMessage);
      setCommitMessage('');
    }
  };

  const generateMessage = async () => {
    const repo = await gitService.discover(workspacePath);
    if (repo) {
      const diff = await repo.diff();
      const message = await aiAssistant.generateCommitMessage(
        diff.map(d => d.path).join('\n')
      );
      setCommitMessage(message);
    }
  };

  if (!status) return <div>Not a git repository</div>;

  return (
    <div className="source-control">
      <div className="branch-info">
        <span className="branch-name">{status.branch}</span>
        {status.ahead > 0 && <span>↑{status.ahead}</span>}
        {status.behind > 0 && <span>↓{status.behind}</span>}
      </div>

      <div className="commit-input">
        <textarea
          value={commitMessage}
          onChange={(e) => setCommitMessage(e.target.value)}
          placeholder="Commit message"
        />
        <div className="commit-actions">
          <button onClick={generateMessage}>✨ Generate</button>
          <button onClick={handleCommit} disabled={!commitMessage}>
            Commit
          </button>
        </div>
      </div>

      <div className="changes">
        <ChangeGroup
          title="Staged Changes"
          changes={status.staged}
          onUnstage={(path) => repo.unstage([path])}
        />
        <ChangeGroup
          title="Changes"
          changes={status.unstaged}
          onStage={(path) => repo.stage([path])}
          onDiscard={(path) => repo.discard([path])}
        />
        <ChangeGroup
          title="Untracked Files"
          changes={status.untracked}
          onStage={(path) => repo.stage([path])}
        />
      </div>
    </div>
  );
}
```

---

## 关键决策清单

1. **内置还是插件？**
   - 内置：开发者工具
   - 插件：笔记应用

2. **使用什么 Git 库？**
   - simple-git：简单易用
   - isomorphic-git：纯 JS
   - libgit2：性能更好

3. **可视化程度？**
   - 基础：状态和提交
   - 完整：Diff、历史、分支图

4. **AI 如何参与？**
   - 提交信息生成
   - 代码审查
   - 冲突解决

---

## 参考资料

- [VSCode SCM API](https://code.visualstudio.com/api/extension-guides/scm-provider)
- [simple-git](https://github.com/steveukx/git-js)
- [isomorphic-git](https://isomorphic-git.org/)
