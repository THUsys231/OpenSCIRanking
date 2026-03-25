<p align="center">
  <img src="assets/opensci-ranking-banner.svg" alt="OpenSCIRanking banner" width="100%" />
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

## Overview | 项目简介

**OpenSCIRanking** is an open repository for ranking journals with a transparent, data-driven workflow rather than top-down expert partitioning.

**OpenSCIRanking** 旨在用透明、可追溯、数据驱动的方法替代传统“自上而下拍脑袋”的期刊分区逻辑。

The current repository keeps the full source code and uses **computer science journals** as the worked example, from journal-list preparation all the way to final ranking export.

当前仓库完整保留了源码，并以**计算机科学期刊**为示例，展示从期刊名单准备、OpenAlex 解析抓取、期刊引用图构建，到最终排名导出的完整流程。

## Why This Repo | 为什么做这个仓库

- Bottom-up rather than top-down: rankings come from citation-network structure and explicit adjustment rules.
- Reproducible workflow: users can clone the repo and rerun the pipeline on their own journal lists.
- Transparent scoring: the method is inspectable, tunable, and easy to audit.
- GitHub-friendly layout: code is versioned, while local data products are excluded.

- 自下而上：基于引用网络与显式规则，而不是专家主观拍板。
- 可复现：用户克隆仓库后即可按流程重跑。
- 可解释：评分规则公开透明，便于审查与调参。
- 适合开源：代码入库，数据产物与本地工作簿不入库。

## Method Snapshot | 方法概览

The default worked-example recipe in this repository is:

- build an inter-journal citation graph from OpenAlex works
- drop journal self-citations
- compute base `PageRank`
- apply `sqrt_norm` to reduce volume bias
- apply a mega-journal penalty to suppress extremely large venues
- apply a mild review-heavy penalty based on average references per paper

当前仓库示例使用的默认方案是：

- 基于 OpenAlex 论文数据构建期刊间引用图
- 去除期刊自引
- 计算基础 `PageRank`
- 用 `sqrt_norm` 抑制超大体量期刊的体积优势
- 加入巨型刊惩罚项
- 对综述占比较重、参考文献极长的期刊加入较轻惩罚

## Repository Structure | 仓库结构

- `inputs/`: example journal lists and seed files
- `src/opensci_v2/`: reusable package code
- `scripts/`: command-line entrypoints for each pipeline stage
- `assets/`: README visual assets
- `data/`: local fetched data and outputs, ignored by git
- `cas_data/`: local raw spreadsheets, ignored by git

## Quick Start | 快速开始

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

## Worked Example: Computer Science | 示例流程：计算机科学

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

## Make Targets | 一键命令

```bash
make example-resolve
make example-sources
make example-fetch
make example-graph
make example-pagerank
make example-final
```

For a smaller smoke test, use `inputs/journals.sample.csv` or pass `--limit` during source resolution.

如果只想快速测试流程，可以改用 `inputs/journals.sample.csv`，或在解析阶段加 `--limit`。

## Design Notes | 设计说明

- `fetch_openalex_works_batch.py` writes one shard per source and persists a CSV state file after every source.
- `build_journal_graph.py` can explicitly remove self-citations.
- `compute_final_ranking.py` is the GitHub-facing entrypoint for the current ranking recipe.
- `igraph` is preferred for PageRank; the code can fall back to `networkx` where applicable.

## Versioning Policy | 版本管理策略

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

## License | 许可

This repository keeps the upstream [LICENSE](LICENSE) file already present in the GitHub repository.
