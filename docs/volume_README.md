# Volume —— 成交量类指标（9 个脚本）

本目录收录 **9 个** Pine Script 成交量指标（全部 `//@version=6`）。

它们回答的是同一个问题：**成交量这一列数字，怎么和价格组合成一个有方向、可比较、能画成曲线的量？**
成交量本身是一根 bar 的绝对数量，不可跨品种比较、不可跨时间比较；这 9 个脚本就是 5 类不同的"归一化/结构化"方案：

| 思路 | 做法 | 脚本 |
| --- | --- | --- |
| **累计**（路径依赖） | 每根 bar 判断价格方向，把成交量带符号地累加成一个不断增长的曲线 | OBV、A/D Line、PVT |
| **条件化**（只在特定量能条件下更新） | 只在"放量日"或"缩量日"更新一个价格指数 | PVI、NVI |
| **单根 bar 归一化** | 只处理当根 bar 的量，不跨 bar 累计 | Net Volume、% Volume |
| **周期分段累计** | 按自然周期把量累加，并与上一周期总量比较 | Volume Accumulation |
| **信号处理** | 把量的"力量"变成振荡器，再做双 EMA 差分 | Klinger Volume Oscillator |

**没有任何一个是 `strategy`**：不能回测、不下单。4 个脚本带 `alertcondition()`（A/D Line 4 条、OBV 3 条、PVT 2 条、Volume Accumulation 1 条，详见 §1.5），其余 5 个只画线/画柱，只有视觉标记。

---

## 目录

