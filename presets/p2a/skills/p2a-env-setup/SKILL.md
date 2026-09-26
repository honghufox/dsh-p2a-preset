---
name: p2a-env-setup
description: Paper2Agent 阶段 2：为代码库建立干净、隔离、可复现的环境（venv/conda/mamba），装全依赖、跑通冒烟测试、产出测试配置与 stage2-env.json，处理依赖冲突、CUDA/GPU、系统库与重型科学依赖。
whenToUse: 克隆好代码库后、准备执行教程之前；或依赖安装失败、版本冲突、import 报错需要修复环境时。
---

# 阶段 2：环境搭建（environment manager 子代理）

目标：**干净、隔离、可复现**的环境 —— 让这个仓库在环境里跑起来，且明天、在另一台机器上跑出同样结果。
产物：`env/`（隔离环境）+ `reports/stage2-env.json`。

## 步骤

1. **探测仓库要求**（不要凭空猜依赖）：
   - 依赖清单：`pyproject.toml`（含 `[project.optional-dependencies]` 与 extras）、`setup.py`、`setup.cfg`、`requirements*.txt`、`environment.yml`、`conda-lock.yml`、`poetry.lock`、`Pipfile`、`DESCRIPTION`（R）、`renv.lock`、`Dockerfile`（描述系统依赖）、`.python-version`；
   - README 的 Installation 段、`docs/install*`、CI 配置（`.github/workflows/*` 往往写明了能跑通的依赖组合——**优先照搬 CI 的安装命令**）。
2. **创建隔离环境**：
   - Python：`python -m venv env` 或 `mamba create -p env python=3.11 -y`；
   - **优先装仓库自己声明的依赖**：`pip install -e .`（可编辑安装），有 extras 时按教程需要加，如 `pip install -e ".[tutorials]"`；
   - **绝对不要**往系统 Python 或其它 preset 的环境里装东西；统一用 `env/` 下的解释器绝对路径调用（如 `<gen>/env/bin/python`，Windows 为 `env\Scripts\python.exe`）。
3. **按需降级/固定版本**：遇到冲突时先尝试按 CI 的组合；仍冲突则逐个固定报错包的版本并把**每次固定的原因**记进 `install_commands[]`。能锁就锁（`pip freeze > env/requirements.lock.txt`）——这是复现性的一部分。
4. **重型/系统依赖**：单细胞、基因组、图形库常需要系统库或 GPU。
   - Linux 上 `R`/`gcc`/`libhdf5`/`hdf5`/`zlib`/`cmake` 等：先用系统包管理器确认已装，缺则记录（**不要**在登录节点跑重编译作业）；
   - GPU 包（torch/tensorflow/jax 的 CUDA 版）：核对 `nvidia-smi` 的驱动与 CUDA 版本，按官方对应表装 wheel；**没有 GPU 时改为装 CPU 版**并把这一降级写进 JSON（会影响运行时间与部分工具可用性）；
   - 巨大的模型权重/数据库：**不要**在环境搭建阶段下载，记入 `external[]`，让教程执行阶段按需取。
5. **冒烟测试**（必须做，且必须真的执行）：导入主包 + 跑仓库自带的最小示例/测试。
   ```bash
   <gen>/env/bin/python -c "import <pkg>; print(<pkg>.__version__)"
   <gen>/env/bin/python -m pytest -q --collect-only   # 能收集到测试即算通路
   ```
   把每条命令的**真实输出**贴进 JSON 的 `commands_run[]`，不要写"应该可以"。
6. **写测试配置**：把环境路径、调用解释器的方式、需要设的环境变量（如 `CUDA_VISIBLE_DEVICES`、`OMP_NUM_THREADS`、`MPLBACKEND=Agg` 以免无头环境画图报错）写进 `env/run_env.sh`（或 `.ps1`），后续阶段统一 source 它。
7. **隔离性检查**：`env` 内 `pip list` 与系统 `pip list` 对比一次，确认没有把依赖装到外面。

## stage2-env.json

```json
{
  "env_kind": "venv",
  "env_path": "<绝对路径>/env",
  "python_exe": "<绝对路径>/env/bin/python",
  "python_version": "3.11.9",
  "install_commands": [
    { "cmd": "pip install -e .", "why": "仓库声明的可编辑安装" },
    { "cmd": "pip install numpy==1.26.4", "why": "1.27 与 numba 冲突，教程用到 numba" }
  ],
  "lockfile": "env/requirements.lock.txt",
  "env_vars": { "MPLBACKEND": "Agg", "OMP_NUM_THREADS": "1" },
  "gpu": { "available": false, "downgrade": "装 CPU 版 torch，教程 3 运行时间约 +6x" },
  "smoke_test": { "cmd": "...", "output": "...", "passed": true },
  "test_config": { "framework": "pytest", "cmd": "<gen>/env/bin/python -m pytest -q" },
  "blockers": []
}
```

## 常见坑

- **报错原文照抄进 JSON**：`ModuleNotFoundError: ...`、`ERROR: Could not find a version that satisfies ...` 本身就是证据；不要改写成"依赖问题"。
- 教程 notebook 常依赖**较老的**包版本（论文发表时的版本）。若新版 API 变了，宁可固定到论文时代的版本，也不要改教程代码——改了会破坏"与原文一致"这个验证基准。
- 不要在集群登录节点跑 `pip install` 之外的编译型重活；需要编译请在计算节点做（见集群相关技能）。
- 环境一旦跑通，**立刻冻结**：后续阶段只读它，不再随意 `pip install`。需要新包时更新 lock 并重跑冒烟测试。