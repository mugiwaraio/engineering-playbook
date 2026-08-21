# Claude Code Skills 插件安装指南

> 本文档记录项目中使用的 Claude Code Skill 插件及其安装方式。

## 1 插件概览

| 插件 | 用途 | 来源 |
|------|------|------|
| superpowers | 开发全流程 Skill 集（brainstorming、TDD、代码审查、Git 等 20+ 个） | [obra/superpowers](https://github.com/obra/superpowers) |
| ui-ux-pro-max | UI/UX 设计智能（67 种风格、161 套行业配色、自动推荐设计系统） | [nextlevelbuilder/ui-ux-pro-max-skill](https://github.com/nextlevelbuilder/ui-ux-pro-max-skill) |
| code-review | 代码审查（安全、性能、正确性检查） | [anthropics/claude-plugins-official](https://github.com/anthropics/claude-plugins-official/tree/main/plugins) |
| code-simplifier | 代码简化（清晰度、一致性、可维护性优化） | [anthropics/claude-plugins-official](https://github.com/anthropics/claude-plugins-official/tree/main/plugins) |
| find-bugs | 本地分支变更的 bug 与安全漏洞扫描 | [anthropics/claude-plugins-official](https://github.com/anthropics/claude-plugins-official/tree/main/plugins) |
| security-review | 安全代码审查（OWASP Top 10） | [anthropics/claude-plugins-official](https://github.com/anthropics/claude-plugins-official/tree/main/plugins) |
| investigate | 系统化调试（调查→分析→假设→实现，根因优先） | [anthropics/claude-plugins-official](https://github.com/anthropics/claude-plugins-official/tree/main/plugins) |
| coding-standards-frontend | TypeScript/React/Node.js 通用编码规范 | [anthropics/claude-plugins-official](https://github.com/anthropics/claude-plugins-official/tree/main/plugins) |
| planning-with-files | 文件式任务规划（task_plan.md / findings.md / progress.md） | OthmanAdi/planning-with-files |
| example-skills | Anthropic 官方技能集（PDF、PPTX、XLSX、前端设计、MCP 等） | [anthropics/skills](https://github.com/anthropics/skills) |
| document-skills | Anthropic 文档类技能集 | [anthropics/skills](https://github.com/anthropics/skills) |
| vercel-composition-patterns | React 组合模式（复合组件、避免 prop 钻取） | [vercel-labs/agent-skills](https://github.com/vercel-labs/agent-skills) |
| ast-grep | AST 级代码搜索与结构化重构 | [ast-grep/agent-skill](https://github.com/ast-grep/agent-skill) |
| webapp-testing | Playwright 自动化测试（含服务生命周期管理） | [anthropics/skills](https://github.com/anthropics/skills) |
| mcp-builder | 构建 MCP Server（让 LLM 调用外部服务） | [anthropics/skills](https://github.com/anthropics/skills) |
| caveman | 极致代码优化与精简 review | [juliusbrussee/caveman](https://github.com/juliusbrussee/caveman) |
| semgrep | 静态分析安全扫描 | [trailofbits/skills](https://github.com/trailofbits/skills) |
| supply-chain-risk-auditor | 依赖供应链安全审计 | [trailofbits/skills](https://github.com/trailofbits/skills) |
| claude-api | Claude API/SDK 开发指南 | [anthropics/skills](https://github.com/anthropics/skills) |
| tdd | 测试驱动开发（红-绿-重构循环） | [mattpocock/skills](https://github.com/mattpocock/skills) |
| web-design-guidelines | Web 界面设计规范审查 | [vercel-labs/agent-skills](https://github.com/vercel-labs/agent-skills) |

## 2 一键安装

复制以下脚本到终端执行，即可安装全部插件：

```bash
# === claude plugin 系列 ===
claude plugin install superpowers
claude plugin marketplace add nextlevelbuilder/ui-ux-pro-max-skill
claude plugin install ui-ux-pro-max@ui-ux-pro-max-skill
claude plugin marketplace add OthmanAdi/planning-with-files
claude plugin install planning-with-files
claude plugin marketplace add anthropics/skills
claude plugin install example-skills@anthropic-agent-skills
claude plugin install document-skills@anthropic-agent-skills

# === npx skills 系列（-g 全局安装，-y 跳过交互确认） ===
npx skills add https://github.com/anthropics/claude-plugins-official --skill code-review -g -y
npx skills add https://github.com/anthropics/claude-plugins-official --skill code-simplifier -g -y
npx skills add https://github.com/anthropics/claude-plugins-official --skill find-bugs -g -y
npx skills add https://github.com/anthropics/claude-plugins-official --skill security-review -g -y
npx skills add https://github.com/anthropics/claude-plugins-official --skill investigate -g -y
npx skills add https://github.com/anthropics/claude-plugins-official --skill coding-standards-frontend -g -y
npx skills add https://github.com/mattpocock/skills --skill tdd -g -y
npx skills add https://github.com/vercel-labs/agent-skills --skill vercel-react-best-practices -g -y
npx skills add https://github.com/vercel-labs/agent-skills --skill vercel-composition-patterns -g -y
npx skills add https://github.com/vercel-labs/agent-skills --skill web-design-guidelines -g -y
npx skills add https://github.com/anthropics/skills --skill webapp-testing -g -y
npx skills add https://github.com/ast-grep/agent-skill --skill ast-grep -g -y
npx skills add https://github.com/anthropics/skills --skill mcp-builder -g -y
npx skills add https://github.com/anthropics/skills --skill claude-api -g -y
npx skills add https://github.com/juliusbrussee/caveman --skill caveman -g -y
npx skills add https://github.com/trailofbits/skills --skill semgrep -g -y
npx skills add https://github.com/trailofbits/skills --skill supply-chain-risk-auditor -g -y
```

> 说明：`-g` 安装到全局（`~/.claude/skills/`），`-y` 跳过所有交互式确认。仅项目级使用时去掉 `-g`。

## 3 分步安装

### 3.1 核心插件：Superpowers

```bash
claude plugin install superpowers
```

### 3.2 UI/UX 设计

```bash
claude plugin marketplace add nextlevelbuilder/ui-ux-pro-max-skill
claude plugin install ui-ux-pro-max@ui-ux-pro-max-skill
```

### 3.3 代码质量与安全

```bash
npx skills add https://github.com/anthropics/claude-plugins-official --skill code-review -g -y
npx skills add https://github.com/anthropics/claude-plugins-official --skill code-simplifier -g -y
npx skills add https://github.com/anthropics/claude-plugins-official --skill find-bugs -g -y
npx skills add https://github.com/anthropics/claude-plugins-official --skill security-review -g -y
npx skills add https://github.com/anthropics/claude-plugins-official --skill investigate -g -y
npx skills add https://github.com/anthropics/claude-plugins-official --skill coding-standards-frontend -g -y
```

### 3.4 文件式规划

```bash
claude plugin marketplace add OthmanAdi/planning-with-files
claude plugin install planning-with-files
```

### 3.5 Anthropic 官方技能集

```bash
claude plugin marketplace add anthropics/skills
claude plugin install example-skills@anthropic-agent-skills
claude plugin install document-skills@anthropic-agent-skills
```

### 3.6 前端开发与测试

```bash
# TDD（Matt Pocock）
npx skills add https://github.com/mattpocock/skills --skill tdd -g -y

# Vercel React 系列
npx skills add https://github.com/vercel-labs/agent-skills --skill vercel-react-best-practices -g -y
npx skills add https://github.com/vercel-labs/agent-skills --skill vercel-composition-patterns -g -y
npx skills add https://github.com/vercel-labs/agent-skills --skill web-design-guidelines -g -y

# Playwright 自动化测试
npx skills add https://github.com/anthropics/skills --skill webapp-testing -g -y

# AST 结构化重构
npx skills add https://github.com/ast-grep/agent-skill --skill ast-grep -g -y
```

### 3.7 AI 集成与 API

```bash
# MCP Server 构建
npx skills add https://github.com/anthropics/skills --skill mcp-builder -g -y

# Claude API/SDK 开发
npx skills add https://github.com/anthropics/skills --skill claude-api -g -y
```

### 3.8 代码优化与供应链安全

```bash
# 极致代码优化
npx skills add https://github.com/juliusbrussee/caveman --skill caveman -g -y

# 静态分析
npx skills add https://github.com/trailofbits/skills --skill semgrep -g -y

# 依赖供应链审计
npx skills add https://github.com/trailofbits/skills --skill supply-chain-risk-auditor -g -y
```

### 3.9 更新所有 npx 安装的技能

```bash
npx skills update
```

## 4 Claude Code 升级与维护

以下命令用于升级 Claude Code CLI 自身（区别于 3.9 的 skill 插件更新）。

日常升级（按顺序执行：先升级 npm 自身 → 升级 Claude Code → 验证安装健康度）：

```bash
npm install -g npm
npm install -g @anthropic-ai/claude-code@latest
claude doctor
```

故障排查（`npm update -g` 批量更新全局包；`npm cache clean --force` 在缓存损坏或安装异常时使用）：

```bash
npm update -g
npm cache clean --force
```

## 5 安装方式说明

项目中混用两种安装方式：

| 方式 | 命令 | 适用场景 |
|------|------|---------|
| claude plugin | `claude plugin install` / `claude plugin marketplace add` | Claude Code 原生插件系统 |
| npx skills | `npx skills add <repo> --skill <name>` | 社区 Skills 仓库（skills.sh 生态） |

两者共存，无冲突。`claude plugin` 管理的插件通过 Claude Code 内置机制加载；`npx skills` 管理的插件写入项目 `.claude/skills/` 目录。

本仓库项目级 skill：

```text
.claude/skills/coding-standards/
```

## 6 参考资料

- Superpowers 仓库：https://github.com/obra/superpowers
- Superpowers 介绍：https://zhuanlan.zhihu.com/p/2015725269667840386
- UI/UX Pro Max 仓库：https://github.com/nextlevelbuilder/ui-ux-pro-max-skill
- UI/UX Pro Max 官网：https://www.uupm.cc/
- Skills 目录：https://skills.sh
