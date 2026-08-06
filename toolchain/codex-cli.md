# Codex CLI

OpenAI 官方终端编码 agent。

## 安装

**推荐：官方安装脚本**

```bash
# macOS / Linux
curl -fsSL https://chatgpt.com/codex/install.sh | sh

# Windows
powershell -ExecutionPolicy ByPass -c "irm https://chatgpt.com/codex/install.ps1 | iex"
```

**备选方式：**

```bash
# npm（注意：包名是 @openai/codex；无 scope 的 codex 是无关的 2012 年老项目，勿装错）
npm install -g @openai/codex

# Homebrew
brew install --cask codex
```

支持平台：macOS（arm64 / x86_64）、Linux（x86_64 / arm64）、Windows。

**Linux 沙箱依赖**（Fedora / RHEL 系）：Codex 在 Linux 上执行命令依赖 bubblewrap 沙箱，需先安装：

```bash
dnf install -y bubblewrap
```

## 验证

```bash
codex --version
```

## 认证

运行 `codex`，两种方式：

1. **Sign in with ChatGPT**（推荐）：支持 Plus / Pro / Business / Edu / Enterprise 计划。
2. **API key**：需按开发者文档做额外配置。

## 更新

```bash
brew upgrade                       # Homebrew 安装时
npm install -g @openai/codex@latest   # npm 安装时
# 脚本安装时重新执行安装脚本即可
```

## 卸载

```bash
npm uninstall -g @openai/codex     # npm 安装时
brew uninstall --cask codex        # Homebrew 安装时
```

---

来源：[openai/codex GitHub README](https://github.com/openai/codex)。核实日期：2026-08-06。
