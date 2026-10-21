# Bands & Channels — 带状与通道类指标

本目录收录 7 个「中轨 + 上下轨 + 半透明填充」形态的 Pine Script 指标（全部 `overlay=true`，直接叠加在价格图上）。
它们回答的是同一个问题：**当前价格相对最近一段时间的"正常区间"偏离了多少？** 区别只在于「正常区间」怎么定义：

- 用**价格振幅**定义 → Acceleration Bands、STARC
- 用**统计分布 / 离散度**定义 → Interquartile Range Bands、Mean Absolute Deviation Bands、Kirshenbaum Bands
- 用**滤波器的残差**定义 → Vortex Bands
- 干脆不做波动度量，只用**平滑后的高/低点包络** → Moving Average Channel

## 文件清单

| 文件 | 指标（简称） | Pine 版本 | 默认参数 | 带宽来源 |
| --- | --- | --- | --- | --- |
| `acceleration_bands.pine` | Acceleration Bands (AB) | 6 | `length=20`, `factor=0.001` | 每日振幅占价格的比例（自适应） |
| `interquartile_range_bands.pine` | Interquartile Range Bands (IQRB) | 6 | `length=14`, `mult=1.5` | 四分位距 IQR |
| `kirshenbaum_bands.pine` | Kirshenbaum Bands (KB) | 6 | `length=30`, `regressionLength=20`, `mult=1.75` | 线性回归残差的标准误 |
| `mean_absolute_deviation_bands.pine` | Mean Absolute Deviation Bands (MADB) | 6 | `length=20`, `mult=2.0` | 平均绝对偏差 |
| `moving_average_channel.pine` | Moving Average Channel (MAC) | 6 | `upper/lowerLength=20`, `offset=0` | 无（SMA(high) / SMA(low) 包络） |
| `stoller_average_range_channels.pine` | Stoller Average Range Channels (STARC) | 6 | `length=6`, `MA=SMA`, `atrLength=15`, `mult=1.33` | ATR（真实波幅） |
| `vortex_bands.pine` | Vortex Bands (VB) | 6 | `length=20`, `mult=2`, `src=hlc3` | MNMA 平滑后的偏离量 |

> 所有脚本的作者版权声明均为 `Alex Orekhov (everget)`，GPL-3.0。

---

## 1. `acceleration_bands.pine` — Acceleration Bands (AB)

**做什么**：构造一组随"当日振幅/价格水平"自动张缩的通道，用来捕捉趋势加速与突破。

**原理**：由 Price Headley 提出。核心是每个 bar 计算一个相对振幅系数，再用它去缩放 high / low，最后分别做 SMA：

```
mult        = 4000 * factor * (high - low) / (high + low)
upperBand   = sma(high * (1 + mult), length)
basis       = sma(close, length)
lowerBand   = sma(low  * (1 - mult), length)
```

默认 `factor = 0.001`，`4000 * 0.001 = 4`，即 `mult ≈ 4 * (H-L)/(H+L)`，与原版 Headley 的 4 倍系数一致。`(H-L)/(H+L)` 是"相对振幅"：高价股与低价股、平静期与剧烈期都能横向可比，不需要额外做百分比换算。

**参数**

| 参数 | 默认 | 说明 |
| --- | --- | --- |
| `Length` | 20 | 上/中/下轨的 SMA 周期 |
| `Factor` | 0.001 | 振幅缩放因子，越大通道越宽（`step=0.0001` 便于微调） |
| `Source` | `close` | 仅中轨使用；上下轨固定用 `high` / `low` |

**读法**：收盘价站上上轨 → 上行动能加速（Headley 的原始用法即"突破上轨做多"）；跌破下轨 → 下行加速。价格在通道内运行时视为无加速趋势。由于带宽与当日振幅正相关，**波动放大时通道自动变宽**，可减少高波动期的假突破。

**注意**：通道整体滞后于价格（三级 SMA 平滑 + 带宽用的也是当根 bar 的振幅）；参数 `factor` 对宽度非常敏感，跨品种需要重新标定。

