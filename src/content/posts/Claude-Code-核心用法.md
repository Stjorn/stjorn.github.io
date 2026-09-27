---
title: 'Claude Code的一些核心用法'
published: 2026-06-20
description: '在Boss上投了一个岗位，是以claude code为核心工具搭建自动化工具链的，所以今天好好研究一下'
tags: [AI]
category: note
draft: false
---

在Boss上投了一个岗位，是以claude code为核心工具搭建自动化工具链的，所以今天好好研究一下

---

## 1. Memory 

Claude Code 每次启动都会自动加载 CLAUDE.md。两个位置：

- `~/.claude/CLAUDE.md` — 用户级，对你所有项目生效
- `./CLAUDE.md` — 项目级，只对当前项目生效，通常随项目一起提交到 Git

两者都会被加载，项目级在用户级之后加载，内容更具体，冲突时以项目级为准。

在 Claude Code 输入框里输 `/init`，它会扫描当前项目，自动生成或更新CLAUDE.md 。之后用 `/memory` 可以查看当前加载了哪些记忆并进行修改。

---

## 2. Plan Mode 

普通模式下说"修 bug"，Claude Code 可能边查边改。Plan Mode 换了一种方式：先只读代码、分析问题、给出方案，等用户批准了再动手。

输入 `/plan` 进入。

即使不在 Plan Mode 下，也可以在 prompt 末尾加一句 "先读代码给方案，不要修改文件"，效果一样的。核心都是让 Claude 知道要先商量好再改。

---

## 3. Skills

用一段时间后你会发现，很多操作每次都要粘贴同样一段长提示词。Skill 就是把这些流程封装成一个可复用的 `/命令`。

项目级 Skill 放在 `.claude/skills/<name>/SKILL.md`，用户级放在 `~/.claude/skills/<name>/SKILL.md`。

创建 Skill 后，输入 `/skill-name` 就能调用。

Skill 是一套流程说明——规定"审查 diff 时查哪些东西、输出什么格式"。subAgent 是一个专门执行者——"负责代码审查的专家，只允许读文件和搜索，不允许写代码"。两者的关系是：可以让 subAgent 按 Skill 规定的流程执行。Skill 是 procedure，subAgent 是 executor。

直接在 Claude Code 里说 "请帮我创建一个项目级 Skill，名字叫 review-diff，放在 .claude/skills/review-diff/SKILL.md。先展示内容，我确认后再创建" 就可以。当然，也可以自己写或者网上下载别人的Skill。

