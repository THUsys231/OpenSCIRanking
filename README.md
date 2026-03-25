<p align="center">
  <a href="#english">
    <img alt="English" src="https://img.shields.io/badge/English-1f6feb">
  </a>
  <a href="#中文">
    <img alt="中文" src="https://img.shields.io/badge/%E4%B8%AD%E6%96%87-c2410c">
  </a>
</p>

<p align="center">
  <img src="assets/banner.png" alt="OpenSCIRanking banner" width="50%" />
</p>

<h1 align="center">OpenSCIRanking</h1>

<p align="center">
  A reproducible, bottom-up journal ranking pipeline built on top of OpenAlex.
  <br />
  一个基于 OpenAlex 的可复现、自下而上的期刊排序流水线。
</p>

<p align="center">
  <img alt="Python" src="https://img.shields.io/badge/Python-3.11%2B-1f6feb">
  <img alt="Data Source" src="https://img.shields.io/badge/Data-OpenAlex-0f766e">
  <img alt="Ranking" src="https://img.shields.io/badge/Method-Citation%20Graph%20%2B%20PageRank-c2410c">
  <img alt="Workflow" src="https://img.shields.io/badge/Workflow-Reproducible-334155">
</p>

## Home

This repository keeps the full source code and uses **computer science journals** as the worked example, from journal-list preparation to final ranking export.

当前仓库完整保留了源码，并以**计算机科学期刊**为完整示例，展示从期刊名单准备到最终排名导出的全流程。

<a id="english"></a>

<details open>
<summary><strong>English</strong></summary>

## Overview

**OpenSCIRanking** is an open repository for ranking journals with a transparent, data-driven workflow rather than top-down expert partitioning.

## Why This Repo

- Bottom-up rather than top-down: rankings come from citation-network structure and explicit adjustment rules.
- Reproducible workflow: users can clone the repo and rerun the pipeline on their own journal lists.
- Transparent scoring: the method is inspectable, tunable, and easy to audit.
- GitHub-friendly layout: code is versioned, while local data products are excluded.

## Method Snapshot

The default worked-example recipe in this repository is:

- build an inter-journal citation graph from OpenAlex works
- drop journal self-citations
- compute base `PageRank`
- apply `sqrt_norm` to reduce volume bias
- apply a mega-journal penalty to suppress extremely large venues
- apply a mild review-heavy penalty based on average references per paper

## Repository Structure

- `inputs/`: example journal lists and seed files
- `src/opensci_v2/`: reusable package code
- `scripts/`: command-line entrypoints for each pipeline stage
- `assets/`: README visual assets
- `data/`: local fetched data and outputs, ignored by git
- `cas_data/`: local raw spreadsheets, ignored by git

## Quick Start

Requirements:

- Python `>= 3.11`
- network access to the OpenAlex API
- enough local disk for Parquet outputs

```bash
python -m pip install -r requirements.txt
```

or:

```bash
make install
```

## Worked Example: Computer Science

The repository already includes:

- [inputs/computer_science_journals.csv](inputs/computer_science_journals.csv)

### 1. Resolve journals to OpenAlex sources

```bash
PYTHONPATH=src python scripts/resolve_openalex_sources.py \
  --input inputs/computer_science_journals.csv \
  --output data/raw/computer_science_sources_resolved.csv \
  --search-fallback
```

### 2. Build the normalized sources table

```bash
PYTHONPATH=src python scripts/build_sources_table.py \
  --input data/raw/computer_science_sources_resolved.csv \
  --output data/bronze/computer_science_sources.parquet
```

### 3. Fetch recent works in resumable batches

```bash
PYTHONPATH=src python scripts/fetch_openalex_works_batch.py \
  --sources data/bronze/computer_science_sources.parquet \
  --output-dir data/bronze/computer_science_works_shards \
  --state-path data/raw/computer_science_fetch_state.csv \
  --merged-output data/bronze/computer_science_works.parquet \
  --start-year 2023 \
  --end-year 2025 \
  --resume \
  --retry-failures
```

### 4. Build the de-self-cited journal graph

```bash
PYTHONPATH=src python scripts/build_journal_graph.py \
  --works data/bronze/computer_science_works.parquet \
  --output data/silver/computer_science_journal_edges.parquet \
  --drop-self-citations
```

### 5. Compute the base PageRank

```bash
PYTHONPATH=src python scripts/compute_pagerank.py \
  --edges data/silver/computer_science_journal_edges.parquet \
  --output data/gold/computer_science_journal_pagerank.parquet
```

### 6. Export the final ranking

```bash
PYTHONPATH=src python scripts/compute_final_ranking.py \
  --rankings data/gold/computer_science_journal_pagerank.parquet \
  --resolved data/raw/computer_science_sources_resolved.csv \
  --works data/bronze/computer_science_works.parquet \
  --output-prefix data/gold/computer_science_journal_ranking_final \
  --size-mode sqrt_norm \
  --mega-journal-cap 12000 \
  --review-penalty \
  --review-threshold-quantile 0.75 \
  --review-penalty-exp 0.15
```

Outputs:

- `data/gold/computer_science_journal_ranking_final.csv`
- `data/gold/computer_science_journal_ranking_final.xlsx`
- `data/gold/computer_science_journal_ranking_final.parquet`
- `data/gold/computer_science_journal_ranking_final_top100.csv`

## Make Targets

```bash
make example-resolve
make example-sources
make example-fetch
make example-graph
make example-pagerank
make example-final
```

For a smaller smoke test, use `inputs/journals.sample.csv` or pass `--limit` during source resolution.

## Design Notes

