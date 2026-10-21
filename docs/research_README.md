# research —— Pine Script 研究 / 试验脚本

本目录不是「可直接交易的指标集」，而是作者（Alex Orekhov / everget）用来**验证 Pine Script 内置函数语义、识别图表类型、试验绘图能力**的一组探针脚本。
几乎每个脚本都在做同一件事：**用两种（或多种）方式算同一个东西，或者用一组特征判断当前图表是什么类型，然后把结果画在图上，让人一眼看出差异。**

所以它们的价值主要在「读代码 + 看图对照」：

- 想知道 `roc()`、`mom()`、`dev()`、`percentrank()`、`percentile_nearest_rank()` 这些内置函数到底怎么算 → 看对应的对比脚本；
- 想知道当前图表是 Heikin-Ashi / Renko / Kagi / P&F / Line Break / Range Bars 中的哪一种 → 看图表类型识别系列；
- 想查阅 `plotshape` 的形状与尺寸、如何画虚线、如何读取符号信息、如何让脚本到期失效 → 看工具类脚本。

> 全部脚本为 `study(...)`（指标），语言版本为 **Pine Script v3 或 v4**（`//@version=` 已标注）。v4 起 `n` 改名为 `bar_index`，v5 起 `transp=` 由 `color.new()` 取代，迁移时注意这些改名。
> 所有脚本都只「画图 / 打标签」，**不下单、不产生交易信号**。

---

## 快速索引

| 脚本 | 版本 | overlay | 一句话作用 | 运行后看到什么 |
| --- | --- | --- | --- | --- |
| `chart_type_identifier.pine` | v6 | 是 | 识别 7 种图表类型（含 Classic）| 最新 K 线最高价处的蓝底标签，如 "This is Renko Chart" |
| `heikin_ashi_chart_identifier.pine` | v6 | 是 | 只判断是否为 Heikin-Ashi | 标签 + 匹配根数 |
| `renko_chart_identifier.pine` | v6 | 是 | 只判断是否为 Renko | 标签 |
| `kagi_chart_identifier.pine` | v6 | 是 | 只判断是否为 Kagi | 标签 + 3 个内部计数 |
| `point_and_figure_pnf_chart_identifier.pine` | v6 | 是 | 只判断是否为 P&F | 标签 + 4 个内部计数 |
| `custom_median_vs_built_in_median.pine` | v6 | 否 | 手写中位数 vs `percentile_nearest_rank` 中位数 | 一条围绕 0 的差值曲线 |
| `mean_absolute_deviation.pine` | v6 | 否 | 7 种平均绝对偏差写法对比 | 7 条曲线，其中 3 条完全重合 |
| `momentum.pine` | v6 | 否 | 4 种动量写法对比 | 4 条曲线，其中 3 条完全重合 |
| `rate_of_change.pine` | v6 | 否 | 4 种变动率写法对比 | 4 条曲线，其中 2 条完全重合 |
| `percentrank_bug.pine` | v6 | 否 | `percentrank()` 与手写实现的差异 | 两条大部分重合、局部错开的曲线 |
| `dashed_line_style.pine` | v6 | 是 | 用 `linebr` 画出「虚线」 | 价格图上的虚线 SMA(14) |
| `fancy_shapes.pine` | v6 | 否 | `plotshape` 形状 × 尺寸样式表 | 7 组、每组 5 层同心图形，沿时间轴排开 |
| `symbol_info_helper.pine` | v6 | 是 | 显示当前品种元信息 | 盘面上的信息卡片 |
| `utc_clock.pine` | v6 | 是 | 图上显示 UTC 时钟 | 大号字体的 `HH:MM:SS` |
| `litecoin_halving_utc_countdown.pine` | v6 | 是 | LTC 减半倒计时 | `Xd HH:MM:SS` 倒计时标签（现已过期） |
| `script_with_expiration_time.pine` | v6 | 否 | 演示「脚本到期失效」 | 到期后整条 close 曲线消失 |

---

## 脚本详解

