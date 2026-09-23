---
name: p2a-tool-extract
description: Paper2Agent 阶段 5a：把已执行的教程转换成独立、可复用的 MCP 工具（tutorial tool extractor）——识别可泛化的分析步骤、参数化硬编码值、强制文件进文件出、逐个加 MCP 装饰器并注明论文出处，产出 tools/<slug>_tools.py。
whenToUse: 教程已端到端跑通并有金标准输出、需要把它固化成 agent 可调用的工具时；或要把已有的分析脚本改造成 MCP 工具时。
---

# 阶段 5a：工具抽取与实现（tutorial tool extractor 子代理）

**核心纪律：只抽取、不发明。** 每个函数体必须能在阶段 4 的已执行教程里找到对应代码；这不是"照着论文写实现"，是"把跑通的代码整理成参数化函数"。找不到对应代码的双目实现一律不做。

产物：`tools/<slug>_tools.py`（一个模块，多个工具函数）+ 每个函数的出处映射。

## 从教程到工具的映射流程

1. **切分步骤**：把已执行 notebook 的 cell（或脚本段落）按"分析步骤"聚成若干逻辑单元：读取数据 → 质控/过滤 → 降维/聚类 → 差异分析 → 统计分析 → 画图 → 导出。
2. **判定可泛化性**：值得做成工具的是**会用到新数据上的通用步骤**；一次性的探索性代码（"随便看看这个基因"）、纯展示性 cell、调试残留不抽。一个步骤若与示例数据强绑定（如"取第 3 号样本"），先想清楚它在新数据上的等价语义，再决定是否抽取。
3. **参数化**（本阶段最容易漏的一步）：逐行找出硬编码值并提成参数 —— 输入/输出路径、阈值、列名、基因集、组织/细胞类型、模型名、随机种子、并行度。给出**有意义的默认值**（取教程里用的那个值），并在 docstring 里说明依据。
4. **强制文件进、文件出**：函数签名只接受路径与标量参数，返回**写好的文件路径或极小 JSON 摘要**（不要返回巨大 DataFrame / 图像对象 / figure 句柄 —— MCP 的返回值要能跨进程传）。典型签名：
   ```python
   def preprocess_counts(input_h5ad: str, output_h5ad: str, min_genes: int = 200,
                         min_cells: int = 3, n_top_genes: int = 2000, seed: int = 0) -> dict: ...
   ```
5. **加 MCP 装饰器**：每个工具一个 `@mcp.tool()`，`description` 必须写清：做什么、每个关键参数的含义与取值建议、输出文件是什么、以及**出处**（论文 + 代码位置）。描述是 agent 选择工具的唯一依据，含糊的描述会让下游 agent 用错工具。
6. **注明出处**：模块内为每个函数保留一行出处注释，格式 `# Source: <repo_url>/blob/<commit>/<path>:<line> | <tutorial id>`，并可附论文的图/表编号。这是可追溯性的要求。
7. **保持确定性**：固定所有随机种子；字典/集合遍历结果排序后再写出；浮点输出统一格式化（如 `%.6g`），避免下游验证因序列化差异失败。

## 模块骨架（照此结构写）

```python
"""<Paper title> — extracted tools.

Source paper : <title> (<venue> <year>, doi:<doi>)
Source code  : <repo_url> @ <commit>
Extracted from executed tutorials under generations/<slug>/tutorials/.
Every function body is derived from code that actually ran (see Source: lines).
"""
from __future__ import annotations
import json
from pathlib import Path
from mcp.server.fastmcp import FastMCP

mcp = FastMCP("<server-name>")

# ── constants captured from the tutorial (record where each came from) ──
DEFAULT_MIN_GENES = 200  # from tutorials/01-.../: cell 7


@mcp.tool()
def preprocess_counts(input_h5ad: str, output_h5ad: str, min_genes: int = 200,
                      min_cells: int = 3, n_top_genes: int = 2000, seed: int = 0) -> dict:
    """Filter cells/genes and select highly variable genes.

    Args:
        input_h5ad: path to the raw count matrix (.h5ad).
        output_h5ad: path to write the preprocessed matrix.
        min_genes/min_cells: QC thresholds (tutorial defaults: 200 / 3).
        n_top_genes: highly-variable gene count (tutorial default: 2000).
        seed: random seed for reproducibility.
    Returns:
        {"output": <path>, "n_cells": int, "n_genes": int, "n_hvg": int}
    Source: <repo>/blob/<commit>/docs/.../quickstart.ipynb:cell7 | tutorial 01
    """
    # body: copied from the executed tutorial cell, with literals replaced by params
    ...


if __name__ == "__main__":
    mcp.run()
```

## stage5-extract.json（每个函数一条）

```json
{
  "tool": "preprocess_counts",
  "module": "tools/<slug>_tools.py",
  "source_tutorial": "01-quickstart-preprocessing",
  "source_location": "<repo>/blob/<commit>/docs/.../quickstart.ipynb:cell7",
  "params": [{ "name": "min_genes", "default": 200, "from": "tutorial literal", "meaning": "..." }],
  "inputs": ["input_h5ad"], "outputs": ["output_h5ad"],
  "paper_reference": "Fig. 1b / Methods 'Preprocessing'",
  "notes": "教程用 sc.pp.filter_cells 后立刻 filter_genes，保持同样顺序"
}
```

## 常见坑

- **不要一个函数干三件事**：单职责函数才好验证、好组合、好描述。若一段教程有 40 行，通常该拆成 2–4 个工具。
- **不要吞掉异常**：让错误抛出去（返回给调用者），不要 `try: ... except: pass` —— 静默失败会污染科学结论。
- **不要省掉中间产物**：把关键中间结果也写到文件（并可返回路径），否则下游没法审计。
- **不要凭论文文字补实现**：论文里没有代码的部分（尤其新提出的方法），若教程也没演示，就明确不抽成工具，写进 `not_covered[]`。