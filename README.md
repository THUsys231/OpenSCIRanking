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
  <img alt="License" src="https://img.shields.io/badge/License-MIT-16a34a">
</p>

<a id="english"></a>

<details open>
<summary><strong>English</strong></summary>

## Overview

**OpenSCIRanking** is an open repository for journal ranking that replaces top-down expert partitioning with a data-driven workflow centered on an improved PageRank-style citation graph. We treat code as the governing rule: the full data pipeline, graph construction logic, and ranking adjustments are open for inspection rather than hidden behind a black box.

This repository keeps the full source code, while **computer science journals** are included only as a worked example subset for demonstrating the end-to-end pipeline.

## Why This Repo

- Bottom-up rather than top-down: rankings come from citation-network structure and explicit adjustment rules.
- Reproducible workflow: users can clone the repo and rerun the pipeline on their own journal lists.
- Transparent scoring: the method is inspectable, tunable, and easy to audit.
- GitHub-friendly layout: code is versioned, while local data products are excluded.

### Pain Points in Traditional Ranking Systems

For years, the academic community has had to rely on journal rankings built around impact factors, expert partitions, or other centralized classification schemes. In real-world use, those systems show increasingly visible weaknesses:

- Vulnerable to manipulation: metrics such as average citations per paper can be gamed through journal self-citation, editorial pressure to cite the host journal, citation cartels, or aggressive review-heavy publishing strategies.
- Opaque and centralized: many classification systems are effectively black boxes. The rules are difficult to inspect, update cycles are slow, and emerging interdisciplinary venues are often misrepresented.
- Hard to audit or customize: commercial data pipelines and proprietary scoring logic prevent ordinary researchers from reproducing the rankings or rebuilding them for a narrower subfield such as computer networks, AI, or security.

OpenSCIRanking is built to address those weaknesses directly. The goal is not to publish yet another opaque leaderboard, but to expose the full chain from raw metadata to final score so that researchers can audit, reproduce, and adapt the ranking logic themselves.

## Method Snapshot

The default worked-example recipe in this repository is:

- build an inter-journal citation graph from OpenAlex works
- drop journal self-citations
- compute base `PageRank`
- apply `sqrt_norm` to reduce volume bias
- apply a mega-journal penalty to suppress extremely large venues
- apply a mild review-heavy penalty based on average references per paper

### Method and Rationale

This project is not just a PageRank wrapper. It is a de-biasing pipeline designed to counter common ranking distortions.

#### 1. Build an inter-journal citation graph from OpenAlex works

Mechanism:

- start from article-level metadata and paper-to-paper citations
- aggregate them into a directed, weighted journal-to-journal graph

Purpose:

- move away from isolated scalar indicators
- evaluate journals by their position inside the full citation network and by the direction of knowledge flow across the field

#### 2. Drop journal self-citations

Mechanism:

- remove edges whose source journal and target journal are the same

Purpose:

- directly block one of the simplest ranking manipulation tactics
- prevent editorial self-citation pressure from inflating journal prestige

#### 3. Compute base PageRank

Mechanism:

- run PageRank on the de-self-cited directed graph

Purpose:

- distinguish citation quality from raw citation count
- give more weight to citations coming from already influential journals
- weaken the value of low-quality mutual-support structures

#### 4. Apply `sqrt_norm` to reduce volume bias

Mechanism:

- divide the base PageRank score by the square root of the journal's publication volume

Purpose:

- avoid a leaderboard dominated by sheer scale
- avoid the opposite extreme where tiny journals with a few outlier papers become unrealistically over-rewarded
- keep a middle ground between total prestige and per-paper efficiency

#### 5. Apply a mega-journal penalty

Mechanism:

- apply an additional penalty to journals above a configurable output-volume threshold such as `12,000` papers

Purpose:

- suppress rank inflation caused by extreme publication volume alone
- reduce the structural advantage of ultra-large open-access venues and paper-factory-like publication models

#### 6. Apply a mild review-heavy penalty

Mechanism:

- use average references per paper as a proxy for review-heavy behavior
- mildly downweight journals whose papers are unusually reference-dense

Purpose:

- reduce the systematic advantage of survey and review venues over ordinary research journals
- make the ranking better reflect how active researchers perceive standard research outlets such as transactions and regular journals

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

Released under the [MIT License](LICENSE).

</details>

<a id="中文"></a>

<details>
<summary><strong>中文</strong></summary>

