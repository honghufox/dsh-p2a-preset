---
name: p2a-tutorial-run
description: Paper2Agent 阶段 4：端到端执行入选教程，生成金标准输出（数值/表格/图）作为后续工具验证的参考真值，记录运行时间、约束与所有被显式化的隐式假设，产出每教程 report.json。
whenToUse: 教程索引已就绪、需要产出"参考真值"再抽工具时；或教程执行报错需要排查（缺数据、API 变更、内存不足）时。
---

# 阶段 4：教程执行与审计（tutorial executor 子代理）

这一阶段产出的是**金标准（gold standard）**：后续每个工具是否"忠实复现论文结果"，全靠这里的输出做基准。因此**输出必须来自真实执行**，不允许手工填数、不允许"跳过报错的那段、凭理解补结果"。

产物：`tutorials/<id>/executed/`、`tutorials/<id>/outputs/`、`tutorials/<id>/figures/`、`tutorials/<id>/report.json`、`reports/stage4-exec.json`。

## 步骤

1. **准备输入**：按 `index.json` 的 `data_refs` 放好示例数据。需要下载的先下载并**校验**（文件大小、行数、checksum）；缺数据且拿不到公开替代 → 该教程标 `blocked`，不要伪造。
2. **转换 notebook 为可执行形式**：优先用 `jupyter nbconvert --to notebook --execute --inplace`（或 `papermill`）。把原始 notebook 备份到 `source/`，执行产物放 `executed/`——**永远保留原始版**，它是"出处"的锚点。
   ```bash
   <gen>/env/bin/python -m jupyter nbconvert --to notebook --execute \
     --ExecutePreprocessor.timeout=1800 \
     --output-dir tutorials/<id>/executed tutorials/<id>/source/<name>.ipynb
   ```
3. **脚本类教程**：在 `executed/` 下以**最小改动**跑通（不要重写逻辑）。所有改动集中记录在 `report.json` 的 `deviations[]`（原行 → 改后 → 为什么必须改）。
4. **解决执行错误（按此顺序）**：
   - 路径/工作目录问题 → 统一改成绝对路径或从 generation 目录运行；
   - 缺包 → 记录后回到阶段 2 补装**并更新 lock**（不要把 `pip install` 写进教程代码里）；
   - API 变更（旧教程 + 新库）→ **优先降级库版本**（回到论文时代的版本）而不是改教程；确实无法降级时改动代码，并在 `deviations[]` 写明；
   - 资源不足（内存/显存/时间）→ 缩小示例规模**并把缩放在 `deviations[]` 中显式记录**（缩放后的输出只能作为"按比例"参考，验证时容差要放宽并说明）；
   - 网络依赖 → 预取一次，之后离线可跑。
5. **完整保存输出**（金标准的全部内容）：
   - 数值/表格 → `outputs/*.csv|tsv|json`（**确定性格式**：排序、浮点位数统一，便于比对）；
   - 图 → `figures/*.png`（强制 `MPLBACKEND=Agg`，固定 `dpi`、尺寸与随机种子）；
   - 日志 → `outputs/run.log`（含完整命令、耗时、内存峰值）。
6. **记录运行元数据与隐式假设**：命令、耗时、机器（CPU/GPU/内存）、随机种子、被跳过的 cell（必须写清为什么跳过——跳过而不记录等于作弊）、每个原本隐式的假设（如"必须先 `sc.tl.pca` 才能 `sc.tl.leiden`"）。
7. **交叉核对**：把执行的数值/图与**论文正文或补充材料里报告的数字**对照一次（如"我们得到 2700 个高变基因，原文报告 2,638"）。数量级一致即可，明显不一致要在 `notes` 里标注（可能用了不同的示例数据）。这一步能早期发现"教程跑通了但跑错了"。

## tutorials/<id>/report.json

```json
{
  "id": "01-quickstart-preprocessing",
  "status": "ok",
  "command": "<完整可复现的命令>",
  "source_path": "tutorials/01-.../source/quickstart.ipynb",
  "executed_path": "tutorials/01-.../executed/quickstart.ipynb",
  "runtime_s": 214,
  "machine": { "cpu": "...", "ram_gb": 8, "gpu": null },
  "seed": 0,
  "outputs": [{ "path": "outputs/clusters.csv", "kind": "table", "rows": 2638 }],
  "figures": [{ "path": "figures/umap.png", "dpi": 150, "size": [1200, 900] }],
  "cells_skipped": [],
  "deviations": [{ "from": "adata = sc.read('../data/x.h5ad')", "to": "sc.read('/abs/path/x.h5ad')", "why": "工作目录不同" }],
  "implicit_assumptions": ["必须先 pca 再 neighbors 再 umap", "leiden 需要 igraph 已安装"],
  "paper_cross_check": { "ours": 2638, "paper_reported": 2638, "note": "与原文 Methods 一致" },
  "blockers": []
}
```

## 常见坑

- **不许编造输出**：`status: ok` 必须对应真实执行的 `run.log`。查不到日志的产物按未验证处理。
- **随机性要固定**：`numpy.random.seed`、`random_state`、`torch.manual_seed`、scanpy `random_state`。否则阶段 5 的验证会随机失败。
- **图不可复现多半是环境差异**：字体缺失、matplotlib 版本、dpi 不一致。固定版本 + 固定 `rcParams`；验证阶段对图只做近似比对（见 p2a-tool-verify）。
- 执行慢的教程放到后台 job（`run_in_background: true`）或 LSF 计算节点，不要阻塞在登录节点上跑重活。