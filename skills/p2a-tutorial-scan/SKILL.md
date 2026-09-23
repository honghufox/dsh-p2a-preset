---
name: p2a-tutorial-scan
description: Paper2Agent 阶段 3：扫描代码库中的教程/示例/文档，区分真正的可复用教程与普通源码，按可执行性、是否自带示例数据、与论文核心贡献的相关度排序，产出 tutorials/index.json。
whenToUse: 环境搭好后，需要决定"先执行哪几个教程、哪些教程值得抽成工具"时；或在大型仓库里找不到可运行示例时。
---

# 阶段 3：教程发现与分类（tutorial scanner 子代理）

产物：`tutorials/index.json` —— 一份分类后的候选教程索引。

## 步骤

1. **枚举候选位置**（按价值排序）：
   - `docs/source/tutorials/`、`docs/tutorials/`、`tutorials/`、`examples/`、`vignettes/`（R）、`gallery/`；
   - `notebooks/`、`*.ipynb`（含 `docs/**/*.ipynb`）；
   - `scripts/`、`demo/`、`case_studies/`、`workflows/` 下带示例数据的脚本；
   - README 里 fenced code block 形式的 "Quick start"（无示例数据的通常是伪代码）；
   - `tests/` 与 `*.t`/`testthat/`（是测试，不是教程，但能揭示预期输出与容差）。
   先 `ls -R` 或 glob 摸清规模，再用少量 `read` 抽样判断，不要逐个文件读完（大仓库读不完）。
2. **逐个候选判定四件事**（这是本阶段的核心）：
   - `kind`：`notebook` | `script` | `markdown-doc` | `r-vignette` | `test` | `pseudocode`；
   - `uses_example_data`：**是否自带可获得的输入数据**（仓库内文件、Zenodo/Figshare 链接、`scanpy.datasets` 之类内置数据集、或明示的下载 URL）。没有数据的教程**无法产出金标准输出**，价值大打折扣；
   - `covers_paper_contribution`：它演示的方法是否是论文的**核心贡献**（对照摘要与 Methods 里的关键方法名）；
   - `runnable`：粗判能否在阶段 2 的环境里跑（依赖是否已装、是否需要 GPU/大内存/网络、是否有交互式输入）。
3. **排序与筛选**：按 `uses_example_data && covers_paper_contribution && runnable` 排最前。目标不是收全，而是**选出覆盖面最广、最可能跑通的一小组**（典型 3–8 个）。若仓库只有零星示例，就把它们全选上。
4. **记录隐式前提**（重要）：教程里常隐含"先跑上一个 cell""数据放在 `../data`""必须先设 API key"。这些在阶段 4 会被显式化，但**现在就要把可疑处记进 `notes`**，让执行者心里有数。
5. **过滤噪音**：`pseudocode`（README 里不能跑的片段）、`test`（仅作参考）、纯 API 参考文档、CHANGELOG、发布说明不要进 `tutorials` 主列表，可放 `excluded[]` 并写明理由。

## tutorials/index.json

```json
{
  "repo_commit": "<与 stage1 一致>",
  "scan_summary": { "files_scanned": 812, "candidates": 14, "selected": 5 },
  "tutorials": [
    {
      "id": "01-quickstart-preprocessing",
      "path": "docs/source/tutorials/quickstart.ipynb",
      "kind": "notebook",
      "language": "python",
      "uses_example_data": true,
      "data_refs": ["docs/source/tutorials/data/pbmc3k.h5ad"],
      "covers_paper_contribution": "对应论文 Fig.2 的预处理与聚类流程",
      "runnable": true,
      "runtime_estimate": "3-5 min, CPU only",
      "implicit_assumptions": ["必须先设 scanpy.settings.datasetdir", "需要网络下载示例数据"],
      "rank": 1,
      "notes": "..."
    }
  ],
  "excluded": [{ "path": "tests/test_core.py", "kind": "test", "why": "是单测不是教程，仅作容差参考" }]
}
```

## 快速判定启发式

- 有 `.ipynb` 且**输出单元已保存**（`outputs` 非空）→ 极有价值：原文作者已跑通，且输出可作为交叉参照。
- `.py` 脚本自带 `if __name__ == "__main__"` 与示例数据路径 → 好候选。
- Markdown 教程带 ```` ```bash ```` + 截图 → 中等，需要人工补输入数据。
- 引用外部私有数据（"our in-house cohort"）→ 标 `runnable: false`，除非能拿到公开替代数据。

## 常见坑

- **别把测试当教程**：`test_*.py` 的断言是"能通过"，不是"产出论文结果"；但它可以告诉你数值容差的合理量级。
- **别把 API 文档当教程**：`docs/api/*.rst` 只有签名，抽不出可复用工作流。
- 教程与代码版本可能不匹配（教程是旧 API）：标 `notes`，把适配留给阶段 4 并在那里显式记录。