### 1. `chart_type_identifier.pine` —— 图表类型综合识别

**做什么**：判断当前 TradingView 图表属于哪种画法：Classic（普通 K 线）、Heikin-Ashi、Renko、Kagi、Point & Figure、Line Break、Range Bars，并在图上打一个标签说明结论。

**原理**：不同图表类型在「K 线的微观结构」上留下的指纹不一样，脚本先累积三个计数器，再用它们组合出判断条件。

三个计数器（都从 `-1` 起算，避免首根 K 线误判）：

| 计数器 | 统计条件 | 含义 |
| --- | --- | --- |
| `horizCount` | `open[1] == close or close[1] == open` | **连续性**：上一根与前一根首尾相接（上一根是十字线，或本根开盘等于上一根收盘） |
| `gapsCount` | `low[1] > high or high[1] < low` | **跳空**：相邻两根的高低区间完全不相交 |
| `noWicksCount` | `max(open,close) == high and min(open,close) == low` | **无影线**：实体吃满整根高低区间（折线型图表的基本特征） |

识别条件：

| 判断函数 | 判定规则 | 为什么 |
| --- | --- | --- |
| `_isHA()` | 除首根外，每根都满足 `open == (open[1] + close[1]) / 2`，且无跳空 | Heikin-Ashi 的定义就是 `HA_open = (HA_open[-1] + HA_close[-1]) / 2`，递归成立 |
| `_isRenko()` | 每根实体长度 `body` 都等于历史平均实体长度（`cum(body)/(bar_index+1) == body`），且无跳空 | Renko 砖块高度恒等于设定的 brick size，所有实体等长 |
| `_isKagi()` | 全部 K 线无影线 + 其余 K 线首尾相接 + 无跳空 | Kagi 是以折线绘制的，没有影线，且线是连续的 |
| `_isPnF()` | 收盘方向累计量 `sum` 始终落在 `{-1, 0, 1}` 内，且 **连续性条件不普遍成立**（`horizCount < bar_index - 1`） | P&F 的格子不是按时间连续衔接的，收盘方向不会连续同向累积 |
| `_isLineBreak()` | 全部无影线 + 连续性不普遍成立 + 无跳空，且不是 P&F | 线形图表但没有 P&F 的格子特征 |
| `_isRangeBars()` | `security()` 取 `"1"/"60"/"D"/"W"/"M"` 得到的收盘价全部等于当前 `close`，且当前周期不是 `"S"` | Range Bars 按价格区间而非时间切分，任何时间周期的回取结果都相同 |
| `isClassic` | 以上全否 | 普通 K 线 |

**输出（执行后得到什么）**：一个 `label.style_labeldown` 蓝色标签，固定在最新 K 线，价格位置取最近 `min(bar_index+2, 50)` 根的最高价，文本为
`\nThis is XXX Chart\n`（K 线少于 2 根时显示 "Not enough bars to identify Chart Type"）。
脚本底部有一段被注释掉的 `debug` 拼接，取消注释后可在同一个标签里追加打印 `Gaps / Horiz / No Wicks` 三个计数器，便于定位误判。

**备注**：这是启发式识别，不是严格证明；`_isPnF()` 的假设在极端行情下（持续单边且 K 线数量很少时）可能失效。`security()` 多次回取在实时行情下还会带来额外的数据请求开销。

---

### 2. `heikin_ashi_chart_identifier.pine` —— Heikin-Ashi 识别

**做什么**：只回答一个问题——当前图表是不是 Heikin-Ashi。

**原理**：Heikin-Ashi 的开盘价定义是上一根 HA 的 `(open + close) / 2`，因此在 HA 图上这个等式对**每一根** K 线都成立；同时 HA 图不会出现跳空。脚本维护两个 `var` 计数器：

- `gapsCount`：`low[1] > high or high[1] < low` 时 +1；
- `matchedCandles`：`open == (open[1] + close[1]) / 2` 时 +1。

