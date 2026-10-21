# statistics —— Pine Script 统计 / 分布 / 基本面聚合脚本

本目录收录 17 个用 Pine Script 写的**统计类**指标（7 个 `//@version=3`、9 个 `//@version=4`、1 个 `//@version=6`）。
它们不是交易信号，也不是技术指标，而是把「历史数据里发生过什么」量化出来：

- 现在离历史最高点有多远 / 多久没创新高；
- 涨跌幅、跳空、连阳连阴的**分布形状**（直方图）；
- Z-Score、Kendall 秩相关、动量加速度（文件名叫 Kurtosis）等**统计量**的计算方式；
- 全历史**线性回归**、ROI、YTD 这类**收益率度量**；
- 把**财报（DPS / EPS）、美债收益率、历届总统任期涨幅**等外部数据聚合成柱状对比图；
- 以及对一段行情做「体检」的 **Candles History** 汇总器。

> 所有脚本都只有 `study()` / `indicator()`，**没有 `strategy()`、不下单、不产生买卖信号**（少数脚本带 `alertcondition`，但那是「统计值穿越阈值」的提醒，不是交易指令）。
> 所有统计都基于**当前图表加载到的历史**：默认状态下 TradingView 只加载有限根 K 线，`cum()` 类累计值会随「图表历史长度 / 付费套餐的历史上限」变化，换周期、换品种、换区间，结果就不同 —— 这是读本目录所有脚本的前提。

---

## 快速索引

**一、历史极值**

| # | 脚本 | 版本 | overlay | 一句话作用 | 运行后看到什么 | 外部数据 |
| --- | --- | --- | --- | --- | --- | --- |
| 1 | `all_time_high_days.pine` | 6 | 否 | 距上一次创新高过了多少根 K 线 | 副图一条紫线：0 = 当根创新高，N = 已 N 根没破前高 | — |
| 2 | `all_time_high_percent_drop.pine` | 6 | 否 | 收盘价相对历史最高价的回撤百分比 | 副图一条紫线 + 0%…−100% 的彩色刻度线 | — |
| 3 | `bitcoin_ath_hash_rate_level.pine` | 6 | **是** | 算力创新高那一刻的 BTC 价格水平线 | 主图一条阶梯黑线；可选按涨跌变色 | `QUANDL:BCHAIN/HRATE` |

**二、分布 / 直方图**

| # | 脚本 | 版本 | overlay | 一句话作用 | 运行后看到什么 | 外部数据 |
| --- | --- | --- | --- | --- | --- | --- |
| 4 | `close_to_close_percent_change_distribution.pine` | 6 | 否 | 日涨跌幅（收盘对收盘）的直方图 | 副图 50 根柱（±0.25% 步长到 ±6%） | — |
| 5 | `gaps_percent_size_distribution.pine` | 6 | 否 | 跳空幅度百分比的直方图（可调分桶） | 副图每边 150 根柱（可调到 250），正跳空在右、负跳空在左 | — |
| 6 | `bullish_bearish_candle_series_distribution.pine` | 6 | 否 | 连续同向 K 线「段长」的频次分布 | 副图 58 根柱：列标签 N = 段长，柱高 = 出现次数 | — |

**三、收益率度量**

| # | 脚本 | 版本 | overlay | 一句话作用 | 运行后看到什么 | 外部数据 |
| --- | --- | --- | --- | --- | --- | --- |
| 7 | `roi_return_on_investment.pine` | 6 | 否 | 从指定日期（或首根）持有至今的收益率 | 副图 ROI% 曲线，盈利绿填充、亏损红填充 | — |
| 8 | `ytd_year_to_date_percent_return.pine` | 6 | 否 | 年初至今收益率 | 副图 YTD% 曲线 + 6 条自定义水平线 | — |
| 9 | `linear_regression_all_data.pine` | 6 | **是** | 全历史最小二乘回归 | 主图一条橙色回归线 + 蓝色信息卡（slope/intercept/r/r²） | — |

**四、统计量**

| # | 脚本 | 版本 | overlay | 一句话作用 | 运行后看到什么 | 外部数据 |
| --- | --- | --- | --- | --- | --- | --- |
| 10 | `z_score.pine` | 6 | 否 | 价格相对均值的 Z 分数 | 副图一条线，\|z\|>2 时变红/绿 | — |
| 11 | `kurtosis_indicator.pine` | 6 | 否 | 动量「加速度」及其两条均线（名字叫峰度） | 副图两条线：Fast / Fast-Slow | — |
| 12 | `kendall_rank_correlation_coefficient.pine` | 6 | 否 | 两个品种的 Kendall τ 与 Pearson r | 副图两条线：橙 = Kendall、紫 = Pearson | 两个 symbol 输入 |

**五、外部数据聚合**

| # | 脚本 | 版本 | overlay | 一句话作用 | 运行后看到什么 | 外部数据 |
| --- | --- | --- | --- | --- | --- | --- |
| 13 | `dividends_per_share_dps_yearly.pine` | 6 | 否 | 每年每股分红总额（近 21 年） | 副图 21 根橙柱，最右 = 今年 | `ESD_FACTSET` |
| 14 | `earnings_per_share_eps_yearly.pine` | 6 | 否 | 每年每股收益总额（近 21 年） | 与 #13 完全相同，只是指标换成 EPS | `ESD_FACTSET` |
| 15 | `us_treasury_yields.pine` | 6 | 否 | 11 个期限的美债收益率快照 | 副图柱状（或折线）排一排：1M→30Y（30Y 在最右） | `TVC:US*Y` |
| 16 | `ticker_performance_by_us_president.pine` | **6** | 否 | 历届美国总统任期内的累计涨幅 | 副图一排 24 根柱，每根一位总统 | — |

**六、行情体检**

| # | 脚本 | 版本 | overlay | 一句话作用 | 运行后看到什么 | 外部数据 |
| --- | --- | --- | --- | --- | --- | --- |
| 17 | `candles_history.pine` | 6 | 否 | 行情「体检」：28 项指标任选其一 | 副图一条红线，**读最后一个值** | — |

细节备注：`bullish_bearish_candle_series_distribution.pine`（58 `plot` + 6 `plotshape` = 64）与 `close_to_close_percent_change_distribution.pine`（50 `plot` + 13 `plotshape` = 63）的绘图对象数量已经顶到 / 逼近 Pine 的绘图数量上限（64），想自己扩桶、扩段长的话先确认还能不能编译通过。

