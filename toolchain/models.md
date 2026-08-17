# 模型档位对照表

配套 `coding/claude/CLAUDE.md` 与 `coding/codex/AGENTS.md` 第 13.3 节「模型与档位选择」：规范只定义四个档位（最强 / 主力 / 经济 / 轻量）和推理档位，具体型号在此维护，型号换代只改本表不改规范。第 7.1 节安全审计要求的"最强模型 + 最高推理档位"也以本表第六节为准。

核实日期：2026-08-17（来源见文末）。价格单位均为美元 / 每百万 token（input / output）。

## 一、档位 → 型号

| 档位 | Claude（Claude Code 别名 / 模型 ID） | OpenAI（Codex CLI 模型 ID） |
|---|---|---|
| 最强档 | `fable` / `claude-fable-5` | `gpt-5.6-sol` |
| 主力档 | `opus` / `claude-opus-5` | `gpt-5.6-sol`（high）或 `gpt-5.6-terra`（xhigh） |
| 经济档 | `opus` medium，或 `sonnet` / `claude-sonnet-5` high | `gpt-5.6-terra` |
| 轻量档 | `haiku` / `claude-haiku-4-5`，或 `sonnet` low | `gpt-5.6-luna` |

## 二、干什么 → 用什么（举例）

| 干什么 | Claude | OpenAI / Codex | 为什么这样配 |
|---|---|---|---|
| 复杂任务规划：需求拆解、架构方案、技术选型、Migration 方案 | Fable 5，xhigh；需大范围通读代码库或多方案并行比选时再开 ultracode | gpt-5.6-sol，xhigh；可并行拆分时 ultra | 决策在最上游、输出量小，值得用最强；Fable 5 官方称"低档位常超过前代的 xhigh"，max 留给最难的比选 |
| 安全审计、漏洞挖掘、疑难 bug 深挖 | Fable 5，max（单代理深挖）；需并行覆盖全库/多模块时改 `/effort ultracode`（= xhigh + workflow 编排，与 max 二选一），或保持 max、只在该条 prompt 加 `ultracode` 关键字触发一次 workflow | gpt-5.6-sol，ultra（= max 级推理 + 自动委派） | 第 7.1 节强制最强 + 最高档，禁止降档 |
| 生产数据修复方案、资金/权限/密钥相关逻辑评估 | Fable 5，xhigh | gpt-5.6-sol，xhigh | 不可逆或安全相关，出错代价最高 |
| 实际开发：多文件功能实现、跨模块重构、按方案落地、疑难 bug 修复 | Opus 5，high 默认；复杂 agentic 编码上 xhigh；仅难度顶端用 max | gpt-5.6-sol high/xhigh，或 gpt-5.6-terra xhigh | Opus 5 官方：high 起步、xhigh 给要求高的编码/agentic、max 易收益递减且简单任务上过度思考 |
| 常规实现：单函数/单接口、补单元测试、小 bug、按既定方案的机械改动、按模板改配置 | Opus 5 medium，或 Sonnet 5 high | gpt-5.6-terra medium/high | Opus 5 官方称 low/medium "应作为成本主控大量使用"；Sonnet 5 是 Opus 5 价格的 40% |
| 独立代码评审（第 10 章第二模型审查） | 与作者不同的模型：Opus 5 high（大 diff 上 xhigh）；或跨厂商用 Codex | gpt-5.6-sol high/xhigh（`review_model` 单独配）；或跨厂商用 Claude | 独立性来自"不同模型或不同会话"，不要求最强档 |
| 文档同步、commit message、CHANGELOG、格式化、样板代码 | Sonnet 5 low，或 Haiku 4.5 | gpt-5.6-luna | 输出可机械校验，推理需求低 |
| 代码库探索、搜索、读文件摘要、日志/报错摘要（Explore 类子代理） | Haiku 4.5，或 Sonnet 5 low | gpt-5.6-luna | input 大、推理少，是最烧钱又最不需要强模型的地方；Haiku 与 Fable 单价差 10 倍 |
| Fast mode（加速） | 只在人在等待的交互场景开；Opus 5 Fast $10/$50（2 倍） | Codex `/fast`，2 倍价 | 加速不加智，后台与批量任务不开 |