---

## 2. `interquartile_range_bands.pine` — Interquartile Range Bands (IQRB)

**做什么**：用非参数的**四分位数**构造通道，把价格映射到"最近 N 根 bar 的经验分布"上，属于稳健统计（robust statistics）思路。

**原理**：

```
q1     = percentile_nearest_rank(src, length, 25)
median = percentile_nearest_rank(src, length, 50)
q3     = percentile_nearest_rank(src, length, 75)
iqr    = q3 - q1
upper  = q3 + mult * iqr
lower  = q1 - mult * iqr
```

`percentile_nearest_rank` 取的是窗口内的次序统计量（把 N 个收盘价排序后按百分比位置取值）。`mult = 1.5` 正好是 Tukey 箱线图判定离群值的经典阈值，所以上下轨可以理解为"最近 N 根 bar 的统计离群边界"。

**参数**

| 参数 | 默认 | 说明 |
| --- | --- | --- |
| `Length` | 14 | 分位数统计窗口 |
| `Multiplier` | 1.5 | IQR 倍数（箱线图惯例：1.5 温和离群，3.0 极端离群） |
| `Source` | `close` | 统计对象 |

**读法**：收盘价高于上轨 → 相对近期分布处于极端强势（可视为超买/突破）；低于下轨 → 极端弱势（超卖/破位）。中轨是**中位数**而非均值，因此不受极端值牵引，作为均值回归的锚点比 SMA 更稳。

**优点与局限**：不假设正态分布、对离群值不敏感、不会因为一根大阴线把整条通道拉宽。代价是 `length` 较小时分位数只有几个离散取值，通道会**呈阶梯状跳变**；`length=14` 时 25%/75% 分位仅由 3~4 个样本决定，噪声较大。

---

## 3. `kirshenbaum_bands.pine` — Kirshenbaum Bands (KB)

**做什么**：以 EMA 为中轨、以**线性回归残差的标准误**为带宽的通道，兼具趋势中轨与"趋势线拟合质量"两层信息。

**原理**：

```
regression = linreg(src, regressionLength, 0)                 // 当前 bar 的回归拟合值
stderror   = mult * sqrt(sma(pow(src - regression, 2), regressionLength))
basis      = ema(src, length)
upper      = basis + stderror
lower      = basis - stderror
```

带宽度量的是 **收盘价围绕回归线的离散程度**（残差平方的均值再开方，即总体标准差，分母为 N 而非 N−1）。它刻意与中轨"解耦"：中轨用 `length`（默认 30）的 EMA 提供趋势基准，带宽用 `regressionLength`（默认 20）衡量短期抖动，两者可以独立调。

**参数**

| 参数 | 默认 | 说明 |
| --- | --- | --- |
| `Length` | 30 | EMA 中轨周期 |
| `Regression Length` | 20 | 回归拟合与残差计算的窗口 |
| `Multiplier` | 1.75 | 标准误倍数 |
| `Source` | `close` | 计算对象 |

**读法**：带宽收窄 = 价格紧贴回归线运行 = 趋势整齐、单边性强；带宽扩张 = 价格反复穿越回归线 = 震荡、趋势不成立。价格突破带宽即偏离"回归通道"，可视为趋势延续信号；结合收窄后的突破（类似挤压/挤压释放）效果更好。

**注意**：`linreg` 描述的是**斜率稳定的线性趋势**，遇到明显弯曲（抛物线式加速或 V 型反转）时残差会骤增，带宽随之突然放开。

---

## 4. `mean_absolute_deviation_bands.pine` — Mean Absolute Deviation Bands (MADB)

**做什么**：布林带（Bollinger Bands）的稳健替代品——结构一模一样，只把标准差换成**平均绝对偏差**。

**原理**：

```
basis = sma(src, length)
dev   = mult * dev(src, length)        // Pine 内置 ta.dev：mean(|src - SMA(src, length)|)
upper = basis + dev
lower = basis - dev
```

