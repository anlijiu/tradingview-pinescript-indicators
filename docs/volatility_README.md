# volatility —— Pine Script 波动率指标（6 个脚本）

本目录收录 **6 个**用 Pine Script 写的波动率指标（5 个 `//@version=3`、1 个 `//@version=4`）。

它们回答的是同一个问题：**「这段时间波动有多大？」** —— 但「波动」不是一个数，必须先做三个选择才能算出一个数：

1. **原子量**：用哪一根 bar 上的什么量当基本单位？（`high - low` / `true range` / 收盘价相对高点的回撤 / 价格本身）
2. **聚合方式**：单根的量怎么变成窗口的量？（SMA / Wilder RMA / 平方均值开方 / 滚动极差）
3. **归一化基准**：除不除、除什么？（除以自身历史 / 除以窗口长度 / 除以均线 / 除以价格 / 除以窗口区间 / 不除）

这 6 个脚本正好覆盖了 6 种不同的组合，所以它们的**单位、量级、可比性完全不同**：

| 脚本 | 原子量 | 聚合 | 归一化 | 输出单位 | 默认窗口 |
| --- | --- | --- | --- | --- | --- |
| `damping_index.pine` | `high - low` | SMA | **除以 6 根前的同一个值** | 比值（≈1） | 5 |
| `gopalakrishnan_range_index.pine` | `highest - lowest` | — | `log(range) / log(L)` | 无量纲对数刻度 | 5 |
| `mayer_multiple.pine` | `close`（BTC） | SMA | 除以 200 根 SMA | 倍数 | 200 |
| `normalized_average_true_range.pine` | `tr` | **_RMA_**（Wilder） | 除以当根 `close` ×100 | **百分比** | 14 |
| `ulcer_index.pine` | 相对滚动高点的回撤 | RMS | 不额外归一化 | **百分比** | 14 / 14 |
| `volatility_ratio.pine` | `tr` | — | 除以窗口区间宽度 | 比例（0~1，可 >1） | 14 |

**没有任何一个是 `strategy`**：不能回测、不下单。
**没有任何一个带 `alertcondition`**：这 6 个脚本**一个警报都设不了**，只能看图（`ulcer_index.pine` 有 `plotshape` 圆点，但那是绘图，不是警报）。
**全部是副图**（没有 `overlay = true`）——包括 `mayer_multiple.pine`，它画的是一条「价格/均线」的比值线，看起来像价格，其实不是。

> **「执行」的含义**：这些是 TradingView 的指标脚本，不是本地可执行文件。仓库里没有 `.py` / CLI 版本，也不会有文件输出。所谓「执行完得到什么」= 在 TradingView 的 Pine Editor 里编译通过 → 添加到图表 → **多出一个副图指标面板**，里面是：一组输入参数、若干条线/柱/背景、以及状态栏上「最后一根 bar 的读数」。想本地复现，只能按附录 A 的口径自己用 pandas 重写。

---

## 一、先读这一节：波动率指标的三种「口径陷阱」

这 6 个脚本的代码都很短（最长 36 行），但几乎每一个都埋了至少一个「不读代码就会理解错」的点。看懂下面三节，6 个脚本就不用逐个啃了。

### 1.1 陷阱一：`high - low` ≠ `true range`

- `damping_index.pine`、`gopalakrishnan_range_index.pine` 用的是 **`high - low`**：只量当根自己的影子长度，**跳空完全不计入**。
- `normalized_average_true_range.pine`、`volatility_ratio.pine` 用的是 **`true range`**：`max(high-low, |high-close[1]|, |low-close[1]|)`，把「今天开盘直接跳过昨天收盘」的那段缺口也算进去。

同一个跳空日，前者会显示「波动没变」，后者会显示「波动暴增」。跨品种对比（A 股有涨跌停、加密 24/7）时这个差别尤其大。

### 1.2 陷阱二：`atr()` 用的是 Wilder RMA，不是 SMA

Pine 的 `atr(length)` = `rma(tr, length)`，递推式是

```text
ATR_t = ATR_{t-1} + (TR_t - ATR_{t-1}) / length        // alpha = 1/length
```

它和 `sma(tr, length)` 的数值**不一样**（RMA 对早期数据权重衰减更慢，且首值初始化方式不同）。自己用 pandas 复现时必须是 `tr.ewm(alpha=1/length, adjust=False).mean()`，用 `rolling(14).mean()` 对不上。

### 1.3 陷阱三：归一化基准决定「能不能跨品种比」

```text
绝对 ATR        →  10（美元），换一个品种就没意义
NATR            →  2.3（%），可以跨品种比
DI / GAPO / VR  →  无量纲，但只和自己的历史比才安全
Ulcer Index     →  百分比，但它衡量的是「回撤」不是「波动」
Mayer Multiple  →  倍数，且只对 BTC 有历史分档
```

**GAPO 是最不能跨品种比的一个**：它的分子是价格区间（美元），分母是 bar 数，量纲不一致。5 美元的股票和 500 美元的 BTC，形状一样，GAPO 会差出一大截（详见 §3.2）。

