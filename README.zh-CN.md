# gnhf 中文文档

`gnhf` 是一个面向代码 Agent 的长时间任务编排工具。它会在 Git 仓库中围绕一个目标反复启动 Agent，每次迭代完成一个小改动，验证后自动提交，并把过程记录到本地运行目录中。

适合用于夜间或长时间无人值守任务，例如：

- 持续降低代码复杂度
- 补充测试和修复测试失败
- 分批完成重构
- 根据较大的需求文档逐步实现功能
- 并行启动多个 Agent 探索不同方向

## 特性

- **一条命令启动**：在 Git 仓库中执行 `gnhf "<目标>"` 即可开始循环。
- **小步提交**：每次成功迭代都会生成一个独立 Git commit，便于审查、挑选和回滚。
- **失败回滚**：普通失败会回滚本轮改动；提交失败会保留未提交内容，让下一轮 Agent 尝试修复。
- **可控停止**：支持最大迭代次数、最大 token 数，以及自然语言停止条件。
- **支持恢复**：在已有 `gnhf/` 分支上再次运行可继续之前的任务。
- **工作树隔离**：`--worktree` 模式可以同时运行多个 Agent，互不干扰。
- **多 Agent 支持**：内置支持 Claude Code、Codex、Rovo Dev、OpenCode、GitHub Copilot CLI、Pi 和 ACP 目标。

## 安装

### 使用 npm 安装

```sh
npm install -g gnhf
```

安装后确认命令可用：

```sh
gnhf --version
```

### 从源码安装

```sh
git clone git@github.com:sharkmabin/gnhf.git
cd gnhf
corepack enable
pnpm install
pnpm run build
pnpm link --global
```

项目要求 Node.js 20 或更高版本。

## 快速开始

进入一个干净的 Git 仓库：

```sh
cd your-repo
git status
```

启动一次任务：

```sh
gnhf "在不改变功能的前提下降低代码复杂度"
```

限制迭代次数和 token：

```sh
gnhf "为核心模块补充测试" \
  --max-iterations 10 \
  --max-tokens 5000000
```

从标准输入传入较长需求文档：

```sh
cat prd.md | gnhf
```

`gnhf` 需要在 Git 仓库内运行。如果当前目录还不是仓库，请先执行：

```sh
git init
```

## 常用模式

### 默认分支模式

默认情况下，`gnhf` 会创建一个 `gnhf/<任务摘要>` 分支，并在这个分支上持续提交。

```sh
gnhf "重构 API 层并保持现有行为"
```

适合希望把 Agent 的工作和当前主分支隔离开的场景。

### 当前分支模式

如果希望直接在当前分支上运行：

```sh
gnhf --current-branch "继续改进这个应用"
```

如果还希望每次成功提交后自动推送：

```sh
gnhf --current-branch --push "继续改进这个应用"
```

注意：`gnhf` 不会自动 force push，也不会自动 pull。推送失败时，本地成功提交会被保留，运行会终止。

### Worktree 模式

使用 `--worktree` 可以为每个任务创建独立 Git worktree，适合并行运行多个 Agent：

```sh
gnhf --worktree "实现功能 X" &
gnhf --worktree "为模块 Y 添加测试" &
gnhf --worktree "重构 API 层" &
```

有提交的 worktree 会被保留，方便审查、合并或 cherry-pick。没有产生提交的 worktree 通常会在退出时自动清理。

### 按条件停止

可以用自然语言指定停止条件：

```sh
gnhf "修复测试失败" --stop-when "所有测试都通过"
```

这个条件会随运行记录保存，后续恢复任务时继续生效。传入空字符串可以清除：

```sh
gnhf --stop-when ""
```

## 命令参考

| 命令 | 说明 |
| --- | --- |
| `gnhf "<prompt>"` | 使用给定目标启动新任务 |
| `gnhf` | 在已有 `gnhf/` 分支上恢复任务 |
| `echo "<prompt>" \| gnhf` | 从标准输入传入目标 |
| `cat prd.md \| gnhf` | 从文件传入较长需求 |

