---
name: journal-full-figures-extract-skill
version: 0.2.0
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
  figures from this paper" / "paper figures gallery". Not for scanned PDFs
  (no text layer) or full-text conversion (use MinerU/Marker for that).
---

# journal-full-figures-extract-skill — 期刊完整图表提取（agent 操作手册 / Agent Operations Manual）

> 驱动 [pdf-figs 工具链](../../)（工具包名 pdf-figs，npm 包名不变）

从学术论文 PDF 中保真重组复杂图组与表格。四个典型触发场景，**先判断用户
属于哪类，再走对应路径**：

| 用户意图 | 路径 | 关键产出 |
|---------|------|---------|
| "把图表抽出来给我看" / "extract and show figures" | A：抽取 → 校验 → gallery | gallery.html + 图片目录 |
| "转成数据集/喂 LLM" / "convert to dataset" | B：抽取 → 校验 → 字段解读 | manifest dict（caption/composition/cells） |
| "分析这篇论文的 Figure N" / "describe Figure N" | C：抽取 → 定位该图 → 读 panels + composition | 逐面板图片路径 + 结构描述 |
| "检查提取质量" / "audit the extraction" | D：抽取 → 逐张读图审计 | failures.jsonl 问题清单 |

## 步骤 0：定位工具（Tool Resolution）

按序探测，取第一个命中的 `PDF_FIGS_HOME`（仓库根目录）：

```bash
# 1. 环境变量
echo $PDF_FIGS_HOME
# 2. 本机常见位置（检查 packages/pdf-figs/src/cli.js 是否存在，替换为本机实际路径）
ls "$HOME/pdf-figure-toolkit/packages/pdf-figs/src/cli.js" 2>/dev/null
# 3. npm 全局安装的 CLI
which pdf-figs
```

- 命中 → `PDF_FIGS_HOME` 设为该仓库根；下文 `<CLI>` = `$PDF_FIGS_HOME/packages/pdf-figs/src/cli.js`，
  `<TOOLS>` = `$PDF_FIGS_HOME/tools`
- 全部未命中 → 一次性安装（约 1 分钟，需 Node ≥18）：

```bash
git clone https://github.com/Nanxzi/pdf-figure-toolkit
cd pdf-figure-toolkit && npm install
export PDF_FIGS_HOME="$(pwd)"
```

前置依赖：Node ≥18（`node --version` 验证）。npm 依赖装在仓库内，clone 后
`npm install` 一次即可。

## 引擎选择

| 条件 | 引擎 |
|------|------|
| 只有 PDF（最常见） | `--engine local`（零依赖，~0.08s/页，纯 CPU） |
| 用户提供了 MinerU content_list zip | `--content-list <zip>`（召回上限更高，表格 HTML 更全） |

## 路径 A：抽取 + 展示（最常见）

### 1. 抽取

```bash
node <CLI> extract <pdf路径> --engine local --formats png -o <输出目录>
```

成功标志：末行 `✅ N 个表格 / M 个图组 / K 张图片`。
常用变体：`--panels`（多面板图逐面板出图）、`--tables-only`、
`--formats png,jpg,svg,pdf`、批量 `batch <pdf...> --concurrency 4`。

### 2. 完整性校验（必做）

```bash
node <TOOLS>/check-manifest.mjs <输出目录>
```

通过标志：无 `❌` 行。

### 3. 画廊（用户要看图）

```bash
node $PDF_FIGS_HOME/packages/paper-gallery/src/cli.js <输出目录>/figures.json -o <输出目录>/gallery.html
```

产出单文件 HTML（缩略图 + 点击放大 + 面板按原布局复原 + 表格渲染），
把文件路径给用户即可。多篇论文可合并一页：gallery 接受多个 figures.json
输入（`gallery a/figures.json b/figures.json -o 合并.html`）。

## 路径 B：转数据集 / 喂 LLM

抽取后**不要只给用户文件路径**——读 manifest 并向用户转述关键内容：