## 项目简介

**OpenSCIRanking** 是一个面向开源使用的期刊排序项目模板，目标是用数据驱动的方法(改进的PageRank)替代传统“自上而下拍脑袋”的期刊分区逻辑。我们坚信“代码即法则”，致力于提供一个完全开源、数据透明、且基于底层论文引用网络自然生长的自下而上评价框架。在这个框架内，没有任何黑箱操作，接受学术共同体的检验。

当前仓库完整保留了源码，而**计算机科学期刊**只是本项目当前附带的一个示例子集，用来演示从期刊名单准备到最终排名导出的完整流程。

## 为什么做这个仓库

- 自下而上：基于引用网络和显式评分规则，而不是专家主观拍板
- 可复现：克隆仓库后即可按流程重跑
- 可解释：评分逻辑清晰，可审查、可调参
- 适合开源：代码入库，数据产物和本地工作簿不入库

### 传统评价体系的核心痛点

长期以来，学术界深受传统期刊评价体系的困扰。当前主流的期刊分区和排名（如基于 JCR 影响因子/中科院分区/新锐分区或各类专家设定的分区）在实际运行中暴露出日益严重的问题：

- 容易被恶意操纵：如果指标过度依赖篇均被引、总被引或若干简单平均数，就会很容易被非学术行为绑架，例如强制作者引用本刊、堆高自引率、形成互引联盟，或者通过大量综述型文章抬高整体表现。
- 中心化与黑盒化：很多分区或排行榜的更新逻辑并不透明，规则带有明显主观性，而且对新兴交叉领域的响应速度很慢，难以及时反映学科结构变化。
- 缺乏开源审计能力：数据和算法长期被少数商业机构或封闭系统掌握，普通研究者很难复现其计算过程，更难针对某个具体子领域做定制化剥离和重排。

## 方法概览

当前仓库示例使用的默认方案是：

- 基于 OpenAlex 论文数据构建期刊间引用图
- 去除期刊自引
- 计算基础 `PageRank`
- 用 `sqrt_norm` 抑制超大体量期刊的体积优势
- 加入巨型刊惩罚项
- 对综述较重、参考文献极长的期刊加入较轻惩罚

### 核心排序方法与反制逻辑

本项目不是简单地统计引用量，也不是只把 `PageRank` 套到期刊名单上。它更接近一套针对常见刷榜路径的去偏引擎，每一步都对应一个明确的结构性目的。

#### 1. 从 OpenAlex 构建期刊间引文图谱

机制：

- 从底层论文元数据出发，收集论文到论文的引用关系
- 再把这些关系聚合成“期刊到期刊”的有向加权图

目的：

- 摆脱孤立的统计数字
- 用整个学科网络的拓扑结构来判断一本期刊在知识传播中的真实位置

#### 2. 剔除期刊自引

机制：

- 删除起点和终点属于同一期刊的边

目的：

- 直接切断最常见、最廉价的操纵方式
- 防止编辑部暗示作者自引或通过流程性安排抬高本刊得分

#### 3. 计算基础 PageRank

机制：

- 在去自引后的有向图上运行 `PageRank`

目的：

- 不再把所有引用视为等价投票
- 引入“被谁引用”比“被引多少次”更重要的质量概念
- 从算法层面削弱低质量期刊之间互相输血的价值

#### 4. 应用平方根归一化

机制：

- 用期刊基础分数除以其发文量的平方根

目的：

- 防止超大发文量期刊靠体积碾压榜单
- 同时避免简单篇均化把超小体量期刊抬得过高
- 在“总量优势”和“篇均效率”之间取一个更稳的数学折中

#### 5. 巨型期刊惩罚

机制：

- 对发文量超过阈值的期刊施加额外惩罚，默认示例中使用 `12,000` 作为量级参考

目的：

- 抑制纯体积效应造成的排名泡沫
- 对超大型开源期刊、海量发文模式或接近“论文工厂”式的结构优势进行额外约束

#### 6. 综述型期刊温和惩罚

机制：

- 用篇均参考文献数作为综述密度的近似代理
- 对参考文献异常密集的期刊进行温和降权

目的：

- 适度削弱综述类期刊天然更容易吸引引用的优势
- 让榜单在综述刊与常规原创研究期刊之间取得更接近一线研究者体感的平衡

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

本项目采用 [MIT License](LICENSE) 开源协议。

</details>
