# dsh-p2a-preset — 文献转 Agent 模式（Paper2Agent）

一个 [DeepSeek Harness](https://github.com/deepseek-ai)（DSH）agent 预设：把一篇研究论文连同它的代码库、补充材料与数据集，转换成**可交互、可复现、可调用**的 paper agent —— 相当于这篇文献的「虚拟通讯作者」。

实现依据：Miao et al., *Reimagining research papers as interactive and reliable AI agents*, **Nature** (2026), doi:[10.1038/s41586-026-11044-y](https://doi.org/10.1038/s41586-026-11044-y)（Paper2Agent）。

## 这个预设做什么

论文是被动的：读者要自己找到代码库、装环境、搞懂 API 和参数语义，才能把方法用到自己的数据上。Paper2Agent 的做法是把论文的贡献封装成一个 **MCP 服务器**（tools + resources + prompts），再把它接到 agent 上，让用户用自然语言复现论文结果、把方法迁移到新数据、并让多个论文 agent 协同工作。

本预设把这条流水线固化成 8 个技能，并内置两条硬约束：

- **禁止代码幻觉**：所有工具必须从**实际执行过的教程代码**中抽取，绝不凭空编写论文方法的实现。
- **验证门禁**：每个工具都要用教程自带示例数据验证 —— 期望文件生成、数值在 **3%** 容差内、图形 perceptual hash 距离 **< 20**；反复失败（最多 6 轮）的工具移除 MCP 装饰器并记录原因，绝不放行。

## 六阶段流水线

| # | 阶段 | 子代理角色 | 技能 |
|---|---|---|---|
| 1 | 定位并下载代码库 | 编排器 | `p2a-extract-codebase` |
| 2 | 环境搭建 | environment manager | `p2a-env-setup` |
| 3 | 教程发现与分类 | tutorial scanner | `p2a-tutorial-scan` |
| 4 | 教程执行与审计（产出金标准） | tutorial executor | `p2a-tutorial-run` |
| 5a | 工具抽取与实现 | tutorial tool extractor | `p2a-tool-extract` |
| 5b | 测试、验证与精炼 | test verifier–improver | `p2a-tool-verify` |
| 6 | MCP 服务器组装 | 编排器 | `p2a-mcp-assembly` |
| — | 接到 agent 上（交付） | 编排器 | `p2a-connect-agent` |

阶段之间只通过文件与 JSON 报告传递数据（`reports/stage1..6-*.json`），每个文献一个 generation 目录，全部产物落盘，保证链路可追溯、可重跑、可审计。`p2a-pipeline` 是总纲。

每个阶段的技能都包含：命令模板、JSON 契约、判据、以及常见坑（依赖冲突、旧教程 + 新库的 API 变更、随机性未固定、图不可复现、GPU 降级等）。

## 目录结构

```
agent.cordis.yml          # 预设组合（persona + 工具行 + 技能目录）
preset.yml                # 显示名与描述
skills/
  p2a-pipeline/           # 六阶段总纲、文件布局、阶段间契约、验证门禁
  p2a-extract-codebase/   # 阶段 1
  p2a-env-setup/          # 阶段 2
  p2a-tutorial-scan/      # 阶段 3
  p2a-tutorial-run/       # 阶段 4
  p2a-tool-extract/       # 阶段 5a
  p2a-tool-verify/        # 阶段 5b
  p2a-mcp-assembly/       # 阶段 6
  p2a-connect-agent/      # 交付：接成 paper agent 预设
```

## 安装

预设是「一个目录 + 一份 composition」。把它放到 DSH 的**用户预设根目录**：

```
${DSH_HOME:-$HOME/.dsh}/.agent-presets/p2a/
```

```bash
# 从本仓库克隆到用户预设根目录（目录名即预设 id，必须是 p2a）
git clone https://github.com/honghufox/dsh-p2a-preset.git \
  "${DSH_HOME:-$HOME/.dsh}/.agent-presets/p2a"
```

Windows（PowerShell）：

```powershell
git clone https://github.com/honghufox/dsh-p2a-preset.git "$env:USERPROFILE\.dsh\.agent-presets\p2a"
```

重启（或新开）一个 DSH 会话，在预设选择里选 **文献转Agent模式** 即可。

> 目录名决定预设 id。放成 `dsh-p2a-preset` 也能用，但预设 id 就会是 `dsh-p2a-preset`，persona 与技能中的 `p2a-*` 名字不变、仍然有效。

## 用法

新开一个该预设的会话，然后把文献交给它，例如：

```
把这篇论文转成 agent：<PDF 路径或 DOI>
```

它会依次：定位代码库 → 建隔离环境 → 找教程 → 执行并保存金标准输出 → 抽取并验证工具 → 组装 MCP → 用 `p2a-connect-agent` 交付一个可挂载的 paper agent 预设。

也可以只调用其中一段，例如「只做阶段 1-4」「这个工具为什么验证没过」「把已有的 MCP 接到一个 paper agent 预设上」。

## 依赖

- 预设本身只依赖 DSH 的基础插件（标准模式的全部能力：shell、文件、搜索、技能、子代理、工作流、后台作业、目标、计划模式、压缩）。
- 流水线在运行时用到：`git`（克隆）、`python`（环境与 MCP 服务器）、以及 `pymupdf`（从 PDF 抽文）与 `Pillow`（图形 perceptual hash）。建议在目标机器上确认：

```bash
python -c "import mcp, PIL, pymupdf; print('ok')"
```

- 目标论文的代码库需要什么（GPU、系统库、大型数据库）取决于论文本身；技能里对 GPU 缺失、依赖冲突、无法执行的降级路径都有明确要求。

## 与其它预设的关系

本预设由 DSH 随部署发布的 `standard` 预设复制而来（**不修改**任何随部署发布的预设），保留其全部编码与子代理能力，只增加了 paper 转换的 persona 与 8 个流水线技能。它可以与本机其它预设（如生物科研、LSF 集群生信分析）并存，互不冲突。

## 可复现性说明

- `p2a-tool-verify` 的三条判据中的 `aHash` 实现只用 Pillow，不引入额外依赖。
- 每个工具在 MCP 描述与模块注释中都保留论文与代码出处（repo URL + commit + 文件:行 + 论文图表编号）。
- 代码库无法执行时（缺数据、缺许可证、依赖不可获取），预设要求**降级交付**而不是伪造实现：MCP 仍提供 resources 与 prompts，并标注 `executable: false`。

## 许可

预设的组合与技能文本以 MIT 许可提供（见 `LICENSE`）。所转换论文及其代码库的许可由各自原作者决定，请遵守其条款；预设不会重新分发论文代码库。