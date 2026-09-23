---
name: p2a-extract-codebase
description: Paper2Agent 阶段 1：从论文正文/参考文献/补充材料自动定位配套代码库与资源，克隆/下载到 generation 目录，识别语言与入口，产出 stage1-codebase.json，含多候选与私有仓库的处理。
whenToUse: 拿到一篇论文 PDF/DOI/标题，需要找到并取得它的代码库与数据时；或自动识别失败、需要向用户确认真实仓库时。
---

# 阶段 1：定位并下载代码库

产物：`generations/<slug>/repo/`（克隆的仓库）与 `reports/stage1-codebase.json`。

## 步骤

1. **先建 generation 目录**，并把论文本体（PDF/HTML）落到 `paper/`。从 PDF 取文可用 Python `pymupdf`（已在本机可用：`import pymupdf`；`fitz` 为旧名）。
2. **在论文文本中找代码仓库**，按可靠性从高到低：
   - "Code availability" / "Data availability" 章节（Nature 系刊固定有，最可靠）；
   - 正文或图注里明写的 `github.com/...`、`gitlab.com/...`、`zenodo.org/...`、`bitbucket.org/...` 链接；
   - 参考文献中指向软件包的条目（如 `Scanpy`、`TISSUE`，多为 "software" / "package" 类型引用）；
   - 补充材料（Supplementary Information / Supplementary Note）中的下载说明。
   用 `grep` 在抽出的论文文本里搜 `github|gitlab|zenodo|bitbucket|Code availability|Code and data` 最省事。
3. **判定唯一性**：只找到 1 个候选 → 直接使用；0 个或多个候选 → 列出候选（仓库名 + 一句说明 + 从哪找到的）并用 `ask_user_question` 让用户选或直接给 URL。不要自己"猜一个最像的"。
4. **克隆**（不要用 `--depth 1` 如果后续需要 git 历史或 tag）：`git clone <url> repo/`；若网络受限或仓库很大，先 `git clone --filter=blob:none`。同时把仓库的 commit hash 记进 JSON（复现性要求）。
5. **下载关联资源**：补充数据表、示例数据、配置模板、预训练权重说明、Zenodo DOI 附件。放到 `data/`（小）或 `external/`（大/外部引用），**不要塞进 repo 里**以免污染上游代码。
6. **识别语言与入口**：`pyproject.toml`/`setup.py`/`environment.yml`/`requirements*.txt`/`DESCRIPTION`/`package.json` 定语言与依赖；README 与 `docs/` 定官方入口；`examples/`、`tutorials/`、`notebooks/`、`*.ipynb`、`vignettes/` 是阶段 3 的重点（此处只记录路径，不展开分析）。
7. **检查许可证**：记录 license 类型。无许可证或为 restrictive license 时**立即标注并告知用户**——下游是否可再分发/部署取决于此。

## stage1-codebase.json

```json
{
  "slug": "alphagenome-avsec-2025",
  "paper": { "title": "...", "doi": "...", "venue": "Nature", "year": 2025,
             "code_availability_statement": "原文 Code availability 段落的原文摘录" },
  "repo_url": "https://github.com/...",
  "commit": "<40 位 hash>",
  "clone_path": "<绝对路径>/repo",
  "language": "python",
  "entrypoints": ["src/pkg/__init__.py"],
  "tutorial_candidates": ["docs/source/tutorials", "notebooks"],
  "install_files": ["pyproject.toml", "requirements.txt"],
  "license": "Apache-2.0",
  "artifacts_downloaded": [{ "kind": "supplementary_data", "path": "data/xxx.xlsx" }],
  "candidates_considered": [{ "url": "...", "why": "...", "rejected_because": "..." }],
  "blockers": []
}
```

`code_availability_statement` 必须**逐字摘录原文**：它是后续所有出处标注的锚点。

## 常见坑

- 论文里写的仓库可能已迁移或私有：克隆失败时报错原文照抄进 `blockers`，然后请用户给可访问的 URL 或压缩包。
- 同一篇论文可能有多个仓库（主方法 + 数据集 + 训练代码）：主方法优先，其余记进 `candidates_considered`。
- 不要 clone 到工作区根目录：一律落到 `generations/<slug>/repo`。
- 不要修改上游代码。所有适配改动写在 `tools/` 或补丁文件里，并在报告中说明改动内容。