---

## 三种「把统计结果画出来」的套路

本目录所有脚本的视觉输出，本质只有三种做法。认清这一点，17 个脚本的原理就都通了。

### 套路 A：累计值曲线（最朴素）

直接用 `cum()` / 递归变量算一条随时间变化的序列，然后 `plot()` 成一条线。

代表：`all_time_high_days`、`all_time_high_percent_drop`、`roi_*`、`ytd_*`、`z_score`、`kendall_*`、`kurtosis_*`、`candles_history`。

局限：它画出的是「**在每根 K 线上，统计量此时的值**」，而不是「一个直方图」。所以要看最终结论，得看**最后一根的值**（数据窗口里的最后一行）。

### 套路 B：offset + show_last=1 —— 用 K 线的位置冒充横轴

这是本目录最有辨识度的技巧，几乎所有柱状/直方图脚本都用它：

```pine
plot(value, offset=(-3 * space), style=plot.style_histogram, linewidth=width, show_last=1)
```

- `show_last=1`：只在**最后一根 K 线**上绘制这个值；
- `offset=-3 * space`：把这一根柱子**向左平移** 3×space 根 K 线的宽度；
- 于是「不同的桶」被摆到当前 K 线的左侧不同位置上，视觉上就成了一张**柱状图**，而横轴其实是时间轴的空白区。

`plotshape(..., location=location.absolute, text=" 10 ")` 再在同样的 offset 位置钉一个标签，就得到刻度文字。

这套做法在 v3 时代几乎是画直方图的唯一手段；v4 起也可以用 `line.new` / `box.new` / `table.new` 做得更灵活（本目录的 `gaps_percent_size_distribution.pine` 就是 v4 的 `line.new` 版本，见套路 C）。

### 套路 C：`line.new()` / `label.new()` 手工绘图

`gaps_percent_size_distribution.pine` 用的是 v4 的绘图对象：在 `barstate.islast` 时用 `line.new(xloc=xloc.bar_time, x=time[n], ...)` 把柱状图画到「最近的 N 根 K 线」的位置上，用 `label.new()` 画刻度，并用 `line.delete(l[1])` 删掉上一轮的对象（实时行情每个 tick 都会重跑最后一段代码，不删就会堆叠）。

好处是桶数可以动态（数组里存计数），不像套路 B 要硬编码每一根 `plot`。

---

## 脚本详解

### 一、历史极值（All-Time High）三件套

#### 1. `all_time_high_days.pine` —— 距上次创新高多少根

**做什么**：回答「这只票/这个币已经多少根 K 线没创历史新高了」。

**原理**：

```pine
ath = high
ath := na(ath[1]) ? ath : (high > ath[1] ? high : ath[1])   // 递归求 high 的历史最大值
cond = high == ath                                          // 当根是否创新高
bars = barssince(cond)                                       // 距上一次 cond 为真的 bar 数
```

三行分别是：**历史最高价（running max）**、**当根是否刷新它**、**自上次刷新以来经过的根数**。`barssince` 在条件成立的那根返回 0，之后逐根 +1，所以它天然就是「多久没创新高」。

**输出**：副图一条紫色曲线。规律是**锯齿状**：创新高的那根落在 0，然后一路爬升，直到下一次创新高再掉回 0。曲线的高度 = 这段「无新高」持续了多久。输入 `Highlight ATH ?` 打开后，创新高的当根会打橙色背景。

**备注**：用的是 `high`，所以「影线扫一下前高」也算创新高；ATL 角度完全对称，把 `high` 全换成 `low`、`>` 换成 `<` 即可。

#### 2. `all_time_high_percent_drop.pine` —— 相对历史高点的回撤百分比

**做什么**：算出「当前收盘价距离历史最高价跌了多少 %」，并配一套从 0% 到 −100% 的彩色刻度线。

**原理**：

```pine
ath = high                      // 同样是 high 的 running max
percent = 100 * (close - ath) / ath
```

分数线上的每个刻度都是「一对」`plot(常量, style=circles, linewidth=0)` + `hline(常量)`：`plot` 只负责在左侧图例里生成一个可点的项，`hline` 才是真正的横线，颜色从灰（0%）经黄、橙（−50%）渐变到红（−100%）。

**输出**：副图一条紫色回撤曲线，恒 ≤ 0；曲线贴近哪条刻度线，就是跌了多少。−10%、−20%、−30% 这几条是加密/股票里常用的「熊市分级」参考。

**备注**：**分子用 `close`、分母用 `high` 的 running max**，所以最大回撤会比你用「收盘价历史最高」算出来的略深一点点（因为基准取的是盘中最高价）。

#### 3. `bitcoin_ath_hash_rate_level.pine` —— 算力新高时的价格水平

**做什么**：每当比特币全网算力（hash rate）创历史新高，就把**当天的 BTC 价格**记下来画成一条水平线；之后价格在这条线上方/下方，用来观察「算力新高 ↔ 价格」的相对位置。

**原理**：

```pine
hashRate = security("QUANDL:BCHAIN/HRATE", "D", close)   // 外部日线算力数据
hrATH = _ath(hashRate)                                   // 算力的 running max
hrathLevel = close
hrathLevel := hrATH > hrATH[1] ? close : nz(hrathLevel[1], close)
                                                 // ↑ 算力创新高 → 记录当根收盘价；否则沿用上一条水平
```

这是一个「**阶梯线**」：只在算力刷新高的那些 bar 上更新数值，其余时间保持不动。`alertcondition(hrATH != hrATH[1], ...)` 在算力刷新高时发提醒。

**输出**：**主图**（`overlay=true`）一条 2px 黑色阶梯线，代表「最近一次算力创新高时的 BTC 价格」。打开 `Highlight Price Movements ?` 后，收盘价高于该水平显示绿色、低于显示红色、持平显示紫色。

**备注**：
- 数据依赖 `QUANDL:BCHAIN/HRATE`，Quandl 的免费数据集在 TradingView 上已大幅收缩，**如果图上什么都没画，先确认这个代码是否还能取到数据**（换成 `BTC.D`、某个算力类指标或自带数据源）。
- `security()` 请求日线数据，在日线以上周期（周/月）会产生「当日未收盘」的取整问题，最后一段会随盘中变化。

---

### 二、分布 / 直方图类

#### 4. `close_to_close_percent_change_distribution.pine` —— 日涨跌幅的直方图