---

## 二、快速索引

| # | 脚本 | 版本 | 一句话作用 | 运行后看到什么 | 输入参数 |
| --- | --- | --- | --- | --- | --- |
| 1 | `damping_index.pine` | 6 | 最近 5 根的平均振幅 ÷ 再往前 5 根的平均振幅 | 副图橙/蓝折线 + 1.5 虚线 + 橙色背景带 | SMA Length, Threshold |
| 2 | `gopalakrishnan_range_index.pine` | 6 | 把 5 根的价格包络取对数压成一个标量 | 副图一条 2px 折线，涨绿跌红 | Length |
| 3 | `mayer_multiple.pine` | 6 | **BTC** 价格 ÷ BTC 200 根 SMA | 副图折线 + 2.4 虚线 + 橙色背景带 | SMA Length, Threshold |
| 4 | `normalized_average_true_range.pine` | 6 | ATR 占价格的百分比 | 副图一条橙色线（无阈值、无变色） | Length |
| 5 | `ulcer_index.pine` | 6 | 相对滚动高点的回撤的 RMS（回撤痛苦度） | 副图 UI 线 + 信号线 + 色带 + 交叉圆点 + 1.5 虚线 + 橙色背景 | 9 个参数 |
| 6 | `volatility_ratio.pine` | 6 | 当根真实波幅 ÷ 最近 14 根区间宽度 | 副图一条 2px 折线（绿/暗红）+ 0.5 虚线 | Length, Breakout Level |

各脚本的图形对象数量（可用来判断「还能不能再加东西」）：

| 脚本 | `plot` | `hline` | `bgcolor` | `fill` | `plotshape` |
| --- | --- | --- | --- | --- | --- |
| `damping_index` | 1 | 1 | 1 | 0 | 0 |
| `gopalakrishnan_range_index` | 1 | 0 | 0 | 0 | 0 |
| `mayer_multiple` | 1 | 1 | 1 | 0 | 0 |
| `normalized_average_true_range` | 1 | 0 | 0 | 0 | 0 |
| `ulcer_index` | 2 | 1 | 1 | 1 | 2 |
| `volatility_ratio` | 1 | 1 | 0 | 0 | 0 |

---

## 三、逐个说明

### 3.1 `damping_index.pine` —— Damping Index（DI，振幅阻尼）

**做什么**：比较「最近 L 根的平均振幅」和「再往前 L 根的平均振幅」，得到一个倍数，用来判断高低点是在**收拢**还是在**张开**。

**输入**：`SMA Length`（默认 5）、`Threshold`（默认 1.5）。

**核心代码**（第 9–11 行）：

```pine
sma = sma(high - low, smaLength)
di  = sma[1] / sma[6]
```

**原理**：

1. **原子量**：单根振幅 `R_t = high_t - low_t`（不含跳空）。
2. **平滑**：`X_t = SMA(R, 5)_t`，把单根噪声压掉。
3. **比值**：`DI_t = X_{t-1} / X_{t-6}`。
   - 分子 `X_{t-1}` 覆盖 bar `t-1 … t-5`；
   - 分母 `X_{t-6}` 覆盖 bar `t-6 … t-10`；
   - 两段**完全不重叠**，正好是「最近 5 根 vs 再往前 5 根」。
4. **与原始公式等价**：因为 SMA 是线性的，`SMA(high - low) = SMA(high) - SMA(low)`，所以这个式子就是 1992 年原文的

   ```text
   DI = (MA(high,4)[1] - MA(low,4)[1]) / (MA(high,4)[6] - MA(low,4)[6])
   ```

5. **出处**：Curtis McKallip Jr., *"The Damping Index"*, Technical Analysis of Stocks & Commodities, V.10:7（1992 年 7 月, pp. 296–299）。原始动机：找「高点与低点越来越靠近」的位置——振幅衰减（damping）之后往往跟着一次扩张，所以它是一个**压缩/释放**的度量，本身不带方向。

**数值怎么读**：

| DI | 含义 |
| --- | --- |
| = 1 | 与 5 根前的振幅持平 |
| > 1 | 振幅在**扩张** |
| = 1.5 | 比 5 根前放大 50%（脚本在此高亮） |
| < 1 | 振幅在**收缩**（真正的 "damping"） |

> ⚠️ **方向最容易搞反**：本实现的分子是「近」、分母是「早」，所以 **>1 = 扩张、<1 = 收缩**。网上（包括部分二手资料）会用「high = 收缩」来描述原指标，那是把分子分母反过来讲的。**以代码为准**：`di >= threshold`（默认 1.5）高亮的是振幅放大。

**运行后得到什么**：

| 元素 | 位置 | 说明 |
| --- | --- | --- |
| `plot(di, ...)` | 副图 | 一条折线：`di >= threshold` 为橙色 `#ff9800`，否则蓝色 `#3c78d8` |
| `hline(threshold)` | 副图 | 灰色点线阈值 |
| `bgcolor` | 副图背景 | `di >= threshold` 时橙色、`transp=90`（很淡）；否则用 `color(white, 100)`（全透明）＝不画 |

