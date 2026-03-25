---
name: task-scheduler
description: 用于管理 Windows 定时任务的技能。当用户描述一个需要定期自动执行的任务（如"每周一归纳新闻"、"每天备份文件"、"每小时检查日志"），或要求修改、删除、列出已有定时任务时，必须触发本技能。即使用户没有说"定时任务"这个词，只要意图是"让某件事定期自动发生"，就应触发。本技能会自动创建 task.md 文件、维护全局 summary.md 注册表、并通过 PowerShell 将任务注册到 Windows Task Scheduler。
---

# Windows 定时任务管理 Skill

## 概述

本 skill 让用户只需用自然语言描述一个定期任务，OpenCode 就会自动完成所有技术细节：创建任务描述文件、更新注册表、注册到 Windows Task Scheduler。

所有任务文件统一存放在全局目录 `~/.config/opencode/tasks/`，日志存放在其下的 `logs/` 子目录。

---

## 触发意图识别

以下表达都应触发本 skill：

- "每周一早上帮我归纳周日的新闻"
- "每天备份一下 D:/projects 文件夹"
- "设置一个每小时检查日志的任务"
- "删掉那个新闻归纳任务"
- "把备份任务改成每天晚上 11 点"
- "列出所有定时任务"
- "查看现在有哪些自动任务"

不触发的情况：用户只是问"怎么用 Task Scheduler"或讨论调度概念，而没有要求创建/修改/删除具体任务。

---

## 文件结构

```
~/.config/opencode/tasks/
  ├── summary.md              ← 所有任务的注册表，OpenCode 负责维护
  ├── <task-id>.md            ← 每个任务一个文件（含 YAML front matter + prompt）
  └── logs/
      └── <task-id>.log       ← 任务执行时的输出日志

C:\OpenCodeTasks\
  └── <task-id>.cmd           ← 实际执行的批处理文件（Task Scheduler 直接调用）
```

### task.md 格式

```markdown
---
id: <task-id>
description: <一句话描述>
schedule: <schtasks 格式的时间参数，见下方参考>
workdir: <绝对路径>
status: enabled
created: <YYYY-MM-DD>
---

<任务的提示词，即 opencode run 执行时的 message>
```

**提示词编写规则（关键）：**

1. **必须是单行英文**。中文会在 Task Scheduler 注册/传递时编码损坏（GBK/UTF-8 混乱）
2. **要简短直接**。不要写冗长的多步骤 Markdown 指令，让 AI 自行判断如何完成
3. **路径使用正斜杠**。Windows 下 `D:/path/to/file` 比 `D:\path\to\file` 安全得多，避免反斜杠在多层 shell 传递中被转义
4. **指定关键命令**。如果需要用特定工具（如 `gh`、`git -C`），在 prompt 里明确写出命令名和参数格式
5. **指定输出行为**。明确写 "write to <path> (overwrite)" 或 "append to <path>"，以及完成后是否打开文件

**好的 prompt 示例：**
```
Read D:/Github/repos/AGENTS.md, get the repos listed. For each repo, run git -C with forward slash path to get local HEAD, then use gh api to get remote commits. Write an update report to D:/Github/repos/UpdateLog.md (overwrite). Then open it with: powershell Start-Process D:/Github/repos/UpdateLog.md
```

**坏的 prompt 示例（会导致执行失败）：**
```
请执行以下任务：
## 目标
读取 `D:\Github\repos\AGENTS.md` 中列出的所有仓库名称...
### 第一步：读取仓库清单
...（多行中文 Markdown 指令）
```

### summary.md 格式

summary.md 的列名和结构**严格固定**，任何操作（增、改、删）后都必须保持以下格式不变：

```markdown
---
last_updated: <YYYY-MM-DD>
---

# 定时任务注册表

| id | 描述 | 触发时间 | 状态 |
|----|------|---------|------|
| <id> | <描述> | <人类可读时间> | 启用/禁用 |
```

**列名不可改变**：`id`、`描述`、`触发时间`、`状态` 这四列永远保持不变。删除任务时只删除对应的数据行，不能修改表头或改变列名。

---

## 操作流程

### 新增任务