判定：`matchedCandles == bar_index - 1`（说明从第 1 根起**每根都匹配**）且 `matchedCandles != -1`（排除首根）且 `gapsCount == -1`（一次跳空都没有）。

**输出（执行后得到什么）**：标签显示 "This is Heikin-Ashi Chart" 或否定结论，并附一行
`Bar Index: <n>` 与 `Matched: <数量>`，方便看出是从第几根开始匹配失败的。

---

### 3. `renko_chart_identifier.pine` —— Renko 识别

**做什么**：只判断当前图表是不是 Renko。

**原理**：Renko 每一块砖的实体高度恒等于砖块尺寸，所以「当前实体长度 == 历史平均实体长度」恒成立；另外 Renko 图块之间不重叠、也不留常规跳空。函数 `_isRenko()` 内部用 `var int gapsCount` 统计跳空（`var` 声明在函数内保证跨 bar 保持状态），用 `cum(body) / (bar_index + 1) == body` 判断等长，两者同时满足即为 Renko。

**输出（执行后得到什么）**：标签显示 "This is Renko Chart" 或否定结论，位置在最近 50 根最高收盘价上方。

**备注**：Renko 的 open/close 都是砖块边界，因此实体恒等长；但若砖块尺寸随时间变化（TradingView 的 ATR 型砖块），平均实体会漂移，早期几根可能判不出来。

---

### 4. `kagi_chart_identifier.pine` —— Kagi 识别

**做什么**：只判断当前图表是不是 Kagi（折线图）。

**原理**：Kagi 以「线」的方式绘制，没有影线，线条首尾连续。脚本据此累加三个计数：

| 计数 | 条件 | 含义 |
| --- | --- | --- |
| `ocCount` | `open[1] == close or close[1] == open` | 上一根与本根首尾相接 |
| `hCount` | `open == high or close == high` | 无上影线（最高价就是开/收盘之一） |
| `lCount` | `open == low or close == low` | 无下影线 |

判定：`hCount == bar_index and lCount == bar_index and ocCount == bar_index - 1` —— 即**每一根**都无上下影线，且从第 2 根起全部首尾相接。

**输出（执行后得到什么）**：标签 "This is Kagi Chart" / "This is not Kagi Chart"，并附 `Bar Index`、`O==C Candles`、`OC==H Candles`、`OC==L Candles` 四项计数，位置在最近 10 根最高收盘价上方。

---

### 5. `point_and_figure_pnf_chart_identifier.pine` —— Point & Figure 识别

**做什么**：只判断当前图表是不是 P&F（点数图）。

**原理**：两个特征量：

- `ocCount`：上一根是十字线（`open[1] == close`）或首尾相接（`close[1] == open`）的累计次数；
- `sum`：收盘方向的累计量，`close > close[1]` 时 +1，`close < close[1]` 时 -1，持平不变。

判定：`sum` 始终落在 `{-1, 0, 1}`（价格从不出现连续同向的两步，即收盘价在「涨/平/跌」之间来回振荡）**且** `ocCount < bar_index - 1`（大多数 K 线之间并不首尾相接）。这两条正好是格子式图表的形态：一格一格地排，不与时间轴连续对齐。

**输出（执行后得到什么）**：标签 "This is P&F Chart" / 否定结论，附 `Bar Index`、`Matched Candles`（用 `cum(isPnF ? 1 : 0)` 统计历史上有多少根满足条件）、`O==C`、`Sum` 四项计数。

**备注**：这是四个单类型识别脚本中最容易误判的一个——`sum` 的逻辑一旦出现连续两根同向 K 线就永久跳出 `{-1,0,1}`，因此它更适合用来「证明不是 P&F」，在样本很短时参考价值有限。

---

### 6. `custom_median_vs_built_in_median.pine` —— 手写中位数 vs 内置中位数

**做什么**：验证「Pine 里没有 `median()`，用 `percentile_nearest_rank(src, length, 50)` 能不能当中位数用」，并画出两者差值。

**原理**：

