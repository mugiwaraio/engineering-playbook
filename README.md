# Engineering Playbook

工程规范模板库。仓库内的文件是**供复制到实际项目使用的模板和说明案例**，本仓库自身不做任何开发，也不包含可运行代码。

模板的核心目标之一：控制 AI 编码工具（codex、Claude Code 等）的上下文读取范围，避免其读入依赖、构建产物等无用内容。

## 目录大类约定

第一层按内容性质分大类，技术栈在 `stacks/` 下平级展开。大类**按需创建**，不预建空目录：

| 大类 | 说明 | 状态 |
|------|------|------|
| `stacks/` | 按技术栈划分的项目级模板（每个栈一个目录，整体复制即可用） | 已建 |
| `toolchain/` | vibe coding 工具的安装、认证与更新说明（每个工具一个文件） | 已建 |
| `configs/` | vibe coding 工具的配置说明模板（按工具分子目录，占位值替换后使用） | 已建 |
| `common/` | 跨栈通用模板（agent 通用规则、git 规范、hooks）。第二个栈出现且与现有栈有重复内容时，把共性部分上提至此 | 按需 |
| `ci/` | CI 流水线模板（覆盖率门禁、依赖漏洞扫描等），按平台分子目录 | 按需 |
| `scripts/` | 脚本模板（如 `check.sh`） | 按需 |
| `docs/` | playbook 自身的说明与规则决策记录 | 按需 |

## 现有模板

### `stacks/nodejs/`

| 文件 | 用途 | 使用方式 |
|------|------|---------|
| `AGENTS.md` | codex 等 AI 工具的上下文控制规则：限定读取范围、搜索纪律、依赖目录与大文件禁读 | 复制到项目根目录，直接可用 |
| `gitignore` | Node.js 项目通用忽略规则 | 复制到项目根目录后**重命名为 `.gitignore`**。模板去点存放是有意为之：避免它在本仓库内实际生效，无声忽略模板目录下的文件 |

### `toolchain/`

四款主流 CLI 工具的安装说明：Claude Code、Codex CLI、Gemini CLI、Copilot CLI。内容从官方文档核实，每篇文末注明来源与核实日期。索引见 [toolchain/README.md](toolchain/README.md)。

## 扩展方式

- **新增技术栈**：在 `stacks/` 下新建目录（如 `stacks/python/`），结构与 `stacks/nodejs/` 同构。
- **agent 工具维度不单独分层**：`AGENTS.md`（codex/copilot 通用）、`CLAUDE.md` 等直接平铺在各栈目录内；仅当同一栈需要为不同工具维护内容明显分化的多套规则时，才在栈目录下建 `agents/` 子目录。
- **忽略类模板一律去点存放**（`gitignore`、`dockerignore` 等），README 或栈目录内注明复制后的重命名方式。