- 状态栏显示最后一根的 DI 值。
- 无信号、无箭头、无表格、无 alert。

**坑**：

1. **参数和硬编码索引是耦合的**。`smaLength` 只改变 SMA 的窗口，`[1]` 和 `[6]` 是写死的。若把 `SMA Length` 改成 10，分子覆盖 bar 1–10、分母覆盖 bar 6–15，**两段重叠 5 根**，比值会被拉向 1，指标基本钝化。想改长度必须同时改索引（保持 `索引差 = smaLength`），或者干脆别动这个参数。
2. **除零**：连续 5 根 `high == low`（一字板、停牌、极低波动）时 `sma[6] == 0`，`di` 变成 `na` / `inf`，图上断线或出现尖峰。
3. **跳空不计入**：用 `high - low`，跳空日 DI 会低估真实波动。
4. 阈值 1.5 是经验值，不随品种/周期自适应。

---

### 3.2 `gopalakrishnan_range_index.pine` —— Gopalakrishnan Range Index（GAPO）

**做什么**：把「最近 L 根的价格包络有多宽」取对数，再除以 `log(L)`，压成一个可比大小的标量。

**输入**：`Length`（默认 5）。

**核心代码**（第 8 行）：

```pine
gapo = log(highest(length) - lowest(length)) / log(length)
```

> `highest(length)` 是 **v3 的遗留简写**，等价于 `highest(high, length)`；v4 中它已 deprecated，v5+ 必须写全两个参数。

**原理**：

1. **包络**：`R_t = highest(high, L) - lowest(low, L)`（窗口含当前 bar）。
2. **取自然对数**：把「R 翻倍」变成固定增量 `+ln2`，压缩量级、让 R 的相对变化近似线性。
3. **除以 `ln(L)`**：抵消窗口长度的影响，使不同 `Length` 的读数大致在同一量级。
4. **由此得到几个可以直接验算的性质**（L = 5，`ln5 ≈ 1.609`）：

   | 情形 | GAPO |
   | --- | --- |
   | `R = 5`（L=5，每根平均走 1 个价格单位且不重叠） | `ln5/ln5 = 1` |
   | `R = 1` | `0`（`ln1 = 0`） |
   | `R < 1`（低价品种、窗口区间小于 1 元） | **负数** |
   | R 每乘以 e（≈2.718） | `+1/ln5 ≈ +0.62` |

5. **出处**：Jayanthi Gopalakrishnan, *"What's Your Tradable?"*, S&C 2001 年 1 月。作者本人的意图**不是买卖信号**，而是给不同品种的「波动个性」排序，用来挑适合自己风格的标的。
6. **它到底是不是「分形维数」？** 形式上是「范围随窗口长度的标度指数」，但**不是严格的分形维数**，原因有两个：
   - **量纲不匹配**：分子是价格（美元），分母是 bar 数；
   - **缺少路径长度项**。真正的对数刻度指标是 Dreiss 的 **Choppiness Index**：

     ```text
     CHOP/100 = log(ΣTR) / log(n) - GAPO
     ```

     也就是说，GAPO 相当于「把 Choppiness 里的路径长度项拿掉，只留包络项」。
7. **正因为缺路径长度**，GAPO 只反映「包络张开/收拢」，不区分「一路单边推进」和「来回震荡但幅度很大」——这两种情况包络一样宽。

**数值怎么读**：

- **只能纵向自比**：当前 GAPO 在自己历史里的高低位置，才有意义。
- **上升** = 包络在张开（新高/新低在扩张）；**走平** = 区间锁死（`highest`/`lowest` 没更新时数值恒定，会画出平台期）。
- **颜色只表示方向**：`gapo >= gapo[1]` 绿色 `#0ebb23`，否则红色——不是「多空」，只是「比上一根高/低」。

**运行后得到什么**：

| 元素 | 说明 |
| --- | --- |
| 一条 2px 折线 | 副图，红绿交替 |
| — | **没有**阈值线、背景、信号、alert |

**坑**：

1. **依赖价格水平**。5 美元的品种和 500 美元的品种，即使走势形状一样，GAPO 会差一大截；同一品种换计价单位（美分→美元）、换拆分，历史读数会跳变。**绝对水平没有跨品种意义**。
2. **`log` 的定义域**：`R < 1` → 负数；`R = 1` → 0；若数据源出现零区间（连续一字板）→ `log(0)` = `na`。
3. **平台期**：窗口内没有新高/新低时 GAPO 完全不动，看起来像「指标坏了」。
4. **v3 简写**：`highest(length)` 在 v4 已 deprecated、v5+ 不合法，迁移时必须补 source 参数。

---

### 3.3 `mayer_multiple.pine` —— Mayer Multiple（MM）

**做什么**：**BTC 价格 ÷ BTC 的 200 根 SMA**，衡量比特币价格偏离长期均线的倍数。

