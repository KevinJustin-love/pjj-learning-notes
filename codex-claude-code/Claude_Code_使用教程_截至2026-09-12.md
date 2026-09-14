# Claude Code 使用教程（截至 2026-09-12）

> 面向 Claude Code CLI、IDE 集成和 Claude Code 网页/桌面工作流。命令、模型和权限模式会随版本、平台、账号及组织策略变化；先输入 `/` 查看当前会话可用命令。

## 1. 基本概念

Claude Code 是 Anthropic 的代码代理，能够读取项目、编辑文件、运行 shell、调用 MCP、派发子代理并审查改动。CLAUDE.md 记录项目规则，权限规则控制工具调用，沙箱限制 Bash 的系统边界。

官方资料：[Commands](https://code.claude.com/docs/en/commands)、[CLI reference](https://code.claude.com/docs/en/cli-usage)、[Interactive mode](https://code.claude.com/docs/en/interactive-mode)、[Permissions](https://code.claude.com/docs/en/permissions)、[Memory](https://code.claude.com/docs/en/memory)。

## 2. 安装、登录、启动

按官方安装页选择系统和发行渠道。安装后进入仓库：

````bash
cd path/to/project
claude
claude "解释这个项目的结构并指出入口文件"
claude update
claude auth status --text
````

首次交互会话会提示登录 Anthropic 账号；也可以用 Console/API 计费方式。安装命令随版本变化，以官方 CLI 文档为准。

## 3. 三种输入方式

**普通提示：**写明目标、范围、约束和验收标准。

````text
请先阅读 README、src/auth 和测试，不要修改数据库 schema。
定位刷新令牌失效后的 401 处理问题，先提出计划，再实现最小修复。
完成后运行相关测试，列出实际命令、退出状态和未验证项。
````

**斜杠命令：**只能在消息开头识别。输入 `/` 后继续输入字母过滤；命令后的文字是参数。Claude 回复期间输入的命令通常排队到当前回合结束；`/status`、`/tasks`、`/usage` 等信息命令可立即显示。

**`!` Shell 模式：**交互式会话中在输入行首加 `!`，直接执行 shell 并把输出加入会话：

````text
! git status
! npm test
! python -m pytest tests/test_login.py -q
! ls -la
````

`!` 不经过 Claude 的解释或审批流程，相当于你亲自在当前终端执行；先检查命令、目录和参数。长命令可按 `Ctrl+B` 放到后台。

## 4. 常用斜杠命令

| 命令 | 作用 |
|---|---|
| `/help` | 查看帮助 |
| `/init` | 生成 CLAUDE.md 起始文件 |
| `/memory` | 查看或改进项目记忆 |
| `/plan` | 进入计划模式，探索和规划但不编辑源文件 |
| `/model` | 切换模型；支持时可调整 effort |
| `/effort` | 调整推理投入 |
| `/permissions` | 管理 allow、ask、deny 规则 |
| `/context` | 查看上下文各部分占用 |
| `/compact` | 总结对话以释放上下文 |
| `/status` | 查看会话、模型和环境状态 |
| `/usage` | 查看用量 |
| `/diff` | 查看 Git 差异 |
| `/code-review`、`/review` | 审查当前 diff；支持强度和 PR 参数 |
| `/security-review` | 检查差异中的安全问题 |
| `/mcp` | 配置或检查 MCP |
| `/agents` | 设置或查看子代理 |
| `/tasks` | 查看后台任务 |
| `/background` | 将整个会话转为后台 agent |
| `/batch` | 将大改动拆成独立 worktree 单元（支持时） |
| `/btw` | 提出不加入主历史的侧问题 |
| `/clear` | 开始新任务并保留项目记忆 |
| `/resume` | 恢复旧会话 |
| `/branch`、`/fork` | 分支或复制会话 |
| `/rewind` | 回到检查点或总结部分工作（支持时） |
| `/doctor`、`/debug` | 诊断安装/运行时问题 |
| `/hooks` | 浏览 hooks 配置 |
| `/config` | 打开配置面板 |
| `/add-dir`、`/cd` | 增加目录或移动主工作目录 |
| `/exit` | 退出会话 |

以当前 `/` 菜单为准；插件和 MCP 也可能贡献额外命令。

## 5. 权限模式

| 模式 | 用途 |
|---|---|
| `default` | 日常开发，按规则询问 |
| `plan` | 只探索和规划 |
| `acceptEdits` | 自动接受文件编辑，其他工具仍受规则约束 |
| `auto` | 使用自动模式和分类器减少提示 |
| `dontAsk` | 未允许的操作直接拒绝，不弹询问 |
| `bypassPermissions` | 隔离 runner/受控 CI，风险最高 |

CLI 中按 `Shift+Tab` 循环可用模式，或启动时指定：

````bash
claude --permission-mode plan
claude --permission-mode acceptEdits
claude --permission-mode dontAsk
````

不要把 bypassPermissions 当作提速开关；优先使用精确的 allowedTools 和权限规则。

## 6. 权限规则与沙箱

规则按 **deny → ask → allow** 判断；deny 优先。示例：

````json
{
  "permissions": {
    "allow": ["Bash(npm test *)", "Bash(git diff *)", "Read"],
    "ask": ["Bash(git commit *)"],
    "deny": ["Bash(git push *)", "Read(.env)"]
  }
}
````

项目共享规则放 `.claude/settings.json`，个人机器规则放 `.claude/settings.local.json` 或用户设置。`--add-dir` 或 `/add-dir` 可增加工作目录。权限控制工具能否调用；沙箱提供 Bash 及子进程的文件/网络限制，两者要一起使用。

## 7. CLI 命令

| 命令 | 说明 |
|---|---|
| `claude` | 启动交互式 REPL |
| `claude "query"` | 带初始提示启动 |
| `claude -p "query"` | 非交互 print 模式，适合脚本/CI |
| `cat file \| claude -p "query"` | 管道输入 |
| `claude -c` | 继续当前目录最近会话 |
| `claude -c -p "query"` | 非交互继续会话 |
| `claude -r ID "query"` | 按 ID/名称恢复 |
| `claude update` | 更新 CLI |
| `claude auth login/logout/status` | 认证管理 |
| `claude mcp` | 管理 MCP 服务器 |
| `claude plugin` | 管理插件 |
| `claude agents`、`attach`、`logs`、`stop` | 管理后台会话 |
| `claude import codex --dry-run` | 预览导入其他 agent 配置 |

示例：

````bash
claude -p "运行测试并总结失败原因"
cat logs.txt | claude -p "按错误类型分组"
claude -p --output-format json "检查当前 diff 的风险"
claude --permission-mode plan "设计迁移方案"
claude -r auth-refactor "完成剩余测试"
````

## 8. 常用参数

| 参数 | 作用 |
|---|---|
| `--add-dir PATH` | 增加可访问目录 |
| `--permission-mode MODE` | 指定权限模式 |
| `--allowedTools` | 指定无需询问的工具规则 |
| `--tools` | 限定可用工具 |
| `--model MODEL` | 指定模型 |
| `--agent NAME`、`--agents JSON` | 使用或临时定义 agent |
| `--effort LEVEL` | 调整推理投入 |
| `--mcp-config FILE` | 加载 MCP 配置 |
| `--plugin-dir PATH` | 增加插件目录 |
| `--settings FILE` | 指定设置文件 |
| `--output-format text\|json\|stream-json` | 控制 print 输出 |
| `--max-turns N` | 限制非交互回合数（支持时） |
| `--verbose` | 详细日志 |
| `--dangerously-skip-permissions` | 跳过询问，仅限隔离环境 |

## 9. CLAUDE.md 项目记忆

运行 `/init` 生成起点，再用 `/memory` 优化。适合记录启动、测试、构建、目录职责、代码风格、兼容性和验收标准。不要放密码、token、个人路径和私有数据。

````markdown
# 项目工作说明
- 先阅读 README 和相关测试。
- 只修改与任务有关的文件。
- Python 测试：python -m pytest tests -q
- 前端检查：npm test && npm run lint
- 完成后报告改动文件、命令、退出状态和未验证项。
- 不直接 push 或部署，先展示 diff。
````

## 10. MCP、Plugins、Skills、Subagents

- MCP 连接外部工具和数据；用 `/mcp` 或 `claude mcp` 管理。
- Plugins 可打包命令、技能、hooks 和 MCP；安装前检查来源和权限。
- Skills 是可复用提示工作流，在 `/` 菜单中查看。
- Subagents 使用独立上下文处理子任务；`/tasks` 查看后台工作。

并行任务应使用独立目录或 worktree，避免多个 agent 同时写同一文件。

## 11. Hooks

Hooks 可在 SessionStart、PreToolUse、PermissionRequest、SessionEnd 等事件运行命令、HTTP、prompt、agent 或 MCP handler。`/hooks` 可只读查看配置。PreToolUse 可以阻止危险调用，但 hook 的 allow 不能绕过 deny/ask；退出码 2 可阻止工具调用。

## 12. 快捷键

| 快捷键 | 作用 |
|---|---|
| `Ctrl+C` | 取消输入或生成 |
| `Ctrl+D` | 退出会话 |
| `Ctrl+O` | transcript viewer |
| `Ctrl+R` | 搜索命令历史 |
| Windows `Alt+V` | 粘贴图片（终端支持时） |
| `Ctrl+B` | Bash 命令后台运行 |
| `Shift+Tab` | 切换权限模式 |
| `?` | transcript viewer 帮助 |

## 13. 推荐工作流

1. `git status` 确认工作树；首次使用运行 `/init` 并检查 CLAUDE.md。
2. 大任务先 `/plan`，要求列出影响范围和验证命令。
3. 通过后执行修改；用 `! git diff`、`! npm test` 快速检查。
4. 长任务用 `/tasks`、`/context`、`/compact` 管理。
5. 完成后运行 `/diff`、`/code-review` 和项目测试。
6. 人工检查高影响操作，再提交、推送或部署。

## 14. 提示词模板

````text
目标：
项目范围：
背景：
约束（禁止修改的文件、版本、兼容性）：
完成条件：
必须运行的验证命令：
输出要求（文件、测试、风险、未验证项）：
````

## 15. 常见问题

**命令不存在：**输入 `/` 查看当前菜单；部分命令取决于版本、平台、计划或插件。
**权限询问太多：**用 `/permissions` 检查规则，增加精确 allow 或 `--add-dir`，不要直接 bypass。
**`!` 是否安全：**它直接交给 shell，不经过 Claude 解释；按亲自执行命令的标准检查。
**如何撤销：**先看 `git diff`，使用 Git 或 `/rewind`（支持时）；删除会话不会自动撤销代码。

## 16. 资料与版本说明

本文根据 Anthropic 官方文档整理，核对日期为 **2026-09-12**。主要来源：[Commands](https://code.claude.com/docs/en/commands)、[CLI reference](https://code.claude.com/docs/en/cli-usage)、[Interactive mode](https://code.claude.com/docs/en/interactive-mode)、[Permissions](https://code.claude.com/docs/en/permissions)、[Hooks](https://code.claude.com/docs/en/hooks)、[Security](https://code.claude.com/docs/en/security)。以本机 `claude --help`、`/help` 和 `/` 菜单为准。