- 内置写法：`median(src, length) => percentile_nearest_rank(src, length, 50)`（最近秩百分位，第 50 百分位）；
- 手写写法：`customMedian()` 通过 `sort(src, length, i)` 取第 `i` 个次序统计量。`sort()` 是一个 O(n²) 的选择过程——外层 `j` 从 0 数到 `i`，每轮在内层 `l` 遍历 `length` 根数据，跳过所有 `<= 当前最小值` 的样本，取「严格大于当前最小值里的最小者」作为下一轮基准，最终得到第 `i` 小的值；
- 偶数长度取中间两个次序统计量的平均（`length/2 - 1` 与 `length/2`），奇数长度取第 `length/2` 个（代码里写成 `length / 2 - 0.5`，靠 Pine 的隐式取整落到同一位置）。

**输出（执行后得到什么）**：单独副图里一条 `diff = 内置 - 手写` 的曲线。

- `length` 为**奇数**时差值基本恒为 0 —— 说明 `percentile_nearest_rank(..., 50)` 与中位数完全等价；
- `length` 为**偶数**时差值可能是一个很小的非零值 —— 最近秩只会返回某个实际样本，而手写版做了两端平均，二者定义不同；
- 若窗口内存在**重复值**，手写 `sort()` 的「严格大于」跳过逻辑会把重复样本算漏，差值会出现尖峰。

---

### 7. `mean_absolute_deviation.pine` —— 平均绝对偏差的 7 种写法

**做什么**：搞清楚内置 `dev()` 到底等价于哪种平均绝对偏差（MAD）算法，顺便排掉几种常见错写。

**原理**：以 `close`、`len = 14`、`sma = sma(close, 14)` 为基准，并列 7 个版本：

| 变量 | 写法 | 说明 |
| --- | --- | --- |
| `dev0` | `dev(src, len)` | 内置基准 |
| `dev1` | `src - sma` | 错：忘了取绝对值 |
| `dev2` | `sum(abs(src - sma), len) / len` | 正确：以 SMA 为中心的**平均绝对偏差** |
| `dev3` | `abs(src - sma) / len` | 错：只用了当前这根，且多除了长度 |
| `dev4` | `(src - sma) / len` | 错：既没绝对值也没求和 |
| `dev5` | `median(src - median(src, len), len)` | 另一个概念：以**中位数**为中心的绝对偏差中位数（MedAD） |
| `dev6` | `for i = 0 to len - 1 { sum += abs(src[i] - sma) }` 后除 `len` | 用循环手写 `dev2` |

**输出（执行后得到什么）**：一张副图 7 条曲线（紫/橄榄/青/绿/红/蓝/橙）。预期看到的结论是：

- `dev0`（内置）、`dev2`（`sum()+abs`）、`dev6`（手写循环）**三条完全重合** → 证明 `dev()` 就是「相对 SMA 的平均绝对偏差」；
- `dev1`、`dev4` 是带正负号的残差类曲线（在 0 附近上下摆动）；
- `dev3` 数值被压缩到很小；
- `dev5` 是另一条不同的、通常更大的曲线（MAD 的中位数版本）。

---

### 8. `momentum.pine` —— 动量的 4 种写法

**做什么**：验证内置 `mom()`、`change()`、手写 `src - src[length]` 三者是否一回事，以及能否用 SMA 的斜率近似动量。

**原理**：`length = 7`，并列四个表达式：

| 变量 | 写法 | 说明 |
| --- | --- | --- |
| `mom` | `src - src[length]` | 手写差分 |
| `mom2` | `change(src, length)` | 内置差分 |
| `mom3` | `mom(src, length)` | 内置动量 |
| `mom4` | `(sma - sma[1]) * length` | 用 `sma(close, 7)` 的单根变化量 × 长度做近似 |

**输出（执行后得到什么）**：副图 4 条曲线（蓝/绿/红/紫）。

- 前三条（`mom`、`mom2`、`mom3`）**完全重合** → 内置 `mom()` 与 `change()` 在数值上等价，都是「当前值减 length 根前的值」；
- `mom4` 是一条更平滑、并且天然带平滑滞后的近似曲线（SMA 斜率 × 长度），只能在趋势中大致贴合，转折处必然落后。