具体的我在另一篇博客里详细写过：[Agent 三件套](https://whyulooksad.github.io/posts/agent%E4%B8%89%E4%BB%B6%E5%A5%97/)

---

## 4. Hook 

Skills是AI选择去调用。Hook 是在某个时机到了之后自动触发一段检查或命令。

### 4.1 结构

- **event** — 什么时候触发
- **matcher** — 匹配什么工具或场景（正则表达式）
- **handler** — 匹配后执行什么（shell 命令 / LLM prompt / agent）

比如下面这个配置写进 `.claude/settings.json`：

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Edit|Write",
        "hooks": [
          {
            "type": "command",
            "command": "echo '文件已修改，记得跑测试'"
          }
        ]
      }
    ]
  }
}
```

PostToolUse（工具执行成功后触发）→ 匹配 Edit 或 Write（写文件时触发）→ 执行 shell 命令打印提醒。Claude 每次改完文件你都能看到这条提示。

### 4.2 常用事件

不需要全记，记住最常用的几个：

| 事件 | 触发时机 | 适合做什么 |
|------|---------|-----------|
| `PreToolUse` | 工具调用前 | 拦截危险命令、阻止修改敏感文件 |
| `PostToolUse` | 工具调用成功后 | 改完文件自动格式化、跑实验后自动读结果 |
| `PostToolUseFailure` | 工具调用失败后 | 记录失败日志 |
| `Stop` | Claude 准备结束本轮回复时 | 结束前再检查一次指标是否达标 |
| `UserPromptSubmit` | 用户提交 prompt 时 | 自动追加规则、记录输入 |
| `SessionStart` | 会话开始时 | 加载自定义环境 |
| `PreCompact` | 上下文压缩前 | 提醒保存关键信息 |
| `Notification` | Claude 需要用户注意时 | 发桌面通知 |
| `SubagentStart` / `SubagentStop` | subAgent 启动/结束时 | 记录 agent 活动 |

### 4.3 Handler 类型

handler 不只是跑 shell 命令，有三种：

- **command** — 跑一个 shell 命令，上下文信息（session_id、tool_name、tool_input 等）以 JSON 通过 stdin 传入，你可以用 `jq` 解析
- **prompt** — 调 LLM 评估条件，比如"这个 bash 命令安全吗？"
- **agent** — 调一个 agent 做验证，比如"检查测试是否通过"

大部分场景用 command 就够了。

### 4.4 实战：实验自动迭代

假设你有一个 ML 训练项目，每次跑 `python train.py` 后结果写入 `results/metrics.json`。你想要的效果是：Claude 每次跑完实验后自动读结果，如果 accuracy < 0.90 或 loss > 0.30，就不要继续做别的事，回头分析原因、改代码、重新实验，直到达标为止。

这个效果不能只靠一个 Hook，需要组合：

**第一步，创建检查脚本** `.claude/hooks/check_result.py`，读取 `results/metrics.json`，判断指标是否达标。不达标就输出 "不要继续，请分析原因并修改代码"，达标就输出 "通过"。

**第二步，写 Hook 配置**——让 PostToolUse 事件在 Bash 命令包含 `python`、`train` 等关键词时，自动跑检查脚本：

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          {
            "type": "command",
            "command": "python .claude/hooks/check_result.py results/metrics.json",
            "if": "Bash(*python*)"
          }
        ]
      }
    ],
    "Stop": [
      {
        "matcher": "",
        "hooks": [
          {
            "type": "command",
            "command": "python .claude/hooks/check_result.py results/metrics.json"
          }
        ]
      }
    ]
  }
}
```

PostToolUse 里加了 `"if": "Bash(*python*)"`，意思是只在 Bash 命令包含 "python" 时才触发检查，避免每次 ls、git 都跑一遍。Stop Hook 在 Claude 准备结束时再检查一次——如果不达标就提醒 Claude 不要结束，继续迭代。

**第三步，开始实验时给 Claude 主规则**，规定它必须根据 Hook 的反馈行动：

> 接下来做实验迭代。每次改代码或参数后必须跑 `python train.py`，每次实验后必须读 `results/metrics.json`。如果 Hook 提示不达标，不要写总结、不要进入下一阶段——先分析原因、改代码、重新实验。只有 accuracy >= 0.90 且 loss <= 0.30 时才允许进入下一阶段。

Hook 不等于完整自动化——它负责"触发检查"这一步。剩下的分析、修改、重新实验，需要主提示词和 Skill 配合。

**不推荐手动改 settings.json**—— JSON 嵌套层级容易出错。直接在 Claude Code 里说 "请帮我创建 Hook 配置，先展示完整内容，我确认后再写" 更稳。

---

## 5. subAgent 

### 5.1 意义

如果主 Claude 自己排查一个复杂 bug，可能读十几个文件、跑好几条搜索、试多个假设，主对话上下文很快塞满。到后面你真正需要的是"根因是什么、怎么改"，但对话里塞的全是搜索记录。

subAgent 的做法是：派一个专门的专家助手自己去查，最后只把结论带回来。主对话不被中间过程塞满。

### 5.2 创建

输入 `/agents`，进入管理界面。默认停在 **Running** 页——这里是看"当前有没有 agent 在跑"的，没有任务时就是空的。按**右方向键 →** 切到 **Library** 页创建。

更推荐的方式是直接用对话创建，因为可以精确描述职责和权限：

