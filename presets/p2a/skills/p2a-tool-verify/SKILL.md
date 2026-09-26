---
name: p2a-tool-verify
description: Paper2Agent 阶段 5b：为每个抽取的工具用教程自带示例数据生成并执行测试（test verifier–improver）——校验输出文件生成、数值在 3% 容差内、图形 perceptual hash 距离 < 20，最多 6 轮迭代修复，反复失败者剔除 MCP 装饰器并记录原因。
whenToUse: 工具刚由 p2a-tool-extract 抽出来后；或需要判断某个工具能否进入最终 MCP 服务器（验证门禁）时。
---

# 阶段 5b：测试、验证与精炼（test verifier–improver 子代理）

这是把"LLM 生成的代码"变成"可信科学工具"的那一步。判据来自 Paper2Agent：**期望文件生成 + 数值在容差内 + 图形一致**。

产物：`tests/test_<fn>.py`、`reports/stage5-verify.json`、每个工具的通过/失败结论。

## 通过判据（三条同时满足才算 pass）

| 检查 | 判据 | 实现要点 |
|---|---|---|
| 输出文件 | 契约声明的每个文件都存在且非空 | 严格按函数的 `Returns` 校验，不要额外放宽 |
| 数值一致 | 与阶段 4 金标准比对：`abs(a-b)/max(1.0, abs(b)) <= 0.03` | 浮点 3% 容差；计数类（细胞数、基因数）**要求完全相等**，不适用 3%；表格按行集合比对，允许行序不同 |
| 图形一致 | 与参考图 perceptual hash 的 Hamming distance `< 20` | 见下方 `image_phash`，只依赖 Pillow；无参考图时退化为"图存在、非空白、分辨率/轴范围一致" |

容差**可以更严但不能更松**：计数与字符串类结果必须精确匹配；只有浮点是 3%。放宽容差必须在报告里写明理由并经编排器复核。

## 测试怎么生成（用教程自己的示例数据，不另造数据）

```python
# tests/test_preprocess_counts.py
import json, subprocess, sys
from pathlib import Path

GEN = Path(__file__).resolve().parents[1]
PY  = GEN / "env/bin/python"                      # Windows: env/Scripts/python.exe
GOLD = GEN / "tutorials/01-quickstart-preprocessing/outputs"
OUT = GEN / "tests/_out"; OUT.mkdir(exist_ok=True, parents=True)

def test_preprocess_counts_matches_gold():
    result = subprocess.run(
        [str(PY), str(GEN / "tools/<slug>_tools.py"), "--selftest", "preprocess_counts",
         "--input", str(GEN / "data/pbmc3k_raw.h5ad"), "--output", str(OUT / "processed.h5ad")],
        capture_output=True, text=True)
    assert result.returncode == 0, result.stderr
    summary = json.loads(result.stdout.strip().splitlines()[-1])

    # 1) 期望文件生成
    assert (OUT / "processed.h5ad").exists() and (OUT / "processed.h5ad").stat().st_size > 0

    # 2) 数值在容差内（计数类要求相等）
    gold = json.loads((GOLD / "summary.json").read_text())
    assert summary["n_cells"] == gold["n_cells"]          # 计数：必须相等
    for key in ("n_hvg_pct",):                            # 浮点：3% 容差
        a, b = float(summary[key]), float(gold[key])
        assert abs(a - b) / max(1.0, abs(b)) <= 0.03, (key, a, b)

    # 3) 图形 perceptual hash 距离 < 20
    assert phash_distance(OUT / "umap.png", GOLD / "figures/umap.png") < 20
```

给工具模块加一个**自测入口**（`--selftest <fn> --input ... --output ...`，打印一行 JSON 摘要）会让验证简单很多：在 `tools/<slug>_tools.py` 里用 `argparse` 实现即可，不要为测试另外复制一份实现。

## `image_phash`（只依赖 Pillow，aHash 64 bit）

```python
from PIL import Image

def image_phash(path, size=8):
    """64-bit average hash. Grayscale, resize to size x size, threshold at mean."""
    img = Image.open(path).convert("L").resize((size, size), Image.LANCZOS)
    px = list(img.getdata())
    mean = sum(px) / len(px)
    bits = 0
    for i, value in enumerate(px):
        if value > mean:
            bits |= (1 << i)
    return bits

def phash_distance(a, b):
    return bin(image_phash(a) ^ image_phash(b)).count("1")
```

`< 20`（64 位里最多 20 位不同）是"同一张图的不同渲染"通常能落在的范围（字体、dpi、抗锯齿差异）。布局不同、颜色映射不同、坐标轴范围不同会明显超过 20 —— 那说明**图确实不一样**，不要靠放宽阈值掩盖。若因环境差异导致真实的渲染不一致（例如色图版本变化），在修复渲染（固定 matplotlib/colormap 版本）后重试，而不是改判据。

## 迭代循环（最多 6 轮）

```
for attempt in 1..6:
    生成/运行测试 → 失败则诊断根因 → 修复（改工具实现 / 改参数化 / 改环境固定版本）
    若通过 → 记录 attempts 并结束
```

**诊断要落到根因**，常见根因与正确处理：

| 现象 | 根因 | 正确修复 |
|---|---|---|
| 数值差一点点（<10%） | 随机种子/浮点顺序/库版本 | 固定种子、统一聚合顺序、固定版本；**不要**放宽容差不改代码 |
| 计数类不一致 | 过滤条件或顺序与教程不同 | 对照已执行教程逐行比对，恢复教程的顺序与阈值 |
| 图差异大 | 布局/配色/dpi 不同 | 固定 `rcParams`（figure size、dpi、colormap、字体）后重试 |
| 缺输出文件 | 函数没走到写出那一步（早退/异常被吞） | 去掉 `except: pass`，让错误显式抛出 |
| 每次结果都不同 | 未固定的随机性或并行归约 | 固定全部种子与 `OMP_NUM_THREADS`，串行归约 |

**6 轮仍失败**：从工具上**移除 `@mcp.tool()` 装饰器**，在其上方加注释 `# VALIDATION FAILED: <一句话原因>`，保留函数（便于人工排查），并记入 `excluded[]`。**绝不**放行未验证的工具——放行一个算错的工具比少一个工具严重得多。

## stage5-verify.json

```json
{
  "results": [
    { "tool": "preprocess_counts", "test": "tests/test_preprocess_counts.py",
      "attempts": 2, "passed": true,
      "checks": { "files": true, "numeric": {"n_hvg_pct": {"ours": 14.2, "gold": 14.19, "rel_err": 0.0007}},
                  "figure": { "phash_distance": 6 } },
      "fixes": ["固定 random_state=0（原实现未设，导致 n_hvg 漂移）"] }
  ],
  "excluded": [ { "tool": "fit_tissue_model", "attempts": 6, "reason": "需要 GPU，CPU 降级后 3 天未收敛" } ],
  "not_covered": ["论文报告的空间不确定度校准曲线：教程未演示该步骤，未抽成工具"]
}
```

## 常见坑

- 测试必须真跑：`pytest -q` 的输出贴进报告，别写"应当通过"。
- 不要用**新造的数据**做验证：判据是"复现教程结果"，只能用教程自带示例数据。
- 不要为了让测试过而改测试期望值 —— 期望值来自阶段 4 的金标准，改它等于改基准。
- 一个工具超过 6 轮就别再纠缠：记录并排除，把时间花在其它工具上（编排器可并行推进）。