- `fetch_openalex_works_batch.py` writes one shard per source and persists a CSV state file after every source.
- `build_journal_graph.py` can explicitly remove self-citations.
- `compute_final_ranking.py` is the GitHub-facing entrypoint for the current ranking recipe.
- `igraph` is preferred for PageRank; the code can fall back to `networkx` where applicable.

## Versioning Policy

Tracked in git:

- source code
- configuration files
- example input lists
- README and visual assets

Ignored locally:

- raw Excel workbooks
- fetched OpenAlex datasets
- intermediate Parquet shards
- local ranking outputs

## License

This repository keeps the upstream [LICENSE](LICENSE) file already present in the GitHub repository.

</details>

<a id="中文"></a>

<details>
<summary><strong>中文</strong></summary>

## 项目简介

**OpenSCIRanking** 是一个面向开源使用的期刊排序项目模板，目标是用透明、可追溯、数据驱动的方法替代传统“自上而下拍脑袋”的期刊分区逻辑。

## 为什么做这个仓库

- 自下而上：基于引用网络和显式评分规则，而不是专家主观拍板
- 可复现：克隆仓库后即可按流程重跑
- 可解释：评分逻辑清晰，可审查、可调参
- 适合开源：代码入库，数据产物和本地工作簿不入库

## 方法概览

当前仓库示例使用的默认方案是：

- 基于 OpenAlex 论文数据构建期刊间引用图
- 去除期刊自引
- 计算基础 `PageRank`
- 用 `sqrt_norm` 抑制超大体量期刊的体积优势
- 加入巨型刊惩罚项
- 对综述较重、参考文献极长的期刊加入较轻惩罚

## 仓库结构

- `inputs/`: 示例期刊名单和种子文件
- `src/opensci_v2/`: 可复用的 Python 包代码
- `scripts/`: 各阶段命令行脚本
- `assets/`: README 视觉资源
- `data/`: 本地抓取数据和输出结果，已被 git 忽略
- `cas_data/`: 本地原始表格，已被 git 忽略

## 快速开始

环境要求：

- Python `>= 3.11`
- 可访问 OpenAlex API 的网络
- 足够的本地磁盘空间用于保存 Parquet 数据

```bash
python -m pip install -r requirements.txt
```

或者：

```bash
make install
```

## 示例流程：计算机科学

仓库中已经包含这份示例期刊名单：

- [inputs/computer_science_journals.csv](inputs/computer_science_journals.csv)

### 1. 解析期刊到 OpenAlex source

```bash
PYTHONPATH=src python scripts/resolve_openalex_sources.py \
  --input inputs/computer_science_journals.csv \
  --output data/raw/computer_science_sources_resolved.csv \
  --search-fallback
```

### 2. 生成规范化的 sources 表

```bash
PYTHONPATH=src python scripts/build_sources_table.py \
  --input data/raw/computer_science_sources_resolved.csv \
  --output data/bronze/computer_science_sources.parquet
```

### 3. 以断点续跑方式抓取近年论文

```bash
PYTHONPATH=src python scripts/fetch_openalex_works_batch.py \
  --sources data/bronze/computer_science_sources.parquet \
  --output-dir data/bronze/computer_science_works_shards \
  --state-path data/raw/computer_science_fetch_state.csv \
  --merged-output data/bronze/computer_science_works.parquet \
  --start-year 2023 \
  --end-year 2025 \
  --resume \
  --retry-failures
```

### 4. 构建去自引的期刊引用图

```bash
PYTHONPATH=src python scripts/build_journal_graph.py \
  --works data/bronze/computer_science_works.parquet \
  --output data/silver/computer_science_journal_edges.parquet \
  --drop-self-citations
```

### 5. 计算基础 PageRank

```bash
PYTHONPATH=src python scripts/compute_pagerank.py \
  --edges data/silver/computer_science_journal_edges.parquet \
  --output data/gold/computer_science_journal_pagerank.parquet
```

### 6. 导出最终排名

```bash
PYTHONPATH=src python scripts/compute_final_ranking.py \
  --rankings data/gold/computer_science_journal_pagerank.parquet \
  --resolved data/raw/computer_science_sources_resolved.csv \
  --works data/bronze/computer_science_works.parquet \
  --output-prefix data/gold/computer_science_journal_ranking_final \
  --size-mode sqrt_norm \
  --mega-journal-cap 12000 \
  --review-penalty \
  --review-threshold-quantile 0.75 \
  --review-penalty-exp 0.15
```

输出文件：

- `data/gold/computer_science_journal_ranking_final.csv`
- `data/gold/computer_science_journal_ranking_final.xlsx`
- `data/gold/computer_science_journal_ranking_final.parquet`
- `data/gold/computer_science_journal_ranking_final_top100.csv`

## 一键命令

```bash
make example-resolve
make example-sources
make example-fetch
make example-graph
make example-pagerank
make example-final
```

如果只想快速冒烟测试，可以改用 `inputs/journals.sample.csv`，或者在解析阶段加 `--limit`。

## 设计说明

- `fetch_openalex_works_batch.py` 为每本期刊写一个 shard，并在每本期刊完成后更新状态文件
- `build_journal_graph.py` 可以显式去除期刊自引
- `compute_final_ranking.py` 是当前 GitHub 版本的最终评分入口
- `igraph` 优先用于 PageRank；缺失时在适用场景下回退到 `networkx`

## 版本管理策略

会纳入 git 的内容：

- 源代码
- 配置文件
- 示例输入列表
- README 和视觉资源

本地忽略的内容：

- 原始 Excel 工作簿
- 抓取得到的 OpenAlex 数据
- 中间 Parquet shards
- 本地排名输出

## 许可

仓库保留远端已有的 [LICENSE](LICENSE) 文件。

</details>
