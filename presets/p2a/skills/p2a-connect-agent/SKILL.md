---
name: p2a-connect-agent
description: Paper2Agent 交付：把验证通过的 MCP 服务器接到 DSH 的 agent preset 上，生成含 MCP 行、paper 专属 persona 与 prompt 模板的 paper agent 预设（stdio 本地或 streamable-http 远端），做挂载校验与端到端冒烟测试，产出交付报告。
whenToUse: MCP 服务器已组装并通过 tools/list 验证、需要交付一个用户真能打开对话使用的 paper agent 时；或用户问"这个 MCP 怎么用/怎么接到 agent 上"时。
---

# 交付：把 MCP 接到 agent 上，成为 paper agent

MCP 只是协议层；用户要的是一个**能对话的专家**。这一步把它接成 DSH 的 agent preset。

## 两种接法（先判断环境）

DSH 的预设是「一份 YAML 组合 + 一堆技能目录」。当用户使用的是本仓库（或本机已装 dsh）时，**优先把 MCP 接到预设里**：

```sh
dsh plugin --profile <profile> add @deepseek-ai/dsh-mcp-client   # 若该 Bundle 尚未安装
```

然后新建（或复制一份）预设目录，加入一行 MCP：

```yaml
- id: mcp-paper-<slug>
  name: '@deepseek-ai/dsh-mcp-client'
  config:
    serverName: <slug>                 # 全局唯一，[A-Za-z0-9_-]{1,32}
    transport: stdio                   # 远端托管时用 streamable-http
    command: python
    args:
      - !!js "process.getBuiltinModule('node:url').fileURLToPath(new URL('tools/<slug>/<slug>_mcp.py', baseUrl))"
    env:
      <SLUG>_API_KEY: !!js "process.env.<SLUG>_API_KEY ?? ''"
    toolCallTimeoutMs: 120000          # 科学计算常超过默认 60s
    failOnStartupError: false          # 服务器暂时不可用时预设仍可挂载
```

远端（Hugging Face Spaces 等）改成：

```yaml
    transport: streamable-http
    url: https://<owner>-<space>.hf.space/mcp
    headers: {}
```

`!!js ... new URL('...', baseUrl)` 是预设里的惯用法：路径相对于预设目录解析，换机器不用改绝对路径。

## 预设里还要写什么

1. **persona**：把这个 paper 的身份写进去 —— 论文说了什么、它的方法适合回答什么问题、什么时候该用哪个工具、结果的正确解读方式与常见误用。**这是 paper agent 与裸 MCP 的差别所在**。
2. **一段"工作流优先"的指引**：让 agent 优先走 MCP prompt 模板（论文里的正确顺序），而不是自己拼工具顺序；并强调"先 inspect 数据，只有在沿用默认值会导致错误结果时才偏离，并说明理由"。
3. **可信度边界**：写明哪些贡献**没有**被工具覆盖（来自 `not_covered[]`）、哪些工具被排除，避免 agent 越界编答案。
4. **可选**：把该论文的技能（若有）一起放进预设的 `skills/`，并加 `customSkillDirs` 让它们可被发现。

## 挂载校验与冒烟测试（必须做）

1. **预设能挂载**：用 roster 的 `standingKeyFor('<preset-id>')` 做真实组合校验 —— 它会拒绝包名解析失败、配置缺字段、以及发布到根 realm 的服务。挂载成功才算配置正确。
2. **工具真的出现在目录里**：新开一个该预设的会话，检查工具列表里出现 `mcp__<serverName>__*`。名称前缀错了说明 `serverName` 配错。
3. **端到端冒烟测试**：用论文自带示例数据跑一个最有代表性的问题（例如"用 <工具> 处理 <示例数据>，报告 <关键数值>"），确认：
   - agent 选对了工具；
   - 输出的数值与阶段 4 的金标准一致；
   - 生成的图表文件真实存在。
4. **失败排查顺序**：服务器起不来（先手跑 `<slug>_mcp.py`）→ 工具没注册（`tools/list`）→ 工具名对但调用失败（路径/依赖/凭证）→ agent 选错工具（改工具 description，它才是选择依据）。

## 交付报告（给用户看的内容）

- **这是什么**：`<论文标题>`（doi:...）被转成的 paper agent；它能回答哪类问题。
- **怎么用**：预设名、怎么启动、示例提问（2–3 条，用论文的真实问题措辞）。
- **工具清单**：通过验证的工具名 + 一句话用途（从 `tools_manifest.json` 取）。
- **能力边界**：被排除的工具及原因、未覆盖的论文贡献、`executable: false` 时的降级说明。
- **可复现信息**：代码库 URL + commit、MCP 服务器文件路径、manifest 路径、凭证从哪个环境变量读。
- **远端部署时的**：endpoint URL 与一条等价调用示例。

## 常见坑

- **一次接多个 MCP**：MCP 是模块化的，多个 MCP 可以接到同一个 chat agent 上（这正是"多篇论文协作"的实现方式：优先给因果基因时同时挂 genomics 与 single-cell 的 MCP）。只要 `serverName` 各不相同。
- **不要把 serverName 取成通用词**（如 `tools`、`mcp`、`paper`）：会与其它预设的 MCP 冲突，且工具前缀难看懂。
- **不要忘记超时**：科学计算（比对、模型推理、大规模矩阵）默认 60s 经常不够，`toolCallTimeoutMs` 给到 120000 起。
- **不要把凭证写进预设文件**：用 `!!js "process.env.X ?? ''"` 读环境变量；预设可能进入版本控制。
- **交付前自己先跑一遍**：以会话方式真实调用一次代表性工具。你没跑过就不要说"可用"。