## 三、价格与能力速查

### Claude

| 模型 | 模型 ID | 价格 input / output | 上下文 / 最大输出 | effort 支持 | 备注 |
|---|---|---|---|---|---|
| Claude Fable 5 | `claude-fable-5` | $10 / $50 | 1M / 128k | low / medium / high / xhigh / max，默认 high | 思考始终开启不可关；自带安全分类器：API 直连可能返回 `refusal`；Claude Code（v2.1.219+）中被网络安全类分类器命中的请求会自动改在 Opus 4.8 重跑（生物类改 Opus 5），transcript 显示提示后**会话即停留在回退模型**，需 `/model fable` 切回；渗透测试、CTF、安全审计类负载常在首个请求就触发（CLAUDE.md、git status 也参与判定）；要改为每次询问，在 `/config` 关闭 Switch models when a message is flagged（`switchModelsOnFlag: false`）；需 30 天数据保留、不支持 ZDR；单次请求可能跑数分钟；官方定位"最高可用能力"，其余场景官方建议从 Opus 5 起步 |
| Claude Opus 5 | `claude-opus-5` | $5 / $25 | 1M / 128k | 五档全支持，默认 high | 默认开思考；`thinking: disabled` 只允许 effort ≤ high；Claude Code 中网络安全类请求同样自动回退到 Opus 4.8（生物类直接 refusal，无回退）；Fast mode $10 / $50（仅 Claude API） |
| Claude Sonnet 5 | `claude-sonnet-5` | $2 / $10 | 1M / 128k | 五档全支持，默认 high | $2/$10 原为限时价，官方已宣布成为标准价（原定 2026-09-01 涨至 $3/$15 取消）；新 tokenizer 同样文本约多 30% token；medium ≈ Sonnet 4.6 high，high ≈ Sonnet 4.6 max |
| Claude Haiku 4.5 | `claude-haiku-4-5`（完整 ID `claude-haiku-4-5-20251001`） | $1 / $5 | 200k / 64k | 不支持 effort（也不支持 adaptive thinking） | 最快最便宜；不在 effort 支持模型列表内，不要传该参数（传入后的具体行为官方未说明） |

### OpenAI / Codex

| 模型 | 模型 ID | 价格 input / output | 上下文 / 最大输出 | Codex 推理档位 | 备注 |
|---|---|---|---|---|---|
| GPT-5.6 Sol | `gpt-5.6-sol` | $5 / $30（>272K 输入按 2× / 1.5× 计） | API 1.05M / 128k；Codex 内 272k | low / medium / high / xhigh / max / ultra | 旗舰；官方"不确定就选 Sol"；Codex 内默认 low，API 默认 medium |
| GPT-5.6 Terra | `gpt-5.6-terra` | $2 / $12 | Codex 内 272k | low / medium / high / xhigh / max / ultra | 均衡款，"日常工作的务实全能型"，能力接近 GPT-5.5 但更便宜 |
| GPT-5.6 Luna | `gpt-5.6-luna` | $0.20 / $1.20 | Codex 内 272k | low / medium / high / xhigh / max（无 ultra） | 轻量款，相当于旧 nano 档；适合抽取、分类、转换类重复任务 |
| GPT-5.5 | `gpt-5.5` | $5 / $30 | — | 最高 xhigh | 上一代旗舰，仍可选 |
| GPT-5.4 / GPT-5.4 mini | `gpt-5.4` / `gpt-5.4-mini` | $2.5 / $15、$0.75 / $4.5 | — | 最高 xhigh | 2026-08-31 从 Codex（ChatGPT 登录）退役，官方建议分别换 terra / luna；OpenAI API 及用 API key 登录的 Codex 不受影响 |