**做什么**：把历史上每一根 K 线的「收盘对收盘涨跌幅」分桶统计，画出**分布直方图**——这只标的平时是 ±0.5% 的小波动为主，还是经常 ±3% 的大波动，一眼可见。

**原理**：

```pine
roc = roc(close, 1)      // 单根涨跌幅（%）
total = bar_index + 1    // 分母：全部 K 线数

p025 = cum(0 <= roc and roc < 0.25 ? 1 : 0)     // 落入 [0, 0.25) 的根数
plot(100 * p025 / total, offset=(-23 * space), style=plot.style_histogram, linewidth=width, show_last=1)
```

- **分桶**：0.25% 一档，正向 24 档（0~6%）+ 「≥6%」溢出档；负向同样 24 档 + 「≤−6%」溢出档，共 **50 个桶**；
- **计数**：每档一个 `cum(条件 ? 1 : 0)`，即「历史上有多少根落进这个区间」；
- **归一化**：除以 `total` 再 ×100，变成**占比 %**（柱高是百分比，不是根数）；
- **布局**：靠 `offset` 把 50 个桶排到当前 K 线左侧（−48×space 到 +1×space），负向在最左、正向在最右，越靠右幅度越大。刻度标签用 `plotshape(0, offset=(-floor(23.5*space)), text=" 0% ")` 钉在正负分界处，`1%`、`2%`…`−5%` 同理。

**输出**：副图一张 50 列的直方图，柱高 = 该幅度区间出现的频率（%）。横轴刻度有 `≤−6% −5% … 0% … 5% ≥6%`。

**备注**：
- 分母是 `bar_index + 1`，其中**第一根 K 线的 `roc` 是 `na`**（没有前收盘），所以所有桶加起来略低于 100%；
- 分桶边界是「左闭右开」：正向 `0 <= roc < 0.25`、负向 `-0.25 < roc <= 0`，正负两侧无重叠也无空档；
- 桶宽写死在代码里，想改步长（比如加密用 1%）需要改 50 行条件 —— 这个脚本是「一次成型」的写法，`gaps_percent_size_distribution.pine` 那个才是可调版本。

#### 5. `gaps_percent_size_distribution.pine` —— 跳空幅度分布（可调分桶）

**做什么**：统计「开盘价相对上一根收盘价」跳空了多少 %，并按可调的桶宽画出直方图。用来判断一个品种是「经常跳空」还是「几乎不跳空」，以及跳空幅度大致多大（对网格/挂单/止盈止损距离设置有直接参考价值）。

**原理**：

```pine
isGapUp   = close[1] < open
isGapDown = close[1] > open
size = 100 * (open / close[1] - 1)          // 跳空幅度（%）
```

- `size` 为正 → 向上跳空，落到第 `i` 个正桶的条件是 `i*step <= size < (i+1)*step`；
- `size` 为负 → 向下跳空，落到第 `i` 个负桶的条件是 `-(i+1)*step < size <= -i*step`；
- 桶数组 `posValues` / `negValues` 各 `floor(maxColumns/2)` 个（默认 300 档 → 每边 150 个桶）；
- 最后在 `barstate.islast` 时用 `line.new()` 把每个桶画成一根柱子（x 坐标取 `time[sizeLimit - i]`，也就是「借最近 N 根 K 线的时间位置」当横轴），用 `label.new()` 每 `nthLabel` 个桶标一次区间文字，tooltip 里放该桶的根数。

**输出**：副图一张镜像直方图 —— **右半边 = 向上跳空**、**左半边 = 向下跳空**，柱高 = 该幅度区间出现过的次数（`tooltip` 悬停可看精确根数）。标签文字形如 `1.5%\n-\n1.75%`。

**输入参数**：`Binning Step`（桶宽，默认 0.25%）、`Columns Number`（总列数上限 500）、`Columns Width`（柱宽像素）、`Show Labels` / `Show Each Nth Label`（刻度密度）。`Reboot` 是个「出错就点一下」的重置开关（v4 里 `var` 输入控件触发重算用）。

**备注**：
- 桶数受 `max_lines_count=500` 限制，`maxColumns` 拉到 500 时线对象正好用满，**再往上加会因为超出上限被截断**；
- 用的是**开盘价**（`open`）对前收盘，不是收盘价对收盘价 —— 它衡量的是「隔夜/隔根跳空」，与 #4 的涨跌幅分布是两个互补的口径；
- 横轴借用的是「最近 N 根 K 线的时间」，所以**图上柱子位置和真实时间无关**，纯当直方图的横轴看；
- 与 #4 一样，第一根 K 线的 `close[1]` 为 `na`，不进入任何桶。

#### 6. `bullish_bearish_candle_series_distribution.pine` —— 连阳/连阴「段长」分布

**做什么**：统计「连续 N 根同向 K 线」这种**段**出现了多少次。用来回答：这品种是走「三连阳后大概率回落」还是「一阳到底」；连阳 5 根以上是不是罕见事件。

**原理**：

```pine
cond = mode == "Bullish" ? close >= open : ...    // 四种模式：Bullish / Strict Bullish / Bearish / Strict Bearish

var int count = 0
count := na(count[1]) ? count : (cond ? count[1] + 1 : 0)   // 当前连续同向计数，反向即归零

len = nz(count[1]) + 1          // 段结束时：上一根的连续数 + 1 = 这段的长度
if count == 0                   // 只在「段刚被打断」的这根 bar 上记账
    serie2 := len == 2 ? serie2 + 1 : serie2
    serie3 := len == 3 ? serie3 + 1 : serie3
    ...
```

- `count` 是标准的「连续同向计数器」；
- 当出现第一根反向 K 线时，`count` 归 0，此刻 `count[1]` 就是刚结束那段的长度，于是把对应的 `serie{len}` 计数器 +1；
- `serie2 … serie59` 共 58 个 `var int` 计数器，写死在代码里（文件里保留了一段被注释掉的 JavaScript 生成脚本，说明这 58 行是代码生成的）；
- 最后每个计数器一个 `plot(..., offset=(-i*space), style=histogram, show_last=1)`，把「2 段、3 段、…、59 段」排成一排柱子；`plotshape` 每 10 段钉一个刻度标签。

**输出**：副图 58 根橙色柱子。**列标签 N 是段长记号（口径见下条备注）、柱高表示出现次数**（绝对次数，不是百分比）。默认 `Bullish` 模式下最常见的通常是 2~3 段，之后快速衰减形成一条**右偏长尾**。