- `figures[].caption`：完整题注段落（多行已聚合）→ 可直接作为图片的检索文本
- `figures[].composition`：`{kind, rows, cols, order}` → 组图的面板排布与阅读顺序
- `tables[].cells`：2D 文本数组 → 可转 pandas DataFrame 或 CSV
  （**仅在 `table-g*` 文本生长表与 `mineru-table` 上可用**；`table-N` 图组
  管道表只有裁剪图 + caption，转述时不要承诺结构化输出）
- `tables[].html`：可直接渲染的表体（可用性同上）
- `tables[].source`：`caption-anchored`（题注锚定）| `mineru-table`（MinerU 原生）| `table-detection`（兜底）

Python 消费（用户环境有 Python 时）：

```bash
python -m pip install -e "$PDF_FIGS_HOME/python"
```

```python
import pdffigures
from pdffigures.tables import tables_to_dataframe
result = pdffigures.extract("paper.pdf")
# tables[].cells → pd.DataFrame：tables_to_dataframe(t)（逐表调用，
# cells 缺失时返回 None——先判空）
```

## 路径 C：分析特定图（"描述 Figure N 的每个子图"）

1. 抽取后读 manifest，找到 `id` 或 `label` 匹配的 figure 条目
2. `composition.kind != "single"` 时：向用户说明面板结构
   （如"这张图是 2×2 网格，阅读顺序为左上→右上→左下→右下"）
3. 面板图片路径在 `figures[].panels[].image.path`——**相对路径，相对
   输出目录解析**（如 `fig-2_panel-1_p4.jpg`）——用 Read 工具逐张查看
   并向用户描述内容；单图时直接看 `image.path`
4. `caption` 是论文作者对图的原版描述——引用它作为分析的权威起点

## 路径 D：质量审计（"检查提取质量" / 批量审计）

agent 的视觉判断是引擎几何信号之外的**第二双眼睛**。审计后产出的
`failures.jsonl` 是修复工作的数据入口——每条可直接转化为回归 fixture。

### 流程

1. 抽取（路径 A 第 1 步）+ 校验（第 2 步）
2. 读 manifest，对**每个** figure/table 条目用 Read 查看裁剪图，按
   **五查清单**判定：
   - **表头切**：图/表顶部文字被水平切断（表头行/首行不完整）
   - **底行缺**：底部内容戛然而止（数据行/图例/x 轴标签缺失）
   - **宽截列**：左右边缘有被切的字符（表格侧列/图旁标签被裁）
   - **题注断**：caption 文本明显不完整（无句号收尾的长题注、
     与图中可见题注文字不一致）
   - **过收**：卷入邻栏正文、相邻资产（另一图/表的局部出现在本
     裁剪内）——双栏排版常见（astro-ph fig-4 吞邻栏 Table 1 实测）
3. 复核 `unassigned[]`：其中含大块图形（宽 >300 或高 >200）时,
   可能是被题注关联遗漏的真图表——逐块查看
4. 每个问题写一条 JSONL 记录到 `<输出目录>/failures.jsonl`：

```json
{"paper": "paper-id", "asset": "table-1", "page": 7, "check": "header-cut",
 "evidence": "top row characters clipped", "image": "table-1_p7.png",
 "captionExcerpt": "Table 1. Mean ranks of models..."}
```

字段：`paper`（pdf 文件名）/ `asset`（manifest 里的 id）/ `check`
（header-cut | bottom-truncated | width-clipped | caption-broken |
over-captured | suspicious-unassigned）/ `evidence`（一句话，引用图中可见证据）/
`image`（证据截图的相对路径）/ `captionExcerpt`（可选）。

### 边界

- 只记录**有图中证据**的问题，不确定的不记（宁缺勿滥）
- 审计结论不回写 manifest、不改变抽取产物——质量判定以 GT 评测为准，
  本路径只产出问题清单
- 批量审计多篇时每篇一个 failures.jsonl，放在各自输出目录

## agent 辅助修复：agent-split（A2，面板拆解）

**适用**：`composition.kind == "single"` 但 `composition.captionPanels >= 2`
的资产——面板烘焙在单张位图/矢量块里，纯几何不可分，agent 视觉是
唯一出路。工作流：

