# Vibe Coding 工具链

各类 AI 编码 CLI 工具的安装、认证与更新说明。每个工具一个文件，内容从官方文档核实，文末注明来源与核实日期。

## 工具索引

| 工具 | 厂商 | 文档 |
|------|------|------|
| Claude Code | Anthropic | [claude-code.md](claude-code.md) |
| Codex CLI | OpenAI | [codex-cli.md](codex-cli.md) |
| Gemini CLI | Google | [gemini-cli.md](gemini-cli.md) |
| Copilot CLI | GitHub | [copilot-cli.md](copilot-cli.md) |

## 通用建议

- **Node.js 统一装 22 或更高**：Copilot CLI 和 Claude Code 的 npm 包均要求 Node ≥ 22，装 22+ 可覆盖全部四款工具的 npm 安装路径。
- **优先使用官方原生安装器**（curl/irm 脚本）：不依赖 Node 环境，且通常带自动更新；npm 全局安装作为备选。
- **npm 全局安装禁止 `sudo`**：会引入权限和安全问题，遇到权限错误应修复 npm 全局目录归属。
- **警惕仿冒包名**：安装前确认 scope（如 `@openai/codex` 而非无 scope 的 `codex`，后者是无关项目），与总规范第 7 章供应链要求一致。
- **安装方式迭代很快**：本目录内容有核实日期，执行前如发现命令失效，以文末来源链接的官方文档为准，并回头更新本文档。

## 扩展方式

- 新增工具：本目录下新建 `<工具名>.md`，包含安装、验证、认证、更新、卸载五节，文末注明来源与核实日期，并在上方索引表加一行。
- 模型档位清单（安全审计要求的最强模型对照表）如需集中维护，可在本目录新增 `models.md`。