Batch / Flex 为标准价 50%，Fast mode 为 2 倍。

## 四、推理档位说明

- **Claude API `effort`**：low / medium / high / xhigh / max，省略等价于 high；影响响应中的所有 token（正文、工具调用、思考），低档位会减少工具调用次数；Opus 5 上改 effort 不可靠地缩短可见回复长度，要短回复靠提示词。官方推荐：Opus 5 从 high 起步、要求高的编码/agentic 上 xhigh、max 仅在值得不计 token 时用、low/medium 大量使用；Fable 5 同样 high 默认，xhigh 只给能力最敏感的场景，routine 工作降到 medium/low；Sonnet 5 默认 high，最难的编码/agentic 用 xhigh。高档位下须给足 `max_tokens`（思考 + 回复共用该上限）：Opus 4.7 / 4.8 / 5 在 xhigh/max 下官方建议从 64k 起调；Fable 5 在 high/xhigh 下官方只说"设大"，未给具体数字。
- **Claude Code**：`/effort` 可选 low / medium / high / xhigh / max / ultracode / auto，默认 high。`max` 与 `ultracode` 是会话级设置，不能写进 `effortLevel` 设置项；`CLAUDE_CODE_EFFORT_LEVEL` 环境变量优先级最高、不接受 ultracode。**ultracode = xhigh + 动态多代理 workflow 编排**，不是第六个档位，`--effort ultracode` 需 v2.1.203+；官方：支持 xhigh 的模型均可开（Fable 5 / Opus 5 / Sonnet 5 / Opus 4.8 / 4.7），其余模型的 `/effort` 菜单不显示 ultracode。ultracode 与 max 是 `/effort` 同一选择器里的互斥取值，选 ultracode 即 xhigh，不能"max 叠加 ultracode"；workflow 关闭时 ultracode 退化为 xhigh。
- **Codex CLI `model_reasoning_effort`**：源码取值 none / minimal / low / medium / high / xhigh / max / ultra。**ultra = 最高推理 + 自动任务委派（子代理并行）**，是 `model_reasoning_effort` 的取值、与 max 互斥（选择器第 6 项 "Maximum reasoning with automatic task delegation"），不是叠加在 max 上的开关；仅 sol / terra 支持，luna 最高 max，5.5 / 5.4 / 5.2 最高 xhigh。官方 config-reference 页表格仍只写到 xhigh（文档滞后），以源码和模型页为准。OpenAI API 的 `reasoning.effort` 只接受到 max，不接受 ultra。

## 五、怎么设置

### Claude Code

- 主会话模型：`/model`（别名 `best`（组织有 Fable 5 权限时解析为 Fable 5，否则最新 Opus）/ `fable` / `opus` / `sonnet` / `haiku` / `opusplan`，或完整 ID）；启动参数 `--model`；`settings.json` 的 `model` 字段；环境变量 `ANTHROPIC_MODEL`（优先级最高）。`ANTHROPIC_DEFAULT_OPUS_MODEL` / `_SONNET_MODEL` / `_HAIKU_MODEL` / `_FABLE_MODEL` 控制各别名解析到哪个具体型号（第三方兼容接入时靠它把别名映射到中转模型）。
- 推理档位：`/effort`；启动参数 `--effort`；`settings.json` 的 `effortLevel`（不含 max / ultracode）；环境变量 `CLAUDE_CODE_EFFORT_LEVEL`。ultracode：`/effort ultracode`、`--effort ultracode`，或 `--settings '{"ultracode": true}'`。只想对单个任务开工作流、不改会话 effort：在 prompt 里加关键字 `ultracode`（或直说 "use a workflow"）；仅对人工键入的 prompt 生效，`-p`、未标记为人工输入的 SDK prompt、定时任务不触发。
- Fast mode：`/fast` 切换，`fastMode` 设置项；仅 Opus 5 / 4.8。
- 子代理模型解析顺序：`CLAUDE_CODE_SUBAGENT_MODEL` 环境变量 > 调用时的 `model` 参数 > `.claude/agents/*.md` frontmatter 的 `model:`（别名 / 完整 ID / `inherit`）> 主会话模型。内置 Explore / Plan / general-purpose 默认继承主会话（Explore 在 Claude API 上封顶 Opus，Plan / general-purpose 完整继承）——所以派发探索类子代理必须显式传 `model`，否则跟着主会话跑高档位；可在 `.claude/agents/` 建同名代理覆盖内置默认。
- 子代理 effort：Agent 工具逐次调用只有 `model` 参数、没有 effort 参数；effort 只能写在 `.claude/agents/*.md` 的 `effort` frontmatter（low / medium / high / xhigh / max，默认继承会话）、Agent SDK 的 `AgentDefinition.effort`，或 Workflow 脚本 `agent()` 的 effort 选项。`CLAUDE_CODE_EFFORT_LEVEL` 优先于 frontmatter effort。要把所有子代理钉到某个模型用 `CLAUDE_CODE_SUBAGENT_MODEL`（优先级最高，`inherit` 恢复正常解析）。