1. [先读这一节：9 个脚本的共同骨架](#一先读这一节9-个脚本的共同骨架)
   - [1.1 面板与版本](#11-面板与版本)
   - [1.2 代码骨架](#12-代码骨架)
   - [1.3 输出与配色约定](#13-输出与配色约定)
   - [1.4 "信号线 + 色带"三件套](#14-信号线--色带三件套)
   - [1.5 告警能力](#15-告警能力)
2. [快速索引](#二快速索引)
3. [逐个说明](#三逐个说明)
   - [3.1 累计型：OBV / A/D Line / PVT](#31-累计型obv--ad-line--pvt)
   - [3.2 条件化指数：NVI / PVI](#32-条件化指数nvi--pvi)
   - [3.3 Klinger Volume Oscillator](#33-klinger-volume-oscillator)
   - [3.4 单根 bar 量能：Net Volume / % Volume](#34-单根-bar-量能net-volume---volume)
   - [3.5 周期累计：Volume Accumulation](#35-周期累计volume-accumulation)
4. [附录 A：内置变量的精确口径](#附录-a内置变量的精确口径)
5. [附录 B：坑与瑕疵清单](#附录-b坑与瑕疵清单)
6. [附录 C：怎么组合着用 / 怎么调参](#附录-c怎么组合着用--怎么调参)

---

## 一、先读这一节：9 个脚本的共同骨架

### 1.1 面板与版本

| 文件 | 版本 | 面板 | 输出类型 | `alertcondition` |
| --- | --- | --- | --- | --- |
| `accumulation_distribution_line.pine` | 6 | 副图 | 线 + 线 + 填充带 + 交叉点 | **4** |
| `klinger_volume_oscillator.pine` | 6 | 副图 | 线 + 线 + 可选柱 + 零轴背景 + 交叉点 | 0 |
| `negative_volume_index.pine` | 6 | 副图 | 线 + 线 + 填充带 + 交叉点 | 0 |
| `net_volume.pine` | 6 | 副图 | 四色柱 | 0 |
| `on_balance_volume.pine` | 6 | 副图 | 线 + 线 + 填充带 + 交叉点 | **3** |
| `percent_volume.pine` | 6 | 副图 | 柱 + 6 档参考线（圆点串 + 虚线） | 0 |
| `positive_volume_index.pine` | 6 | 副图 | 线 + 线 + 填充带 + 交叉点 | 0 |
| `price_volume_trend.pine` | 6 | 副图 | 线 + 线 + 填充带 + 交叉点 | **2** |
| `volume_accumulation.pine` | 6 | 副图 | 柱 + 主图 K 线着色 + 周期分隔背景 | **1** |

- **全部是副图**（没有 `overlay = true`）。`volume_accumulation.pine` 唯一会"跑到主图"的地方是它调用了 `barcolor()`，把成交量周期状态刷到 K 线颜色上。
- **单位差异极大**：从 0–100 指数（NVI/PVI）、百分比（% Volume）、单根量（Net Volume），到亿级的累计值（OBV/A/D/PVT）。看图前先看坐标轴，否则会误判"涨得很快"。
- 三个脚本声明了 `format = format.volume`（`on_balance_volume`、`price_volume_trend`、`volume_accumulation`），TradingView 会把刻度显示成 `12.3M` / `1.02B`。

### 1.2 代码骨架

除了 `net_volume.pine`（12 行）、`percent_volume.pine`（柱状图 + 参考线）、`volume_accumulation.pine`（周期累计）这 3 个，其余 **6 个**几乎是同一个模板，只有中间一段公式不同：

```pine
study("XXX", shorttitle="XXX", precision=0)        // v3/v4；v6 用 indicator(...)

// ---- 输入：几乎都是同一组 ----
showSignal          = input(..., defval=true)      // 是否画信号线
signalType          = input(..., defval="SMA")     // EMA / SMA 二选一
signalLength        = input(..., defval=21)        // 信号线长度
highlightCrossovers = input(..., defval=false)     // 是否在交叉处画圆点
applyFilling        = input(..., defval=true)      // 色带是否跟随趋势变色

// ---- 公式：唯一有差异的部分 ----
xxx = ...                                          // OBV / A/D / PVT / NVI / PVI / KVO

// ---- 三件套：信号线 + 趋势色 + 填充带 ----
signal     = signalType == "EMA" ? ema(xxx, signalLength) : sma(xxx, signalLength)
trendColor = xxx > signal ? #0ebb23 : red
xxxPlot    = plot(xxx,    color = applyFilling ? trendColor : #0094ff, linewidth=2)
signalPlot = plot(showSignal ? signal : na, color = applyFilling ? trendColor : #ff6a00)
fill(xxxPlot, signalPlot, color = applyFilling ? trendColor : color(white, 100), transp=70)
plotshape(crossover(xxx, signal)  and highlightCrossovers ? xxx : na, style=shape.circle, color=green)
plotshape(crossunder(xxx, signal) and highlightCrossovers ? xxx : na, style=shape.circle, color=red)
```

所以对任一脚本，"干什么 + 原理"= 中间那段公式，"跑出来什么"= **一条主线 + 一条均线 + 一个方向色 + 一条半透明色带**。

默认参数也高度一致，记住这一张表即可覆盖 6 个脚本：

| 输入 | OBV | A/D Line | PVT | NVI | PVI | KVO |
| --- | --- | --- | --- | --- | --- | --- |
| 信号线类型 | SMA | **SMA** | SMA | **EMA** | **EMA** | **EMA** |
| 信号线长度 | 21 | 21 | 21 | **255** | **255** | 13 |
| Show Signal | true | true | true | 固定 true | 固定 true | true |
| Highlight Crossovers | false | false | false | false | false | **true** |
| Apply Ribbon Filling | true | true | true | true | true | **false** |

（`net_volume` / `percent_volume` / `volume_accumulation` 不属于这套模板，参数见 §3.4、§3.5。）

### 1.3 输出与配色约定

| 形态 | 触发条件 | 颜色 | 脚本 |
| --- | --- | --- | --- |
| **趋势双色线**（默认） | `applyFilling = true`，`主线 > 信号线` → `#0ebb23` 绿，否则 `red` | 两条线同色，用色带方向表达多空 | OBV、A/D、PVT、NVI、PVI |
| **分离配色** | `applyFilling = false` | 主线 `#0094ff` 蓝、信号线 `#ff6a00` 橙 | 同上 |
| **零轴配色** | KVO 在 `applyFilling = false` 时 | KVO `> 0` 绿 / `< 0` 红；信号线固定 `#512da8` 紫 | KVO |
| **四色柱** | 值 + 前值比较 | `#26A69A` 深绿（正且走高）/ `#B2DFDB` 浅绿（正但回落）/ `#FFCDD2` 浅红（负但回升）/ `#EF5350` 深红（负且走低） | `net_volume`、KVO 的 histogram |
| **阴阳柱** | `close > open` | 阳线 `#92df8bff` 绿 / 阴线 `#fe9695ff` 红 | `percent_volume` |
| **环比柱** | 本周期累计 vs 上一周期总量 | 放大 `#92df8bff` 绿 / 缩小 `#fe9695ff` 红 | `volume_accumulation` |
| **圆点标记** | `crossover` / `crossunder` | 绿点 = 上穿、红点 = 下穿，画在指标读数的高度（`location.absolute`），`size.tiny` | 6 个模板脚本 |
| **背景着色** | KVO 零轴、Volume Accumulation 周期切换 | 绿/红 `transp=85`；橙 `#ffcc80` `transp=60` | KVO、`volume_accumulation` |

### 1.4 "信号线 + 色带"三件套

这是本目录最容易被忽略的设计，也是这 6 个脚本真正的**信号来源**：

1. **主线** = 指标本体（累计量 / 指数）；
2. **信号线** = 主线的 EMA 或 SMA。它唯一的职责是把主线**平滑**，制造一个"慢半拍"的参照；
3. **`xxx > signal`** 这一句被复用了三次：决定两条线的颜色（`trendColor`）、决定色带填充、并在 `highlightCrossovers` 打开时决定圆点。所以"颜色翻转"和"圆点出现"其实是同一件事的两种画法。

由此带来一个必须知道的性质：**信号线越长，交叉越少、越滞后**。NVI/PVI 默认 `255`，意味着信号线几乎是"多年均线"，交叉一年也就几次；OBV/A/D/PVT 默认 `21`，交叉频率高得多。

### 1.5 告警能力

| 文件 | 告警条件 | 说明 |
| --- | --- | --- |
| `on_balance_volume.pine` | `Bullish Color Change` / `Bearish Color Change` / `Any Color Change` | 由 `trendColor` 与 `trendColor[1]` 比较得出，消息含 `{{exchange}}:{{ticker}}`、`{{close}}` |
| `price_volume_trend.pine` | `Alert: Buy`（上穿） / `Alert: Sell`（下穿） | 直接是 `crossover(pvt, signal)` / `crossunder(...)` |
| `volume_accumulation.pine` | `Higher Accumulation` | `acc > accPrev`，**注意这是逐 bar 判定**，见附录 B-6 |

其余 5 个脚本**没有** `alertcondition()`。它们的 `crossover` / `crossunder` 只喂给了 `plotshape`，画个圆点而已，挂不了告警。想告警就把那一行复制成 `alertcondition(...)`。

---

## 二、快速索引

| # | 脚本 | 指标 | 版本 | 核心公式（一句话） | 输出刻度 | 默认关键参数 |
| --- | --- | --- | --- | --- | --- | --- |
| 1 | `on_balance_volume.pine` | On Balance Volume (OBV) | 6 | 收盘涨则加量、跌则减量，累计 | 累计量（亿级），0 起 | SMA 21 |
| 2 | `accumulation_distribution_line.pine` | Accumulation/Distribution (AD) | 6 | 用收盘在当日区间的位置加权成交量，累计 | 累计量（亿级），0 起 | SMA 21 |
| 3 | `price_volume_trend.pine` | Price Volume Trend (PVT) | 6 | 用涨跌幅百分比加权成交量，累计 | 累计"量×%"，量纲混合 | SMA 21 |
| 4 | `positive_volume_index.pine` | Positive Volume Index (PVI) | 6 | **只在放量日**按涨跌幅更新指数 | 指数，100 起 | EMA 255 |
| 5 | `negative_volume_index.pine` | Negative Volume Index (NVI) | 6 | **只在缩量日**按涨跌幅更新指数 | 指数，100 起 | EMA 255 |
| 6 | `klinger_volume_oscillator.pine` | Klinger Volume Oscillator (KVO) | 6 | 量的"力量"做 34/55 双 EMA 差，再取 13 信号线 | 围绕 0 | 34 / 55 / 13 |
| 7 | `net_volume.pine` | Net Volume | 6 | `sign(涨跌) × volume`，只看当根 | ±volume，符号即方向 | src = close |
| 8 | `percent_volume.pine` | % Volume | 6 | `100 × volume / SMA(volume, 21)` | 百分比，100 = 均量 | SMA 21 |
| 9 | `volume_accumulation.pine` | Volume Accumulation | 6 | 按分钟/小时/时段/日/月/季/年累计成交量 | 量（周期内累加） | Daily |

---

## 三、逐个说明

### 3.1 累计型：OBV / A/D Line / PVT

三个脚本是同一族：**一个从 0 或 100 开始的累加器，每根 bar 往里加一个"带符号的成交量"**。区别全在"符号"怎么算 —— 即"这根 bar 的量应该算给多头还是空头"。

#### 3.1.1 `on_balance_volume.pine` — On Balance Volume (OBV)

| 项 | 内容 |
| --- | --- |
| 版本 / 面板 | `//@version=6`，副图，`format = format.volume`，`timeframe = ''`（跟随图表周期） |
| 用途 | 判断"涨跌是否有量配合"：价涨量增 → 资金流入；价涨量缩 → 上涨乏力 |
| 默认参数 | 信号线 **SMA 21**（可选 EMA）、`Show Signal = true`、`Highlight Crossovers = false`、色带填充 `true` |
| 输出 | OBV 主线（2px）+ SMA21 信号线 + 半透明色带；OBV > 信号线时两条线 `#0ebb23` 绿并填浅绿(`transp=85`)，否则 `color.red` 并填浅红；可选绿/红圆点；3 个告警 |
| 读法 | OBV 与价格**同向创新高** = 趋势健康；价格新高但 OBV 不新高 = **顶背离**；价格新低但 OBV 不新低 = 底背离 |

**原理**

```
OBV_t = OBV_{t-1} + volume_t × sign(close_t − close_{t-1})
      其中 sign(x) = +1 / 0 / −1
```

脚本直接用内置变量 `ta.obv`，**没有自己写公式**，所以口径完全由 TradingView 决定：

- 收盘价**上涨** → 整根 bar 的成交量记为 `+volume`；
- 收盘价**下跌** → 整根 bar 的成交量记为 `−volume`；
- 收盘价**不变** → `+0`，OBV 原地不动；
- 首根 bar `change(close)` 为 `na`，所以 OBV **从 0 起算**，不是从第一根的量开始。

注意这是**零和记账**：不管当根涨跌幅度有多大，量都按全量记给一方。所以一根 0.01% 的微涨和一根 9% 的大涨对 OBV 的贡献相同 —— 这也是 OBV 最大的局限，A/D 和 PVT 都在试图修正这一点。

**v6 版本特有的地方**（相比本目录其他脚本）：

- `enum MovingAverageType` + `input.enum(...)`，下拉框是枚举而非字符串；
- 颜色用 `const color` 常量、`color.new(bullishColor, 85)` 处理透明度（而不是 v3 的 `fill(..., transp=70)`）；
- 有 3 条 `alertcondition`（Bullish / Bearish / Any Color Change）；同目录 `accumulation_distribution_line.pine` 是同一模板的另一次实现，告警更多（4 条）。

#### 3.1.2 `accumulation_distribution_line.pine` — Accumulation/Distribution Line

| 项 | 内容 |
| --- | --- |
| 版本 / 面板 | `//@version=6`，副图，`precision = 0`（坐标轴取整） |
| 用途 | 用**收盘价在当日区间中的位置**来衡量多空，比 OBV 的"只看收盘涨跌"更细腻 |
| 默认参数 | 信号线 **SMA 21**（可选 EMA）、`Show Signal = true`、`Highlight Crossovers = false`、色带填充 `true` |
| 输出 | A/D 主线 + 信号线 + 色带；配色与 OBV 完全一致（绿 `#0ebb23` / 红，非填充时 `#0094ff` 蓝 + `#ff6a00` 橙） |
| 读法 | A/D 上行而价格横盘 = **吸筹**；价格上行而 A/D 走平或下行 = 上涨靠"少量成交推动"，警惕派发 |

**原理**

```
CLV_t = ((close − low) − (high − close)) / (high − low)        ∈ [−1, +1]
A/D_t = A/D_{t-1} + CLV_t × volume_t
```

`CLV`（Close Location Value）就是**收盘价在当日区间里的相对位置**，映射到 `[−1, +1]`：

- 收在最高价 → `+1`（全部量算流入）；
- 收在最低价 → `−1`（全部量算流出）；
- 收在区间中点 → `0`。

脚本同样用内置变量 `accdist`，没有自己展开公式。由此产生的两个后果：

1. **`high == low` 的 bar（一字板、极低流动性品种）在公式里是除零**。TradingView 内置实现对这种情况如何处理并未公开，如果你依赖这条线做量化，建议自己展开成显式公式并对 `high == low` 显式取 0（详见附录 B-2）；
2. **A/D 的绝对值没有意义，只有斜率和背离有意义** —— 它是个累计量，图中起点取决于你加载了多少历史。

#### 3.1.3 `price_volume_trend.pine` — Price Volume Trend (PVT)

| 项 | 内容 |
| --- | --- |
| 版本 / 面板 | `//@version=6`，副图，`format = format.volume` |
| 用途 | 给成交量乘上**涨跌幅百分比**，让"大涨放量"比"微涨放量"贡献更大 |
| 默认参数 | 信号线 **SMA 21**（可选 EMA）、`Show Signal = true`、`Highlight Crossovers = false`、色带填充 `true`、**`Source = close`（实际未生效，见附录 B-3）** |
| 输出 | PVT 主线 + 信号线 + 色带（配色同 OBV）；另有两个 `alertcondition`：`Buy`（上穿）、`Sell`（下穿） |
| 读法 | 与 OBV 类似看背离，但对高波动品种更敏感；**不适合比较不同品种的绝对水平**，因为量纲是"成交量 × 变化率" |

**原理**

```
PVT_t = PVT_{t-1} + volume_t × (close_t − close_{t-1}) / close_{t-1}
```

脚本使用内置变量 `pvt`。三种累计量的差异可以用一张表看清：

| 指标 | 每根 bar 贡献 | 对"涨跌幅度"敏感？ | 对"收盘在区间位置"敏感？ | 绝对刻度 |
| --- | --- | --- | --- | --- |
| **OBV** | `±volume` | 否 | 否 | 累计量 |
| **A/D Line** | `CLV × volume`，`CLV ∈ [−1,1]` | 否 | **是** | 累计量 |
| **PVT** | `volume × Δclose/close` | **是**（百分比） | 否 | 量×%，量纲混合 |

选型建议：**A 股/加密等波动大、常有一字板的品种用 A/D 要小心除零；跨品种比较用 %Volume；判断资金进出方向用 OBV 足够；想看"资金推动效率"用 PVT。**

#### 3.1.4 三者共有的读法

- 看**斜率**，不看绝对高度；
- 看**背离**（价格与指标不同步）而不是交叉 —— 默认 `highlightCrossovers = false` 其实反映了作者的倾向：交叉只是噪声，背离才是信息；
- 打开 `Apply Ribbon Filling = true` 时，图的颜色就是"指标在信号线上方/下方"，**不是**价格趋势。别把它当多空信号直接下单。

---

### 3.2 条件化指数：NVI / PVI

两个脚本互为镜像：`diff` 下来，除了变量名/标题和 `> 0` ↔ `< 0` 这一处判断，代码完全相同。它们源自 1970 年代的"smart money / public"理论：**机构在缩量时慢慢吸筹，散户在放量时追高**。

#### 3.2.1 `positive_volume_index.pine` — Positive Volume Index (PVI)

| 项 | 内容 |
| --- | --- |
| 版本 / 面板 | `//@version=6`，副图，`precision = 0` |
| 用途 | 跟踪"**放量日**"的价格累积表现，用来代表**散户/公众**的参与结果 |
| 默认参数 | `Initial Value = 100`、信号线 **EMA 255**（可选 SMA）、`Highlight Crossovers = false`、色带填充 `true` |
| 输出 | PVI 主线 + EMA255 信号线 + 色带（绿 `#0ebb23` / 红；非填充时 `#0094ff` + `#ff6a00`）；PVI `showSignal` 固定为 true，没有开关 |
| 读法 | 按该理论，PVI 长期**跑输** NVI 是常态（散户整体亏钱）；PVI 相对 NVI 突然走强 = 市场进入"公众主导"的过热阶段 |

**原理**

```
if volume_t > volume_{t-1}:   PVI_t = PVI_{t-1} × close_t / close_{t-1}
else:                          PVI_t = PVI_{t-1}          ← 保持前值，当根涨跌幅不计入
PVI_0 = 100（首根 bar）
```

关键点：**缩量日 PVI 是平的一条直线**（`close` 的变化被完全忽略），这正是"只在放量日记录价格变化"的含义。

#### 3.2.2 `negative_volume_index.pine` — Negative Volume Index (NVI)

| 项 | 内容 |
| --- | --- |
| 版本 / 面板 | `//@version=6`，副图 |
| 用途 | 跟踪"**缩量日**"的价格累积表现，代表**机构/聪明钱**的行为 |
| 默认参数 | `Initial Value = 100`、信号线 **EMA 255**、`Highlight Crossovers = false`、色带填充 `true` |
| 输出 | 与 PVI 完全对称（同色系） |
| 读法 | 经典规则：**NVI 在 255 日 EMA 之上 → 机构在吸筹，偏多**。NVI 跌破 EMA255 是少数被广泛引用的"卖出"信号之一 |

**原理**

```
if volume_t < volume_{t-1}:   NVI_t = NVI_{t-1} × close_t / close_{t-1}
else:                          NVI_t = NVI_{t-1}
NVI_0 = 100
```

**两者的三个共同注意点**

1. **`signalLength = 255` 是"一年交易日数"**，配 EMA 使用是 NVI 的经典参数（`ema(nvi, 255)`）。信号线画出来几乎是一条水平慢线，**交叉极其稀少**——这是设计意图，不是 bug。在 4 小时/分钟图上，255 根不等于一年，含义会失真。
2. **指数是"乘性"的**：`PVI × close_t/close_{t-1}` 等于"把放量日的收益率复利累积"，所以数值会随时间漂移，**不同起点、不同历史长度的值不可比**。
3. **缩量/放量判定只看 `change(volume)` 的符号**（PVI 用 `> 0`、NVI 用 `< 0`），没有"放量多少倍才算放量"的阈值。低量震荡期，量的小幅抖动会让指数在"更新/不更新"之间频繁切换。

---

### 3.3 Klinger Volume Oscillator

#### `klinger_volume_oscillator.pine`

| 项 | 内容 |
| --- | --- |
| 版本 / 面板 | `//@version=6`，副图，`precision = 0` |
| 用途 | 把"成交量 × 趋势方向 × 量能强度"合成一个**围绕 0 波动**的振荡器，用双 EMA 差找量能的转向 |
| 默认参数 | Fast **34**、Slow **55**、信号线 **EMA 13**、`Show Histogram = false`、`Highlight KVO/Signal Crossovers = true`、`Highlight Zero Line Crossovers = true`、色带填充 `false` |
| 输出 | ① KVO 线（`applyFilling=false` 时按零轴配色：`>0` 绿 / `<0` 红；设为 `true` 时改为按 KVO 与信号线的相对位置）② EMA13 信号线（紫 `#512da8`）③ 可选 histogram 四色柱 ④ **零轴背景色**（KVO>0 浅绿 / <0 浅红，`transp=85`，默认开启）⑤ 交叉圆点（`transp=20`，比别处更实） |
| 读法 | KVO 上穿信号线 = 量能转多；**零轴**才是多空分界；histogram 由浅变深 = 量能加速 |

**原理（四步）**

```
① 趋势方向（带记忆）
   mom   = change(hlc3)                       ← 用 (H+L+C)/3，不是 close
   trend = mom > 0 ? +1 : mom < 0 ? −1 : trend[1]

② 当日量程 / 累计量程
   dm = high − low
   cm = (trend == trend[1]) ? cm[1] + dm       ← 趋势不变：累加
                            : dm + dm[1]       ← 趋势翻转：重置，并带上上一根的 dm

③ 量能力量（Volume Force）
   vf = 100 × volume × trend × |2 × dm / cm − 1|

④ 振荡器
   KVO    = EMA(vf, 34) − EMA(vf, 55)
   signal = EMA(KVO, 13)
   hist   = KVO − signal
```

**为什么要这么绕？** 拆开看，`2×dm/cm − 1` 是"当天量程占累计量程的比重，再映射到 `[−1, +1]`"：

- 当根价格区间很小（`dm ≈ 0`，十字星/窄幅整理）→ 系数 `≈ −1`，**即使趋势向上也给出负力量**（波动没被这波量确认）；
- 当天量程≈全部累计量程（`dm ≈ cm`）→ 系数 `≈ +1`，量能集中，力量打满；
- 再乘上 `trend`（±1），就得到"这个方向的量能有多真"。

所以 KVO 与 OBV 的本质区别是：**OBV 只问"涨还是跌"，KVO 还要问"这根 bar 的量能是不是集中在当根"**。

**输出解读**

| 视觉元素 | 含义 |
| --- | --- |
| KVO 线颜色 | 只表示 KVO 的正负（`applyFilling=false` 时）。正 = 快线量能强于慢线 |
| 信号线（紫） | 13 周期 EMA，交叉即"量能拐点" |
| 零轴背景 | 默认开启，**整张副图会长期带淡绿/淡红底色**，视觉噪声较大，可关掉 `Highlight Zero Line Crossovers` |
| histogram | 默认**关闭**。打开后四色柱表达"KVO 与信号的差在扩大还是收敛"，是动量加速度的可视化 |
| 交叉圆点 | 默认**开启**（与其他脚本相反），且透明度 20（更醒目） |

**注意 `cm` 的本质**：它不是"全历史累计量程"，而是**当前趋势段内的累计量程** —— `trend` 一翻转，`cm` 就重置为 `dm + dm[1]`。所以趋势延续得越久，`cm` 越大，`2×dm/cm − 1` 越贴近 `−1`，这就是 Klinger 表达"趋势老化、量能摊薄"的方式。另：第一根 bar `cm = 0`，脚本用 `cm != 0 ? ... : 0` 保护，首根 `vf = 0`。

---

### 3.4 单根 bar 量能：Net Volume / % Volume

这两个脚本不做累计，每根 bar 独立计算，所以**可以跨时间直接比较**。

#### 3.4.1 `net_volume.pine` — Net Volume

整个文件只有 12 行，是本目录最短的脚本。

| 项 | 内容 |
| --- | --- |
| 版本 / 面板 | `//@version=6`，副图，`precision = 0` |
| 用途 | 把成交量**带上正负号**：涨的 bar 量为正、跌的 bar 量为负，一眼看出多空量能对比 |
| 默认参数 | `Source = close`（可改成 `hl2`、`hlc3`、`open` 等） |
| 输出 | 单组四色柱：`#26A69A`（正且放大）/ `#B2DFDB`（正但缩小）/ `#FFCDD2`（负但回升）/ `#EF5350`（负且扩大） |
| 读法 | 柱子在零轴上下交替；**连续同色深柱**（深绿连片或深红连片）= 单边量能占优 |

**原理**

```
NV_t = sign(src_t − src_{t−1}) × volume_t
```

- `src` 默认 `close`，所以就是 `sign(涨跌) × 量`；
- 改成 `hl2` 可获得更平滑的方向判断（减少收盘价噪声）；
- **首根 bar**：`change(src)` 为 `na`，`sign(na)` 为 `na`，该柱不显示。

**四色规则（KVO histogram 也用同一套）**

```pine
nvColor = nv >= 0
        ? (nv[1] < nv ? #26A69A : #B2DFDB)      // 正：走高深绿 / 回落浅绿
        : (nv[1] < nv ? #FFCDD2 : #EF5350)      // 负：回升浅红 / 走低深红
```

注意 `nv[1] < nv` 对负值是"在**回升**（变得不那么负）"，所以是浅红 —— 颜色编码的是**变化方向**，不只是符号。

#### 3.4.2 `percent_volume.pine` — % Volume

| 项 | 内容 |
| --- | --- |
| 版本 / 面板 | `//@version=6`，副图，`precision = 2` |
| 用途 | **相对量**：当根成交量是近期均量的百分之多少。解决"成交量绝对数不可跨品种/跨时间比较"的问题 |
| 默认参数 | `SMA Length = 21` |
| 输出 | 柱状图：阳线（`close > open`）绿 `#92df8bff`，阴线红 `#fe9695ff`；外加 **50/100/150/200/250/300** 六条参考水平（黄色圆点 `#f9ce02` + 紫色虚线 `#674ea7`） |
| 读法 | `100` = 均量；`< 50` 极度缩量（常见于变盘前）；`> 200` 放量异动（突破/恐慌）；**同一根柱子同时看高度和颜色**：高绿柱 = 放量上涨，高红柱 = 放量下跌 |

**原理**

```
%V_t = 100 × volume_t / SMA(volume, 21)_t
```

就本目录范围而言这是**最实用的一个脚本**，因为：

- 输出是**无量纲的百分比**，可以在茅台和比特币之间直接比；
- `100` 这条线有绝对含义（均量），不是"某根 bar 的成交量"；
- 分母是 21 期 SMA（不是 EMA）：一根巨量只会抬高基准 21 根 bar，之后自动移出窗口，且不像 EMA 那样按权重长期残留。

**关于那 6 条参考线**：脚本为每一档画了**两个东西** —— `plot(常量, style=circles)` 和 `hline(常量)`。前者每根 bar 画一个黄色小圆点，串起来像虚线；后者才是真正的水平虚线。**功能上完全重复**，属于装饰冗余（见附录 B-4）。

---

### 3.5 周期累计：Volume Accumulation

#### `volume_accumulation.pine`

| 项 | 内容 |
| --- | --- |
| 版本 / 面板 | `//@version=6`，副图（但会用 `barcolor()` 给主图 K 线上色），`format = format.volume` |
| 用途 | 把成交量**按自然周期（分钟/小时/时段/日/月/季/年）分段累加**，看"这一段时间里总共成交了多少"，以及"比上一段多还是少" |
| 默认参数 | `Type = Daily`、`Session = '0900-1530'`、`Include Weekends = true`、`Apply Bar Colors = true` |
| 输出 | ① 累计量柱状图（本段累计 > 上段总量 → 绿 `#92df8bff`，否则红 `#fe9695ff`）② 可选 `barcolor` 把主图 K 线染成同色 ③ **周期切换处**画一条橙色背景带 `#ffcc80`（`transp=60`）④ `Higher Accumulation` 告警 |
| 读法 | 柱子在周期内**递增**（不是在周期末尾才画一根），到下一周期**重置为当根的量重新累计**；柱子颜色告诉你"这个周期的量能否超过上一个周期"——放量突破/缩量回调的直接量化 |

**原理（分段递归）**

```pine
// ① 判断"当前处于哪个周期"，得到周期编号 period
min = minute(time)                      // Minutely
h   = hour(time)                        // Hourly
s   = _inSession(timeframe.period, session) ? 1 : 0   // Session
d   = dayofmonth(time)                  // Daily
m   = month(time)                       // Monthly
q   = int(math.ceil(m / 3))             // Quarterly → 1..4
y   = year(time)                        // Yearly

// ② 周期编号一变，累加器就重置为"当根 volume"；否则继续累加
acc := period != period[1] ? volume : nz(acc[1]) + volume

// ③ 记住"上一周期的最终累计值"，用来给本周期上色
accPrev := period != period[1] ? acc[1] : accPrev
```

`Session` 类型的实现值得一提：

```pine
_inSession(res, sess) => not na(time(res, sess))
var session = sessionInput + (includeWeekends ? ':1234567' : '')
```

`'0900-1530'` + `':1234567'` 是 TradingView 的 session 语法（时段 + 生效的星期几，`1234567` = 周一到周日）。`time(timeframe.period, session)` 在该 bar 落在时段内时返回时间、否则返回 `na`，取反得到布尔值，再转成 `0/1` 作为"周期编号"。

**由此带来的一个副作用**：`s` 只有 `0` 和 `1` 两个值，所以在 Session 模式下，**时段内累计一份、时段外又累计一份**（进入/离开时段各触发一次重置），并不是"一次会话结束就停"。这与 `Daily`/`Monthly` 的直觉不同。

---

## 附录 A：内置变量的精确口径

三个脚本（OBV、A/D、PVT）依赖 TradingView 内置变量，源码里看不到公式。它们在 v3/v4 是全局变量，v5+ 搬到 `ta.` 命名空间：

| 脚本 | v3/v4 变量 | v5/v6 等价写法 | 定义 |
| --- | --- | --- | --- |
| `on_balance_volume.pine` | `obv` | `ta.obv` | `cum(sign(change(close)) × volume)`，首根为 0 |
| `accumulation_distribution_line.pine` | `accdist` | `ta.accdist` | `cum(((close−low)−(high−close))/(high−low) × volume)`，`high==low` 时行为未文档化 |
| `price_volume_trend.pine` | `pvt` | `ta.pvt` | `cum(volume × (close−close[1])/close[1])` |

**为什么值得单独列出来**：这三个累计量在本目录里没有被展开成显式公式，所以

- 你**无法**在脚本里替换 `src`（PVT 的 `Source` 输入因此形同虚设，见附录 B-3）；
- A/D 的除零、OBV 的平盘处理都藏在平台实现里，**自写回测时数值必然对不上**，除非按上表口径复现。

---

## 附录 B：坑与瑕疵清单

| # | 文件 | 问题 | 影响 / 建议 |
| --- | --- | --- | --- |
| 1 | `net_volume.pine` | 首根 bar `sign(na) = na`，柱消失 | 无实际影响，滚动画布即可 |
| 2 | `accumulation_distribution_line.pine` | `high == low`（一字板、停牌复牌、低流动性）导致 `CLV` 除零 | 内置 `accdist` 的处理未公开。若依赖此指标，建议自写 `high != low ? ((close-low)-(high-close))/(high-low) : 0` |
| 3 | `price_volume_trend.pine` | **`src = input(...)` 声明了但从未使用**，公式用的是内置 `pvt`（始终基于 `close`） | 改 `Source` 不会有任何变化，容易误以为脚本支持 `hl2`。要真正可配置，需展开成 `cum(volume × change(src)/src[1])` |
| 4 | `percent_volume.pine` | 每一档参考线同时用 `plot(..., style=circles)` 和 `hline(...)` 画了两遍 | 纯装饰冗余：画面上是多了一层点状线，图例里多出 6 个空标题条目。可删掉全部 `plot(常量)` 只留 `hline` |
| 5 | `percent_volume.pine` | 颜色写 8 位十六进制 `#92df8bff` / `#fe9695ff` | v3 文档只定义 `#RRGGBB`，8 位（含 alpha）是 v4 起才正式支持。若编译报错就删掉末尾 `ff` |
| 6 | `volume_accumulation.pine` | `alertcondition(acc > accPrev)` 是**逐 bar** 判定，周期内 `acc` 几乎一直在增长 | 一个周期会触发几十次告警。要只保留"周期结束时对比"，改成 `period != period[1] and acc[1] > accPrev[1]`（切换 bar 上 `acc[1]` 是刚结束周期的总量，`accPrev[1]` 是再上一个周期的总量） |
| 7 | `volume_accumulation.pine` | 能选的周期**比图表周期细**时（如日线图上选 `Minutely`） | `minute(time)` 在所有 bar 上都是开盘分钟数，`period` 几乎不变 → 累加器永不清零，柱子变成一条单调上升的斜线，毫无意义。**粒度必须 ≥ 图表周期** |
| 8 | `volume_accumulation.pine` | `Session` 类型的 `period` 只有 0/1 | 时段内、时段外各累计一份，不是"一天一份"。要按交易日累计请选 `Daily` |
| 9 | `positive_volume_index.pine` / `negative_volume_index.pine` | `signalLength` 默认 255，在非日线上没有"一年"的含义 | 分钟图上这根信号线几乎是常数，交叉全是噪声 |
| 10 | 全部累计型脚本 | 累计量的绝对值取决于**加载了多少历史** | 不同人看到的 OBV/A-D/PVT 曲线高度不同，只有形状可比。做图截图对比前先统一 `bar_index` 起点 |
| 11 | `klinger_volume_oscillator.pine` | `Highlight Zero Line Crossovers` 默认 `true`，整张副图带淡色背景 | 视觉噪声大，且与"KVO 上穿信号线"的信号不是一回事，容易混淆 |
| 12 | 全部 9 个脚本 | 已全部迁移到 v6 | 原 v3/v4 的 `study()`、`transp=`、`input(type=...)`、`fill(color=..., transp=...)` 已替换为 `indicator()`、`color.new()`、`input.*()`、`ta.` 前缀 |

---

## 附录 C：怎么组合着用 / 怎么调参

**这 9 个脚本没有一个是策略**，它们描述的是"量在做什么"，不是"该不该买"。真要组合使用，按下面三层搭配：

1. **方向层**（谁在推动）→ `on_balance_volume.pine` 或 `accumulation_distribution_line.pine`
   看背离：价创新高而指标不创新高。默认 `Highlight Crossovers = false` 是对的，交叉噪声太大。

2. **相对量层**（今天算不算放量）→ `percent_volume.pine`
   这是唯一可以在不同品种之间横向比较的脚本。经验阈值：`< 50` 缩量、`100` 均量、`> 200` 异动。

3. **结构层**（这一段的量比上一段多还是少）→ `volume_accumulation.pine`（`Type = Daily`）
   柱子颜色就是"放量/缩量"的直接答案，比任何均线交叉都直观。记得按附录 B-6 改掉告警条件。

**调参建议**

| 脚本 | 参数 | 建议 |
| --- | --- | --- |
| OBV / A/D / PVT | 信号线长度 21 | 短线 10–13，中线 21，长线 50+。**信号线只影响颜色和填充，不改变指标值** |
| OBV / A/D / PVT | `Apply Ribbon Filling` | 打开 = 双色趋势图（默认）；关闭 = 蓝橙分离配色，更容易看清两条线的距离 |
| NVI / PVI | `Signal Length = 255` | 只在**日线**上保留 255；其他周期按"该周期一年有多少根"换算（如 4 小时 → 约 1500） |
| NVI / PVI | `Initial Value = 100` | 不用改。它是指数的起点，改动只改变曲线的绝对刻度，形状与交叉位置完全不变 |
| KVO | 34 / 55 / 13 | Klinger 原版参数（快 34、慢 55、信号 13），改动会偏离原著设定的量能周期。`Show Histogram` 建议打开，四色柱的信息量比两条线更大 |
| % Volume | `SMA Length = 21` | 越短越敏感；`5` 适合看日内，`21` 是月度基准 |
| Volume Accumulation | `Type = Daily` | 与图表周期保持一致或更粗；`Include Weekends` 在加密市场建议 `true`，股票市场建议 `false` |

**移植到其他平台时最容易踩的一步**：OBV / A/D / PVT 这三个用的是**平台内置变量**，不是脚本里可见的公式。换平台（Python、TradeStation、同花顺）时必须按附录 A 的口径手动实现，并显式处理 `high == low` 与平盘 bar，否则数值一定对不上。
