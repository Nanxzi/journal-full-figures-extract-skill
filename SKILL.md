---
name: journal-full-figures-extract-skill
version: 0.2.1
license: MIT
description: >
  Extract figures and tables from academic PDFs with bbox precision —
  composite multi-panel figures reassembled to their paper-faithful form,
  complete captions, structured table bodies (cells/HTML), and a single-file
  interactive HTML gallery. Use when the user asks to extract figures, tables,
  or charts from a paper/PDF; rebuild multi-panel figures; build a paper
  figure gallery; get full captions of figures; convert PDF figures/tables
  for LLM pipelines or RAG corpora; or analyze/describe the figures in a
  paper (panel crops + composition metadata). Triggers include: 提取图表 /
  抽取图片 / 论文图组 / 重组图 / 图表画廊 / 表格提取 / PDF图表 / "extract
  figures from this paper" / "paper figures gallery" / 论文插图. Not for scanned PDFs
  (no text layer) or full-text conversion (use MinerU/Marker for that).
---

# journal-full-figures-extract-skill — 期刊完整图表提取（agent 操作手册）

> 驱动 pdf-figs 工具链（npm 包名不变）。细节按需读 `references/`。

## Design principles（设计宪法）

1. **确定性优先**：同 PDF 同结果；非确定性能力（agent 视觉判断、自由
   裁量修正）不进主路径——做成分体 skill 或 opt-in 协议，产物打
   `agentAssisted` 标记且不进默认评测口径；
2. **能力边界自白**：每个新功能同时声明它不做什么；
3. **每条路径有验收判据**：交差前按该路径自检清单核验；
4. **渐进披露**：本手册 ≤6KB，细节在 `references/`（manifest-fields /
   troubleshooting / python-consumption）；
5. **试验先行**：手册改动也走 Spike→Integrate→Measure。

## 交互规则（前置集中，中途零打断）

**开始时一次问清**（每问带默认建议，用户零输入=全默认）：
交付形态（默认：画廊展示；转结构化数据/逐图描述请说明）、范围（默认：
单篇；批量请给目录）、引擎（默认 local；有 MinerU 解析结果请提供）、
质量要求（默认：抽取+校验；需逐张目检请说明）、边界偏好（默认：扫描件
跳过并告知）。**开始之后中途不停等提问**——可疑项按保守默认继续并记入
"待确认清单"，交付时随过程报告（修正动作留待用户事后授权）。

从学术论文 PDF 中保真重组复杂图组与表格。三个典型触发场景，**先判断
属于哪类，再走对应路径**：

| 意图 | 路径 | 产出 |
|---|---|---|
| "把图表抽出来给我看" | A：抽取→校验→gallery | gallery.html + 图片 |
| "转成数据集/喂 LLM" | B：抽取→校验→字段解读 | manifest（caption/composition/cells）|
| "分析这篇论文的 Figure N" | C：定位→读 panels+composition | 逐面板描述 |

## 步骤 0：定位工具

```bash
echo $PDF_FIGS_HOME
ls "$HOME/pdf-figure-toolkit/packages/pdf-figs/src/cli.js" 2>/dev/null
which pdf-figs
```

命中 → `<CLI>` = `$PDF_FIGS_HOME/packages/pdf-figs/src/cli.js`，`<TOOLS>`
= `$PDF_FIGS_HOME/tools`。全未命中则安装（Node ≥18）：

```bash
git clone https://github.com/Nanxzi/pdf-figure-toolkit
cd pdf-figure-toolkit && npm install && export PDF_FIGS_HOME="$(pwd)"
```

## 路径 A：抽取 + 展示

1. `node <CLI> extract <pdf> --engine local --formats png -o <输出目录>`
   （变体：`--panels` 逐面板、`--tables-only`、`batch` 批量）
2. `node <TOOLS>/check-manifest.mjs <输出目录>`（通过=无 ❌ 行，必做）
3. `node $PDF_FIGS_HOME/packages/paper-gallery/src/cli.js <输出目录>/figures.json -o <输出目录>/gallery.html`
   （多篇可合并：gallery 接受多个 figures.json 输入）
4. **自检**：无 ❌、gallery 存在、抽查 1 图 1 表裁剪完整（可疑则按交互
   规则记入待确认清单，不停等）

## 路径 B：转数据集 / 喂 LLM

抽取后**不要只给文件路径**——读 manifest 转述关键内容：
`figures[].caption`（完整题注，检索文本）、`composition`（kind/rows/
cols/order，面板排布与阅读序）、`tables[].cells/.html`（**仅 `table-g*`
生长表与 mineru-table 携带**，转述勿对 `table-N` 承诺结构化输出）。
字段语义详见 `references/manifest-fields.md`。Python 消费见
`references/python-consumption.md`。
**自检**：逐资产转述、无 cells 的表已明确标注。

## 路径 C：分析特定图

1. manifest 找到 `id`/`label` 匹配条目；`composition.kind != "single"`
   时先说明面板结构（如"2×2 网格，阅读顺序左上→右下"）
2. `panels[].image.path`（**相对输出目录**）逐张 Read 描述；单图看
   `image.path`
3. `caption` 是作者原版描述——作为分析权威起点
4. **自检**：每张面板已 Read、描述引用了 composition

## 质量基线（90 篇 pdffigures2 GT，2026-10）

Figure F1 **96.0** · Table F1 **96.7**（超 MinerU basic 94.1）· 双轴
完整率 **99.0%** · 组合识别 **92.0%**（竞品 0）· ~0.08s/页（快 ~80×）·
117 单测。评测：`$PDF_FIGS_HOME/fixtures/pdffigures2-eval/`。

## 已知边界

扫描件不可用（依赖文本层）；纯图片表/手写表不在范围；数学上下标密集
符号表行聚合可能欠佳；图组管道表 rows/columns 为 null；紧凑排版偶有
`-2` 伴生碎组；中文题注已支持，德/日文未覆盖。故障排查见
`references/troubleshooting.md`。

---
*Validated against toolkit v0.2.0+（117 unit tests / 90-paper GT /
completeness & composition 专项）· 源文件：仓库
`skills/journal-full-figures-extract-skill/SKILL.md`，本文件为安装副本。*
