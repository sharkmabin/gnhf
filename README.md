<p align="center">睡前，我会对我的 agents 说：</p>
<h1 align="center">good night, have fun</h1>

<p align="center">
  <a href="https://www.npmjs.com/package/gnhf"
    ><img
      alt="npm"
      src="https://img.shields.io/npm/v/gnhf?style=flat-square"
  /></a>
  <a href="https://github.com/kunchenguid/gnhf/actions/workflows/ci.yml"
    ><img
      alt="CI"
      src="https://img.shields.io/github/actions/workflow/status/kunchenguid/gnhf/ci.yml?style=flat-square&label=ci"
  /></a>
  <a href="https://github.com/kunchenguid/gnhf/actions/workflows/release-please.yml"
    ><img
      alt="Release"
      src="https://img.shields.io/github/actions/workflow/status/kunchenguid/gnhf/release-please.yml?style=flat-square&label=release"
  /></a>
  <a
    href="https://img.shields.io/badge/platform-macOS%20%7C%20Linux%20%7C%20Windows-blue?style=flat-square"
    ><img
      alt="Platform"
      src="https://img.shields.io/badge/platform-macOS%20%7C%20Linux%20%7C%20Windows-blue?style=flat-square"
  /></a>
  <a href="https://x.com/kunchenguid"
    ><img
      alt="X"
      src="https://img.shields.io/badge/X-@kunchenguid-black?style=flat-square"
  /></a>
  <a href="https://discord.gg/Wsy2NpnZDu"
    ><img
      alt="Discord"
      src="https://img.shields.io/discord/1439901831038763092?style=flat-square&label=discord"
  /></a>
</p>

<p align="center">
  <img src="docs/splash.png" alt="gnhf - Good Night, Have Fun" width="800">
</p>

别空手醒来。