1. 读 manifest 找到这类资产（captionPanels 是题注声明的面板数 N）
2. Read 该资产裁剪图，肉眼划分 N 个面板，记录每个面板的归一化 bbox
   （0-1000，阅读顺序）
3. 执行拆解：

```bash
node <CLI> agent-split <输出目录> <figId> \n  --bboxes "x0,y0,x1,y1 x0,y0,x1,y1 ..." --pdf <原PDF路径>
```

4. 命令内置四条防呆（captionPanels 数量一致 / bbox 合法 / 面板重叠
   IoU≤0.5 / 离心 ≤200），违规即拒；`--force` 可跳过数量与离心检查
5. 产物：`{figId}_agent-panel-N_p{P}.jpg` 面板文件 + manifest 的
   `panels[]`/`composition` 重算 + `agentAssisted: true` 标记；
   之后重新生成 gallery 即可按面板展示

**注意**：agent 返回的 bbox 要贴合面板边缘（含面板子标签如 "(a)"）；
拆解结果以 Read 复核为准，有疑问宁可不拆。

## manifest 字段速查

| 字段 | 含义 |
|------|------|
| `figures[].composition` | 组图布局元数据（kind/rows/cols/order）——本工具独有 |
| `figures[].caption` | 完整题注段落（多行聚合，中英文题注均支持） |
| `figures[].panels[].image.path` | 逐面板裁剪图（需 `--panels`） |
| `tables[].cells` / `.html` | 结构化表体（仅 `table-g*` 生长表与 MinerU 块携带） |
| `tables[].source` | `caption-anchored` \| `mineru-table` \| `table-detection` |
| `*.bbox` | 页面归一化坐标 0-1000 |
| `unassigned[]` | 未能关联到题注的图块（供用户判断是否重要） |
| `panels[].image.path` | 相对路径——相对输出目录解析 |

## 质量基线（90 篇 pdffigures2 GT 评测，2026-10）

- Figure P 97.4% / R 95.6% / **F1 96.5%**
- Table P 97.2% / R 96.3% / **F1 96.7%**（超 MinerU basic 94.1%）
- 表格双轴完整率 **99.0%**（区域检出表中 bbox 覆盖 GT ≥95% 的比例）
- 组图组合识别 **92.0%** / 组合语义覆盖 **88.9%**（竞品 0——无产出该结构者）
- 速度 ~0.08s/页（比 MinerU basic 快 ~80×），110/110 单测

完整评测：`$PDF_FIGS_HOME/fixtures/pdffigures2-eval/`（eval-pdffigures2.mjs /
eval-table-completeness.mjs / eval-composition.mjs）。

## 已知边界（向用户主动说明）

- 扫描件不可用（local 引擎依赖文本层）
- 纯图片表（表体是位图）与手写/扫描表格不在范围
- 数学上下标密集的符号表（θ/+/* 碎块错位）行聚合可能欠佳
- 表格 `rows/columns` 在图组管道表中为 null（面板语义≠表格行列）
- 同页紧凑排版偶有 `-2` 伴生碎组
- 中文题注（图N/表N）已支持；其他语言（德/日文）未覆盖

## 故障排查

| 症状 | 处置 |
|------|------|
| `无可提取图组` | 先确认有文本层（扫描件不支持）；文本层正常则可能是纯理论论文（无图表，如实告知用户，math.PR 实测）；再试 heuristic 引擎 |
| `heuristic 引擎需要 --content-list` | 缺 MinerU zip；改用 `--engine local` |
| 输出目录 Permission denied | 输出目录不要预创建，让工具自建 |
| `pdf-figs CLI 未找到` | 未定位到工具——回到步骤 0 检测/安装 |
| 题注被切断 | 确认用最新代码（pitch 行距版 assembleCaption）；仍切则记档 |
| 表格范围偏小/缺行 | 确认用最新代码（TC 线：前瞻续收/栏带锚/表体文本吸收）；仍缺则记档 |

---
*Validated against toolkit v0.2.0+（110 unit tests / 90-paper GT eval /
completeness & composition 专项评测）· Skill 源文件：仓库
`skills/journal-full-figures-extract-skill/SKILL.md`，本文件为安装副本。*
