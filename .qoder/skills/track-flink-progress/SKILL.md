---
name: track-flink-progress
description: 追踪 Apache Flink 项目研发进展：拉取最新代码合并到本地，分析每次新增的 commit 变更内容，并将分析结果写入 track 文件夹内的 changelog-日期时间.md 文件。当用户说"追踪 flink 进展"、"拉取最新变更"、"分析新增 commit"或"生成 changelog"时使用此技能。
---

# Track Apache Flink Progress

追踪 Apache Flink 项目的研发进展，自动拉取最新代码、分析新增 commit、生成结构化 changelog 文件。
支持 **Windows（PowerShell）**、**macOS** 和 **Linux** 三个平台。

## 跨平台说明

所有步骤均优先使用纯 `git` 命令实现，避免依赖特定 shell 语法。  
唯一需要获取当前时间的地方（Step 4 文件命名），根据运行环境自动选择对应命令：

| 平台 | 获取当前时间命令 |
|------|-----------------|
| Windows PowerShell | `Get-Date -Format "yyyy-MM-dd-HHmm"` |
| macOS / Linux (bash/zsh) | `date +"%Y-%m-%d-%H%M"` |
| Git Bash on Windows | `date +"%Y-%m-%d-%H%M"` |

执行前先通过以下方式判断当前平台，选择对应命令：
- **PowerShell**：使用 `$env:OS` 判断，值为 `Windows_NT` 时为 Windows 环境，使用 `Get-Date`。
- **bash/zsh**：使用 `uname` 判断，返回 `Darwin` 为 macOS，返回 `Linux` 为 Linux，使用 `date`。
- **Git Bash on Windows**：`uname` 返回 `MINGW*` 或 `MSYS*` 时，视为 Windows 环境；但 Git Bash 自带 `date` 命令，可直接使用 `date +"%Y-%m-%d-%H%M"`，无需切换到 PowerShell。

## 工作流程

### Step 1：确保 upstream 已配置，记录当前 HEAD，拉取最新代码

```bash
# 检查是否存在 upstream remote（三平台通用）
git remote -v
```

若没有 `upstream`，先添加 Apache 官方仓库：

```bash
git remote add upstream https://github.com/apache/flink.git
```

然后记录拉取前的 HEAD 并拉取：

```bash
# 记录拉取前的 HEAD commit（三平台通用，纯 git 命令）
git rev-parse HEAD
# 将输出结果记录为 OLD_HEAD，供后续步骤使用

# 从 Apache 官方仓库拉取最新代码，冲突以官方为准（三平台通用）
git pull -X theirs upstream master
```

> **平台差异**：`git rev-parse HEAD` 在三个平台输出相同，直接使用其输出值替换后续命令中的 `<OLD_HEAD>`。

若 `git pull` 失败（如非内容冲突的结构性错误），提示用户具体错误信息，不继续执行。

### Step 2：找出新增的 commit 列表

```bash
# 列出 <OLD_HEAD>..HEAD 之间所有新增的 commit（从旧到新排列，三平台通用）
git log <OLD_HEAD>..HEAD --oneline --reverse
```

若输出为空，说明本地已是最新，提示用户"已是最新版本，无新增 commit"，终止流程。

### Step 3：逐个分析每个 commit

对每个新增 commit，执行（三平台通用）：

```bash
git show <commit_hash> --stat
git show <commit_hash>
```

分析时关注以下维度：
- **类型判断**：新功能（Feature）/ Bug 修复（Fix）/ 性能优化（Perf）/ 重构（Refactor）/ 文档注释（Docs）/ 测试（Test）/ 其他
- **影响模块**：从提交信息的 `[模块]` 前缀和改动文件路径推断（如 `[runtime]`、`[connectors]`、`[table]`、`[streaming]`、`[checkpoint]`、`[network]`、`[sql]`、`[client]` 等）
- **核心变更**：用 1-3 句话描述主要改动内容
- **关键文件**：列出最核心的改动文件（最多 5 个）

### Step 4：生成 changelog 文件

文件命名格式：`changelog-{YYYY-MM-DD-HHmm}.md`  
文件存放路径：项目根目录下的 `track/` 文件夹

**获取项目根目录（三平台通用）：**

```bash
git rev-parse --show-toplevel
```

> **平台差异**：在 Windows 上 `git rev-parse --show-toplevel` 返回 Unix 风格路径（如 `E:/project/java-project/flink`），在 Git 命令中可直接使用；若需传递给 PowerShell 原生命令，需将 `/` 替换为 `\`。

**获取当前时间（根据平台选择）：**

```powershell
# Windows PowerShell
Get-Date -Format "yyyy-MM-dd-HHmm"
```

```bash
# macOS / Linux (bash/zsh)
date +"%Y-%m-%d-%H%M"
```

将时间值记为 `NOW`，changelog 文件完整路径为：`<项目根目录>/track/changelog-<NOW>.md`

- 若 `track` 文件夹不存在，先创建该文件夹，再写入文件：

  ```powershell
  # Windows PowerShell
  New-Item -ItemType Directory -Path track -Force
  ```

  ```bash
  # macOS / Linux / Git Bash
  mkdir -p track
  ```
- **不得手动填写固定日期**，必须通过上述命令动态获取

## changelog 文件模板

```markdown
# Apache Flink 研发进展 - {YYYY-MM-DD HH:mm}

> 拉取时间：{YYYY-MM-DD HH:mm}
> 新增 commit 数量：{N}
> commit 范围：{oldHead_short}..{newHead_short}

---

## 变更列表

### 1. {commit_type} | {module} | {commit_subject}

- **commit**：`{hash_short}`（作者：{author}，时间：{date}）
- **类型**：{Feature / Fix / Perf / Refactor / Docs / Test}
- **摘要**：{1-3句核心描述}
- **关键文件**：
  - `{file_path_1}`：{一句话说明}
  - `{file_path_2}`：{一句话说明}

---

### 2. {commit_type} | {module} | {commit_subject}

...（以此类推）

---

## 模块统计

| 模块 | 新增 commit 数 | 类型分布 |
|------|--------------|---------|
| runtime | N | Feature×a, Fix×b |
| table | N | Feature×a |
| ...  | N | ... |
```

### Step 5：提交并推送到远程仓库

将新生成的 changelog 文件提交并推送（三平台通用，纯 git 命令）：

```bash
# 添加 changelog 文件
git add track/changelog-*.md

# 提交（将 <NOW> 替换为实际时间字符串）
git commit -m "track: add changelog <NOW>"

# 推送到 origin
git push origin master
```

若推送失败（如远程有新提交），提示用户具体错误信息。

## 注意事项

- commit 描述使用**中文**，保持简洁专业
- 若某个 commit 的 diff 过大（超过 500 行），只分析 `--stat` 部分，不逐行阅读全量 diff
- 文件路径引用使用 Markdown 链接格式指向实际文件
- 若新增 commit 超过 20 个，优先分析功能类（Feature/Fix/Perf）commit，测试和文档类可简略处理
- changelog 文件创建后，告知用户文件的完整路径