**输入**：`SMA Length`（默认 200）、`Threshold`（默认 2.4）。

**核心代码**（第 9–11 行）：

```pine
price = nz(security("BTCUSD", period, close))
mm = price / sma(price, smaLength)
```

**原理**：

1. **分母**是长期均线（趋势中枢），**分子**是现价；比值 = 价格偏离长期均值的程度。它其实就是「均线乖离率」的倍数写法：`MM - 1 ≈ 乖离率`。
2. **200 根**是按「200 日」的惯例设的（在日线图上正好是 200 个交易日），不是数学推导出来的。
3. **出处**：Trace Mayer（约 2015 年），比特币圈最常用的宏观估值指标之一。
4. **经典分档**（**只对 BTC 日线成立**）：

   | MM | 含义 |
   | --- | --- |
   | < 0.8 | 历史上极度低估区 |
   | 0.8 – 1.0 | 低于 200 日均线（熊市/恐慌） |
   | 1.0 – 2.4 | 正常区间（长期均值约 1.4–1.5） |
   | ≥ 2.4 | 历史过热区（脚本在此高亮） |

   2.4 的来源是 Mayer 自己的回测：**在 MM < 2.4 时定投**长期收益最好。所以 2.4 是「定投区间的上沿」，不是机械的卖出信号。
5. **为什么要 `security()`**：脚本把标的**写死成 `BTCUSD`**，所以在任何图表（包括 AAPL、EURUSD）上画的都是 BTC 的 MM。`period` 是**当前图表周期**。

**运行后得到什么**：

| 元素 | 说明 |
| --- | --- |
| `plot(mm)` | 折线：`mm > threshold` 橙色 `#ff9800`，否则蓝色 `#3c78d8` |
| `hline(threshold)` | 点线阈值 |
| `bgcolor` | `mm > threshold` 时橙色、`transp=90`；否则全透明 |

- 状态栏显示最后一根 MM。
- 无 alert、无信号。

**坑**：

1. **画的是 BTC，不是当前品种**——这是本目录最容易误解的脚本第一名。在股票图上看到一条 1.2 附近的线，那是比特币相对它自己 200 根均线的倍数。
2. **`period` 是当前周期**：在 1 分钟图上算的是「BTC 的 200 根 1 分钟均线」，与「200 日」毫无关系。**Mayer Multiple 只有日线才有意义**。
3. **`nz()` 会把缺失值变成 0**：图表早期 BTC 无数据时 `price = 0`，SMA 被 0 污染，MM 会出现假的低值/断崖。
4. **时间轴对齐**：非 24/7 品种（股票、期货）的 bar 序列与 BTC 不同，`security` 对齐后可能出现重复 bar 或空洞。
5. 图表历史不足 200 根时，前段是 `na`，没有曲线。

---

### 3.4 `normalized_average_true_range.pine` —— Normalized ATR（NATR）

**做什么**：把 ATR（美元波幅）除以收盘价，转成**百分比波幅**，使波动率可以跨品种、跨时间比较。

**输入**：`Length`（默认 14）。

**核心代码**（第 8 行）：

```pine
natr = 100 * atr(length) / close
```

**原理**：

1. **原子量 TR**（真实波幅，含跳空）：

   ```text
   TR_t = max(high_t - low_t, |high_t - close_{t-1}|, |low_t - close_{t-1}|)
   ```

2. **ATR**：`RMA(TR, 14)`（Wilder 平滑，见 §1.2）。
3. **归一化**：除以当根 `close`，×100 变成百分比。
4. **意义**：`NATR = 5` 读作「平均每根 bar 的波动幅度约等于当前价格的 5%」。
5. **和 ATR 的分工**：ATR 是绝对价格（「平均波动 3.2 美元」），NATR 是相对比例（「平均波动 2.1%」）。**跨品种对比只能看 NATR**。

**数值量级**（经验值，取决于品种与周期）：

| 场景 | NATR 大致范围 |
| --- | --- |
| 日线大盘股 / 指数 ETF | 1% – 3% |
| 日线小盘股 / 高波动个股 | 3% – 8% |
| 日线加密 | 3% – 10% |
| 1 分钟股票 | 0.05% – 0.3% |

> 不同周期的 NATR 不能直接比：波动率大致按 √时间 缩放，1 分钟 0.1% 换到 390 分钟的交易时段约 `0.1% × √390 ≈ 2%`。

**运行后得到什么**：

| 元素 | 说明 |
| --- | --- |
| `plot(natr, color=#ff9800)` | 副图一条橙色曲线 |
| — | **没有**阈值线、没有变色、没有背景、没有 alert |

- 状态栏显示最后一根 NATR（百分比）。
- 这是本目录**输出最朴素**的脚本：它只提供一个「当前波动率水平」的读数，任何判断都要自己加。

**坑**：