**输入参数**：`Mode`（`Bullish` = `close >= open` 含十字星；`Strict Bullish` = 严格 `close > open`；Bearish 两档同理）、`Columns Width`、`Space Between Columns`、`Hide 2-Series ?`（隐藏最短的那一档，因为它的柱子通常压得别的柱子看不见）。

**备注**：
- 记账口径是 `len = count[1] + 1`，即**把「打断这段的那根 K 线」也算进段长**。所以列标签 `N` 对应的实际连续根数是 `N − 1`；自己对照 TradingView 的「连阳统计」时会有 1 根的偏差；
- 只统计「已经结束」的段：**数据最后一根仍在延续的段不会被记账**（所以要判断「当前这段已经 7 连阳了」，得看图上实时值或另写脚本）；
- 连续多根反向 K 线时，`count == 0` 会反复成立，但每次 `len` 都算成 1（无对应桶），不会污染统计；
- 58 个 `plot` + 6 个 `plotshape` = **64 个绘图对象，正好是 Pine 的绘图数量上限**。这也解释了作者为什么把段长停在 59：再加一根 `plot` 就编译不过了。

---

### 三、收益率度量

#### 7. `roi_return_on_investment.pine` —— 指定起点持有至今的 ROI

**做什么**：假设在某个日期（或数据第一根）的**开盘价**买入并一直持有，现在赚了多少 %。附带「止盈/止损水平」的可视化与两条提醒。

**原理**：

```pine
startTime = timestamp(yearStart, monthStart, dayStart, 0, 0)      // 起点（交易所时区）

index := na(index[1]) ? valuewhen(time(timeframe.period) >= startTime, bar_index, 0) : index[1]
initialCond = useDate ? bar_index == index : bar_index == 0          // 定位起点那根
initialRegular = valuewhen(initialCond, open, 0)                     // 起点那根的开盘价 = 成本

roiRegular = 100 * (close - initialRegular) / initialRegular
```

- `index` 用 `na(index[1]) ? ... : index[1]` 这种写法「只求值一次并记住」，得到**起点所在 bar 的序号**；
- `valuewhen(initialCond, open, 0)` 取出那根的开盘价当成本；
- 之后 `roi` 就是最朴素的 `(现价 − 成本) / 成本 × 100`。

**输出**：副图一条黑色 ROI% 曲线，配合三层填充：

| 元素 | 含义 |
| --- | --- |
| 绿色/红色填充（曲线与 0 线之间） | ROI 为正填绿、为负填红（`transp=80`） |
| 黄色小圆点虚线 `tp`（`Profit Level, %`） | 止盈水平；`roi > tp` 时曲线与止盈线之间填绿（`transp=60`） |
| 黄色小圆点虚线 `sl`（`Loss Level, %`，只能是 ≤0） | 止损水平；`roi < sl` 时填红 |
| 橙色竖条背景 | 每换一年打一根，方便数「持有几年」 |

另外 `alertcondition(crossover(roi, tp))` / `(crossunder(roi, sl))` 可以挂提醒。

**备注**：
- 代码里还留着一份 `Annualized`（年化）实现，但 `mode` 输入和 `roiAnnualized` 都被注释掉了，**当前只有普通口径**；想恢复年化，取消注释 `mode`、`bars`、`power`、`roiAnnualized` 并把 `roi = roiRegular` 改成三目判断；
- `tp` 默认 0、`sl` 默认 0，此时填充条件都不成立 —— **要看到止盈/止损填充必须自己填正/负值**；
- `index` 只在第一次满足 `time >= startTime` 时取到值，若把起点设得比数据还早，`valuewhen` 会返回 `bar_index = 0`，等价于「从数据首根算起」。

#### 8. `ytd_year_to_date_percent_return.pine` —— 年初至今收益率

**做什么**：每年年初重新起算的收益率曲线（YTD），并给 6 条可自定义的水平线（比如 ±10%、±20%、±30%）。

**原理**：

```pine
_ytd(yearTarget) =>
    yearCond = year != year[1]                              // 每年第一根
    yearOpen = valuewhen(yearCond, open, yearTarget)         // yearTarget=0 → 最近一次跨年的开盘价
    ytd = 100 * (close - yearOpen) / yearOpen
    ytd
```

核心就一句 `valuewhen(跨年条件, open, 0)`：取**最近一次跨年那根的开盘价**当基准。每到新年，基准自动重置，所以曲线**每年重新起算**（年初第一根的值是 `(首根收盘 − 首根开盘)/首根开盘`，接近但一般不等于 0），形成一条条「年度锯齿」。

**输出**：副图一条黑色 YTD% 曲线（正绿负红填充），叠加 6 条紫色点线水平线（默认全 0，都在中线），以及每年的橙色背景分界。6 组 `crossover`/`crossunder` 提醒分别对应 6 个水平。

**备注**：跨年那根的 `open` 是新年第一根的开盘价 —— 如果图表周期很大（月线以上）或数据有缺口，「年初」会落在第一根可用的 K 线上。

#### 9. `linear_regression_all_data.pine` —— 全历史线性回归

**做什么**：对**图表加载到的全部历史数据**做一次最小二乘线性回归，画出回归线，并把斜率、截距、相关系数 r、决定系数 r² 显示成一张信息卡。

**原理**：用 `cum()` 累积和实现「一遍扫描」的最小二乘，不需要循环：

```pine
n = bar_index + 1
X = n;  Y = close

sumX = cum(X);      sumY = cum(Y)
sumX2 = cum(pow(X,2));  sumY2 = cum(pow(Y,2))
sumXY = cum(X * Y)

meanX = sumX / n;   meanY = sumY / n

slope     = (n*sumXY - sumX*sumY) / (n*sumX2 - pow(sumX,2))
intercept = meanY - slope * meanX
r  = (n*sumXY - sumX*sumY) / sqrt((n*sumY2 - pow(sumY,2)) * (n*sumX2 - pow(sumX,2)))
r2 = pow(r, 2)
```

这就是教科书的 OLS 闭式解（把 `X` 取成 bar 序号，`Y` 取成收盘价）。`r` 用同一个分子除以 `X`、`Y` 各自离差平方和的几何平均，`r2 = r²` 表示「时间能解释多少价格方差」。

**输出**（**主图**，`overlay=true`）：
1. 一条橙色回归直线：从「最近 5000 根之前」画到最新一根（`MAX_BARS_BACK = 5000`，为了不画出几千根的长线也为了符合 `max_bars_back` 限制）；
2. 一张蓝底白字的 `label`，贴在回归线右端（`endY`）向右偏移约 10 根的位置，内容是
   `Slope / Intercept / r / r2`（分别保留 6/6/3/3 位小数）。

