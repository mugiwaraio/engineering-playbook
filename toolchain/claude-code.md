# Claude Code

Anthropic 官方 CLI 编码工具。

## 安装

**推荐：原生安装器**（无 Node 依赖，后台自动更新）

```bash
# macOS / Linux / WSL
curl -fsSL https://claude.ai/install.sh | bash

# Windows PowerShell
irm https://claude.ai/install.ps1 | iex

# Windows CMD
curl -fsSL https://claude.ai/install.cmd -o install.cmd && install.cmd && del install.cmd
```

安装 stable 渠道（约滞后一周，跳过有重大回归的版本）：

```bash
curl -fsSL https://claude.ai/install.sh | bash -s stable
```

**备选方式**（均不自动更新，需手动升级）：

```bash
# Homebrew（claude-code 为 stable 渠道，claude-code@latest 为 latest 渠道）
brew install --cask claude-code

# WinGet
winget install Anthropic.ClaudeCode

# npm（要求 Node ≥ 22，禁止 sudo）
npm install -g @anthropic-ai/claude-code
```

Debian/Ubuntu（apt）、Fedora/RHEL（dnf）、Alpine（apk）有官方签名仓库，命令见来源文档。

## 验证

```bash
claude --version
claude doctor    # 安装与配置健康检查
```

## 认证

运行 `claude`，按浏览器提示登录。要求 Pro / Max / Team / Enterprise / Console 账号（免费版无 Claude Code 权限）。设置了 `ANTHROPIC_API_KEY` 环境变量时会提示确认使用该 key。也可对接 Amazon Bedrock / Google Vertex / Microsoft Foundry。

## 更新

```bash
claude update    # 手动立即更新；原生安装本身会后台自动更新
brew upgrade claude-code          # Homebrew 安装时
winget upgrade Anthropic.ClaudeCode   # WinGet 安装时
npm install -g @anthropic-ai/claude-code@latest   # npm 安装时（勿用 npm update -g）
```

## 卸载

```bash
# 原生安装
rm -f ~/.local/bin/claude
rm -rf ~/.local/share/claude

# npm 安装
npm uninstall -g @anthropic-ai/claude-code

# 彻底清除配置（会删除设置、MCP 配置和会话历史）
rm -rf ~/.claude && rm ~/.claude.json
```

---

来源：[Claude Code 官方安装文档](https://code.claude.com/docs/en/setup)。核实日期：2026-08-06。
