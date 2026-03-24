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
  ├── <task-id>.md            ← 每个任务一个文件
  └── logs/
      └── <task-id>.log       ← 任务执行时的输出日志
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

<任务的详细提示词，这是 opencode run 执行时发送给 LLM 的内容>
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
   - 涉及的路径（如有，转换为绝对路径）

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

5. **更新 summary.md**：追加一行到表格

6. **注册到 Task Scheduler**：执行 PowerShell 脚本（见下方）

7. **反馈**：告知用户任务已创建，并说明日志路径

### 修改任务

1. 读取 summary.md，定位目标任务
2. 展示当前配置，询问修改内容
3. 更新 task.md 对应字段
4. 更新 summary.md 对应行
5. 重新注册：与新增任务完全相同的 PowerShell 命令（含 `Get-Credential` 弹框、`cmd /c` 包装、`OpenCode\<id>` 任务名），加 `-Force` 参数覆盖已有任务

### 删除任务

1. 读取 summary.md，定位目标任务
2. 向用户确认删除
3. 删除 `tasks/<id>.md`
4. 询问是否保留日志文件，按用户意愿处理
5. 从 summary.md 移除对应行
6. 执行 `Unregister-ScheduledTask`

### 列出任务

直接读取 summary.md，以清晰格式展示给用户。

---

## PowerShell 注册脚本

注册任务时，构造并执行以下 PowerShell 代码。

**关键点：**
- 任务名必须使用 `OpenCode\<id>` 格式（含子文件夹前缀），方便在 GUI 中统一管理
- 执行器必须用 `cmd /c` 包装，才能实现日志重定向（`opencode` 本身不支持 `>>` 重定向）
- 凭据通过 `Get-Credential` 弹框输入，每次注册/更新弹一次

```powershell
# 填入实际值
$taskId   = "<id>"
$taskFile = "C:\Users\<username>\.config\opencode\tasks\$taskId.md"
$logFile  = "C:\Users\<username>\.config\opencode\tasks\logs\$taskId.log"
$workdir  = "<workdir的绝对路径>"

# 确保日志目录存在
New-Item -ItemType Directory -Force -Path (Split-Path $logFile) | Out-Null

# 执行动作：用 cmd /c 包装以支持日志重定向
# 注意：-Execute 必须是 "cmd"，不是 "opencode"
$action = New-ScheduledTaskAction `
    -Execute "cmd" `
    -Argument "/c opencode run -f `"$taskFile`" `"请执行文件中描述的任务`" >> `"$logFile`" 2>&1" `
    -WorkingDirectory $workdir

# 触发器：根据 task.md 的 schedule 字段构造（见下方参考表）
$trigger = <见 Trigger 构造参考>

$settings = New-ScheduledTaskSettingsSet `
    -ExecutionTimeLimit (New-TimeSpan -Hours 2) `
    -StartWhenAvailable  # 机器关机错过触发时，开机后补执行

# 弹出密码输入框（用户输入一次，密码加密存储）
$cred = Get-Credential

Register-ScheduledTask `
    -TaskName "OpenCode\$taskId" `
    -Action   $action `
    -Trigger  $trigger `
    -Settings $settings `
    -RunLevel Highest `
    -Force `
    -User     $cred.UserName `
    -Password $cred.GetNetworkCredential().Password
```

### Trigger 构造参考

根据 task.md 的 schedule 字段，构造对应的 trigger：

| 用户描述 | schedule 字段 | PowerShell Trigger |
|---------|--------------|-------------------|
| 每天 08:00 | `DAILY /ST 08:00` | `New-ScheduledTaskTrigger -Daily -At "08:00"` |
| 每周一 08:00 | `WEEKLY /D MON /ST 08:00` | `New-ScheduledTaskTrigger -Weekly -DaysOfWeek Monday -At "08:00"` |
| 每小时 | `HOURLY` | `New-ScheduledTaskTrigger -RepetitionInterval (New-TimeSpan -Hours 1) -Once -At (Get-Date)` |
| 每月1日 09:00 | `MONTHLY /D 1 /ST 09:00` | `New-ScheduledTaskTrigger -Weekly` 不支持月度，改用 `-AtLogOn` 或手动构造 |
| 工作日每天 | `WEEKLY /D MON,TUE,WED,THU,FRI /ST 08:00` | `New-ScheduledTaskTrigger -Weekly -DaysOfWeek Monday,Tuesday,Wednesday,Thursday,Friday -At "08:00"` |

---

## 路径处理规则

用户在描述中提到的相对路径，**必须在创建 task.md 时转换为绝对路径**，避免运行时因工作目录不同导致找不到文件。

转换逻辑：
- `src/` → 基于用户当前工作目录拼接为绝对路径
- `~/` → 展开为用户 home 目录
- 已经是绝对路径 → 保持不变

---

## 错误处理

- **注册失败**：展示 PowerShell 错误信息，提示用户检查权限或路径
- **task.md 已存在同名文件**：提示用户该 id 已被占用，建议换一个名称或改为修改操作
- **summary.md 不存在**：自动创建（含表头），不报错
- **logs/ 目录不存在**：自动创建

---

## 注意事项

- Task Scheduler 任务名统一使用 `OpenCode\<id>` 格式，方便在任务计划程序 GUI 中统一查看
- `-StartWhenAvailable` 确保机器关机期间错过的任务在下次开机后补执行
- 执行超时设为 2 小时，防止 LLM 任务卡死占用资源
- 日志文件会累积追加（`>>`），不会自动清理，长期运行需提醒用户定期清理