**备注**：
- 线的**起点被限制在最近 5000 根**，但**斜率和 r 仍是用全部历史算的** —— 所以图上那截线可能看起来和整体趋势不完全贴合；
- `X` 用的是 bar 序号（等间距时间轴），所以**「非交易日被跳过」的品种（股票）等价于按交易日回归**，不是按日历日；
- 价格本质是随机游走，`r` 通常很低且**完全没有预测含义**，这里只用来看「这段历史趋势的线性程度」；
- `Y = input(title="Source", defval=close)` 只是个 float 输入（默认 close），不像 v4 的 `type=input.source` 那样给下拉选择器。

---

### 四、统计量计算

#### 10. `z_score.pine` —— Z 分数

**做什么**：`(价格 − 均值) / 标准差`，即「当前价格偏离均值的几个标准差」，经典均值回归度量。

**原理**：

```pine
// 样本标准差（分母 n−1）的手写实现
sampleStdev(src, length) =>
    dev = src - sma(src, length)
    sqrt(sum(dev*dev, length) / (length - 1))

selectedStdev = stdevInput == "Sample" ? sampleStdev(src, length) : stdev(src, length)
zsc = (src - sma(src, length)) / selectedStdev
```

内置 `stdev()` 是**总体标准差**（分母 `n`），手写函数是**样本标准差**（分母 `n−1`），脚本用输入项让用户二选一。二者比值是 `sqrt(n/(n-1))`，`length=14` 时差约 4%，`length` 越小差别越大。

**输出**：副图一条 2px 曲线（`zsc`），配合 `±2.0` 两条虚线。颜色是**分段**的：`zsc > 2` 绿色、`zsc < −2` 红色、其余紫色。所以「超过 2 个标准差」的两段会明显变色。

**备注**：这里是**滚动窗口**（`length`，默认 14）的 z 分数，不是全历史 z 分数；想用「全历史均值/标准差」的话，把 `sma`/`stdev` 换成 `cum` 版本（做法可参考 #9）。

#### 11. `kurtosis_indicator.pine` —— 名字叫峰度，其实是「动量加速度」

**做什么**：一个**动量二阶差分**指标（版权标注 2018）。虽然标题写着 Kurtosis（峰度），但代码里**没有任何统计峰度的计算**（没有四阶矩、没有 `(x−μ)⁴/σ⁴`），它本质是：

```pine
K = change(src, 3) - change(src[1], 3)      // 一阶动量的变化 = 二阶差分（加速度）
FK  = MA(K, 66)                              // 快线：对加速度做长平滑
FSK = MA(FK, 3)                              // 慢线：对快线再做短平滑
```

`change(src, n)` 是 `src − src[n]`，所以 `K ≈ (P_t − P_{t−3}) − (P_{t−1} − P_{t−4})`，衡量的是**涨速本身在加速还是减速**。作者用「长周期平滑（66）+ 短周期再平滑（3）」把这个噪声很大的二阶量压成两条可读的曲线，常见的读法是 FK 与 FSK 的交叉、以及二者与 0 线的关系。

**输出**：副图两条 2px 曲线（`Fast` 橙粉、`Fast / Slow` 深紫）+ 一条 0 虚线。四条 MA 类型（EMA/SMA/VWMA/WMA）都可以通过输入切换，默认 `EMA(66)` 与 `SMA(3)`。

**备注**：
- **名字与内容不符**：这不是峰度。真正的滚动峰度需要四阶中心矩，本目录没有实现；
- 两条线的**量纲随价格绝对水平变化**（是价格差，不是百分比），所以在 40000 的 BTC 和 1.1 的 EURUSD 上数值完全不可比；
- 常用参数 `momentumLength=3 / fast=66 / fastSlow=3` 是作者为日线外汇调的，换品种要重调。

#### 12. `kendall_rank_correlation_coefficient.pine` —— Kendall τ vs Pearson r

**做什么**：把两个品种的**秩相关**（Kendall τ）和**线性相关**（Pearson r）画在一起对比。秩相关只看「涨跌顺序是否一致」，对异常值和量纲不敏感；Pearson 看的是线性关系强弱。

**原理**：

```pine
data1 = security(sym1, period, src)     // period = 当前图表周期（v3 内置变量）
data2 = security(sym2, period, src)

getKendall(data1, data2, length) =>
    totalPairs = length * (length - 1) / 2          // 所有无序对的数目
    numerator = 0.0
    for i = 0 to length - 1
        for j = 0 to i                              // j <= i，只枚举一半的配对
            numerator := numerator + sign(data1[i] - data1[j]) * sign(data2[i] - data2[j])
    numerator / totalPairs

kendallCorr = sym1 == sym2 ? 1 : getKendall(data1, data2, length)
pearsonCorr = correlation(data1, data2, length)
```

- 双重循环枚举窗口内所有 K 线对，`sign(ΔA) * sign(ΔB)` 是 +1（同向）、−1（反向）、0（持平）；
- 全部累加后除以「无序对总数」`n(n−1)/2`，得到 **Kendall τ-a**，取值在 [−1, 1]；
- `j` 从 0 到 `i`（含 `i == j`）时 `sign(0)*sign(0) = 0`，对分子无贡献，所以归一化还是对的；
- Pearson 直接用内置 `correlation()`。

**输出**：副图两条曲线：**橙色 = Kendall τ**、**紫色 = Pearson r**，都在 [−1, 1]。两者同高说明关系既单调又线性；Kendall 明显高于 Pearson 说明关系是单调但非线性的（比如黄金与白银、BTC 与 ETH 这类）。

**备注**：
- 默认 `sym1 = FX:AUDUSD`、`sym2 = FOREXCOM:XAUUSD`、`length = 20`；比较同一品种时 `kendallCorr` 直接返回 1；
- **它比较的是价格水平，不是收益率**：两条带趋势的序列即使毫无因果关系也会得到很高的秩相关（伪相关）。要做严谨的比较，把 `src` 换成 `roc(close,1)`、`change(close,1)` 之类的收益率；
- 嵌套循环是 O(n²)，`length` 拉到几百会明显拖慢图表；
- v3 里 `security(sym, period, src)` 的第三参没写 `lookahead`，属于「不预知未来」的默认行为，但仍然会在高周期品种和当前品种数据对齐时产生轻微错位。

