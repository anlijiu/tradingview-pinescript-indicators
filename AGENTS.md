# AGENTS.md

## 目录与文档对应关系

每个指标分类目录在 `docs/` 下有且仅有一个对应的说明文档，一一对应：

| 指标分类目录 | 对应文档 |
| --- | --- |
| `bands_and_channels/` | `docs/bands_and_channels_README.md` |
| `highlighters/` | `docs/highlighters_README.md` |
| `movings/` | `docs/movings_README.md` |
| `oscillators/` | `docs/oscillators_README.md` |
| `research/` | `docs/research_README.md` |
| `statistics/` | `docs/statistics_README.md` |
| `trailing_stops/` | `docs/trailing_stops_README.md` |
| `volatility/` | `docs/volatility_README.md` |
| `volume/` | `docs/volume_README.md` |

## 强制约束：指标改动必须同步更新文档

上述分类目录中的**指标一旦更新，必须在同一次变更中同步更新 `docs/` 下对应的文档**，不允许只改 `.pine` 文件而留下过期文档。

「指标更新」包括但不限于：

- 新增、删除、重命名分类目录下的 `.pine` 脚本
- 修改指标计算逻辑、默认参数、`input` 项（名称、类型、默认值）
- 升级 Pine Script 版本，或改动 `//@version` 相关的语法/函数
- 改动 `plot` / `fill` / `hline` 等绘图输出、颜色、样式
- 新增或修改 `alertcondition` / 告警逻辑、`strategy` 行为
- 修复会改变指标数值结果的 bug

文档中需要同步的内容：

- **文件清单表格**：文件名、指标（简称）、Pine 版本、默认参数、关键来源/带宽来源
- **指标说明段落**：做什么 / 原理与公式 / 参数含义 / 用法与解读
- 分类内的指标数量、概览分类描述（如「本目录收录 N 个……」）
- 若文档结构或目录发生变化，同步更新本文件末尾的 `docs/` 目录树

## `docs/` 目录树

```
docs/
├── bands_and_channels_README.md
├── highlighters_README.md
├── movings_README.md
├── oscillators_README.md
├── research_README.md
├── statistics_README.md
├── trailing_stops_README.md
├── volatility_README.md
└── volume_README.md
```

> `utils/` 为公共工具脚本目录，当前没有对应文档，不在上述约束范围内。
