# Copilot CLI

GitHub 官方终端编码 agent。前提：有效的 GitHub Copilot 订阅。

## 安装

```bash
# npm（要求 Node ≥ 22）
npm install -g @github/copilot

# 安装脚本（macOS / Linux）
curl -fsSL https://gh.io/copilot-install | bash

# Homebrew（macOS / Linux）
brew install --cask copilot-cli

# WinGet（Windows，要求 PowerShell ≥ 6）
winget install GitHub.Copilot
```

## 验证

```bash
copilot --version
```

## 认证

1. **交互登录**：进入项目目录运行 `copilot`，首次使用输入 `/login` 按提示完成 GitHub 认证。
2. **细粒度 PAT**：创建带 "Copilot Requests" 权限的 fine-grained personal access token，通过环境变量传入，优先级从高到低：`COPILOT_GITHUB_TOKEN` > `GH_TOKEN` > `GITHUB_TOKEN`。

## 更新

```bash
npm install -g @github/copilot@latest   # npm 安装时
brew upgrade --cask copilot-cli         # Homebrew 安装时
winget upgrade GitHub.Copilot           # WinGet 安装时
```

## 卸载

```bash
npm uninstall -g @github/copilot
brew uninstall --cask copilot-cli    # Homebrew 安装时
```

---

来源：[GitHub Copilot CLI 官方安装文档](https://docs.github.com/en/copilot/how-tos/copilot-cli/set-up-copilot-cli/install-copilot-cli)。核实日期：2026-08-06。