---

### 五、外部数据聚合（财报 / 宏观 / 政治周期）

#### 13 & 14. `dividends_per_share_dps_yearly.pine` / `earnings_per_share_eps_yearly.pine` —— 年度 DPS / EPS

这两个文件**除了标题和数据字段名以外完全相同**（`diff` 只差 3 行：版权注释行、`study()` 里的标题、`_id("DIVIDENDS")` vs `_id("EARNINGS")`），所以放在一起讲。

**做什么**：把个股的**每股分红**（DPS）或**每股收益**（EPS）按年累加，画成「今年、1 年前、2 年前 … 20 年前」的柱状对比图，直观看出分红/盈利是逐年增长还是在衰退。

**原理**：

```pine
_id(type) => "ESD_FACTSET:" + syminfo.prefix + ";" + syminfo.ticker + ";" + type
id = _id("DIVIDENDS")                                            // 例：ESD_FACTSET:NASDAQ;AAPL;DIVIDENDS
esd      = security(id, "D", close, true, lookahead=true)        // 实际值
estimate = security(id, "D", open,  true, lookahead=true)        // 分析师预期值

_count(src, yearTarget) =>
    var float counter = na
    counter := year != year[1] and year == yearTarget ? 0 : counter[1]     // 目标年第一根 → 清零
    counter := year == yearTarget ? counter + (na(src) ? 0 : src) : counter // 目标年内 → 累加
    counter
```

1. **数据源**：TradingView 的 `ESD_FACTSET` 系列（Estimate / Surprise Data，来自 FactSet 的财报事件数据）。`close` 是实际值、`open` 是预期值 —— 这是一个「借 K 线两根价格位分别装两个字段」的小技巧；
2. **按年累加**：`_count()` 是一个「带年重置的累加器」。每个目标年有一个独立实例（`var` 在函数内每个调用点各自持有状态），在目标年第一根把计数清零，之后把当根的值累加进去；过了目标年就冻结在那个累计值上 —— 于是每根柱子显示的就是**那一整年的总和**；
3. **取数**：`_countESD(year)` 一次返回 `[esd累计, estimate累计]` 两个值，代码为 1990…2030 每年各展开一行（41 组硬编码，同样是代码生成的写法）；
4. **`_getCounter(mode, yearTarget)`** 是一长串 `yearTarget == 20xx ? ... : out` 的取值器，把「年份参数」映射到上一步的 41 个变量之一；
5. **绘图**：`plot(_getCounter("esd", year - N), offset=(-N*space), style=histogram, show_last=1)`，即「今年在最右、每年往左一格」，共 21 根柱子（0~20 年前）；每 5 年钉一个标签 `Current / 3Y / 5Y / 10Y / 15Y / 20Y`。

**输出**：副图 21 根橙色柱子，**最右 = 今年（`Current`）**，往左依次是 1 年前…20 年前。柱高 = 该年 DPS（或 EPS）合计。

**备注**（这两个脚本的坑最多）：
- **只对个股有效**：`ESD_FACTSET` 是财报数据，forex / crypto / 指数上取不到值，脚本会一片空白；
- **需要足够长的图表历史**：脚本要求图表能覆盖到 1990 年之后的数据，否则那些年的柱子是 `na`（`_count` 的 `var float counter` 初值是 `na`，如果图表第一根就落在目标年中间，累加会一直是 `na`）；
- **`lookahead=true`**：`security()` 第 5 参打开了 `barmerge.lookahead_on`。这在纯展示时是为了让「财报事件」和 K 线对齐，但**如果拿它做回测就是未来函数**，务必注意；
- **`estimate`（预期值）算出来了但没有画**：文件里只 `plot` 了 `esd` 系列；被注释掉的生成器代码里有半透明蓝色的 `estimate` 柱，取消注释即可恢复「实际 vs 预期」的双柱；
- 年份范围写死 1990–2030，**2030 年之后脚本需要手动加年份分支**；
- 脚本没有 `overlay`，是副图；`space` 默认 8 根 K 线间距，在小周期图上 21 根柱子会挤在一起，调大 `Space Between Columns`。

#### 15. `us_treasury_yields.pine` —— 美债收益率曲线快照

**做什么**：把 11 个期限的美债收益率（1M / 2M / 3M / 6M / 1Y / 2Y / 3Y / 5Y / 7Y / 10Y / 30Y）取过来，**同一根 K 线上并排成一张柱状图**，一眼看出收益率曲线的形状（正常向上 / 平坦 / 倒挂）。

**原理**：

```pine
_get(sym) => nz(security(sym, timeframe.period, close, barmerge.gaps_off, barmerge.lookahead_on))

US30Y = _get("TVC:US30Y")
...
plot(not modeLines ? US30Y : na, offset=(0 * space), color=..., style=plot.style_histogram, linewidth=columnWidth, show_last=1)
plot(modeLines ? US30Y : na, color=..., linewidth=lineWidth)
plotshape(not modeLines ? min(US30Y, 0) : na, offset=(0 * space), text=" 30Y ", style=shape.labelup, location=location.absolute, ...)
```

- 每个期限用 `security()` 取 `TVC:USxxY` 的收盘（收益率，单位 %）；
- **柱状模式**：`offset` 依次取 `0, −1, −2 … −10` × `space`，把 11 个期限的最后一根值排开成 11 根柱子；`show_last=1` 只画最后一根；
- **折线模式**：不用 offset，直接 `plot` 成 11 条历史曲线；
- 标签用 `min(值, 0)` 锚定在 y=0（柱子底端），`shape.labelup` 向上标注期限名，避免标签遮住柱子；
- `nz(...)` 处理没有数据的期限（返回 0，不会报 `na` 错误）。

**输出**：
- 默认**柱状**：副图 11 根等宽柱子，从左到右依次是 `1M → 2M → 3M → 6M → 1Y → 2Y → 3Y → 5Y → 7Y → 10Y → 30Y`（`offset` 越负越靠左：30Y 的 offset 是 0 所以落在最右，1M 是 −10 落在最左），底部有 `30Y / 10Y / 7Y / … / 1M` 标签，另有一条 0 线；
- 切到 `Lines` 模式：变成 11 条收益率历史曲线，可以直接看长期走势和倒挂。