> 请帮我创建一个项目级 subAgent，名字叫 experiment-runner。放在 .claude/agents/experiment-runner.md。
>
> 作用：负责实验迭代。每轮运行实验命令、读取 results/metrics.json、判断 accuracy/loss 是否达标，不达标就提出修改方向。
>
> 权限：Read、Grep、Glob、Bash、Edit、Write
>
> 约束：修改代码前必须先说明要改什么和为什么。创建前先展示 agent 文件内容，我确认后再写。

### 5.3 agent 权限

这是 subAgent 最重要的设计原则——不是所有 agent 都该有全权限：

- **debugger**：Read、Grep、Glob、Bash — 只读，不允许改代码
- **code-reviewer**：Read、Grep、Glob、Bash — 只输出建议，不允许写代码
- **test-writer**：Read、Grep、Glob、Edit、Write、Bash — 可以写测试文件
- **security-reviewer**：Read、Grep、Glob — 只读，Bash 都不给
- **experiment-runner**：全集，但受实验规则约束

### 5.4 调用

创建后不会自动运行。可以在Claude Code输入中指定，比如调用方式：

> 请使用 experiment-runner agent 做实验迭代。不要由主会话直接去做迭代。目标：accuracy >= 0.90 且 loss <= 0.30。实验命令：python train.py。

---

## 6. MCP 

前面的 Memory、Skill、Hook、subAgent 都在 Claude Code 自己的世界里。MCP（Model Context Protocol）让 Claude Code 能接外部系统：GitHub、PostgreSQL、Figma、Slack、Sentry、Jira 等。

`/mcp` 查看当前连接了哪些服务和状态。在终端添加新连接：

```bash
# HTTP 类型
claude mcp add --transport http github https://example.com/mcp

# stdio 类型（本地进程）
claude mcp add --transport stdio myserver -- node server.js
```

MCP安全风险其实挺多的，建议在 CLAUDE.md 里写死 MCP 安全规则：

- 所有 MCP 默认只读
- 数据库 MCP 默认只允许 SELECT
- GitHub MCP 默认只读 issue，不直接修改 issue 或 PR
- 涉及写操作必须先给方案、影响范围和确认点