---

### 9. `rate_of_change.pine` —— 变动率的 4 种写法

**做什么**：验证内置 `roc()` 返回的是「百分比」而不是「倍数」，并给出与 `change()` 的关系。

**原理**：`length = 7`，并列四个表达式：

| 变量 | 写法 | 量纲 |
| --- | --- | --- |
| `roc` | `roc(src, length)` | 内置，**百分数**（%） |
| `roc2` | `change(src, length)` | 绝对差额 |
| `roc3` | `change(src, length) / src[length]` | 纯倍数（ratio） |
| `roc4` | `100 * change(src, length) / src[length]` | 百分数 |

**输出（执行后得到什么）**：副图 4 条曲线（蓝/绿/红/紫）。预期：`roc4` 与内置 `roc` **完全重合**，证明 `roc() = 100 * (close - close[length]) / close[length]`；`roc3` 是它的 1/100；`roc2` 是带价格量纲的绝对差额。

---

### 10. `percentrank_bug.pine` —— `percentrank()` 的手写对照

**做什么**：把内置 `percentrank()` 与一个朴素手写实现画在一起，用来排查作者怀疑的「内置百分位排名行为不符合预期」。

**原理**：`src = volume`（成交量重复值多，最容易暴露边界差异），`length` 可调（默认 14）：

- 内置：`pr = percentrank(src, length)`；
- 手写：`_percentrank()` 内部循环 `i = 1 to length`，用 `src >= nz(src[i])` 计数后 `100 * count / length`。

**输出（执行后得到什么）**：副图两条曲线（红=内置，绿=手写）。两者大体同形，但会在**成交量出现重复值**的位置错开——手写版用 `>=` 把「相等」也算作命中，且比较窗口是前 `length` 根（不含自身），而内置实现的计数口径与之不同。

**备注**：这个脚本的价值是把「口径差异」可视化，而不是给出谁对谁错的结论。若要自己实现与内置一致的行为，需要对齐三点：窗口是否包含当前值、平局（相等）算不算命中、除以 `length` 还是 `length - 1`。

---

### 11. `dashed_line_style.pine` —— 用 `linebr` 伪造虚线

**做什么**：在 Pine v3 没有 `plot(..., style=dashed)` 的年代，画出「虚线的 SMA」。

**原理**：`plot()` 的 `style=linebr`（line-break）会在遇到 `na` 时断开线段。脚本利用 v3 的内置变量 `n`（当前 bar 序号，v4 起改名为 `bar_index`）：

```
plot(n % 3 != 0 ? sma : na, style=linebr, linewidth=3, color=black, transp=0)
```

即**每 3 根 K 线里藏掉 1 根**的 `sma(close, 14)`，`linebr` 把空位变成断口，视觉上就成了「2 段实线 + 1 段空隙」重复的虚线。

**输出（执行后得到什么）**：价格主图上一条黑色加粗的虚线状 SMA(14)。修改 `n % 3` 的模数与比较值可以控制虚线的疏密（例如 `n % 2` 得到长短均匀的虚线）。

**备注**：v4+ 可以直接用 `plot(..., style=plot.style_linebr)` 配合 `na` 屏蔽，或用 `line.new` / 折线样式实现，这个技巧现在主要用于「兼容老版本」和「理解 `na` + `linebr` 的断路机制」。

---

### 12. `fancy_shapes.pine` —— `plotshape` 形状 / 尺寸样式表

**做什么**：把 Pine 可用的 7 种 `plotshape` 形状 × 5 种尺寸画成一张对照表，写指标时直接挑样式。

**原理**：7 个形状依次是 `shape.flag`、`shape.square`、`shape.diamond`、`shape.triangleup`、`shape.circle`、`shape.labelup`、`shape.labeldown`；每组再叠 5 层尺寸 `size.huge / large / normal / small / tiny`。

关键参数：