1. **没有阈值、没有信号**：想让它可操作，得自己补 `hline()` / `alertcondition()`。
2. **分母是当根 `close`**：收盘价剧烈变化会给 NATR 带来额外噪声（严格说应该用窗口均价，但那是另一种定义）。
3. **Wilder RMA**：自己复现时不能用 `rolling().mean()`（见 §1.2）。
4. **周期不可比**：短周期的 NATR 天然更小，别拿 1 分钟和日线比高低。
5. **低价品种放大**：`close` 很小时，`100/close` 会把噪声放大（仙股上 NATR 可能长期两位数）。

---

### 3.5 `ulcer_index.pine` —— Ulcer Index（UI，回撤痛苦度）

**做什么**：把「相对最近 L 根最高价的回撤」平方、平均、再开方，得到一个**非负的百分比**，衡量「回撤有多深、持续多久」。

**输入**（9 个）：

| 参数 | 默认 | 作用 |
| --- | --- | --- |
| `Length` | 14 | 计算回撤的滚动窗口（找窗口最高价） |
| `Smoothing Length` | 14 | 对回撤平方做 SMA 的窗口 |
| `Breakout Level` | 1.5 | 高亮阈值（%） |
| `Signal Smoothing Type` | SMA | 信号线用 SMA 还是 EMA |
| `Signal Smoothing Length` | 52 | 信号线窗口 |
| `Source` | close | 用哪个价格序列算回撤 |
| `Show Signal ?` | true | 是否画信号线 |
| `Highlight Signal Crossovers ?` | true | 是否在交叉处画圆点 |
| `Apply Ribbon Filling ?` | true | 是否用趋势色填充两条线之间 |

**核心代码**（第 16–19 行）：

```pine
highest  = highest(src, length)
drawdown = 100 * (src - highest) / highest
ulcer    = sqrt(sma(pow(drawdown, 2), smoothLength))
signal   = signalType == "SMA" ? sma(ulcer, signalLength) : ema(ulcer, signalLength)
```

**原理**：

1. **滚动高点**：`H_t = max(src_{t-L+1}, …, src_t)`（含当前 bar）。
2. **百分比回撤**：`DD_t = 100 × (src_t - H_t) / H_t`，恒 **≤ 0**（创新高时为 0）。
3. **平方**：`DD²` 去掉符号，并且**对大回撤给平方级惩罚**（回撤 10% 的惩罚是 5% 的 4 倍）。
4. **时间平均**：`SMA(DD², 14)`。
5. **开方回百分比量纲** → 得到回撤序列的 **RMS（均方根）**：

   ```text
   UI_t = sqrt( mean(DD²) ) = 100 × RMS( (src - H) / H )
   ```

6. **出处**：Peter G. Martin（与 Byron McCann），1987 年提出，最初用来评估共同基金。作者的原话大意：「回撤越深、恢复到前高耗时越久，UI 越高」。
7. **和波动率的区别**：标准差对称看待涨跌；UI **只惩罚下跌**（相对高点的回撤），属于「下行波动率」。
8. **和最大回撤（MDD）的区别**：MDD 只看最痛的那一刻；UI 同时反映**深度 × 持续时间**——一条慢慢阴跌半年的曲线，MDD 可能不大，UI 会很高。
9. **信号线（52 根）**：UI 自身的长期均值，用来判断「当前回撤压力相对长期水平是高还是低」。

**数值怎么读**：

- `UI` 的值 = 「平均而言，当前价格比近期高点低了多少个百分点」。
- **UI 上升** = 回撤在加深/持续；**UI 回到低位** = 价格贴近窗口高点且稳定。
- `UI > signal` → 趋势色 **绿色**；`UI < signal` → **红色**。
- `UI > breakout`（默认 1.5）→ 背景橙色（突破级别）。
- 绿色圆点 = UI 上穿 signal；红色圆点 = 下穿。

> ⚠️ **配色语义反直觉**：这里的绿色表示「UI 高于信号线」，也就是**回撤压力在上升**，**不是看涨**。作者用的 `trendColor` 描述的是「UI 自身的趋势方向」，不是多空方向。第一次看图很容易读反。

**运行后得到什么**（本目录输出最丰富的脚本）：

| 元素 | 说明 |
| --- | --- |
| `plot(ulcer)` | 2px 主曲线；ribbon 模式下颜色 = `trendColor`，否则按「比上一根高低」变色 |
| `plot(signal)` | 信号线；ribbon 模式下颜色 = `trendColor`，否则蓝色 `#1155cc` |
| `fill(ulcer, signal)` | 两线之间的色带，`transp=70`（ribbon 模式下才填充） |
| `plotshape` ×2 | UI 上穿/下穿 signal 处的绿色/红色小圆点（`size.tiny`） |
| `hline(breakout)` | 点线阈值 |
| `bgcolor` | `UI > breakout` 时橙色、`transp=90` |

- 状态栏显示 `Ulcer` 和 `Signal` 两个读数。
- **没有 alert**。

**坑**：

