# journal-full-figures-extract-skill — 期刊完整图表提取 Agent Skill

> （skill 名；驱动的工具链叫 pdf-figs，npm 包名不变）

> **别的工具给你图的碎片，这个 skill 给你论文里的原图。**
>
> 从学术论文 PDF 提取图表的 agent 能力：多面板组图按原文排版重组、完整
> 题注、结构化表格（cells/HTML）、单文件交互式画廊。Agent skill 标准格式
> （`SKILL.md`），适用于 Claude Code / ZCode / 其他支持 agent skills 的环境。

## 为什么不是又一个抽图工具

文档解析器（MinerU / Marker / Docling）输出的是"文档流"：一张 4 子图的
`Fig. 2` 变成 5+ 个互相独立的块。本 skill 的差异化全部有实测数字：

| 特点 | 说明 | 对比 |
|---|---|---|
| **组图保真重组（独占）** | 面板按原文布局还原，输出 `composition` 元数据（rows×cols + `panel-1..N` 阅读顺序），VLM 可逐面板消费 | 组合识别 **92.0%**，竞品 **0**——MinerU [#5031](https://github.com/opendatalab/MinerU/issues/5031) 用户请求此能力，官方确认无修复计划 |
| **表格完整交付** | 题注锚定 + 跨页拼接 + cells/HTML 结构化，双轴完整率 **99.0%** | Table F1 **96.7%**，超 MinerU basic（94.1%） |
| **完整题注段落** | 多行聚合、中英双语题注，可直接作检索文本 | pdffigures2/PicAxe 均无聚合 |
| **零部署重量** | `npm install` 秒级可用，纯 CPU | ~0.08s/页，比 MinerU basic 快 **~80×**；MinerU VLM 模式需 GPU + 1.2B 模型 |
| **与 MinerU 互补** | `--content-list` 引擎直接吸收 MinerU 的块做重组 | 它的文档流 + 本 skill 的结构层 = 完整管线 |
| **为 agent 而生** | 不只是 CLI——SKILL.md 是操作手册：三路径分流、校验强制、故障排查、能力边界自白 | 抽图工具没有"agent 会不会用"这一层 |
| **评测驱动、可复现** | 117 单测 + 90 篇 GT F1 + 完整率/组合专项，评测脚本全部公开 | 抽图类仓库普遍无公开量化评测 |

诚实边界：扫描件（无文本层）与公式/正文流解析不做——定位换取速度与轻量。

## 这是什么

一个让 agent **即取即用**的操作手册：用户说"把这篇论文的图表抽出来"，
agent 自动加载本 skill，按手册驱动 [pdf-figs 工具链](../../)完成抽取、
校验、展示——不需要读项目文档，也不会踩已知的坑。

三条路径：

| 用户说 | agent 做 | 产出 |
|---|---|---|
| "把图表抽出来给我看" | 抽取 → 完整性校验 → 画廊 | 单文件 gallery.html |
| "转成数据集/喂 LLM" | 抽取 → 读 manifest 转述字段 | caption / composition / cells |
| "描述这篇论文的 Figure 2" | 定位 → 按面板结构逐张读图 | 逐面板描述 |

## 质量基线（90 篇 pdffigures2 GT 评测，2026-10）

- Figure F1 **96.5%** · Table F1 **96.7%**（超 MinerU basic 94.1%）
- 表格双轴完整率 **99.0%** · 组图组合识别 **92.0%**（竞品 0）
- 速度 ~0.08s/页（零模型纯 JS），117/117 单测

## 安装

> 本 skill 也可从独立仓库安装：`git clone https://github.com/Nanxzi/journal-full-figures-extract-skill`（内容与本目录同步，`git subtree` 发布）。

前置：Node ≥18。

```bash
# 1. 安装工具链（约 1 分钟）
git clone https://github.com/Nanxzi/pdf-figure-toolkit
cd pdf-figure-toolkit && npm install

# 2. 安装 skill（复制到 agent 的 skills 目录）
mkdir -p ~/.agents/skills
cp -r skills/journal-full-figures-extract-skill ~/.agents/skills/
```

Claude Code 用户复制到 `~/.claude/skills/journal-full-figures-extract-skill/`。

## 验收示例

安装后对 agent 说：

1. **"提取这篇论文的图表并生成画廊"** + 任意学术论文 PDF
   → 应得到 gallery.html（缩略图 + 面板按原布局复原 + 表格渲染）
2. **"这篇论文里有几张表？Table 3 的内容是什么"**
   → agent 应读 manifest 的 `tables[]`，报出表格数量和 cells 内容
3. **"Figure 2 由几个子图组成？阅读顺序是什么"**
   → agent 应按 `composition` 回答（如"2×2 网格，左上→右上→左下→右下"）

## 触发词

提取图表 / 抽取图片 / 论文图组 / 重组图 / 图表画廊 / 表格提取 / PDF图表 /
extract figures / paper figures gallery / rebuild multi-panel figures

## 已知边界

扫描件（无文本层）不可用；纯图片表不在范围；详见 `SKILL.md` 的
"已知边界"与"故障排查"章节。

## 版本

与工具链同步发版：skill v0.2.0 ↔ toolkit v0.2.0。变更记录见仓库
[CHANGELOG](../../)（todo.md 各线交付记录）。