内置 `dev()` 计算的是 `1/n * Σ|x_i − SMA|`（平均绝对偏差，MAD），不是标准差，也不做平方。

**参数**

| 参数 | 默认 | 说明 |
| --- | --- | --- |
| `Length` | 20 | SMA 与 MAD 的窗口 |
| `Bands Multiplier` | 2.0 | 与布林带默认的 2σ 对齐，便于直接对比 |
| `Source` | `close` | 计算对象 |

**读法**：与布林带用法一致——触上轨偏热、触下轨偏冷、带宽收缩代表波动率收敛（squeeze）。真正的区别在带宽稳定性：标准差对极端值做了平方放大，一根大 K 线就能把布林带撑开；MAD 只线性加权，所以 **MADB 在跳空、暴涨暴跌后不会突然"鼓包"**，突破/回归判断更连续。

**注意**：`mult=2` 的 MAD 带在覆盖概率上并不等价于 2σ（正态假设下 MAD ≈ 0.8σ），因此默认 MADB 比默认布林带**更窄**，触轨更频繁；跨指标比较时不要直接套用同一个倍数。

---

## 5. `moving_average_channel.pine` — Moving Average Channel (MAC)

**做什么**：最朴素的一种通道——把「N 期最高价的均值」和「N 期最低价的均值」当作上下轨。它唯一的"波动度量"就是高低的自然分离。

**原理**：

```
upper = sma(upperSrc, upperLength)   // 默认 sma(high, 20)
lower = sma(lowerSrc, lowerLength)   // 默认 sma(low, 20)
```

可以理解为 **Donchian Channel（唐奇安通道）的平滑版**：Donchian 用 `highest(high, n)` / `lowest(low, n)`，这里改用 SMA，避免单根极值造成的台阶式跳变。上、下轨的 source / length / `offset` 完全独立，因此可以做一个"不对称通道"（例如上轨 20 期、下轨 55 期），也可以把某一条轨水平前后平移。

**参数**

| 参数 | 默认 | 说明 |
| --- | --- | --- |
| `Upper Source` / `Upper Length` / `Upper Offset` | `high` / 20 / 0 | 上轨的取样源、周期、水平平移（正数向右/未来方向） |
| `Lower Source` / `Lower Length` / `Lower Offset` | `low` / 20 / 0 | 下轨同上 |

**额外的柱状着色**：

```
close > upper → lime（绿）
close < lower → red（红）
其余          → gray（灰）
```

即"收盘价站上上轨"标记为强势、"跌破下轨"标记为弱势、"夹在通道内"标记为中性，便于用 `barcolor` 快速目测趋势状态。

**读法**：收盘价上穿上轨 → 突破近期平均高点、动量向上；下破下轨 → 相反。通道本身的宽窄反映的是近期**区间振幅的平均水平**（趋势扩张时开口、横盘时收窄）。因为中轨缺失（画面中间是空白带而非一条线），它更适合判断突破，而不是做均值回归锚点。

---

## 6. `stoller_average_range_channels.pine` — STARC Bands

**做什么**：中轨 = 均线，带宽 = **ATR 的倍数**。以"真实波幅"而非"收盘价离散度"衡量风险，是趋势跟踪类通道（与 Keltner Channel 同族，Keltner 早期版本同样用 SMA ± ATR）。

**原理**：

```
basis     = SMA/EMA/WMA/VWMA(src, length)      // 默认 SMA(close, 6)
upperBand = basis + atr(length) * mult
lowerBand = basis - atr(length) * mult
```

ATR 基于 `max(high-low, |high-prevClose|, |low-prevClose|)`，因此**把跳空缺口也计入波动**——这是它相对布林带（只看收盘价）最关键的优势。

**参数**

| 参数 | 默认 | 说明 |
| --- | --- | --- |
| `Length` | 6 | 中轨均线与 ATR 的周期（见下方注意） |
| `MA` | `SMA` | 中轨均线类型：EMA / SMA / VWMA / WMA |
| `ATR Length` | 15 | **声明了但当前实现未使用** |
| `Multiplier` | 1.33 | ATR 倍数 |
| `Source` | `close` | 仅中轨使用 |

