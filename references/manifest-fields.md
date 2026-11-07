# manifest 字段速查

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

composition.kind 取值：single / row / column / grid / irregular-grid /
freeform / hybrid。order 元素为 `panel-N`（阅读顺序号，与
`--panels` 产出的 `{figId}_panel-N_p{P}.jpg` 一一对应）。
