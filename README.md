# journal-full-figures-extract-skill — Full-Figure Extraction from Journal Papers (Agent Skill)

> (The skill name; the toolchain it drives is called pdf-figs — npm package name unchanged.)

[中文](./README.zh-CN.md) · English

> **Other tools give you figure fragments — this skill gives you the original
> figure as the reader sees it.**
>
> Agent capability for extracting figures and tables from scholarly PDFs:
> multi-panel composite figures reassembled in their original layout, complete
> captions, structured tables (cells/HTML), and a single-file interactive
> gallery. Standard agent-skill format (`SKILL.md`), works with Claude Code,
> ZCode, and other agent-skills environments.

## Why this isn't yet another figure extractor

Document parsers (MinerU / Marker / Docling) output a *document stream*: a
4-panel `Fig. 2` becomes 5+ mutually independent blocks. Every differentiator
below is backed by measured numbers:

| Feature | What it does | Comparison |
|---|---|---|
| **Faithful composite-figure reassembly (unique)** | Panels restored in their original layout, `composition` metadata emitted (rows×cols + `panel-1..N` reading order) so VLMs can consume per panel | Composition recognition **92.0%** vs **0** for competitors — [MinerU #5031](https://github.com/opendatalab/MinerU/issues/5031) users request exactly this; confirmed no fix planned |
| **Complete table delivery** | Caption-anchored + cross-page stitching + cells/HTML structuring, axis-completeness **99.0%** | Table F1 **96.7%**, ahead of MinerU basic (94.1%) |
| **Full caption paragraphs** | Multi-line aggregation, Chinese & English captions, directly usable as retrieval text | pdffigures2 / PicAxe have no aggregation |
| **Zero deployment weight** | `npm install`, seconds to first use, pure CPU | ~0.08s/page, **~80× faster** than MinerU basic; MinerU VLM mode needs a GPU + a 1.2B model |
| **Complementary to MinerU** | The `--content-list` engine ingests MinerU blocks directly for reassembly | Its document stream + this skill's structure layer = a complete pipeline |
| **Built for agents** | Not just a CLI — SKILL.md is an operations manual: three-path triage, mandatory validation, troubleshooting, honest capability boundaries | Figure extractors don't cover "can an agent drive this" |
| **Evaluation-driven, reproducible** | 117 unit tests + 90-paper GT F1 + completeness & composition benchmarks, all scripts public | Figure-extraction repos generally publish no quantitative evaluation |

Honest boundaries: scanned PDFs (no text layer) and formula/body-text parsing
are out of scope — focus traded for speed and lightness.

## What it is

A drop-in operations manual for agents: the user says "extract the figures
from this paper" and the agent loads this skill, drives the
[pdf-figs toolchain](../../) through extraction, validation, and
presentation — no project docs to read, none of the known pitfalls.

Three paths:

| User says | Agent does | Output |
|---|---|---|
| "Extract the figures and show me" | Extract → validate → gallery | Single-file gallery.html |
| "Convert to a dataset / feed an LLM" | Extract → read manifest, report fields | caption / composition / cells |
| "Describe Figure 2 of this paper" | Locate → read panels via composition | Per-panel description |

## Quality baseline (90-paper pdffigures2 GT benchmark, 2026-10)

- Figure F1 **96.5%** · Table F1 **96.7%** (ahead of MinerU basic 94.1%)
- Table axis-completeness **99.0%** · composition recognition **92.0%** (competitors: 0)
- ~0.08s/page (zero models, pure JS), 117/117 unit tests

## Install

> This skill is also installable from its standalone repo:
> `git clone https://github.com/Nanxzi/journal-full-figures-extract-skill`
> (kept in sync with this directory via `git subtree`).

Prerequisite: Node ≥18.

```bash
# 1. Install the toolchain (~1 minute)
git clone https://github.com/Nanxzi/pdf-figure-toolkit
cd pdf-figure-toolkit && npm install

# 2. Install the skill (copy into your agent's skills directory)
mkdir -p ~/.agents/skills
cp -r skills/journal-full-figures-extract-skill ~/.agents/skills/
```

Claude Code users: copy to `~/.claude/skills/journal-full-figures-extract-skill/`.

## Acceptance examples

After installing, say to your agent:

1. **"Extract the figures from this paper and build a gallery"** + any
   scholarly PDF → you should get gallery.html (thumbnails + panels restored
   in original layout + rendered tables)
2. **"How many tables does this paper have? What's in Table 3?"**
   → the agent should read `tables[]` from the manifest and report the count
   and cell contents
3. **"How many panels does Figure 2 have, and in what reading order?"**
   → the agent should answer from `composition` (e.g., "a 2×2 grid,
   top-left → top-right → bottom-left → bottom-right")

## Trigger phrases

提取图表 / 重组图 / 论文图组 / 表格提取 / PDF图表 /
extract figures / paper figures gallery / rebuild multi-panel figures /
composite figure extraction

## Known boundaries

Scanned PDFs (no text layer) are unsupported; image-only tables are out of
scope; see the "Known boundaries" and "Troubleshooting" sections in
`SKILL.md`.

## Version

Released in lockstep with the toolchain: skill v0.2.0 ↔ toolkit v0.2.0.
Change history lives in the main repo ([CHANGELOG](../../), per-line delivery
notes in todo.md).