我这篇博客也详细讲了MCP: [Agent 三件套](https://whyulooksad.github.io/posts/agent%E4%B8%89%E4%BB%B6%E5%A5%97/)

---

## 7. worktree 

### 7.1 概念

Claude Code 的 worktree 是基于 Git worktree的。你的项目必须有 Git 仓库才能用。

普通情况下你只有一个项目目录。如果开两个终端跑两个 Claude 会话，都在同一个目录里改文件，改的是同一份文件，很容易互相覆盖。

worktree 让你从一个 Git 仓库开出多个独立目录，每个目录有自己的文件副本：

```
my-project/                           # 主目录
my-project/.claude/worktrees/exp-a    # worktree 目录 A
my-project/.claude/worktrees/exp-b    # worktree 目录 B
```

### 7.2 启动

在终端：

```bash
cd my-project
claude --worktree exp-a        # 创建或进入 worktree exp-a
```

再开另一个终端：

```bash
cd my-project
claude --worktree exp-b
```

exp-a 里的 Claude 改 exp-a 目录下的文件，exp-b 里的 Claude 改 exp-b 目录下的文件，主目录不受影响。

### 7.3 关键认知

这是很容易踩坑的地方。三个概念要分清：

- **worktree 名字**：启动时写的，比如 `exp-a`
- **worktree 目录**：磁盘路径，比如 `.claude/worktrees/exp-a/`
- **worktree 对应的分支**：不一定叫 `exp-a`，必须查出来

在主项目目录下跑：

```bash
git worktree list
```

输出示例：

```
D:/my-project                         abc1234 [main]
D:/my-project/.claude/worktrees/exp-a def5678 [worktree-exp-a]
D:/my-project/.claude/worktrees/exp-b 8910111 [worktree-exp-b]
```

方括号 `[]` 里的才是分支名。**合并时合并的是方括号里的分支名，不是 worktree 名字，不是目录路径。** 几个例子：

| git worktree list 输出 | 分支名 | 合并命令 |
|---|---|---|
| `[login-fix]` | `login-fix` | `git merge login-fix` |
| `[worktree-exp-a]` | `worktree-exp-a` | `git merge worktree-exp-a` |
| `[claude/test1]` | `claude/test1` | `git merge claude/test1` |

不要猜。先查再 merge。

### 7.4 合并和冲突

```bash
cd D:/my-project
git checkout main
git merge worktree-exp-a     # 查出来的分支名
# 跑测试确认没问题
git merge worktree-exp-b
```

如果两个 worktree 改了同一个文件的同一段代码，Git 会报 CONFLICT。这时候让 Claude 分析冲突：

> 当前合并出现冲突。分析冲突文件：A 改了什么、B 改了什么、可以同时保留吗、推荐合并方案。先不要修改文件。

最好开 worktree 时就明确分工，给每个 Claude 下限制：

> 你现在在 worktree exp-a 中工作。只允许修改模型结构相关文件。不要修改数据处理文件。如果确实需要改，先停下来说明原因。

完成后清理：

```bash
git worktree remove .claude/worktrees/exp-a
git branch -d worktree-exp-a   # 分支也不要了就删
```

---

## 8. Multi-Agent 

### 8.1 概念

- **subAgent**：主 Claude **在一个会话里**派一个专家去查，专家返回结果。专家不和其他专家讨论。
- **worktree**：隔离的代码目录。不是助手，是文件系统层面的隔离。
- **Agent Team**：**多个 Claude 实例**组成团队，各自领任务，lead 汇总。这是多会话之间的协作。这个功能现在还不成熟，Antropic也说了现在是在实验阶段。

**适用场景：**大型重构、前后端同时分析、复杂 bug 多方向排查、实验多方向并行、一个写代码一个写测试一个 review。它是为复杂任务设计的，不要为了用而用。

### 8.2 用法

关键不是"多叫几个 Claude"，而是拆任务 + 定角色 + 定权限 + 定输出 + lead 汇总 + 你批准后再改。缺一步就是多份混乱。

给 lead 的提示词：

> Use an agent team for this task.
>
> 目标：重构用户登录模块，保持现有行为不变。
>
> 角色：
> 1. frontend-agent — 只分析登录页面、表单组件、路由跳转。只读，不修改。
> 2. api-agent — 只分析 auth API、token 存储、错误处理。只读，不修改。
> 3. test-agent — 只分析现有测试覆盖，提出需要补的测试。只读，不修改。
> 4. reviewer-agent — 在前三个完成后，审查汇总方案是否有风险。只读，不修改。
>
> 规则：
> - 第一轮所有人只读代码，不修改文件
> - 每个 teammate 输出自己的发现
> - lead 汇总最终计划
> - 我批准前不要改代码

---

## 核心速查

| 功能 | 作用 | 怎么访问 |
|------|---------|---------|
| Memory | 项目说明书，启动自动加载 | `/init` 生成，`/memory` 查看 |
| Plan Mode | 先读代码给方案，批准再改 | `/plan`，或 `claude --permission-mode plan` |
| Skill | 封装重复流程为 /命令 | `.claude/skills/<name>/SKILL.md` |
| Hook | 到时机自动触发检查 | `.claude/settings.json`，`/hooks` 查看 |
| subAgent | 派专家查问题，只带回结论 | `/agents` → Library 创建 |
| MCP | 连接 GitHub、数据库等 | `/mcp` 查看，`claude mcp add` 添加 |
| worktree | Git worktree 隔离目录并行工作 | `claude --worktree <name>`，`git worktree list` 查分支 |
| Agent Team | 多 Claude 组队分工协作 | 拆任务 → 定角色 → 定权限 → lead 汇总 |
