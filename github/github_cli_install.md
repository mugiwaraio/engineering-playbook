# GitHub CLI（gh）安装

GitHub 官方命令行工具，用于操作 PR、issue、GitHub API 等。以下步骤适用于 Amazon Linux 2023 / Fedora / RHEL 等使用 dnf 的发行版。

## 安装

```bash
# 1. 安装 dnf config-manager 插件（用于添加第三方仓库）
sudo dnf install 'dnf-command(config-manager)'

# 2. 添加 GitHub CLI 官方仓库
sudo dnf config-manager --add-repo https://cli.github.com/packages/rpm/gh-cli.repo

# 3. 安装 gh
sudo dnf install gh -y
```

## 验证

```bash
gh --version
```

## 认证

```bash
gh auth login    # 按提示选择 GitHub.com，浏览器或 token 方式登录
gh auth status   # 查看当前认证状态
```