1. **配色反直觉**（见上）。
2. **阈值 1.5% 不自适应**：对低波动个股可能永远不触发，对加密可能长期亮着。
3. **`Show Signal = false` 时圆点仍然会画**：`plotshape` 用的是 `signal` 变量，不受 `showSignal` 控制，只隐藏了线。
4. **`Apply Ribbon Filling = false` 时**：UI 颜色改成「与上一根比」，signal 变蓝色，填充消失——三处视觉同时变化。
5. **滚动窗口 vs 全历史**：这里用的是「最近 14 根的最高价」，与 StockCharts 的 14 期口径一致；但它**不会记住更早的深回撤**。想要「全历史回撤痛苦度」，得把 `highest` 换成 running max。
6. **创新高时 UI 会被拉低**：`DD = 0` 会稀释均值，这是定义使然，不是 bug。
7. **`src = 0` 的数据源**会让 `highest` 为 0 触发除零（正常价格序列不会）。
8. v4 的 `transp=` 参数在 v5+ 已移除（见附录 C）。

---

### 3.6 `volatility_ratio.pine` —— Volatility Ratio（VR）

**做什么**：**当根真实波幅 ÷ 最近 L 根的价格区间宽度**，衡量「今天这一根走完了近期区间多少比例」。

**输入**：`Length`（默认 14）、`Breakout Level`（默认 0.5）。

**核心代码**（第 9–12 行）：

```pine
max = max(highest(high, length), close[length + 1])
min = min(lowest(low, length), close[length + 1])

vr = tr(true) / (max - min)
```

**原理**：

1. **分子**：当根真实波幅 `TR_t`（含跳空）。
2. **分母**：最近 L 根 `high`/`low` 的包络，并额外把 **`close[length + 1]`**（窗口前一根的收盘）纳入——因为 TR 的参照点是「上一根收盘」，而跳空可能发生在窗口之外，不把收盘算进来，分母会漏掉那段缺口。
3. **比值含义**：

   | 当根走势 | VR |
   | --- | --- |
   | 在区间中间小幅震荡 | 小（0.1–0.3） |
   | 从区间一端打到另一端 | 接近 1 |
   | 跳空 + 大实体 | **可能 > 1** |

4. **出处**：这个指标在文献里被归到 **Jack D. Schwager**（MQL5、Investopedia 的表述），**不是** Donald Dorsey——Dorsey 做的是 RVI（相对波动率指数，用标准差替代 RSI 的涨跌幅，阈值 50/60/40）和 Mass Index，两者不是一回事。
5. **注意存在两种互不相容的定义**：
   - `TR / EMA(TR, N)`：无上界，阈值在 1 附近；
   - `TR / (HH - LL over N)`：**本脚本用的是这种**，天然落在 0–1（可 >1），阈值 0.5。

     所以「0.5」这个数**只对第二种定义有意义**。另外 Investopedia 版本的分母是「平均高点 − 平均低点」，本脚本是「最高点 − 最低点」，两者也不完全相同。
6. **0.5 的含义**：当根波幅超过近期区间的一半 → 常被当作**波动率突破日**（趋势跟随体系里常见的配套确认条件，通常还会叠加成交量或 RSI 过滤）。

**运行后得到什么**：

| 元素 | 说明 |
| --- | --- |
| `plot(vr)` | 2px 折线：`vr >= breakoutLevel` 绿色 `#39b361`，否则暗红 `#cc4125` |
| `hline(breakoutLevel)` | 灰色点线阈值 |

- 状态栏显示最后一根 VR。
- 无 alert、无信号。

**坑**：

1. **VR 会 > 1**：跳空、或窗口极窄（连续一字板后突然放量）时都可能超过 1，不要以为它被限制在 0–1。
2. **`close[length + 1]` 不是笔误**：那是窗口外一根的收盘，专门用来补齐 TR 的参照。改 `Length` 时这个 `+1` 要保留。
3. **它是「事件指标」不是「状态指标」**：单根尖峰后通常立刻回落，适合「今天是不是突破日」的判断，不适合「当前波动率水平」的判断（后者用 NATR）。
4. **阈值 0.5 与周期强相关**：短周期噪声大，0.5 的触发频率很高。
5. 分母为 0（连续 L 根一字板）→ `na` / `inf`。

---

## 四、6 个脚本合起来看

### 4.1 每个指标回答哪个问题

| 你关心的问题 | 用哪个 | 看什么 |
| --- | --- | --- |
| 振幅是在收拢还是张开？ | `damping_index` | DI 相对 1 的位置、是否过阈值（默认 1.5） |
| 包络整体多宽？ | `gopalakrishnan_range_index` | GAPO 在自己历史里的高低 |
| 当前波动率是几个百分点？ | `normalized_average_true_range` | NATR 的绝对水平与历史分位 |
| 今天这根算不算「猛」？ | `volatility_ratio` | VR 是否 ≥ 0.5 |
| 拿着这个标的有多少「回撤痛苦」？ | `ulcer_index` | UI 与其信号线、是否过阈值（默认 1.5） |
| 价格偏离长期均值多远？ | `mayer_multiple` | MM 相对 1 与 2.4（**仅 BTC 日线**） |