**注意（实现现状）**：脚本里带宽写的是 `atr(length)`，也就是跟随 `Length`（默认 6），而输入项 `atrLength`（默认 15）在后续计算中没有被引用。经典 STARC 的组合是 **SMA(6) ± 1.33 × ATR(15)**：短周期均线提供灵敏的趋势基准，长周期 ATR 提供更平滑的带宽。当前默认值下带宽用的是 6 期 ATR，比经典设定**敏感得多、更容易被单根大 K 线带偏**。若想还原经典行为，需要把 `atr(length)` 改为 `atr(atrLength)`。

**读法**：与 Keltner / 布林带一致——沿上轨运行是强趋势；回归中轨是常态；上下轨之间的宽度直接等价于"以 ATR 计的风险"，因此也常被用来做止损距离（`mult` 倍的 ATR 距中轨）与仓位换算。默认中轨只有 6 期，价格会频繁上下穿越它，**不要把穿越中轨读成趋势反转**。

---

## 7. `vortex_bands.pine` — Vortex Bands (VB)

**做什么**：用一条**低滞后均线（MNMA）**做中轨，再用同一条均线去平滑"价格相对中轨的偏离量"作为带宽。是目录里唯一自带源码级自定义函数、也是唯一带多种柱状着色模式的脚本。

**原理**：

```
_ema(src, alpha) =>                     // 以给定 alpha 递归平滑
    out = alpha * out + (1 - alpha) * nz(out[1], out)
    out

_mnma(src, length) =>                   // McNicholl MA
    alpha = 2 / (length + 1)
    ema1  = _ema(src, alpha)
    ema2  = _ema(ema1, alpha)
    ((2 - alpha) * ema1 - ema2) / (1 - alpha)

basis = _mnma(src, length)
dev   = mult * _mnma(src - basis, length)   // 对"偏离序列"再做一次 MNMA
upper = basis + dev
lower = basis - dev
```

- `_mnma` 结构上是 **Tillson T3 / 广义 DEMA 的无体积因子形式**（即 `(2-α)·EMA1 − EMA2)/(1-α)`），比同周期 EMA 滞后更小，用来做中轨可以更快跟随价格。
- 带宽不是对 `src` 求标准差，而是先取 `src - basis`（带符号的偏离量），再对这条偏离序列做同样的平滑。由于滤波器是线性的，`dev` **会保留符号**：价格长期在中轨下方时 `src - basis` 持续为负，`dev` 可能为负，于是 `upper` 反而跑到 `lower` 下方（通道"倒挂"）。这也是脚本里第一个着色选项要判断 `upper > lower` 的原因。

**参数**

| 参数 | 默认 | 说明 |
| --- | --- | --- |
| `Length` | 20 | 中轨与带宽的 MNMA 周期 |
| `Multiplier` | 2 | 偏离量倍数 |
| `Source` | `hlc3` | 计算对象（默认典型价，比 close 更平滑） |
| `Bar Coloring` | `None` | 柱状着色模式，见下 |

**柱状着色模式**

| 选项 | 逻辑 |
| --- | --- |
| `None` | 不着色 |
| `Upper > Lower` | 上轨在上的常态区间内为绿，通道倒挂时为红（用于警示带宽异常） |
| `Close Above/Below Basis` | `close > basis` 绿，否则红（简单的多空状态） |
| `Close Inside Cloud` | 收盘价落在上下轨之间（无论是否倒挂）为绿，出带为红 |
| `Wicks Inside Cloud` | 整个高低点都落在带内为绿，任一影线出带为红 |

**读法**：因为带宽来自"平滑后的偏离量"，它和价格偏离程度**同向自适应**——单边行情中 `dev` 会持续累积放大，通道随之张开；震荡行情中偏离量正负抵消，带宽收窄。使用时重点看中轨方向 + 价格是否保持在带内。`Upper > Lower` 着色是一种实用的自检开关，能立刻暴露倒挂区间。