**备注**：
- 与图表本身的品种无关 —— 在任何图表上加载，画出来的都是美债收益率；
- `barmerge.lookahead_on` 在这里用于让稀疏的宏观数据和 K 线对齐，属于常见的展示用法，但同样不适合直接拿来回测；
- 想加期限（比如 20Y、5Y TIPS）照抄一组三行 `plot/plot/plotshape` 并把 offset 往下顺延即可 —— 注意 `space` 与 `columnWidth` 的配合，柱子太宽会互相压住。

#### 16. `ticker_performance_by_us_president.pine` —— 历届总统任期涨幅

**做什么**：把标的（默认适合 SPX / DJI / 美股指）在**每一位美国总统任期内的累计涨幅**算出来，排成一排柱子对比。属于「政治周期」类的趣味统计。

**原理**：

```pine
_perf(presidency) =>
    var float start = na
    var float out = na
    start := na(start) and presidency ? open : start                    // 任期第一根 → 记下开盘价
    out := not na(start) and presidency ? 100 * (close - start) / start : out   // 任期内 → 算涨幅；任期外 → 冻结
    out

trump2 = _perf(t > _ts(2025, 1, 20))
biden  = _perf(_ts(2021, 1, 20) <= t and t <= _ts(2025, 1, 20))
...
```

- `presidency` 是一个「当前 bar 是否落在该任期区间内」的布尔值；
- `_perf()` 内部用两个 `var`：`start`（任期第一次满足条件时的 `open`）和 `out`（任期内的累计涨幅）。任期结束后 `out` 不再更新，**冻结在任期最后一天的涨幅**；
- 每位总统一个调用点 → Pine 会为每个调用点独立保存 `start`/`out`，互不干扰；
- `plot(..., offset=trump2Offset, show_last=1)` 把 24 位总统的值排成一排（offset 依次递减 `space`，默认 150 根 K 线间距），`plotshape(math.min(值, 0))` 在 0 处钉名字标签；
- 版本是 **v6**（`math.min`、`input.int`、`indicator()`、带名字的 `_ts(y, m, d, h=0, min=0, s=0)` 默认参数），是本目录唯一一个 v6 脚本。

**输出**：副图一排 24 根柱子，从右到左：**Trump(2025–) → Biden → Trump(2017–2021) → Obama → G.W. Bush → Clinton → G.H.W. Bush → Reagan → Carter → Ford → Nixon → Johnson → Kennedy → Eisenhower → Truman → F. Roosevelt → Hoover → Coolidge → Harding → Wilson → Taft → T. Roosevelt → McKinley → Cleveland**（代码里共 24 个 `_perf()` 调用点，最早只列到 Cleveland 1893）。每根柱高 = 该任期结束时相对任期起点的累计涨幅，底部有姓氏标签。

**备注**：
- **需要图表历史覆盖该任期**：如果图表数据从 2000 年才开始，早于 2000 年的总统柱全是 `na`（`start` 永远没机会被赋值）。只有把图表拉到大周期（周/月）并加载足够长的历史，才能看到全部 24 位；
- `timestamp` 不带时区参数 → 使用**交易所时区**，与就职日的实际时刻会有小时级偏差；
- 柱高是「任期终点相对起点的涨幅」，**不是年化**，也没有扣除任期中间的回撤 —— 任期 8 年和任期 4 年的柱子在视觉上是不可比的；
- 名字里带 `ticker_performance` 但默认就是给股指用的：个股上市晚于任期时同样会出 `na`。

---

### 六、行情体检

#### 17. `candles_history.pine` —— 28 项行情指标汇总器

**做什么**：一个「下拉菜单式」的行情体检工具。`Display Mode` 里有 28 个选项，选哪一项就把那一项的统计值画成一条红线。

**原理**：所有指标都先算好，最后用一串「覆盖赋值」选出当前模式对应的那一个：

```pine
out = 0.0
out := mode == "Available Candles" ? n + 1 : out
out := mode == "Bullish Candles"   ? bullSum : out
out := mode == "Longest Candle Range" ? longestRange : out
...
plot(out, title="Metric", color=red)
```

内部指标可以分四类：

| 类别 | 指标 | 实现要点 |
| --- | --- | --- |
| **起点信息** | Starting Year / Month / Day / Weekday / Hour / Minute | `valuewhen(n == 0, year/month/..., 0)`，即「第一根 K 线的时间分量」，常量曲线 |
| **极值** | All-Time High / All-Time Low / Highest Volume / Lowest Volume | 递归 running max / min（`lowestVolume` 从 `pow(10,10)` 起算并排除 0 成交量） |
| **K 线形态计数** | Available / Bullish / Bearish / Doji / Bullish Trend / Bearish Trend / Bullish Marubozu / Bearish Marubozu | `cum(条件 ? 1 : 0)`；「Trend」= 实体 ≥ 50% 全幅；「Marubozu」= 无影线（`high==close and low==open`） |
| **长度统计** | Longest / Average Candle Range（及 Pips 版）、Longest / Average Candle Body Range（及 Pips 版） | `range = high−low`、`body = abs(open−close)`；平均用 `cum(range)/(n+1)`；Pips 版 = 值 ÷ `syminfo.mintick`（严格说是「mintick 个数」，外汇 5 位报价下 `1 pip = 10 mintick`，所以标着 Pips 的数是 pip 的 10 倍） |
| **连续段** | Longest Bullish / Bearish Serie | `bullCount` 连续性计数 + `bullMax` 递归取最大 |

**输出**：副图**一条红色曲线**（不是表格！）。要读出结论，看**最后一根 K 线的值**（打开「数据窗口」看最后一个数）。选择「累计类」指标（如 Bullish Candles）时曲线单调上升；选择「常量类」指标（如 Starting Year）时曲线是一条水平直线；选择「极值类」时是阶梯线。

**备注**：
- 这个脚本最容易误解的地方就是**它不画表格**，所有信息都编码在「这条线最后一根的值」里；
- 「Longest Bullish/Bearish Serie」的记账同样是 `bullMax := bullCount[1] + 1`，**把打断连续段的那根也算了进去**，与 #6 的段长口径同源，会比实际多 1 根；而且**数据末尾仍在延续的段不计入**；
- 「Average」用的是 `cum(range)/(n+1)`（全历史简单平均），不是滚动均线；
- `precision=1` 是全局精度，选「Available Candles」这类整数指标时会显示成 `1234.0`；
- 想一次看到全部指标，可以复制这个脚本、把每个指标各自 `plot` 成一条线（注意 v3 的绘图数量上限），或者用 `label`/`table` 改写成信息面板（v4+ 支持 `table`）。README 无法替你完成这一步 —— 这个脚本的定位始终是「一次看一个」。