### 4.2 两个常见组合读法（经验约定，非回测结论）

**「压缩 → 释放」**：

```text
压缩期：GAPO 走平/处于历史低位 + DI < 1 + NATR 处于低位 + VR 长期低位
释放期：VR 上穿 0.5 + DI 上穿 1.5 + GAPO 转上行
```

这是 McKallip 原意（damping 之后跟扩张）和 VR 型突破确认的混合读法——**注意这只是读图约定，本目录没有任何一个脚本实现了完整策略**。

**「趋势里的风险监控」**：

```text
UI 持续低于信号线（红）→ 回撤压力小，趋势相对健康
UI 上穿信号线（绿）  → 回撤开始加深，考虑减仓/收紧止损
UI > 1.5（默认阈值）  → 进入历史级别的回撤区
```

**再次强调**：这些脚本都没有 `alertcondition`、都不是 `strategy`，所谓「信号」全部只存在于看图人的脑子里。要变成可执行逻辑，得自己补条件（见附录 D）。

---

## 附录 A：内置函数的精确口径（自己复现时照这个来）

| 函数 | Pine 口径 | 复现注意 |
| --- | --- | --- |
| `sma(src, n)` | 算术平均，窗口含当前 bar | `rolling(n).mean()` |
| `atr(n)` | `rma(tr, n)`，**Wilder 平滑** | `ewm(alpha=1/n, adjust=False).mean()`，不是 `rolling().mean()` |
| `tr(true)` | `max(high-low, abs(high-close[1]), abs(low-close[1]))` | 首根 bar 无前收，Pine 用 `high-low` 兜底 |
| `tr(false)` | 仅 `high - low` | — |
| `highest(src, n)` / `lowest(src, n)` | 窗口极值，**含当前 bar** | `rolling(n).max()` / `.min()` |
| `highest(n)`（v3 简写） | 等价 `highest(high, n)` | v4 deprecated，v5+ 必须写全 |
| `log(x)` | **自然对数**（不是 log10） | 换底不影响比值型指标，但影响 GAPO 的绝对刻度 |
| `pow(x, 2)` | 幂 | `x**2` |
| `sqrt(x)` | 开方；`x < 0` 时为 `na` | — |
| `nz(x)` | `na → 0` | MM 里会把缺失变成 0，注意污染 |
| `security(sym, res, expr)` | 取其他品种；v3 **没有** `lookahead` 参数（默认不前瞻） | v4+ 是 `request.security()` |
| `color(white, 100)` | v3 的「全透明颜色」 | 常用于让 `bgcolor` 在条件不满足时不画 |
| 历史引用 `x[n]` | 第 n 根之前的值 | 在 `sma[1]` 这种写法里，索引是**写死的常量**，不能是输入变量 |

---

## 附录 B：坑与瑕疵清单

| 脚本 | 坑 | 后果 | 处理 |
| --- | --- | --- | --- |
| `damping_index` | `SMA Length` 与 `[1]`/`[6]` 耦合 | 改成非 5 时比值被拉向 1，指标钝化 | 保持 = 5，或手动同步改索引 |
| `damping_index` | `sma[6] == 0` 除零 | `na`/断线/尖峰 | 加 `sma[6] == 0 ? na : ...` |
| `damping_index` | 方向定义与部分资料相反 | 把「扩张」读成「收缩」 | 以代码为准：>1 = 扩张 |
| `gopalakrishnan_range_index` | 量纲不一致（价格 ÷ bar 数） | 绝对值不可跨品种比 | 只看自身历史分位 |
| `gopalakrishnan_range_index` | `R < 1` 时为负 | 低价品种读数全负，不理解就以为坏了 | 用百分比分位替代绝对水平 |
| `gopalakrishnan_range_index` | 区间内无新高新低时恒定 | 平台期，看起来没反应 | 正常现象 |
| `mayer_multiple` | 硬编码 `BTCUSD` | 任何图表上都画 BTC 的 MM | 改字符串，或当作 BTC 参考线 |
| `mayer_multiple` | `period` = 当前周期 | 非日线时 200 根 ≠ 200 日 | **只在日线图使用** |
| `mayer_multiple` | `nz()` 把缺失变 0 | 早期数据段 MM 出现假低值 | 去掉 `nz` 或加 `na` 判断 |
| `normalized_average_true_range` | 无阈值、无变色 | 只能读数值，不能直接判断 | 自己加 `hline`/`alertcondition` |
| `normalized_average_true_range` | 分母是当根 `close` | 收盘剧烈变化时噪声大 | 接受，或改用均价 |
| `ulcer_index` | 绿色 = 回撤压力上升 | 读反多空 | 记住配色语义 |
| `ulcer_index` | `showSignal=false` 后圆点仍在 | 视觉不一致 | 把 `plotshape` 也乘上开关 |
| `ulcer_index` | 滚动窗口，不记全历史 | 早期深回撤不影响当前 UI | 需要全历史就换 running max |
| `ulcer_index` | 阈值 1.5% 不自适应 | 有的品种永远不触发 | 改成历史分位阈值 |
| `volatility_ratio` | 可 > 1 | 以为越界就是 bug | 跳空/窄窗口时正常 |
| `volatility_ratio` | 两种定义混用 | 0.5 阈值被误用到 `TR/ATR` 版本 | 确认分母是「区间」还是「平均 TR」 |
| `volatility_ratio` | 单根尖峰后立刻回落 | 当状态指标用会误判 | 当「事件」用（今天是否突破日） |