**注意**：脚本名里的 "Vortex" 与常见指标 **Vortex Indicator (VI, Botes & Siepman)** 无关，只是借用了同一名词，不要混淆。倒挂现象说明带宽度量本身不保证非负，用它做程序化信号前应先对 `dev` 取绝对值或裁剪。

---

## 执行后得到什么？（输出清单）

把这些脚本加载到图表并执行完，产物只有两类：**价格图上的可视化叠加层**，以及 **Data Window（数据窗口）里的一组数值序列**。全部 7 个脚本都没有 `alertcondition` / `plotshape` / `label` / `table`，因此**不产生买卖信号、告警或回测收益**。其中 2 个会额外改写 K 线颜色。

| 脚本 | 图上画出的线 | 填充区 | K 线着色 | Data Window 列名 | 首根有效 bar |
| --- | --- | --- | --- | --- | --- |
| `acceleration_bands` (AB) | `Upper` 青绿 `#138484`、`Basis` 酒红 `#741b47`、`Lower` 青绿，均 1px | 淡黄 `#ffd966`，transp 84（很淡） | 无 | Upper / Basis / Lower | 第 20 根起 |
| `interquartile_range_bands` (IQRB) | `Upper`、`Median`、`Lower`（青绿 / 酒红 / 青绿），1px | 淡黄，transp 84 | 无 | Upper / Median / Lower | 第 14 根起 |
| `kirshenbaum_bands` (KB) | `Basis` 深紫 `#351c75`、`Upper` / `Lower` 蓝灰 `#45818e`，1px | 淡黄，transp 84 | 无 | Basis / Upper / Lower | 第 30 根起（`max(length, regressionLength)`） |
| `mean_absolute_deviation_bands` (MADB) | `Basis` 深紫、`Upper` / `Lower` 蓝灰，1px | 淡黄，transp 84 | 无 | Basis / Upper / Lower | 第 20 根起 |
| `moving_average_channel` (MAC) | **只有** `Upper` 中蓝 `#3c78d8`、`Lower` 橙 `#f57c00`，1px，可带 `offset` 平移 | 淡黄，transp 90（更淡） | **有（强制）**：`close > Upper` 亮绿、`close < Lower` 红、其余灰 | Upper / Lower | 第 20 根起 |
| `stoller_average_range_channels` (STARC) | `Upper` 蓝、`Basis` 紫 `#6d1e7f`、`Lower` 红，**全部 2px** | **无填充**（源码没有 `fill`） | 无 | Upper / Basis / Lower | 约第 7 根起（ATR 依赖 `prevClose`） |
| `vortex_bands` (VB) | `Basis` 暗红 `maroon` 1px、`Upper` 蓝 2px、`Lower` 橙 2px | 淡黄，transp 90 | 有，但默认 `None` 不着色 | Basis / Upper / Lower | 第 1 根起（`nz(out[1], out)` 自初始化） |

### 每条线的数值形态（真正的差别在这里）

- **AB**：三条线**互不对称**。`Upper = SMA(high · (1 + mult))`、`Lower = SMA(low · (1 − mult))`，而 `mult = 4000 · factor · (H−L)/(H+L)` 每根 bar 都在变，所以通道宽度 ≈ 近期相对振幅的加权平均；`Basis` 只是 `SMA(close)`，**不保证落在带宽正中**。
- **IQRB**：严格有序 `Upper ≥ q3 ≥ Median ≥ q1 ≥ Lower`，带宽恒为 `(1 + 2 · mult) · IQR`（默认 `mult = 1.5` → 4 倍 IQR）；因为分位数取的是次序统计量，线呈**阶梯状**而非平滑曲线。
- **KB**：上下轨**关于 `Basis` 严格对称**，半宽 `= mult · sqrt(mean(残差²))`。带宽放大即"价格离开回归线更远/趋势更不整齐"。
- **MADB**：上下轨关于 `Basis` 对称，半宽 `= mult · MAD`（线性、无平方），因此跳空或大阳线之后不会出现布林带那种「鼓包」。
- **MAC**：上下轨来自两个独立 SMA，**没有共享中轨**，间距 = 近期「平均高点」与「平均低点」的分离度。视觉上最抢眼的是它立刻把全图 K 线改成绿 / 红 / 灰三态；`offset` 设为正数会把对应那条轨画到未来区域。
- **STARC**：上下轨关于中轨对称，半宽 `= 1.33 · ATR(length)`（用的是 `length`，而非那个未生效的 `atrLength`）。三条都是 2px 粗线且**没有云带**，中间是空白。
- **VB**：半宽 `= mult · MNMA(src − basis)`，**可正可负**，所以会周期性看到 `Upper` 掉到 `Lower` 下方（通道倒挂）——这正是它第一个着色选项 `Upper > Lower` 专门检测的状态。