1. **信息提取**：从用户描述中提取：
   - 任务内容（做什么）
   - 触发时间（何时触发，无则默认每天 08:00）
   - 涉及的路径（如有，转换为绝对路径，使用正斜杠）

2. **确认**：向用户展示提取的信息，确认无误后继续：
   ```
   即将创建任务：
   - 名称：weekly-news-summary
   - 描述：每周一早上归纳周日新闻
   - 触发时间：每周一 08:00
   - 工作目录：C:\Users\LZhang4
   是否确认？
   ```

3. **生成 task-id**：英文小写+连字符，如 `weekly-news-summary`

4. **创建 task.md**：写入 `~/.config/opencode/tasks/<id>.md`

5. **创建 .cmd 文件**：写入 `C:\OpenCodeTasks\<id>.cmd`（见下方模板）

6. **更新 summary.md**：追加一行到表格

7. **注册到 Task Scheduler**：使用 `schtasks` 命令（见下方）

8. **立即验证**：手动触发一次，确认任务能正常执行（见下方验证流程）

9. **反馈**：告知用户任务已创建并验证通过

### 修改任务

1. 读取 summary.md，定位目标任务
2. 展示当前配置，询问修改内容
3. 更新 task.md 和 .cmd 文件
4. 更新 summary.md 对应行
5. 重新注册（`schtasks /Create ... /F` 覆盖）
6. 立即验证

### 删除任务

1. 读取 summary.md，定位目标任务
2. 向用户确认删除
3. 删除 `tasks/<id>.md` 和 `C:\OpenCodeTasks\<id>.cmd`
4. 询问是否保留日志文件，按用户意愿处理
5. 从 summary.md 移除对应行
6. 通过管理员权限执行 `schtasks /Delete /TN "OpenCode\<id>" /F`

### 列出任务

直接读取 summary.md，以清晰格式展示给用户。

---

## .cmd 批处理文件模板

每个任务对应一个 `.cmd` 文件，存放在 `C:\OpenCodeTasks\`。这是 Task Scheduler 直接执行的入口。

```batch
@echo off
chcp 65001 >nul
opencode run "<英文单行 prompt>"
```

**关键设计原则：**

- **不使用 `>>` 日志重定向**。`opencode` 通过 npm `.cmd` 包装调用 node.js，`>> log 2>&1` 会在外层 cmd 退出后断开，导致输出截断。让 opencode 在 prompt 中自行完成文件写入
- **不使用 `-f` 参数**。`-f` 是 array 类型，会把后续所有位置参数都当成文件路径。直接把 prompt 作为 message 参数传入
- **不使用中文 prompt**。中文在 Task Scheduler → cmd.exe → opencode 的多层传递中会编码损坏
- **prompt 必须单行**。多行字符串在 cmd 参数传递中会被截断
- `chcp 65001` 设置 UTF-8 代码页，确保 opencode 输出不乱码

---

## 注册方式

### 推荐方式：schtasks 命令行（简单可靠）

通过管理员 PowerShell 执行 `schtasks` 命令。不需要 `Get-Credential` 弹窗，不需要额外的注册脚本文件。

```powershell
# 用管理员权限执行
Start-Process powershell -Verb RunAs -ArgumentList '-Command schtasks /Create /TN ''OpenCode\<id>'' /TR ''cmd.exe /d /c C:\OpenCodeTasks\<id>.cmd'' /SC DAILY /ST 08:00 /RL HIGHEST /F' -Wait
```

**schtasks 参数参考：**

| 用户描述 | /SC 和相关参数 |
|---------|--------------|
| 每天 08:00 | `/SC DAILY /ST 08:00` |
| 每周一 08:00 | `/SC WEEKLY /D MON /ST 08:00` |
| 每小时 | `/SC HOURLY` |
| 每月1日 09:00 | `/SC MONTHLY /D 1 /ST 09:00` |
| 工作日每天 | `/SC WEEKLY /D MON,TUE,WED,THU,FRI /ST 08:00` |

**注意：** `schtasks` 在 Git Bash 中会被路径展开（`/Create` → `C:/Program Files/Git/Create`），必须在 PowerShell 中执行。

### 备选方式：Register-ScheduledTask（仅在 schtasks 不满足时使用）

只有当 schtasks 的参数不够用（如需要设置 ExecutionTimeLimit、StartWhenAvailable 等高级选项）时，才使用 PowerShell `Register-ScheduledTask`。此时必须把注册代码写到一个 `.ps1` 文件，通过管理员权限执行：

```powershell
$action   = New-ScheduledTaskAction -Execute 'cmd.exe' -Argument '/d /c "C:\OpenCodeTasks\<id>.cmd"' -WorkingDirectory '<workdir>'
$trigger  = New-ScheduledTaskTrigger -Daily -At '08:00'
$settings = New-ScheduledTaskSettingsSet -ExecutionTimeLimit (New-TimeSpan -Hours 2) -StartWhenAvailable

