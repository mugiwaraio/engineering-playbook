# 工具配置模板

vibe coding 工具的配置说明模板，第一层按中转/接入方式分目录，第二层按工具分。取用方式：复制到对应位置，把占位内容（中转域名、API key）替换为自己的实际值。

| 目录 | 用途 | 目标位置 |
|------|------|---------|
| `sub2api/codex/config.toml` | Codex CLI 走 sub2api 中转的配置 | `~/.codex/config.toml` |
| `sub2api/codex/auth.json` | Codex CLI 的认证文件模板（key 为占位值） | `~/.codex/auth.json` |

注意：模板中的密钥和中转域名一律用占位值（`sk-xx`、`coding.xx.com`），禁止提交真实值——本仓库是公开的。