常用参数：

| 参数 | 说明 |
| --- | --- |
| `--agent <agent>` | 指定 Agent，例如 `claude`、`codex`、`opencode` 或 `acp:<target>` |
| `--max-iterations <n>` | 最多运行多少轮迭代 |
| `--max-tokens <n>` | 达到 token 上限后停止 |
| `--stop-when <条件>` | 当 Agent 报告满足条件后停止 |
| `--prevent-sleep <on|off>` | 是否阻止系统睡眠 |
| `--worktree` | 使用独立 Git worktree 运行 |
| `--current-branch` | 在当前分支上运行 |
| `--push` | 每次成功提交后推送 |
| `--meteor-frequency <n>` | 设置 TUI 流星动画频率，`0` 表示关闭 |
| `--version` | 输出版本号 |

## 配置

配置文件位于：

```sh
~/.gnhf/config.yml
```

示例：

```yaml
# 默认 Agent
agent: claude

# 最多连续失败次数
maxConsecutiveFailures: 3

# 运行期间阻止系统睡眠
preventSleep: true

# 自定义 Agent 可执行文件路径
agentPathOverride:
  claude: ~/bin/claude-code-switch
  codex: /usr/local/bin/my-codex-wrapper

# 自定义原生 Agent 参数
agentArgsOverride:
  codex:
    - -m
    - gpt-5.4
    - -c
    - model_reasoning_effort="high"
```

命令行参数优先级高于配置文件。首次运行时，如果配置文件不存在，`gnhf` 会按默认值创建。

## 支持的 Agent

| Agent | 参数 | 前置要求 |
| --- | --- | --- |
| Claude Code | `--agent claude` | 安装并登录 `claude` CLI |
| Codex | `--agent codex` | 安装并登录 `codex` CLI |
| Rovo Dev | `--agent rovodev` | 安装 Atlassian `acli` 并完成认证 |
| OpenCode | `--agent opencode` | 安装并配置可用模型提供方 |
| GitHub Copilot CLI | `--agent copilot` | 安装并登录 GitHub Copilot CLI |
| Pi | `--agent pi` | 安装 `pi` CLI 并配置 provider/model |
| ACP 目标 | `--agent acp:<target-or-command>` | 安装并认证对应 ACP 目标 |

## 运行产物

每次运行都会在本地 `.gnhf/runs/<runId>/` 下保存运行数据，包括：

- `prompt.md`：本次任务目标
- `notes.md`：跨迭代共享的记录
- `gnhf.log`：JSONL 调试日志
- `iteration-<n>.jsonl`：单轮迭代的 Agent 输出

这些文件用于恢复、调试和审查，一般不需要提交到业务仓库。

## 本地开发

克隆仓库后：

```sh
corepack enable
pnpm install
```

常用脚本：

```sh
pnpm run build          # 构建
pnpm run dev            # 监听构建
pnpm test               # 构建并运行测试
pnpm run test:e2e       # 运行端到端测试
pnpm run lint           # 运行 ESLint
pnpm run format:check   # 检查格式
pnpm run typecheck      # TypeScript 类型检查
```

贡献代码前请阅读 [CONTRIBUTING.md](./CONTRIBUTING.md)。

## 故障排查

### 提示不是 Git 仓库

请进入已有 Git 仓库，或先执行：

```sh
git init
```

### 工作区不干净

默认模式要求工作区干净。请先提交、暂存或清理当前改动：

```sh
git status
```

### Agent 启动失败

确认对应 Agent CLI 已安装、在 `PATH` 中，并且已经完成登录或模型配置。

### 需要查看详细错误

查看本地运行日志：

```sh
cat .gnhf/runs/<runId>/gnhf.log
```

提交 issue 时，附上相关日志片段通常最有帮助。如需关闭匿名遥测，可设置：

```sh
GNHF_TELEMETRY=0
```

## 许可证

本项目使用 MIT License。详情见 [LICENSE](./LICENSE)。
