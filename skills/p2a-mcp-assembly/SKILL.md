---
name: p2a-mcp-assembly
description: Paper2Agent 阶段 6：把验证通过的工具模块组装成一个完整 MCP 服务器（tools + resources + prompts + 清单 + 版本 + 安全默认值），本地 stdio 冒烟测试并用 MCP inspector 列工具，可选部署到 Hugging Face Spaces 远端托管。
whenToUse: 工具已通过阶段 5 验证、需要产出可挂载的 MCP 服务器文件与清单时；或 MCP 服务器启动失败、工具没出现在工具列表里需要排查时。
---

# 阶段 6：MCP 服务器组装

产物：`generations/<slug>/<slug>_mcp.py`（可直接运行的 MCP 服务器）、`tools_manifest.json`、`reports/stage6-mcp.json`。

## MCP 服务器的三个组成部分

Paper2Agent 把每个 MCP 明确定义为三块，三者缺一不可 —— 只给 tools 的服务器会让下游 agent 变成"只会算、不懂论文"的工具箱：

1. **tools**：阶段 5 验证通过的可执行函数（论文的方法学贡献）；
2. **resources**：静态资产 —— 手稿文本、代码库快照/链接、补充材料、数据集说明、图表索引；
3. **prompts**：把工具按**论文里的正确顺序**编排好的多步工作流模板，用来降低下游 agent 的提示负担、保证分析可复现。

## 组装步骤

1. **合并工具模块**：把 `tools/*.py` 里带 `@mcp.tool()` 的函数收进统一的 `<slug>_mcp.py`。失败/被排除的函数**不要**进服务器（阶段 5 已移除它们的装饰器，此处再确认一次）。
2. **补 resources**：至少提供手稿与代码库引用；补充材料、数据字典、图表清单有多少给多少。
   ```python
   @mcp.resource("paper://manuscript")
   def manuscript() -> str:
       """Full manuscript text extracted from the published PDF."""
       return (GEN / "paper/manuscript.md").read_text(encoding="utf-8")

   @mcp.resource("paper://code-repo")
   def code_repo() -> str:
       """Source repository and the exact commit the tools were extracted from."""
       return json.dumps({"url": REPO_URL, "commit": COMMIT,
                          "license": LICENSE, "tutorials": SELECTED_TUTORIALS}, indent=2)

   @mcp.resource("paper://figures-index")
   def figures_index() -> str:
       """Index of paper figures/tables with the tutorial that regenerates each."""
       return (GEN / "tools_manifest.json").read_text(encoding="utf-8")
   ```
3. **补 prompts**：每个"论文里的完整科学任务"一个 prompt 模板，把工具串成有序步骤，并写明**先 inspect 数据、只有在沿用默认值会导致错误结果时才偏离默认**。
   ```python
   @mcp.prompt()
   def reproduce_main_analysis(input_h5ad: str) -> str:
       """Replicate the paper's main analysis pipeline on new data.

       Steps mirror tutorials/01..03 in the order the authors used them.
       """
       return f"""Analyse {input_h5ad} by reproducing the paper's workflow.
       1. preprocess_counts(file_path='{input_h5ad}')  # QC + HVG; inspect n_cells first
       2. compute_embedding(...)                       # keep defaults unless data is sparse
       3. cluster_cells(...)                           # then annotate
       4. summarize_markers(...)
       Always inspect the data before changing a default; if you deviate, say why and
       record the reason. Report results with the tool outputs as evidence."""
   ```
4. **清单与版本**：`tools_manifest.json` 是下游 agent 与审计者的权威列表 —— 工具名、参数、出处、验证状态、是否可执行都要在里面。服务器里加 `SERVER_VERSION` 常量（与论文 DOI、代码 commit 一起）。
5. **安全默认值**：
   - 文件操作限定在显式传入的路径，**不要**实现"删除/覆盖任意路径"的工具；
   - 不把 API key / 凭证写进代码：从环境变量读（`os.environ.get("XXX_API_KEY")`），缺失时抛出**清晰可读**的错误说明怎么配；
   - 不执行 shell 字符串拼接；不做任意 URL 抓取；
   - 长任务的超时与内存上限写成可配置参数并在 docstring 里说明。
6. **本地冒烟测试（必做）**：
   ```bash
   # 服务器能起来
   timeout 20 python generations/<slug>/<slug>_mcp.py   # 阻塞式 stdio 服务器，能被超时终止即说明启动成功
   # 工具/资源/prompt 都能被列举（MCP inspector，或 SDK 客户端）
   npx -y @modelcontextprotocol/inspector --cli python generations/<slug>/<slug>_mcp.py --method tools/list
   ```
   把真实的 `tools/list` 输出（工具名列表）贴进 `stage6-mcp.json`。工具数为 0 或少于清单里的通过数 → 不要交付，先修。
7. **（可选）远端部署**：Hugging Face Spaces 等平台托管可免去本地依赖问题（论文即如此做的）。部署后记录 endpoint URL，并确认它支持 Streamable HTTP transport。

## tools_manifest.json

```json
{
  "server_name": "<slug>",
  "server_version": "1.0.0",
  "paper": { "title": "...", "doi": "...", "venue": "...", "year": 2025 },
  "repo": { "url": "...", "commit": "...", "license": "..." },
  "executable": true,
  "tools": [
    { "name": "preprocess_counts", "module": "tools/<slug>_tools.py",
      "source": "<repo>/blob/<commit>/docs/.../quickstart.ipynb:cell7",
      "params": ["input_h5ad", "output_h5ad", "min_genes"],
      "validated": true, "attempts": 2, "validation": "files+numeric(3%)+phash<20" }
  ],
  "excluded": [{ "name": "fit_tissue_model", "reason": "需要 GPU，CPU 未收敛" }],
  "resources": ["paper://manuscript", "paper://code-repo", "paper://figures-index"],
  "prompts": ["reproduce_main_analysis"],
  "not_covered": ["论文 Fig.4 的校准曲线：教程未演示，未抽成工具"]
}
```

## stage6-mcp.json

```json
{
  "server_file": "generations/<slug>/<slug>_mcp.py",
  "transport": "stdio",
  "server_name": "<slug>",
  "tool_count": 18,
  "validated_tool_count": 18,
  "excluded": ["fit_tissue_model"],
  "resources": 3,
  "prompts": 4,
  "smoke_test": { "startup": "ok", "tools_list_first_10": ["preprocess_counts", "compute_embedding", "..."] },
  "deployment": { "kind": "local", "url": null }
}
```

## 常见坑

- **serverName 冲突**：DSH 侧 `@deepseek-ai/dsh-mcp-client` 的 `serverName` 必须全局唯一（`[A-Za-z0-9_-]{1,32}`），工具公开名为 `mcp__<serverName>__<rawName>`。组装时就把 serverName 定成论文短名（如 `alphagenome`），与预置里其它 MCP（biotools / pubmed / ncbi / zotero）区分开。
- **服务器起来了但工具没注册**：装饰器没生效（缺 `FastMCP` 实例、函数不在导入路径上、`if __name__ == "__main__"` 里 `mcp.run()` 没调用）。用 `tools/list` 验证，不要靠"应该没问题"。
- **不许把失败工具放进来充数**：`tool_count` 与 `validated_tool_count` 不一致时，必须解释差额。
- **resources 指向的路径要真的存在**：交付前逐个读一次，路径写死成相对 `generations/<slug>/` 的相对路径更稳（用 `Path(__file__).parent` 解析）。