### Codex CLI

- 交互中 `/model` 选模型并同时选推理档位；启动 `codex -m <id>` / `codex --model <id>`；非交互 `codex exec -m <id> "..."`；临时覆盖 `-c model="..."`。
- `~/.codex/config.toml`：`model`、`model_reasoning_effort`、`review_model`（评审用的模型可单独指定，本仓库 `configs/sub2api/codex/config.toml` 即此写法）。
- 子代理模型解析顺序：自定义代理文件（`~/.codex/agents/*.toml` 个人、`.codex/agents/*.toml` 项目）里写的 `model` / `model_reasoning_effort` > 派发时显式值 > `~/.codex/config.toml` `[agents]` 的 `default_subagent_model` / `default_subagent_reasoning_effort` > 父会话值；同名自定义代理可覆盖内置 default / worker / explorer。不显式指定也不设 `[agents]` 默认，子代理就跟着主会话跑最强档。
- `/fast` 切换 Fast 服务档。

## 六、安全审计最强档（第 7.1 节）

| 厂商 | 最强模型 | 最高推理档位 |
|---|---|---|
| Claude | Fable 5（`claude-fable-5`） | max（单代理最高档）；需并行覆盖全库时改 ultracode（= xhigh + workflow 编排，与 max 二选一）。注意：Claude Code 内安全类请求可能被自动回退到 Opus 4.8 且会话停留，审计中须留意 transcript 提示并用 `/model fable` 切回，或预先设 `switchModelsOnFlag: false` |
| OpenAI | GPT-5.6 Sol（`gpt-5.6-sol`） | ultra（Codex 内最高取值，= max 级推理 + 自动委派）；不可并行时 max；API 侧 max |

## 来源

- Anthropic：platform.claude.com/docs/en/about-claude/models/overview、/about-claude/pricing、/build-with-claude/effort、/build-with-claude/thinking-troubleshooting、/about-claude/models/introducing-claude-fable-5-and-claude-mythos-5、/build-with-claude/prompt-engineering/prompting-claude-sonnet-5
- Claude Code：code.claude.com/docs/en/model-config、/sub-agents、/workflows、/settings、/env-vars、/commands、/cli-reference、/fast-mode
- OpenAI：learn.chatgpt.com/docs/models（原 developers.openai.com/codex/models）、developers.openai.com/api/docs/pricing、developers.openai.com/api/docs/models/gpt-5.6-sol、learn.chatgpt.com/docs/developer-commands?surface=cli、learn.chatgpt.com/docs/agent-configuration/subagents、learn.chatgpt.com/docs/config-file/config-reference、github.com/openai/codex（`codex-rs/protocol/src/openai_models.rs`、`codex-rs/models-manager/models.json`）
- 核实日期：2026-08-17。执行前如发现型号或价格已变，以上述官方页面为准，并回头更新本表。