---

## 附录 A：口径与实现细节速查

做统计最怕口径不清楚。下表把本目录里所有「容易差一、差一点」的地方集中列出，方便和 TradingView 内置统计或自己的 Python 脚本对账。

| 位置 | 细节 | 后果 |
| --- | --- | --- |
| `close_to_close_percent_change_distribution` | 分母 `bar_index + 1`，首根 `roc` 为 `na` | 所有桶合计 < 100% |
| 同上 | 分桶左闭右开（正向）/ 左开右闭（负向） | 正负两侧无重叠、无空档 |
| `gaps_percent_size_distribution` | 用 `open` 对前 `close` | 衡量的是「跳空」，不是涨跌幅 |
| 同上 | x 轴借用 `time[n]` | 柱子位置与真实时间无关 |
| `bullish_bearish_candle_series_distribution` | `len = count[1] + 1` | 列标签 N 实际是 N−1 根连续 |
| 同上 | 只统计「已结束」的段 | 末尾未结束的段不记账 |
| `candles_history` | `bullMax := bullCount[1] + 1` | 最长连续段多算 1 根 |
| `candles_history` | 「Average」用 `cum()/(n+1)` | 全历史平均，不是滚动 |
| `all_time_high_percent_drop` | 基准是 `high` 的 running max，被减数是 `close` | 回撤比用收盘价基准略深 |
| `roi_return_on_investment` | 基准是起点那根的 `open` | 与「收盘买入」略有差异 |
| `ytd_year_to_date_percent_return` | 基准是跨年那根的 `open` | 数据有缺口时「年初」偏移 |
| `z_score` | 内置 `stdev` = 总体（n）、手写 = 样本（n−1） | 两者相差 `sqrt(n/(n−1))` |
| `linear_regression_all_data` | 回归用全历史，画线只画最近 5000 根 | 线看起来比统计结论「短」 |
| `kurtosis_indicator` | 不含任何四阶矩计算 | 名不副实，是动量加速度 |
| `dividends_per_share_dps_yearly` / `earnings_..._eps_...` | `lookahead=true` | 展示可接受，回测不可用（未来函数） |
| 同上 | `ESD_FACTSET` 数据源 + 1990–2030 硬编码年份 | 只适用个股、需订阅、2030 年后失效 |
| `bitcoin_ath_hash_rate_level` | `QUANDL:BCHAIN/HRATE` | 数据集可能已下线 |
| `kendall_rank_correlation_coefficient` | 比较的是价格水平不是收益率 | 易产生伪相关 |
| `ticker_performance_by_us_president` | 需要图表历史覆盖任期；终点涨幅、非年化 | 早期总统柱为 `na`，任期长短不可比 |

---

## 附录 B：共性技术点

1. **`cum()` 是本目录的骨架**。`cum(条件 ? 1 : 0)` 是「计数」，`cum(x)` 是「求和」，`cum(x*y)` 配合 `cum(x)`/`cum(x²)` 得到回归所需的全部矩。一遍扫描、无循环，是 Pine 里性能最好的写法。
2. **递归 running max/min** 的固定写法：
   ```pine
   ath = high
   ath := na(ath[1]) ? ath : (high > ath[1] ? high : ath[1])
   ```
   三个脚本（ATH Days、ATH % Drop、Candles History）完全一致，`low` 版本把 `>` 改 `<`。
3. **`var` 在函数内同样保持跨 bar 状态**，且**每个调用点各自独立**。`_count()`、`_perf()`、`_ytd()` 都靠这一点实现「每个年份/每位总统一个独立累加器」。
4. **`offset` + `show_last=1` 是 v3 时代画柱状图的主要办法**（见「套路 B」，v4 起可改用 `line.new`）。理解这一点，所有直方图脚本的「横轴」都不再神秘：那不是真实时间轴，而是把当前 bar 的左侧空白当画布。
5. **`valuewhen(cond, src, 0)` 是「取最后一次满足条件时的值」**，本目录用它取年初开盘价、起点开盘价、第一根的时间分量。`0` 是出现次数参数（0 = 最近一次）。
6. **`security()` 在这里有三种完全不同的用法**：取其他品种的价格（Kendall）、取宏观/财报数据（DPS/EPS、Treasury、Hash Rate）、用「同一价格在不同周期上是否一致」做图表类型判断（见 `../research/`）。其中 DPS/EPS 与美债收益率用 `lookahead=true` 做对齐，展示可以、回测危险。
7. **版本迁移注意**：
   - v3 → v4：`n` → `bar_index`、`transp=` → `color.new()`；`iff()` → `?:`；`input(type=integer)` → `input.int()`；
   - v4 → v5/v6：`study()` → `indicator()`、`security()` → `request.security()`、`change()`/`cum()` → `ta.change()`/`ta.cum()`、`abs`/`pow`/`sqrt` → `math.abs`/`math.pow`/`math.sqrt`（本目录的 v6 脚本已经这样写）。
8. **输出形态决定读法**：曲线类看最后一根；柱状类看柱高和分布形状；标签类看文字。误把曲线当表格读，是这个目录最常见的困惑来源。

---

## 使用建议

- **这不是策略目录**。所有脚本都在描述「历史统计长什么样」，没有任何一个是「统计值 → 买卖决策」的完整逻辑。别拿 ATH 回撤 −80% 当抄底信号。
- **先确认数据可用**：`QUANDL:`、`ESD_FACTSET:`、`TVC:US*Y` 三类外部代码是脚本能否出图的前提。图上空白时，第一件事是去 TradingView 搜索那个代码能不能打开。
- **对账时先对齐口径**：想验证「连阳统计」「涨跌幅分布」这类数字，先看附录 A 对应的那几行，确认分母、闭开区间、±1 偏移之后再比对，否则永远对不上。
- **想要可调的分布图，优先改 `gaps_percent_size_distribution.pine`**：它的分桶靠数组循环，改桶宽/桶数只需改两个输入参数；而 `close_to_close_percent_change_distribution.pine` 是 50 行硬编码，改步长要重写整个文件。
- **做回测请把所有统计值导出成序列**：这些脚本都只 `plot`，若要在策略里复用某个统计量（比如「距上次创新高 N 根」），把对应的 `plot(...)` 换成函数返回、再用 `ta.` 前缀迁移到 v5/v6 即可。
