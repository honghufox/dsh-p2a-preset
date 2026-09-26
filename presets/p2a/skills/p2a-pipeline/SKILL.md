---
name: p2a-pipeline
description: Paper2Agent 主流水线：把一篇文献（论文 + 代码库 + 补充材料/数据/图表）转换成可交互、可复现的 paper agent（MCP 服务器 + agent 预设）的六阶段总纲，含子代理分工、文件布局、阶段间 JSON 契约、验证门禁与交付报告。
whenToUse: 用户要求把某篇论文/文献/代码库「转成 agent」「做成 MCP」「做成虚拟通讯作者」，或要复现某篇论文的分析流程时；也是本模式所有其他 p2a-* 技能的入口。
---

# Paper2Agent 主流水线（文献 → Agent）

把论文的被动知识变成可执行、可对话的 agent：把论文的贡献封装成 MCP 服务器（tools + resources + prompts），再把服务器接到 agent 上。每个阶段都要落盘产物，让整条链路可追溯、可重跑、可审计。

## 六阶段与责任划分

| # | 阶段 | 子代理 | 技能 | 主要产物 |
|---|---|---|---|---|
| 1 | 定位并下载代码库 | 编排器自己 | `p2a-extract-codebase` | `repo/`、`reports/stage1-codebase.json` |
| 2 | 环境搭建 | environment manager | `p2a-env-setup` | `env/`、`reports/stage2-env.json` |
| 3 | 教程发现 | tutorial scanner | `p2a-tutorial-scan` | `tutorials/index.json` |
| 4 | 教程执行与审计 | tutorial executor | `p2a-tutorial-run` | `tutorials/<id>/`、`runs/`、`reports/stage4-exec.json` |
| 5 | 工具抽取 + 测试精炼 | tool extractor / test verifier | `p2a-tool-extract`、`p2a-tool-verify` | `tools/`、`tests/`、`reports/stage5-*.json` |
| 6 | MCP 服务器组装 | 编排器自己 | `p2a-mcp-assembly` | `<slug>_mcp.py`、`tools_manifest.json` |
| 7 | 接到 agent 上（交付） | 编排器自己 | `p2a-connect-agent` | paper agent 预设 + 冒烟测试 + `reports/stage6-mcp.json` |

**动手前先加载对应技能**：每个阶段的做法、命令模板、判据都在技能里，不要凭记忆临场发挥。

## 固定文件布局（每个文献一个 generation 目录）

```
generations/<slug>/                  # slug = 论文短名-一作-年份，如 alphagenome-avsec-2025
├── paper/                           # 手稿全文、补充材料、图表
├── repo/                            # 克隆的代码库（只读参考，不要改上游代码）
├── data/  external/                 # 示例数据、下载的外部资源
├── env/                             # 隔离环境（.venv 或 conda env）
├── tutorials/
│   ├── index.json                   # 阶段 3：分类后的教程索引
│   └── <tutorial-id>/
│       ├── source/  executed/       # 原始 + 已执行 notebook
│       ├── outputs/ figures/        # 金标准数值与图（阶段 4 的参考真值）
│       └── report.json             # 输入/输出/耗时/隐式假设
├── tools/<slug>_tools.py            # 阶段 5：抽取出的独立函数（MCP 装饰）
├── tests/test_<fn>.py               # 阶段 5：逐函数验证
├── <slug>_mcp.py                    # 阶段 6：组装好的 MCP 服务器
├── tools_manifest.json              # 工具清单（名称/参数/出处/验证状态）
└── reports/stage{1..6}-*.json       # 阶段间契约 + 运行日志
```

## 阶段间 JSON 契约

阶段之间**只**通过文件与 JSON 传递，不靠对话上下文（子代理各自独立，看不到你的上下文）：

- `stage1-codebase.json`：`{ repo_url, clone_path, language, entrypoints[], license, papers_artifacts[] }`
- `stage2-env.json`：`{ env_kind, env_path, python_version, install_commands[], test_config, smoke_test_passed }`
- `tutorials/index.json`：`{ tutorials: [{ id, path, kind, language, uses_example_data, data_refs[], rank, notes }] }`
- `stage4-exec.json` 与每教程 `report.json`：`{ id, status, runtime_s, outputs[], figures[], implicit_assumptions[], blockers[] }`
- `stage5-extract.json` / `stage5-verify.json`：`{ tool, module, source_tutorial, params[], test, attempts, passed, failure_reason }`
- `stage6-mcp.json`：`{ server_file, transport, server_name, tool_count, validated_tool_count, excluded[], smoke_test }`

## 编排纪律

- **阶段 2→4 必须串行**（后一步依赖前一步产物）。阶段 4 内多个教程、阶段 5 内多个函数可以**并行**：用 `workflow` 工具扇出（每个教程/函数一个子代理），或对少量对象用几个 `subagent` 并行调用。
- 子代理要收到**完整独立提示**：给它 generation 目录的绝对路径、当前阶段技能名、上游 JSON 路径、以及它必须产出的 JSON 字段。它看不到本对话。
- 每个子代理返回后，**你亲自校验它的 JSON 与产物是否真的落盘**（文件存在、字段齐全、失败有理由）。子代理说"完成"不等于完成。
- **禁止代码幻觉**：见下节。这是全流水线的最高优先约束。
- 每个阶段结束后用 `todo_write` 更新进度；阶段失败时先尝试修复一次，仍失败则记录 blocker 并按「降级交付」继续，不要静默跳过。

## 验证门禁（阶段 5 → 阶段 6 的唯一入口）

一个工具能进入 MCP 服务器，必须**同时**满足：

1. **期望文件生成**：函数按其契约产出了声明的输出文件；
2. **数值一致**：与阶段 4 的金标准在容差内 —— 浮点 `|a-b|/max(1,|b|) <= 3%`；
3. **图形一致**：与参考图 perceptual hash 的 Hamming distance `< 20`（无参考图时，退化为图存在且非空白且分辨率/轴范围一致）。

未通过的函数：跑满 **最多 6 轮**"生成测试→执行→诊断→修复"；仍失败则**移除 MCP 装饰器**、加 `# VALIDATION FAILED: <原因>` 注释、写入 `excluded[]`，并在最终报告里单独列出。绝不因为"只差一点"而放行。

## 降级交付（不可执行时）

缺数据 / 缺许可证 / 依赖不可获取导致代码库跑不通时，**不要伪造实现**：

1. 在 `stage2-env.json` / `stage4-exec.json` 里如实记录 blocker 与解除条件；
2. 仍然组装 MCP，但只含 resources（手稿、补充材料、数据集、图表索引）与 prompts（论文工作流的可复现步骤说明）；
3. 在 `tools_manifest.json` 与 MCP 描述里标注 `executable: false`，让下游 agent 知道"只能答疑指路，不能执行"；
4. 最终报告写明"部分交付"，以及需要用户提供什么才能升级为完整交付。

## 收尾报告（交付给用户时必须包含）

- 论文标识（标题/DOI/代码库 URL）与 generation 目录路径；
- 产物清单：工具数（通过/被排除）、resources 数、prompts 数、MCP 文件路径；
- 验证通过率与**被排除工具的原因**；
- 未覆盖的论文贡献（哪些方法没有变成工具，为什么）；
- 可执行性状态（完整 / 部分 / 不可执行）与解除阻塞所需条件；
- 下一步：`p2a-connect-agent` 生成的 paper agent 预设怎么启动、怎么冒烟测试；
- 若远端部署：endpoint URL 与基本调用示例。