- `location=location.absolute` + `y = 0`：所有图形固定在 y=0 这条水平线上；
- `offset = 0, -20, -40, …, -120`：**offset 是时间轴（横向）偏移**，把 7 个形状组沿时间轴往左依次错开 20 根，避免全部堆在最新一根上；
- `show_last=1`：只有最后一根 K 线上生效，历史 K 线不重复绘制；
- `transp=80`：80% 透明，多层叠加时能看出嵌套关系。

**输出（执行后得到什么）**：副图（无 `overlay`）里出现 7 个「靶心」状图元，从右到左依次是 flag、square、diamond、triangleup、circle、labelup、labeldown，每个都是 5 种尺寸同心叠加。用它可以直接比对「某个形状在 huge 与 tiny 下差多少」，从而决定信号标记该用哪一档尺寸。

---

### 13. `symbol_info_helper.pine` —— 品种信息卡片

**做什么**：在图上打一张信息卡，显示当前品种的元数据，写需要区分市场/品种的指标时用来确认取值来源。

**原理**：把 `syminfo.*` 内置字段拼成多行文本，放在一个 `style=label_left`、左对齐、蓝底白字的标签里，位置固定在 `ohlc4`（(O+H+L+C)/4）价格处：

| 字段 | 内容 |
| --- | --- |
| `syminfo.tickerid` | 带交易所前缀的代码，如 `BINANCE:BTCUSDT` |
| `syminfo.description` | 品种全称 |
| `syminfo.type` | 品种类型（stock / crypto / forex …） |
| `syminfo.pointvalue` | 每点价值（合约乘数） |
| `syminfo.prefix` | 交易所前缀 |
| `syminfo.currency` | 计价货币 |
| `syminfo.mintick` | 最小价格变动单位 |
| `syminfo.timezone` | 交易所时区 |

`Listed Exchange`、`Sector`、`Industry`、`Session` 四项在 Pine 里没有对应内置字段，脚本里用 `"-"` 占位，提醒这一点。

**输出（执行后得到什么）**：盘面 `ohlc4` 附近出现一张信息卡，内容是当前品种的上述字段；切换品种/交易所时会同步变化。

---

### 14. `utc_clock.pine` —— 图上 UTC 时钟

**做什么**：在图表上显示一个实时刷新的 UTC 时钟（跨时区看盘、对齐宏观事件时间用）。

**原理**：`timenow` 是「脚本执行时刻」的毫秒时间戳（Unix epoch，本身即 UTC 基准，与图表时区设置无关）。脚本逐级整除得到总秒/分/时，再取模得到 `% 60`、`% 60`、`% 24`：

```
secondsRaw = floor(timenow / 1000)
hours = floor(minutesRaw / 60) % 24   // 天然是 UTC 小时
```

`f_format()` 负责补零：小于 10 的非零值前面补 `"0"`，等于 0 时补 `"0"`（配合 `tostring(_val, "##")` 输出两位）。

**输出（执行后得到什么）**：主图最高价附近（`highest(high, 50)`）一个无边框大字标签，显示 `HH:MM:SS`。历史 K 线上显示的是**本次脚本执行时刻**的同一个时间（`timenow` 不是 bar 时间），只有最后一根实时 K 线会随 tick 跳动。

**备注**：想要「每根 K 线的收盘时间」应该用 `time` / `time_close`，不是 `timenow`。

---

### 15. `litecoin_halving_utc_countdown.pine` —— LTC 减半倒计时

**做什么**：显示距离莱特币减半事件（2019-08-05 11:55 GMT）还剩多少天多少小时，用来演示「按固定 UTC 时间点做倒计时」。

**原理**：

1. 用 `timestamp('GMT', 2019, 8, halvingDay, halvingHour, halvingMinute)` 构造目标时刻（日/时/分可输入，默认 5 / 11 / 55）；
2. `halvingStamp - timenow` 得到毫秒差，`/1000` 后逐级 `floor` 取整，再用 `% 60 / % 60 / % 24` 拆成天、时、分、秒；
3. `f_format()` 补零后拼成 `"Xd  HH:MM:SS"` 字符串；
4. 用 `style=label.style_none`（无背景）+ `size=size.huge` 的大号标签显示，位置在 `highest(100)`（最近 100 根 K 线的最高价）。

