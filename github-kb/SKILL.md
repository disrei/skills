---
name: github-kb
description: 用于处理 GitHub 仓库检索、本地仓库优先分析、下载 repo、查询 issue/PR/release/branch/commit、以及围绕某个 GitHub 项目回答问题的技能。只要用户提到 github、GitHub、repo、repository、仓库、项目源码、owner/repo、GitHub 链接、issue、PR、pull request、release、branch、fork、clone、下载仓库、某个仓库怎么实现、某个 GitHub 项目现在什么状态，或明显希望基于本地 `C:\Projects\Github_knowledge` 中的仓库来定位并分析问题时，就优先使用本技能，即使用户没有明确说“请用 github-kb”。先检查 `@C:\Projects\Github_knowledge\AGENTS.md` 和本地仓库目录，尽量从本地仓库回答；需要联网时优先使用 `gh`，若 `gh` 不可用或未登录，则退化为 `curl` 查询 GitHub。仅在用户只是询问通用 git 命令语法、且不涉及 GitHub 仓库内容时，才不要触发。
---

# GitHub 本地仓库知识技能（github-kb）

## 目标

让代理在处理 GitHub / repo / 仓库相关任务时，优先使用用户本地仓库目录 `C:\Projects\Github_knowledge` 作为一手上下文来源，并在需要时结合 GitHub 远程信息回答问题。

优先读取：@C:\Projects\Github_knowledge\AGENTS.md

这个文件记录了本地仓库目录中的仓库清单与一句话摘要，能帮助你先判断用户提到的仓库是否已经在本地存在。

## 触发规则

当用户出现以下任一意图时，直接启用本技能，不要等用户明确点名：

- 提到 `github`、`repo`、`repository`、`仓库`
- 提到 `issue`、`pr`、`pull request`、`release`、`tag`、`branch`
- 提到“下载一个 repo”“clone 一个仓库”“看看这个 GitHub 项目”
- 询问某个 GitHub 仓库的结构、实现、问题、提交、PR、issue、版本信息
- 希望基于本地 GitHub 仓库目录找项目、读代码、做摘要、回答问题

如果只是普通 git 本地命令解释、且不涉及 GitHub 仓库内容，可不必触发本技能。

## 核心原则

- 本地优先：先检查 `C:\Projects\Github_knowledge` 与 `AGENTS.md`，再决定是否访问 GitHub。
- 仓库优先匹配：若用户提到仓库名、owner/repo、URL、缩写或明显别名，优先在本地目录寻找对应仓库。
- 证据优先：回答仓库问题时，优先引用本地代码、README、配置、提交记录或 GitHub API 返回结果。
- 低惊扰原则：已有同名本地仓库时，不要直接重复 clone；优先使用本地副本，必要时询问是否需要更新。
- 目录自维护：当本地目录路径失效时，先向用户确认新路径，再把新路径写回当前技能的 `SKILL.md`，避免下次继续失效。

## 标准流程

### 1. 识别目标仓库

优先从用户输入中提取以下信息：

- `owner/repo`
- GitHub URL
- 仓库名
- 组织名 / 用户名
- 相关关键词（例如项目功能、技术栈、README 里的叫法）

如果用户没有明确仓库，但问题明显指向本地某个仓库集合，先读取 `@C:\Projects\Github_knowledge\AGENTS.md` 判断最可能目标，再继续分析。

### 2. 先查本地目录

先检查 `C:\Projects\Github_knowledge` 是否存在。

#### 若目录存在

1. 读取 `@C:\Projects\Github_knowledge\AGENTS.md`。
2. 根据仓库名、别名、owner/repo 或 URL，在本地目录中找最匹配的仓库。
3. 若找到本地仓库：优先读取本地文件、git 信息和项目结构回答问题。
4. 若本地未找到，再决定是否联网查询 GitHub。

#### 若目录不存在

1. 不要假装目录存在。
2. 明确告诉用户：预期目录 `C:\Projects\Github_knowledge` 不存在。
3. 询问用户新的本地 GitHub 仓库根目录路径。
4. 用户确认后，更新当前技能的 `SKILL.md` 中所有相关路径为新路径，并继续使用新路径。

### 3. 当用户要求“下载一个 repo”

当用户要求下载 / clone 仓库时：

1. 先确认目标仓库标识（优先 `owner/repo` 或完整 URL）。
2. 使用 `git` 命令把仓库下载到 `C:\Projects\Github_knowledge`。
3. 若目标目录已存在同名仓库：默认不要覆盖；优先告知用户本地已存在，可改为使用现有副本或执行 `git pull`。
4. clone 完成后，重新生成或更新 `C:\Projects\Github_knowledge\AGENTS.md`。
5. 在回复中说明下载位置、默认分支（若已知）以及本地目录名。

推荐命令形式：

```bash
git clone https://github.com/<owner>/<repo>.git "C:\Projects\Github_knowledge\<repo>"
```

### 4. 当用户询问 GitHub 信息

如果问题无法仅靠本地仓库回答，按下面顺序处理：

1. 先尝试使用 `gh` 查询 GitHub。
2. 若 `gh` 不存在，或存在但未登录，改用 `curl` 请求 GitHub API / 页面。
3. 将远程查询结果与本地仓库信息结合，给出结论。

常见场景：

- 查仓库：`gh repo view <owner/repo>`
- 查 issue：`gh issue list --repo <owner/repo>` / `gh issue view`
- 查 PR：`gh pr list --repo <owner/repo>` / `gh pr view`
- 查 release：`gh release list --repo <owner/repo>`
- 查搜索：`gh search repos <query>` / `gh search issues <query>`

若 `gh` 不可用或未登录，可退化为：

- `curl https://api.github.com/repos/<owner>/<repo>`
- `curl https://api.github.com/repos/<owner>/<repo>/issues`
- `curl https://api.github.com/search/repositories?q=<query>`

注意 GitHub API 速率限制。未登录时优先缩小查询范围，避免无意义大搜索。

### 5. 回答方式

回答时尽量按这个顺序组织：

1. 先说你使用了哪些信息源（本地仓库 / `AGENTS.md` / GitHub 远程）。
2. 再给出结论。
3. 若结论依赖猜测或匹配不完全，明确说明不确定性。
4. 若本地存在多个可能仓库，列出最可能候选并说明判断依据。

## `AGENTS.md` 维护规则

`C:\Projects\Github_knowledge\AGENTS.md` 是本地仓库索引，不是泛泛说明文档。每次 clone 新仓库、重命名仓库目录、或发现摘要明显过时，都应更新它。

建议格式：

```md
# Github_knowledge 仓库索引

- `repo-name`: 一句话摘要
- `another-repo`: 一句话摘要
```

生成一句话摘要时，优先依据：

- README 标题与简介
- 仓库描述（若通过 GitHub 查询到）
- 核心目录 / 配置文件 / 技术栈

摘要应简短、可区分，不要写空话。

## 冲突与澄清

遇到以下情况时再提问：

- 用户给了多个可能仓库，且本地匹配结果不唯一
- 用户只说“下载这个 repo”但没有任何可定位信息
- 本地已有同名目录，但无法确认是否就是目标仓库
- 远程和本地仓库状态明显不一致，且会影响结论

除此之外，优先自己根据本地索引和 GitHub 信息做出合理判断。

## 输出偏好

- 默认用中文回答。
- 涉及本地仓库时，给出明确路径。
- 涉及 GitHub 远程对象时，尽量给出完整 `owner/repo`。
- 若执行了 clone / 搜索 / 查询，简要说明用了什么命令或数据源。
