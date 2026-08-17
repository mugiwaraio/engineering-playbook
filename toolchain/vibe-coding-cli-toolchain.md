# Vibe Coding CLI 工具链安装与多模型配置

## 一.基础环境安装

### 1.1 依赖安装（红帽系）

适用于 Fedora / RHEL / CentOS Stream 等使用 `dnf` 的发行版。

```bash
dnf install -y libatomic
dnf install -y bubblewrap
```

### 1.2 NVM安装

```bash
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.40.3/install.sh | bash
source ~/.bashrc
nvm install node
```

### 1.3 Vibe Coding CLI 安装

```bash
npm install -g @openai/codex@latest
npm install -g @anthropic-ai/claude-code@latest
npm install -g @google/gemini-cli@latest
npm install -g @vibe-kit/grok-cli
```

### 1.4 Claude Code 切到 native stable 安装

```bash
curl -fsSL https://claude.ai/install.sh | bash -s stable
claude install
npm uninstall -g @anthropic-ai/claude-code
```

## 二.工具链统一管理

### 2.1 CC Switch 管模型和 MCP

- 安装说明：https://ccswitch.io/zh/docs?section=getting-started&item=installation
- Releases：https://github.com/farion1231/cc-switch/releases

### 2.2 CCS 管账户和代理

- GitHub：https://github.com/kaitranntt/ccs

## 三.多 LLM API 接入

> 所有 `你的_KEY` 都是占位符，替换成本机真实 Key 时不要提交入库。

### 3.1 MiniMax

| 项目 | 内容 |
| --- | --- |
| 官方文档 | https://platform.minimaxi.com/docs/token-plan/intro |
| Anthropic 兼容 Base URL | https://api.minimaxi.com/anthropic |
| OpenAI 兼容 Base URL | https://api.minimaxi.com/v1 |
| 接入类型 | Coding Plan |

写入 shell 配置：

```bash
cat >> ~/.bashrc <<'EOF'

# MiniMax for Claude Code
export ANTHROPIC_BASE_URL="https://api.minimaxi.com/anthropic"
export ANTHROPIC_AUTH_TOKEN="你的_KEY"
EOF
source ~/.bashrc
```

写入 Claude Code 配置：

```bash
mkdir -p ~/.claude && cat > ~/.claude/settings.json <<'EOF'
{
  "env": {
    "ANTHROPIC_MODEL": "MiniMax-M3",
    "ANTHROPIC_DEFAULT_SONNET_MODEL": "MiniMax-M3",
    "ANTHROPIC_DEFAULT_OPUS_MODEL": "MiniMax-M3",
    "ANTHROPIC_DEFAULT_HAIKU_MODEL": "MiniMax-M3",
    "API_TIMEOUT_MS": "3000000",
    "CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC": "1"
  }
}
EOF
```

跳过引导并启动：

```bash
[ -f ~/.claude.json ] || echo '{"hasCompletedOnboarding": true}' > ~/.claude.json
claude
```

### 3.2 智谱 BigModel

| 项目 | 内容 |
| --- | --- |
| 官方文档 | https://docs.bigmodel.cn/cn/coding-plan/quick-start |
| Anthropic 兼容 Base URL | https://open.bigmodel.cn/api/anthropic |
| OpenAI 兼容 Base URL | 待补充 |
| 接入类型 | Coding Plan |

写入 shell 配置：

```bash
cat >> ~/.bashrc <<'EOF'

# 智谱 GLM for Claude Code
export ANTHROPIC_BASE_URL="https://open.bigmodel.cn/api/anthropic"
export ANTHROPIC_AUTH_TOKEN="你的_KEY"
EOF
source ~/.bashrc
```

写入 Claude Code 配置：

```bash
mkdir -p ~/.claude && cat > ~/.claude/settings.json <<'EOF'
{
  "env": {
    "ANTHROPIC_MODEL": "GLM-5.2",
    "ANTHROPIC_DEFAULT_SONNET_MODEL": "GLM-5.2",
    "ANTHROPIC_DEFAULT_OPUS_MODEL": "GLM-5.2",
    "ANTHROPIC_DEFAULT_HAIKU_MODEL": "GLM-5.2",
    "API_TIMEOUT_MS": "3000000",
    "CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC": "1"
  }
}
EOF
```

跳过引导并启动：

```bash
[ -f ~/.claude.json ] || echo '{"hasCompletedOnboarding": true}' > ~/.claude.json
claude
```

### 3.3 DeepSeek

| 项目 | 内容 |
| --- | --- |
| 官方文档 | https://api-docs.deepseek.com/zh-cn |
| Anthropic 兼容 Base URL | https://api.deepseek.com/anthropic |
| OpenAI 兼容 Base URL | https://api.deepseek.com |
| 接入类型 | 普通 API |

写入 shell 配置：

```bash
cat >> ~/.bashrc <<'EOF'

# DeepSeek for Claude Code
export ANTHROPIC_BASE_URL="https://api.deepseek.com/anthropic"
export ANTHROPIC_AUTH_TOKEN="你的_KEY"
EOF
source ~/.bashrc
```

写入 Claude Code 配置：

```bash
mkdir -p ~/.claude && cat > ~/.claude/settings.json <<'EOF'
{
  "env": {
    "ANTHROPIC_BASE_URL": "https://api.deepseek.com/anthropic",
    "ANTHROPIC_AUTH_TOKEN": "你的_KEY",
    "ANTHROPIC_MODEL": "deepseek-v4-pro[1m]",
    "ANTHROPIC_DEFAULT_OPUS_MODEL": "deepseek-v4-pro[1m]",
    "ANTHROPIC_DEFAULT_SONNET_MODEL": "deepseek-v4-pro[1m]",
    "ANTHROPIC_DEFAULT_HAIKU_MODEL": "deepseek-v4-flash",
    "CLAUDE_CODE_SUBAGENT_MODEL": "deepseek-v4-flash",
    "CLAUDE_CODE_EFFORT_LEVEL": "max"
  },
  "skipDangerousModePermissionPrompt": true
}
EOF
```

跳过引导并启动：

```bash
[ -f ~/.claude.json ] || echo '{"hasCompletedOnboarding": true}' > ~/.claude.json
claude
```

## 四.安全审计模型档位清单

已迁至 [models.md](models.md)（第六节「安全审计最强档」），与编码规范第 7.1 节、第 13.3 节「模型与档位选择」配套；型号更新只改该表，不改规范正文。