### 不会得到的东西

- 买卖箭头 / 入场出场标记 / 收益曲线：全部是 `study`（indicator），不能作为 `strategy` 回测。
- 内置告警：没有 `alertcondition`，只能基于已有 plot 在 TradingView 的「创建警报」里手动挂。
- 独立副图：全部 `overlay=true`，直接画在价格图上。
- 多脚本共存时的稳定着色：`barcolor` 会互相覆盖（MAC 的灰 / 绿 / 红与 VB 的设置后加载者优先），同时对比多个指标时建议只保留一个负责着色。
- 顺带提醒：`MAC` 的缩写与 MACD 极易混淆，图表左上角显示的是 `MAC`，不是 MACD。

## 横向对比：该选哪一个？

| 需求场景 | 推荐 | 理由 |
| --- | --- | --- |
| 突破 / 趋势加速 | `acceleration_bands.pine` | 带宽随相对振幅自适应，趋势期不会因波动放大而频繁假突破 |
| 趋势跟随 + 止损距离 | `stoller_average_range_channels.pine` | ATR 含跳空，带宽可直接换算成风险单位 |
| 均值回归 / 超买超卖 | `interquartile_range_bands.pine`、`mean_absolute_deviation_bands.pine` | 中位数/平均绝对偏差都是稳健统计量，不会被单根暴涨暴跌扭曲 |
| 判断"趋势是否整齐" | `kirshenbaum_bands.pine` | 带宽 = 回归残差，能直接读出拟合质量 |
| 纯价格结构包络、要不对称或平移 | `moving_average_channel.pine` | 上下轨 source/length/offset 完全解耦，等价于平滑版 Donchian |
| 需要低滞后中轨 | `vortex_bands.pine` | MNMA 比同周期 EMA 滞后更小 |

## 本目录脚本的共同约定

- 全部 `overlay=true`，输出三件套：`Basis`（中轨，深紫/暗红系）、`Upper` / `Lower`（青蓝/蓝/橙）、以及两者之间的半透明 `fill`（`#ffd966` 淡黄）。
- 参数模式统一为 **`Length` + `Multiplier` + `Source`**，便于横向替换测试。
- `MAC` 与 `VB` 额外通过 `barcolor()` 提供柱状着色；其余脚本无信号输出。

## 迁移到 Pine v5 / v6 的改动清单

本目录为历史脚本（v3 / v4 语法），在新版编辑器中需做如下机械替换：

| 旧（v3/v4） | 新（v5+） |
| --- | --- |
| `study(title=..., overlay=true)` | `indicator(..., overlay=true)` |
| `input(type=integer / type=float)` | `input.int(...)` / `input.float(...)` |
| `input(type=source, defval=close)` | `input.source(close)` |
| `color(#ffd966, 84)` | `color.new(#ffd966, 84)` |
| `plot(..., transp=0)` | 去掉 `transp`，改用 `color.new(color, 0)` |
| `sma` / `ema` / `wma` / `vwma` / `atr` / `dev` / `linreg` / `percentile_nearest_rank` | 加 `ta.` 前缀 |
| `iff(cond, a, b)` | 三元 `cond ? a : b` |
| `pow(x, 2)` | `math.pow(x, 2)` 或 `x * x` |
