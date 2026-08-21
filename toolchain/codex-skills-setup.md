# Codex CLI Skills 安装指南

> 本文档记录在 OpenAI Codex CLI 中使用 Agent Skills 的方式，是 [`claude-code-skills-setup.md`](./claude-code-skills-setup.md) 的 Codex 对应版本。
>
> **核心前提**：Codex 与 Claude Code 共用同一套 [Agent Skills 开放标准](https://developers.openai.com/codex/skills)（`SKILL.md` + frontmatter，必填 `name` / `description`）。因此**纯指令型 skill 可跨平台复用**，只是安装目录、安装命令、调用方式和工具名映射不同。下文已据 OpenAI / vercel-labs 官方文档逐项核实（非训练记忆，呼应规范第 13.1 章）。

## 1 Codex 与 Claude Code 的关键差异

| 维度 | Claude Code | Codex CLI |
|------|-------------|-----------|
| 项目级 skill 目录 | `.claude/skills/` | `.agents/skills/`（自当前目录向上扫描到仓库根） |
| 用户级 skill 目录 | `~/.claude/skills/` | `~/.agents/skills/` |
| 系统级 skill 目录 | —— | `/etc/codex/skills` |
| 配置文件 | `settings.json` | `~/.codex/config.toml` |
| 项目指令文件 | `CLAUDE.md` | `AGENTS.md`（本仓库见 [`../codex/AGENTS.md`](../codex/AGENTS.md)） |
| 原生安装命令 | `claude plugin install` | `$skill-installer <name>`（在 Codex 会话内执行） |
| 调用方式 | `Skill` 工具 / 自动触发 | `$<skill-name>` 显式调用 / 自动触发 / `/skills` 列出 |
| 子 agent 支持 | 内置 | 需在 config.toml 开启 `multi_agent`（见第 5 节） |

> Codex 自动发现上述目录下的所有 `SKILL.md`，**无需在 config.toml 中逐个登记**。新装 skill 若未出现，重启 Codex。

## 2 三种安装方式

Codex 下安装 skill 有三条路径，按来源选择：

### 2.1 Codex 官方目录（`$skill-installer`）

OpenAI 维护的 [openai/skills](https://github.com/openai/skills) 目录分三类：`.system`（随 Codex 自带）、`.curated`（手动安装）、`.experimental`（实验性）。在 **Codex 会话内**执行：

```text
# 安装 curated 技能
$skill-installer gh-address-comments

# 安装 experimental 技能（指定文件夹或 GitHub URL）
$skill-installer install the create-plan skill from the .experimental folder
$skill-installer install https://github.com/openai/skills/tree/main/skills/.experimental/create-plan
```

> `$skill-installer` 与 `$skill-creator` 本身就是 Codex 内置 skill，用自然语言驱动。安装后需重启 Codex。

### 2.2 社区技能（`npx skills`，skills.sh 生态）

[vercel-labs/skills](https://github.com/vercel-labs/skills) 的 `npx skills` CLI 跨平台，用 `-a codex` 指定目标 agent，是把社区技能搬到 Codex 的主要方式。

常用参数：

- `-s, --skill <name>`：指定 skill；可重复；`'*'` 表示该仓库全部。不带 `--skill` 会弹出交互式多选菜单。
- `-a, --agent codex`：目标 agent，可重复（如 `-a claude-code -a codex` 同时装两端）。
- `-g, --global`：装到用户级目录；不带则装到项目级 `./.agents/skills/`。
- `-y, --yes`：跳过所有交互确认。

> 不要用 `--all`：它等价于 `--skill '*' --agent '*' --yes`，会装到**所有检测到的 agent**。要 Codex 专属，固定用 `--skill … -a codex -y`。
>
> `-g` 的 Codex 落地目录文档间有出入（`~/.agents/skills/` vs `~/.codex/skills/`）。装完用 `/skills` 确认已识别；未识别则手动移到 `~/.agents/skills/`。要目录归属绝对确定就去掉 `-g` 改项目级。

本仓库选用清单（逐条复制执行；用户级、仅 Codex、非交互）：

```bash
npx skills add https://github.com/obra/superpowers                     --skill '*'             -g -a codex -y
npx skills add https://github.com/anthropics/skills                    --skill frontend-design -g -a codex -y
npx skills add https://github.com/nextlevelbuilder/ui-ux-pro-max-skill --skill ui-ux-pro-max   -g -a codex -y
npx skills add https://github.com/coderabbitai/skills                  --skill code-review     -g -a codex -y
npx skills add https://github.com/mattpocock/skills                    --skill tdd             -g -a codex -y
npx skills add https://github.com/vercel-labs/skills                   --skill find-skills     -g -a codex -y
```

> obra/superpowers 为 Claude 插件式结构，若 CLI 未发现其 skill，回退 2.3 用 `git clone` 放入 `.agents/skills/`。

### 2.3 手动放置（git clone / 复制）

任何符合标准的 skill 仓库，直接克隆或复制到目标目录即可，Codex 自动发现：

```bash
# 项目级
mkdir -p .agents/skills
git clone https://github.com/obra/superpowers .agents/skills/superpowers

# 用户级（跨仓库通用）
mkdir -p ~/.agents/skills
cp -r ./my-skill ~/.agents/skills/
```

## 3 其他可按需补充的技能

第 2.2 节是已选清单。下表是可按同样方式（`npx skills add <repo> --skill <name> -g -a codex -y`）补充的常用技能，与已选清单不重复。

| 技能 | 用途 | 来源仓库 |
|------|------|---------|
| find-bugs / security-review / investigate | 代码质量与安全 | `anthropics/claude-plugins-official` |
| coding-standards-frontend | TS/React/Node 编码规范 | `anthropics/claude-plugins-official` |
| ast-grep | AST 级结构化重构 | `ast-grep/agent-skill` |
| webapp-testing / mcp-builder / claude-api | 测试 / MCP / API | `anthropics/skills` |
| semgrep / supply-chain-risk-auditor | 静态分析 / 供应链审计 | `trailofbits/skills` |
| vercel-react-best-practices / vercel-composition-patterns / web-design-guidelines | 前端规范 | `vercel-labs/agent-skills` |

> 来自 Claude 生态的技能，正文可能写死 Claude 工具名（Read/Edit/Bash 等），跨平台时按第 4 节工具映射理解。
>
> `caveman`、`planning-with-files`、`example-skills`、`document-skills` 等若只在 Claude marketplace 提供，可用 2.3 从对应 GitHub 仓库 `git clone` 到 `.agents/skills/`；但 Claude 专属的插件机制（`plugin.json` 的 `agents` 字段等）在 Codex 不通用。

## 4 工具名映射（适配 Claude 系技能）

来自 Claude 生态的 skill 正文可能引用 Claude 工具名，在 Codex 中按下表理解执行（摘自 superpowers 的 `references/codex-tools.md`）：

| Skill 中的写法 | Codex 等价 |
|----------------|-----------|
| `Read` / `Write` / `Edit` | Codex 原生文件工具 |
| `Bash` | Codex 原生 shell 工具 |
| `TodoWrite` | `update_plan` |
| `Skill`（调用技能） | 原生加载，直接照指令执行 |
| `Task`（派发子 agent） | `spawn_agent` + `wait` + `close_agent`（需开启 multi_agent） |

## 5 配置（`~/.codex/config.toml`）

```toml
# 启用子 agent，供 dispatching-parallel-agents、subagent-driven-development 等技能使用
[features]
multi_agent = true

# 按路径禁用某个 skill（默认全部启用，无需逐个登记）
[[skills.config]]
path = "/path/to/skill/SKILL.md"
enabled = false
```

> 改动 `config.toml` 后需重启 Codex 生效。

## 6 维护

```bash
# 升级 Codex CLI 自身
npm install -g @openai/codex@latest

# 更新 npx skills 安装的技能
npx skills update

# 会话内列出当前可用技能，确认安装是否被识别
/skills
```

## 7 参考资料

- Codex Agent Skills 官方文档：https://developers.openai.com/codex/skills
- Codex AGENTS.md 指南：https://developers.openai.com/codex/guides/agents-md
- OpenAI 官方技能目录：https://github.com/openai/skills
- skills CLI（vercel-labs）：https://github.com/vercel-labs/skills
- skills.sh Codex 安装页：https://www.skills.sh/agent/codex
- Superpowers（自带 codex-tools 适配）：https://github.com/obra/superpowers