**输出（执行后得到什么）**：主图顶部一个巨大的 `Xd HH:MM:SS` 倒计时标签。

**备注**：目标时间写死在 2019 年，**现在运行只会得到负数**（`floor` 负数后 `%` 运算在 Pine 中的结果与补零函数配合会显示成奇怪的值），因此这个脚本如今只剩「写法参考」价值。要做通用的倒计时，把 `timestamp()` 的日期改成输入参数并对负差值做 `na` 处理即可。

---

### 16. `script_with_expiration_time.pine` —— 脚本到期失效

**做什么**：演示如何给脚本加一个「到期时间」，过期后脚本在图上什么都不画（常用于限时试用、授权控制）。

**原理**：

```
expired = timenow >= timestamp("GMT", 2018, 12, 1, 13, 0) and (barstate.ishistory or barstate.isrealtime)
plot(expired ? na : close, color=purple)
```

- `timenow` 取当前执行时刻，与 GMT 硬编码的截止时间比较；
- `barstate.ishistory or barstate.isrealtime` 保证历史 K 线与实时 K 线都参与判断（否则某些状态下 `expired` 可能为 `na` 而失效）；
- `expired ? na : close`：一旦到期，所有 bar 的输出都是 `na`，`plot` 便整段不绘制。

**输出（执行后得到什么）**：未到期时主图/副图上是一条紫色 `close` 曲线；**过了 2018-12-01 13:00 GMT 之后，整条曲线完全消失**（因为 `timenow` 与具体 bar 无关，一旦过期就所有点都是 `na`），指标仍可加载但什么都不显示。

**备注**：这是「全局开关」式到期，不是「按 bar 截断」。如果想保留历史只停掉未来，应比较 `time`（bar 时间）而不是 `timenow`。

---

## 共性技术点速查

1. **跨 bar 保持状态**：`var int cnt = -1` 配合条件自增，是本目录识别类脚本的统一写法；注意 `var` 放在函数内部同样能保持状态（见 `renko_chart_identifier.pine`）。
2. **从 `-1` 起算**：这样「第 0 根什么都不可能匹配」自然成立，再用 `!= -1`、`== bar_index - 1` 之类条件排除首根，避免首根误判为「全部匹配」。
3. **两种实现并排画**：对比类脚本一律用同一条 `plot` 序列 + 不同颜色，重合即证明等价，错开即说明口径不同——这是读本目录脚本最快的判断方式。
4. **`overlay` 决定画在哪**：`study(..., overlay=true)` 画在价格上（识别、时钟、信息卡）；不写 `overlay` 则单独开副图（对比类、样式表）。
5. **`na` + 专用样式**：`linebr`（断层）、`style_none`（无背景标签）、`na` 遮罩，是 v3 时代实现虚线/隐藏/条件绘制的主要手段。
6. **`security()` 的用途不限于取高周期数据**：`chart_type_identifier.pine` 用它「同一价格在不同时间周期上是否一致」来反推 Range Bars。
7. **版本迁移注意**：v3 的 `n` 在 v4 改名为 `bar_index`；v4 的 `transp=` 在 v5 由 `color.new(color, alpha)` 取代；`input(type=integer)` → `input.int()`；`style=linebr` → `style=plot.style_linebr`。

## 使用建议

- 这些脚本**不是交易策略**，不要据此下单；它们的作用是「验证语义 / 认图表 / 查样式」。
- 想验证某个内置函数：直接复制对应对比脚本，把你的表达式加进去、换个颜色 `plot` 出来，看是否与内置重合。
- 图表类型识别系列建议在「同一品种、同一段数据、分别切换图表类型」的条件下测试，观察标签能否随类型正确翻转；`chart_type_identifier.pine` 底部的 `debug` 注释块是排错的主要工具。