$cred = Get-Credential

Register-ScheduledTask `
    -TaskName 'OpenCode\<id>' `
    -Action   $action `
    -Trigger  $trigger `
    -Settings $settings `
    -RunLevel Highest `
    -Force `
    -User     $cred.UserName `
    -Password ($cred.GetNetworkCredential().Password)
```

---

## 注册后验证（必须执行）

注册成功不代表执行能成功。每次新增或修改任务后，**必须立即手动触发一次验证**：

```powershell
# 1. 触发任务
Start-ScheduledTask -TaskName '<id>' -TaskPath '\OpenCode\'

# 2. 等待执行完成（根据任务复杂度调整等待时间）
Start-Sleep -Seconds 120

# 3. 检查结果
$info = Get-ScheduledTask -TaskName '<id>' -TaskPath '\OpenCode\' | Get-ScheduledTaskInfo
Write-Host "LastTaskResult: $($info.LastTaskResult)"  # 0 = 成功
```

**验证标准：**
- `LastTaskResult` 为 `0`
- 任务预期的输出文件已生成且内容正确

如果验证失败，检查以下常见原因：
1. `.cmd` 文件中的 prompt 是否被截断（检查引号匹配）
2. `opencode` 命令是否在 Task Scheduler 的 PATH 中可用
3. 路径中是否有反斜杠转义问题（改用正斜杠）
4. prompt 中是否包含中文（改为英文）

---

## 路径处理规则

用户在描述中提到的相对路径，**必须在创建 task.md 和 .cmd 时转换为绝对路径**。

转换逻辑：
- `src/` → 基于用户当前工作目录拼接为绝对路径
- `~/` → 展开为用户 home 目录
- 已经是绝对路径 → 保持不变

**所有路径在 prompt 和 .cmd 中使用正斜杠**（`D:/path/to/file`），避免反斜杠在 shell 传递中被转义。

---

## 错误处理

- **注册失败（Access Denied）**：需要管理员权限。用 `Start-Process -Verb RunAs` 提权执行
- **schtasks 参数被 Git Bash 路径展开**：在 PowerShell 中执行 schtasks，不要在 Git Bash 中执行
- **opencode 在 Task Scheduler 中找不到**：检查 npm bin 目录（`%APPDATA%\npm`）是否在系统 PATH 中
- **task.md 已存在同名文件**：提示用户该 id 已被占用，建议换名或改为修改操作
- **summary.md 不存在**：自动创建（含表头），不报错
- **logs/ 目录不存在**：自动创建
- **C:\OpenCodeTasks 不存在**：自动创建

---

## 注意事项

- Task Scheduler 任务名统一使用 `OpenCode\<id>` 格式，方便在任务计划程序 GUI 中统一查看
- `-StartWhenAvailable`（Register-ScheduledTask 方式）确保机器关机期间错过的任务在下次开机后补执行
- 执行超时设为 2 小时，防止 LLM 任务卡死占用资源
- **不要在 .cmd 中使用 `>>` 日志重定向**。opencode 是通过 npm 的 .cmd 包装调用 node.js 的，外层 cmd 会在包装脚本执行完毕后退出，而 node.js 进程可能还在运行，此时重定向已断开
- **不要在 prompt 中使用中文**。中文字符串在 Task Scheduler 注册时会经过多次编码转换（UTF-8 → GBK → UTF-8），导致乱码
- 日志管理：如果需要日志，在 prompt 中让 opencode 自行 append 到日志文件，而不是依赖 shell 重定向