---

## 附录 C：迁移到 v5 / v6 的对照

本目录 6 个脚本已全部迁移到 v6，均可直接在 TradingView 编译；下面的替换对照表保留为参考，用于理解旧写法或改写为 strategy 时的对应关系：

| v3 / v4 写法 | v5 / v6 写法 |
| --- | --- |
| `study("X", shorttitle="Y")` | `indicator("X", shorttitle="Y")` |
| `input(title=, type=integer, defval=)` | `input.int(defval=, title=)` |
| `input(title=, type=float, defval=)` | `input.float(...)` |
| `input(title=, type=source, defval=close)` | `input.source(defval=close, title=)` |
| `input(title=, type=bool, defval=)` | `input.bool(...)` |
| `sma()` / `ema()` / `atr()` | `ta.sma()` / `ta.ema()` / `ta.atr()` |
| `highest()` / `lowest()` / `tr()` | `ta.highest()` / `ta.lowest()` / `ta.tr()` |
| `highest(length)`（v3 简写） | `ta.highest(high, length)` |
| `security(...)` | `request.security(...)` |
| `pow(x, 2)` / `sqrt` / `log` / `max` / `min` / `abs` | `math.pow` / `math.sqrt` / `math.log` / `math.max` / `math.min` / `math.abs` |
| `transp=0`（plot/fill/bgcolor 的命名参数） | 删除该参数，颜色改用 `color.new(c, 0)` |
| `color(white, 100)` | `color.new(color.white, 100)` 或直接 `na` |
| `linestyle=dotted`（hline） | `hline.style_dotted` |
| `plotshape(..., transp=0)` | `plotshape(...)` + `color.new()` |
| `overlay=false`（默认） | 不变：`indicator(..., overlay=false)` |

**另外 v6 对 `var` 的初始化时机与 `na` 的类型推断更严格**——迁移 `ulcer_index` 这类多对象脚本时，建议先复制到新脚本、逐个对象迁移、每步编译，不要一次性全改。

---

## 附录 D：想把读数变成信号 / 回测怎么办

这 6 个脚本都只 `plot`，没有任何可执行逻辑。三条改造路径：

**1. 加警报（最轻）**

```pine
// v5+ 写法；v3/v4 里去掉 ta. 前缀
alertcondition(ta.crossover(ulcer, signal), title="UI 上穿信号线", message="回撤压力上升")
```

**2. 在当前脚本里直接当变量用**

`di`、`gapo`、`mm`、`natr`、`ulcer`、`vr` 都是普通序列变量，复制到一个 `strategy()` 里就能参与条件判断：

```pine
longCond = vr >= 0.5 and di >= 1.5 and gapo > gapo[1]
```

**3. 把口径搬到 pandas / 回测框架**

按附录 A 的公式逐条实现，重点核对：`atr` 用 RMA、`highest/lowest` 含当前 bar、`tr` 的跳空项、`drawdown` 的窗口基准。**先拿一段 TradingView 的读数对齐，再回测**，否则口径差异会让结果完全对不上。

---

## 使用建议

- **先看单位再看线**：6 个脚本从「比值≈1」到「百分比」到「倍数」都有，坐标轴量级差几个数量级。看到一条 1.2 的线，先确认它是 DI、GAPO、MM 还是 VR。
- **不要跨品种比 GAPO / MM / DI 的绝对值**：GAPO 依赖价格水平，MM 只对 BTC 有历史分档，DI 理论上是无量纲比值，但不同品种的振幅扩张节奏差别很大，跨品种比较意义有限。能放心跨品种比的只有 NATR（百分比）和 UI（百分比，但语义是回撤）。
- **`mayer_multiple.pine` 只在 BTC 日线图上用**；在其他图表上它是「BTC 参考线」，不是当前标的的指标。
- **`ulcer_index.pine` 是本目录最值得深读的一个**：它把「波动」换成了「下行痛苦」，并且自带信号线、色带、交叉点，已经接近一个完整指标；但记住它的绿色 = 压力上升。
- **想监控「当前波动率水平」，用 NATR；想监控「今天是否异常」，用 VR**。两者不可互相替代。
- **所有阈值（1.5 / 2.4 / 0.5 / 1.5%）都是原作者的经验值**，不是从数据里推出来的最优参数。换品种、换周期后应当重新看分位数，而不是照抄。
- **这些脚本不是回测组件**：没有 `strategy()`、没有 `alertcondition`，任何「信号」都需要自己补（附录 D）。