`gnhf` 是一个面向代码 Agent 的长时间任务编排工具，灵感来自 [ralph](https://ghuntley.com/ralph/) 和 [autoresearch](https://github.com/karpathy/autoresearch)。它会在你休息时围绕一个目标反复调用 Agent，每一轮完成一个小的、可提交、可追踪的改动。

你醒来时，会得到一个包含干净提交的分支，以及完整的运行记录。

- **一条命令启动**：在 Git 仓库中执行 `gnhf "<目标>"` 即可开始自主循环，直到你请求停止或达到配置的运行上限。
- **适合长时间运行**：成功迭代会自动提交；普通失败会回滚；提交失败会保留未提交改动交给下一轮修复；可重试的硬错误会指数退避，Agent 主动报告的失败则立即进入下一轮。
- **实时终端标题**：交互式运行会在终端标题中显示状态、token 总量和提交数量；以 `~` 开头的 token 数表示估算值。
- **退出摘要**：每次运行结束都会输出分支、耗时、迭代次数、token、diff 统计、日志路径和审查命令。
- **Agent 无关**：开箱支持 Claude Code、Codex、Rovo Dev、OpenCode、GitHub Copilot CLI、Pi 和 ACP 目标。

## 安装

### 使用 npm

```sh
npm install -g gnhf
```

安装后检查命令是否可用：

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

项目要求 Node.js 20 或更高版本，支持 macOS、Linux 和 Windows。

## 快速开始

进入一个干净的 Git 仓库后运行：

```sh
gnhf "在不改变功能的前提下降低代码复杂度"
```

限制迭代次数和 token：

```sh
gnhf "在不改变功能的前提下降低代码复杂度" \
  --max-iterations 10 \
  --max-tokens 5000000
```

并行启动多个 Agent，可以使用 worktree 模式：

```sh
gnhf --worktree "实现功能 X" &
gnhf --worktree "为模块 Y 添加测试" &
gnhf --worktree "重构 API 层" &
```

直接在当前分支运行，并在每次成功迭代后推送：

```sh
gnhf --current-branch --push "继续改进这个应用"
```

`gnhf` 需要在 Git 仓库内运行，并且默认要求工作区干净。如果当前目录还不是仓库，请先执行：

```sh
git init
```

## Agent Skill

npm 包内置一个给 Agent 使用的 skill，路径为 `skills/gnhf/SKILL.md`。支持本地 skill 的 Agent 可以复制或引用这个文件，学习如何以 Hands-Off 模式运行有边界的夜间任务，或以 Companion 模式监督、引导和审查一次长时间的 GNHF 运行。

从 npm 安装后，skill 位于已安装包目录中；从源码检出时，可以直接使用 `skills/gnhf/SKILL.md`。

## 工作机制

```text
                    ┌─────────────┐
                    │  gnhf start │
                    └──────┬──────┘
                           ▼
                ┌──────────────────────┐
                │  validate clean git  │
                │  create or use branch│
                │  write prompt.md     │
                └──────────┬───────────┘
                           ▼
              ┌────────────────────────────┐
              │  build iteration prompt    │◄──────────────┐
              │  inject notes.md context   │               │
              └────────────┬───────────────┘               │
                           ▼                               │
              ┌────────────────────────────┐               │
              │  invoke your agent         │               │
              │  non-interactive mode      │               │
              └────────────┬───────────────┘               │
                           ▼                               │
                    ┌─────────────┐                        │
                    │  success?   │                        │
                    └──┬──────┬───┘                        │
                  yes  │      │  no                        │
                       ▼      ▼                            │
              ┌──────────┐  ┌───────────┐                  │
              │  commit  │  │ reset or  │                  │
              │  append  │  │ repair    │                  │
              │ notes.md │  │ maybe wait│                  │
              └────┬─────┘  └─────┬─────┘                  │
                   │              │                        │
                   │   ┌──────────┘                        │
                   ▼   ▼                                   │
              ┌────────────┐    yes   ┌──────────┐         │
              │ 3 consec.  ├─────────►│  abort   │         │
              │ failures   │          └────▲─────┘         │
              │ or perm.   ├───────────────┘               │
              │ error?     │                               │
              └─────┬──────┘                               │
                 no │                                      │
                    └──────────────────────────────────────┘
```

- **增量提交**：每次成功迭代都会生成一个独立的未签名 Git commit，便于 cherry-pick 或 revert。若 `git commit` 失败，`gnhf` 会保留未提交改动，并要求下一轮 Agent 修复。
- **失败处理**：普通失败会使用 `git reset --hard` 回滚本轮改动；提交失败除外，会保留现场。连续失败达到上限或出现永久性 Agent 错误时会中止。完全没有改动的迭代会被视为失败。
- **运行上限**：`--max-iterations` 会在下一轮开始前停止；`--max-tokens` 可在本轮中达到上限后中止；`--stop-when` 会在 Agent 输出满足自然语言条件后停止。
- **迭代收尾**：Agent 应该完成验证、停止自己启动的后台进程，然后再输出本轮最终 JSON 结果。
- **优雅中断**：交互式 TUI 中第一次 Ctrl+C 请求优雅停止，等待当前迭代完成；第二次 Ctrl+C 立即强制停止；`SIGTERM` 也会强制停止。
- **共享记忆**：Agent 会读取由前序迭代累积的 `notes.md`，用于跨轮沟通。
- **本地运行元数据**：`gnhf` 会把 prompt、notes、停止条件和 commit message 约定保存到 `.gnhf/runs/`，这些内容默认只用于本地恢复和调试。
- **恢复支持**：在已有 `gnhf/` 分支上再次运行 `gnhf` 可以继续之前的任务。如果传入不同 prompt，`gnhf` 会询问是更新保存的 prompt 继续、创建新分支，还是退出。

## 常用模式

### 默认分支模式

默认情况下，`gnhf` 会创建 `gnhf/<任务摘要>` 分支，并在该分支上持续提交。

```sh
gnhf "重构 API 层并保持现有行为"
```

如果生成的分支已存在，新运行会使用 `gnhf/<任务摘要>-1` 这样的数字后缀。

### 当前分支模式

传入 `--current-branch` 可以直接在当前分支运行：

```sh
gnhf --current-branch "继续改进这个应用"
```

配合 `--push` 时，每次成功迭代后会推送当前分支：

```sh
gnhf --current-branch --push "继续改进这个应用"
```

注意：

- 重新以相同 prompt 运行 `--current-branch` 会复用已有 `.gnhf/runs/<runId>/` 历史，并继续迭代编号。
- 推送失败会中止运行，但成功的本地提交会被保留。
- `gnhf` 不会自动 force push，也不会自动 pull。
- `--push` 也可用于默认的 `gnhf/` 分支模式，并会在需要时设置 `origin` 为 upstream。
- `--current-branch` 不能与 `--worktree` 同时使用。

### Worktree 模式

传入 `--worktree` 可以让每个 Agent 在独立的 [Git worktree](https://git-scm.com/docs/git-worktree) 中运行。这样可以在同一个仓库上并行启动多个任务，互不干扰。

```text
<repo>/
<repo>-gnhf-worktrees/
  ├── <run-slug-1>/
  └── <run-slug-2>/
```

- 有提交的 worktree 会在运行结束后保留，方便审查、合并或 cherry-pick。
- 重新以同一 prompt 运行 `--worktree` 时，会尽量恢复匹配的已保留 worktree，否则创建带数字后缀的新 worktree。
- 没有产生提交的 worktree 通常会在退出时自动清理，除非有提交失败留下的未提交改动需要检查或修复。
- `--worktree` 应该从非 `gnhf/` 分支运行，通常是 `main`。

## 命令参考

| 命令 | 说明 |
| --- | --- |
| `gnhf "<prompt>"` | 使用给定目标启动新运行 |
| `gnhf` | 在已有 `gnhf/` 分支上恢复运行 |
| `echo "<prompt>" \| gnhf` | 从标准输入传入 prompt |
| `cat prd.md \| gnhf` | 从文件传入较长规格或 PRD |

如果在已有 `gnhf/` 分支上用不同 prompt 运行，`gnhf` 会询问是更新 `prompt.md` 并继续现有历史、创建新分支，还是退出。当 prompt 来自 stdin 时，确认输入会从控制终端读取，因此需要有可用的交互终端。

### 参数

| 参数 | 说明 | 默认值 |
| --- | --- | --- |
| `--agent <agent>` | 指定 Agent，可选 `claude`、`codex`、`rovodev`、`opencode`、`copilot`、`pi` 或 `acp:<target-or-command>` | 配置文件，默认 `claude` |
| `--max-iterations <n>` | 最多运行多少轮迭代 | 不限制 |
| `--max-tokens <n>` | 输入和输出 token 总量达到上限后停止 | 不限制 |
| `--stop-when <条件>` | 当 Agent 报告满足条件后停止；恢复运行时继续生效 | 不限制 |
| `--prevent-sleep <mode>` | 运行期间阻止系统睡眠，支持 `on`、`off`、`true`、`false` | 配置文件，默认 `on` |
| `--worktree` | 使用独立 Git worktree 运行 | `false` |
| `--current-branch` | 在当前分支运行，而不是创建 `gnhf/` 分支 | `false` |
| `--push` | 每次成功提交后推送 | `false` |
| `--meteor-frequency <n>` | 设置 TUI 流星动画频率，范围 0 到 5，`0` 表示关闭 | `3` |
| `--version` | 输出版本号 |  |

## 配置

配置文件位于 `~/.gnhf/config.yml`：

```yaml
# 默认使用的 Agent：claude、codex、rovodev、opencode、copilot、pi 或 acp:<target-or-command>
agent: claude

# 自定义原生 Agent 可执行文件路径，可选
# agentPathOverride:
#   claude: /path/to/custom-claude
#   codex: /path/to/custom-codex
#   copilot: /path/to/custom-copilot
#   pi: /path/to/custom-pi

# 自定义原生 Agent CLI 参数，可选
# agentArgsOverride:
#   codex:
#     - -m
#     - gpt-5.4
#     - -c
#     - model_reasoning_effort="high"
#     - --full-auto
#   copilot:
#     - --model
#     - gpt-5.4
#   pi:
#     - --provider
#     - openai-codex
#     - --model
#     - gpt-5.5
#     - --thinking
#     - high

# 自定义 ACP 目标命令，可选
# acpRegistryOverrides:
#   my-fork: "/usr/local/bin/my-claude-code-fork --acp"
#   staging: "node /opt/staging/agent.mjs"

# Commit message 约定，可选
# 默认：gnhf <iteration>: <summary>
# 使用 conventional preset 可生成 semantic-release 兼容标题：
# commitMessage:
#   preset: conventional

# 连续失败达到该次数后中止
maxConsecutiveFailures: 3

# 运行期间阻止系统睡眠
preventSleep: true
```

如果配置文件不存在，`gnhf` 会在首次运行时按解析后的默认值创建。

命令行参数优先级高于配置文件。`--prevent-sleep` 接受 `on`、`off`、`true`、`false`；配置文件中始终使用布尔值。迭代次数和 token 上限只作为运行时参数，不会持久化到 `config.yml`。`--stop-when` 会按单次运行持久化，供恢复时继续使用。

`agentArgsOverride.<name>` 可为原生 Agent 传入额外 CLI 参数，例如模型、profile 或 reasoning 设置。ACP 目标本版本不支持 path 或 arg override，请使用 `acpRegistryOverrides` 映射 `acp:<target>` 名称到自定义启动命令，也可以直接传入带引号的自定义 ACP server 命令：

```sh
gnhf --agent 'acp:./bin/dev-acp --profile ci' "修复测试"
```

`commitMessage` 控制每次成功迭代的 commit subject：

- 省略时使用默认格式 `gnhf <iteration>: <summary>`。
- 设置 `preset: conventional` 后，Agent 会提供 `type` 和可选 `scope`，最终生成 `type(scope): summary`。有效类型包括 `build`、`ci`、`docs`、`feat`、`fix`、`perf`、`refactor`、`test` 和 `chore`。
- 已解析的 commit message 约定会随运行保存，恢复运行时会沿用原来的格式，即使之后 `config.yml` 发生变化。

### 自定义 Agent 路径

`agentPathOverride` 可把原生 Agent 指向自定义二进制，适合 Claude Code Switch 或自定义 Codex 包装器等场景：

```yaml
agentPathOverride:
  claude: ~/bin/claude-code-switch
  codex: /usr/local/bin/my-codex-wrapper
  copilot: ~/bin/copilot-wrapper
  pi: ~/bin/pi-wrapper
```

路径可以是绝对路径、`PATH` 中已有的可执行文件名、以 `~` 开头的路径，或相对于配置目录 `~/.gnhf/` 的路径。该 override 只替换二进制名称，标准参数仍由 `gnhf` 保留，因此替代程序必须与原 Agent CLI 兼容。Windows 上支持 `.cmd` 和 `.bat` 包装器。对 `rovodev` 来说，override 必须指向兼容 `acli` 的二进制，因为 `gnhf` 会以 `<bin> rovodev serve ...` 方式调用。

启用防睡眠时，`gnhf` 会使用系统原生机制：macOS 使用 `caffeinate`，Linux 使用 `systemd-inhibit`，Windows 使用基于 `SetThreadExecutionState` 的 PowerShell helper。

## 调试日志

每次运行都会在 `.gnhf/runs/<runId>/gnhf.log` 写入 JSONL 调试日志，并在同目录保存 `notes.md`。日志会记录 orchestrator、Agent 和 HTTP 请求的生命周期事件、耗时，以及失败时完整的 `error.cause` 链。Agent 自身的流式输出会写到相邻的 `iteration-<n>.jsonl`。

原始 ACP 自定义命令会在调试日志和相关错误中被脱敏为 `acp:custom` 或 `custom`，避免把本地路径或秘密写入 `gnhf.log`。

提交 issue 时，附上一段相关 `gnhf.log` 通常最有帮助。

## 遥测

`gnhf` 会向作者自托管的分析服务发送匿名使用遥测，用于了解功能使用情况。不会发送 prompt、仓库路径或分支名。

如需关闭：

```sh
GNHF_TELEMETRY=0
```

## 支持的 Agent

`gnhf` 支持六种原生 Agent 和 ACP 目标。ACP 支持由内置的 [`acpx`](https://github.com/openclaw/acpx) 提供，负责 `acp:<target-or-command>` 规格的运行时和 Agent 注册表。

| Agent | 参数 | 前置要求 | 说明 |
| --- | --- | --- | --- |
| Claude Code | `--agent claude` | 安装 Anthropic `claude` CLI 并先登录 | `gnhf` 会以非交互模式直接调用 `claude`。Claude 输出成功的结构化结果后，`gnhf` 会把该结果视为最终结果，并在短暂宽限期后关闭残留的 Claude 进程树。 |
| Codex | `--agent codex` | 安装 OpenAI `codex` CLI 并先登录 | `gnhf` 会以非交互模式直接调用 `codex exec`。 |
| GitHub Copilot CLI | `--agent copilot` | 安装 GitHub Copilot CLI 并先登录 | `gnhf` 会以 JSONL 模式直接调用 `copilot`。Copilot 当前暴露 assistant 输出 token，但不暴露完整 input/cache token 总量。 |
| Pi | `--agent pi` | 安装 `pi` CLI 并先配置可用 provider/model | `gnhf` 会以 JSON 模式调用 `pi`，把最终输出 schema 附加到 prompt，并通过 `--no-session` 禁用 Pi session 持久化。 |
| Rovo Dev | `--agent rovodev` | 安装 Atlassian `acli` 并完成 Rovo Dev 认证 | `gnhf` 会在仓库工作区自动启动本地 `acli rovodev serve --disable-session-token <port>` 进程。 |
| OpenCode | `--agent opencode` | 安装 `opencode` 并配置至少一个可用模型提供方 | `gnhf` 会自动启动本地 `opencode serve --hostname 127.0.0.1 --port <port> --print-logs`，创建每次运行的 session，并应用允许规则避免工具调用阻塞。 |
| ACP 目标 | `--agent acp:<target-or-command>` | 安装并认证内置 `acpx` 注册表支持的目标，或传入带引号的自定义 ACP server 命令 | `gnhf` 会通过 ACP 运行目标，并在 `.gnhf/runs/<runId>/acp-sessions` 保存每次运行的持久 session。 |

## 本地开发

贡献代码前请阅读 [CONTRIBUTING.md](./CONTRIBUTING.md)。面向 `main` 的人工 PR 必须通过 `git push no-mistakes` 打开，以满足必需的 `Require no-mistakes` 检查。

常用命令：

```sh
corepack enable
pnpm install

pnpm run build          # 使用 tsdown 构建
pnpm run dev            # 监听构建
pnpm test               # 构建后运行全部 vitest 测试
pnpm run test:e2e       # 构建后运行端到端测试
pnpm run lint           # ESLint
pnpm run format:check   # 检查 Prettier 格式
pnpm run typecheck      # TypeScript 类型检查
```

## 故障排查

### 提示不是 Git 仓库

进入已有 Git 仓库，或先执行：

```sh
git init
```

### 工作区不干净

默认模式要求工作区干净。请先提交、暂存或清理当前改动，再运行：

```sh
git status
```

### Agent 启动失败

确认对应 Agent CLI 已安装、位于 `PATH` 中，并且已经完成登录或模型配置。

### 需要查看详细错误

查看本地运行日志：

```sh
cat .gnhf/runs/<runId>/gnhf.log
```

## Star History

<a href="https://www.star-history.com/?repos=kunchenguid%2Fgnhf&type=date&legend=top-left">
 <picture>
   <source media="(prefers-color-scheme: dark)" srcset="https://api.star-history.com/chart?repos=kunchenguid/gnhf&type=date&theme=dark&legend=top-left" />
   <source media="(prefers-color-scheme: light)" srcset="https://api.star-history.com/chart?repos=kunchenguid/gnhf&type=date&legend=top-left" />
   <img alt="Star History Chart" src="https://api.star-history.com/chart?repos=kunchenguid/gnhf&type=date&legend=top-left" />
 </picture>
</a>

## 许可证

本项目使用 MIT License，详情见 [LICENSE](./LICENSE)。
