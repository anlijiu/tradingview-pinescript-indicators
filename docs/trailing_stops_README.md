# Trailing Stops — 移动止损 / 趋势反转类脚本

本目录收录 5 个脚本，全部是**趋势跟随型的移动止损（trailing stop）指标**，全部 `//@version=6`、全部 `overlay = true`（直接叠加在价格图上）。

它们的共同骨架只有一句话：

> **用某种尺度（ATR 倍数 / 百分比 / 抛物线加速因子）从价格极值处"退开一段距离"画出一条止损线，并让这条线只朝对持仓有利的方向移动（棘轮）；价格反向击穿这条线时翻转方向状态，同时打出多/空信号。**

因此每个脚本产出的东西也是同一套：**一条阶梯状止损线 + 一个二元方向状态 + 反转当根的标签/圆点 + 填充区 + `alertcondition` 告警**。
它们都是 `indicator`，**不含下单逻辑，不能回测**（详见文末「执行完会得到什么」）。

按"退开一段距离"所用的尺度，5 个脚本可以分为三族：

| 家族 | 脚本 | 止损线距价格的量度 |
| --- | --- | --- |
| **波动率（ATR）族** | `chandelier_exit.pine`、`supertrend.pine`、`halftrend.pine` | 若干倍 ATR，随波动率自适应 |
| **百分比族** | `nrtr_nick_rypock_trailing_reverse.pine` | 极值收盘价的固定百分比 |
| **加速因子族** | `parabolic_sar.pine` | 抛物线加速（AF 逐级递增，上限封顶） |

---

## 文件清单

| 文件 | 图表上显示的名字 | 简称 | 核心参数（默认） | 止损线公式（默认参数下） | 信号 | 告警 |
| --- | --- | --- | --- | --- | --- | --- |
| `chandelier_exit.pine` | Chandelier Exit | CE | ATR Period = 22，ATR Multiplier = 3.0，Use Close Price = true | `highest(close,22) − 3×ATR(22)` | Buy / Sell | 3 |
| `supertrend.pine` | SuperTrend | ST | ATR Period = 22，ATR Multiplier = 3，Source = hl2，Take Wicks = true | `hl2 − 3×ATR(22)` | Buy / Sell | 3 |
| `halftrend.pine` | HalfTrend | HT | Amplitude = 2，Channel Deviation = 2 | `up = max(maxLowPrice, up[1])`（极值跟踪线），通道 = `±2×ATR(100)/2` | Buy / Sell | 2 |
| `nrtr_nick_rypock_trailing_reverse.pine` | Nick Rypock Trailing Reverse | NRTR | Coefficient of Correction = 2（%） | `hp × (1 − 2%)`，`hp` = 最高收盘价 | Buy / Sell | 3 |
| `parabolic_sar.pine` | Parabolic SAR | PSAR | Start = 0.02，Increment = 0.02，Maximum = 0.2 | `SAR_t = SAR_{t−1} + AF × (EP − SAR_{t−1})`（内置 `ta.sar`） | Buy / Sell | 3 |

> 版权声明均为 `Alex Orekhov (everget)`，GPL-3.0。数学来源：CE 出自 Chuck LeBeau（吊灯止损）；PSAR 出自 Welles Wilder（1978《New Concepts in Technical Trading Systems》）；NRTR 与 HalfTrend 是从 MetaTrader / MQL 社区流传的经典写法移植；SuperTrend 通常归功于 Olivier Seban。

---

## 共同骨架：5 个脚本其实是同一个模板

读懂其中任何一个，其余四个只是在三处做替换：**止损线怎么算、方向状态怎么翻、可视元素怎么配色**。

### 1) 统一的参数分组

```pine
const string calcGroup = 'Calculation'     // 计算参数
const string visualGroup = 'Visuals'       // 显隐开关
const string alertGroup = 'Alerts'         // 仅 chandelier_exit 有
```

`Calculation` 里的参数决定信号密度，`Visuals` 里的开关只影响画不画，不影响计算。

### 2) 统一的三段式计算

```pine
// ① 算止损线（含棘轮）
longStop  = <极值或基准> - <距离>
shortStop = <极值或基准> + <距离>

// ② 用"上一根"的止损线判断方向翻转
var int dir = 1
dir := <突破条件> ? 1 : <跌破条件> ? -1 : dir

// ③ 取"方向变化"的那一根出信号
buySignal  = dir == 1  and dir[1] == -1
sellSignal = dir == -1 and dir[1] == 1
```

**为什么翻转判断用 `[1]`（上一根的值）而不是当根的值**：止损线是"事后用当根 K 线算出来"的，若用当根线判断当根收盘，会自相矛盾（线本身就是由这根 K 线的高低价推出来的）。用上一根已知的线 + 当根价格做穿越判断，才是可执行、无前视的定义。

### 3) 统一的绘图语言

| 元素 | 实现 | 说明 |
| --- | --- | --- |
| 止损线 | `plot(..., style = plot.style_linebr, linewidth = 2)` | `linebr` = 值为 `na` 时断线，所以多头段只画绿线、空头段只画红线 |
| 启动点 | `plotshape(..., location = location.absolute, style = shape.circle, size = size.tiny)` | 反转当根在止损线位置打一个小圆点 |
| 买卖标签 | `plotshape(..., style = shape.labelup/labeldown, text = 'Buy'/'Sell')` | 白字，颜色随方向（绿/红/蓝） |
| 状态填充 | `fill(midPricePlot, stopPlot, color = color.new(color.green, 85))` | 单边状态时给"价格 ↔ 止损线"之间涂一层 85% 透明的底色 |
| 隐藏锚点 | `plot(ohlc4, display = display.none, editable = false)` | `ohlc4` 只作填充的另一侧边界，不在图上显示、也不出现在 Data Window |

> 配色统一为多头 `color.green` / 空头 `color.red`，填充透明度统一 85；只有 HalfTrend（蓝/红）与 PSAR（`#3388bb` / `#fdcc02`）用自己的一套颜色。

### 4) 统一的告警语义

| 告警 | 触发条件 | 有它的脚本 |
| --- | --- | --- |
| `... Direction Change` / `Direction Change` | `dir != dir[1]`（任意方向翻转） | CE、NRTR、PSAR |
| `... Buy` / `Long` | `buySignal`（空翻多当根） | 全部 5 个 |
| `... Sell` / `Short` | `sellSignal`（多翻空当根） | 全部 5 个 |

告警 message 都带 `{{exchange}}:{{ticker}}` 占位符，方便在 TradingView 的 webhook 里区分品种。HalfTrend 只有 Buy/Sell 两条，没有方向变化告警。

---

## 1. `chandelier_exit.pine` — Chandelier Exit（吊灯止损）

**做什么**：把止损线挂在"最近 N 根 K 线极值下方 k 倍 ATR 处"，形态像从天花板吊下来的吊灯——**多头时吊在最高价下方、空头时吊在最低价上方**，行情走得越远线跟得越高，回撤到线就翻空（反之翻多）。

**原理**：

```pine
atr = mult * ta.atr(length)                                        // 距离：k × ATR(22)

longStop  = (useClose ? ta.highest(close, length) : ta.highest(length)) - atr
longStopPrev = nz(longStop[1], longStop)
longStop  := close[1] > longStopPrev ? math.max(longStop, longStopPrev) : longStop

shortStop = (useClose ? ta.lowest(close, length) : ta.lowest(length)) + atr
shortStopPrev = nz(shortStop[1], shortStop)
shortStop := close[1] < shortStopPrev ? math.min(shortStop, shortStopPrev) : shortStop

var int dir = 1
dir := close > shortStopPrev ? 1 : close < longStopPrev ? -1 : dir
```

四点关键：

1. **极值基准**：`Use Close Price for Extremums = true`（默认）时用 `highest(close, 22)` / `lowest(close, 22)`；关掉则用 `highest(high, 22)` / `lowest(low, 22)`（无 source 参数的 `ta.highest(length)` 默认取 `high`）。用收盘价更平滑、止损更紧；用最高/最低价更保守、更抗插针。
2. **棘轮（只升不降）**：`close[1] > longStopPrev ? math.max(longStop, longStopPrev) : longStop`——上一根收盘仍在旧止损线上方时，只允许线**上移**；一旦上一根收盘已跌破旧线（说明方向已经翻过去了），就放弃旧线、用当根重新锚定的 `highest − ATR` 回落下来，为下一次转多重新定位。这就是"止损线是一级级台阶"的来源。
3. **双边状态机**：`dir` 由两个不同条件翻转——`close > shortStopPrev` 转多、`close < longStopPrev` 转空；两条线都在每根 bar 上同时维护，但只按 `dir` 显示其中一条。由于两个阈值不同（上方用空头止损线、下方用多头止损线），中间存在一个"维持原方向"的滞回区，天然抑制来回打脸。
4. **信号**：`buySignal = dir == 1 and dir[1] == -1`，即状态翻转的那一根。

**参数**

| 参数 | 默认 | 说明 |
| --- | --- | --- |
| `ATR Period` | 22 | 同时决定极值回看窗口和 ATR 周期。LeBeau 原始推荐值 |
| `ATR Multiplier` | 3.0 | 越大越宽松（少信号、更能扛回撤）；越小越敏感（信号多、容易被噪声扫） |
| `Use Close Price for Extremums` | true | true = 用收盘极值；false = 用 high/low 极值 |
| `Show Buy/Sell Labels` | true | 是否显示 Buy/Sell 文字标签（圆点照常显示） |
| `Highlight State` | true | 是否给"价格 ↔ 止损线"之间填充色带 |
| `Await Bar Confirmation` | true | true = 只在 K 线收盘确认后触发告警（`barstate.isconfirmed`）；false = 实时 tick 就触发 |

**输出（执行完你会看到/拿到什么）**

- **图上的线**：多头段一条绿色阶梯线（`Long Stop`），空头段一条红色阶梯线（`Short Stop`），`style_linebr` 保证切换时断开而不连线；
- **翻转标记**：翻转当根在止损线位置一个同色小圆点 + `Buy` / `Sell` 标签（朝上/朝下）；
- **状态色带**：多头段价格与止损线之间淡绿色，空头段淡红色（透明度 85）；
- **状态行 / Data Window**：两列 `Long Stop`、`Short Stop`，未处于该方向的段显示 `n/a`（`n/a` 就是这个脚本输出的"当前无该方向"的表示）；
- **告警**：`CE Direction Change`、`CE Buy`、`CE Sell` 三条 `alertcondition`，可直接在「创建警报 → 条件」里选到。

**注意**

- 这是 5 个脚本里**唯一提供 `Await Bar Confirmation` 开关**的：默认只在收盘 K 线上触发告警，因此实时行情里不会因为影线瞬间穿越而反复告警；其余 4 个脚本在盘中未收盘的 bar 上就可能触发（见文末「重绘与告警」）。
- `dir` 初值为 1，即历史上第一根 bar 默认按多头处理，所以图表最左端可能先显示一段绿线而非"空白"。
- ATR 周期 22 与乘数 3 是 LeBeau 对**股票日线**的经验值；换到加密货币或更小周期，通常要按"愿意扛多大回撤"重新调乘数，而不是调周期。

---

## 2. `supertrend.pine` — SuperTrend

**做什么**：与 Chandelier Exit 思路相同（ATR 距离 + 棘轮 + 双边状态机），但**基准不是"最近极值"，而是当根价格的平滑中值 `hl2`**——止损线 = `hl2 ∓ k×ATR`，再叠一层"前一根价格有没有穿过旧线"的棘轮。因此它的止损线离价格的距离更稳定（始终约 k 倍 ATR），不像 CE 会因为极值窗口内出现远高点而突然拉开。

**原理**：

```pine
atr = mult * ta.atr(length)
highPrice = wicks ? high : close          // 参与判断的"高点"
lowPrice  = wicks ? low  : close          // 参与判断的"低点"
doji4price = open == close and open == low and open == high   // 四价合一（一字板）

longStop = src - atr                      // src 默认 hl2
longStopPrev = nz(longStop[1], longStop)
if longStop > 0
    if doji4price
        longStop := longStopPrev           // 一字板：价格无效，保持原线
    else
        longStop := lowPrice[1] > longStopPrev ? math.max(longStop, longStopPrev) : longStop
else
    longStop := longStopPrev               // 算出负值（价格 < k×ATR）时保持原线

// shortStop = src + atr，对称处理：highPrice[1] < shortStopPrev ? min(...) : shortStop

var int dir = 1
dir := dir == -1 and highPrice > shortStopPrev ? 1 : dir == 1 and lowPrice < longStopPrev ? -1 : dir
```

与 CE 的**三处实质差别**：

1. **基准**：CE 用 `highest/lowest`（极值），ST 用 `src = hl2`（当根价格的位置），因此 CE 是"极值 + ATR"，ST 是"价格 + ATR"；
2. **穿透判定用 `high/low` 而不是 `close`**：`dir == -1 and highPrice > shortStopPrev` 意味着**影线刺穿就翻转**（`wicks = true` 时）。把 `Take Wicks into Account` 关掉，`highPrice/lowPrice` 退化为 `close`，就变成"收盘确认才翻转"，抗毛刺能力更强；
3. **两道保护**：一字板（`open==close==low==high`，加密小币、涨跌停板常见）时保持原止损线不动；止损线算出负数时也保持原值——否则 `longStop` 穿越 0 会让后续所有 `max` 比较失效。

此外翻转条件写成 `dir == -1 and 跌破条件 ? 1 : dir == 1 and 突破条件 ? -1 : dir`，要求"上一根是反向状态"才允许翻转，是一台更严格的二态机。

**参数**

| 参数 | 默认 | 说明 |
| --- | --- | --- |
| `ATR Period` | 22 | ATR 周期 |
| `ATR Multiplier` | 3 | 距离倍数 |
| `Source` | `hl2` | 止损线的基准价。改成 `close` 更敏感，改成 `hl2`/`hlc3` 更平滑 |
| `Take Wicks into Account` | true | true = 用 high/low 判断棘轮与翻转（影线有效）；false = 只用 close（收盘有效） |
| `Show Buy/Sell Labels` | true | 显隐 Buy/Sell 文字标签 |
| `Highlight State` | true | 显隐状态色带 |

**输出**

- 图上的线：`Long Stop`（绿，多头段可见）/ `Short Stop`（红，空头段可见）两条阶梯线，切换处断开；
- 翻转当根：小圆点 + `Buy` / `Sell` 标签；
- 状态色带（85% 透明）；
- 状态行 / Data Window：`Long Stop`、`Short Stop` 两列（非当前方向为 `n/a`）；
- 告警：`SuperTrend Direction Change`、`SuperTrend Buy`、`SuperTrend Sell`。

**注意**

- `Source` 与 `Take Wicks into Account` 两个参数会显著改变信号数量：默认组合（`hl2` + wicks）最灵敏；想要"少而稳"就改成 `close` + 关闭 wicks。
- 因为没有 `Await Bar Confirmation` 之类的开关，实时盘中一旦影线刺穿就会触发告警，随后价格收回时告警已经发出（无法撤回）。做自动交易时必须自行加收盘确认。
- SuperTrend 的止损线在趋势中会不断被 `max/min` 抬高/压低，因此它是**单边持仓视角**的指标：它只回答"现在应该在哪一边、在哪止损"，不回答仓位大小。

---

## 3. `nrtr_nick_rypock_trailing_reverse.pine` — Nick Rypock Trailing Reverse（NRTR）

**做什么**：不依赖 ATR，用**固定百分比**从"本轮极值收盘价"退开一条反转线：多头时 `最高收盘 × (1 − p%)`，空头时 `最低收盘 × (1 + p%)`。价格反向击穿它就翻转。适合"波动率相对稳定、想用统一百分比控制回撤"的场景。

**原理**：

```pine
percentage = input.float(2, ...) / 100      // 2% → 0.02

var int   trend = 0                          // 0 = 初始/多头侧，-1 = 空头，1 = 多头
var float hp = close                         // 本轮最高收盘（high price）
var float lp = close                         // 本轮最低收盘（low price）
nrtr = close

if trend >= 0                                 // 多头侧：跟踪最高收盘
    if close > hp
        hp := close
    nrtr := hp * (1 - percentage)             // 多头止损线
    if close <= nrtr                          // 收盘击穿 → 翻空
        trend := -1
        lp := close
        nrtr := lp * (1 + percentage)
else                                          // 空头侧：跟踪最低收盘
    if close < lp
        lp := close
    nrtr := lp * (1 + percentage)             // 空头止损线
    if close > nrtr                           // 收盘突破 → 翻多
        trend := 1
        hp := close
        nrtr := hp * (1 - percentage)
```

三个要点：

1. **棘轮是天然内建的**：`hp` 只在 `close > hp` 时更新，所以多头止损线只会跟着创新高而抬升，回调时不动；`lp` 同理只会下压。不需要额外写 `math.max/min`。
2. **翻转当根就重新锚定**：一旦击穿，立刻把 `hp`/`lp` 重置为当根收盘，并直接用反向公式算出新线——所以翻转那根 bar 的止损线在价格另一侧，`nrtr` 在图上表现为"跳一下"。
3. **只有收盘价参与**：与 ST 不同，NRTR 不看 high/low，影线刺穿不算，**只看收盘**。这是它抗插针的原因，也是它反应偏慢的原因。

**参数**

| 参数 | 默认 | 说明 |
| --- | --- | --- |
| `Coefficient of Correction, %` | 2 | 距离百分比。全脚本唯一的计算参数；同一数值在不同品种上含义差别巨大（2% 对股指日线合适，对山寨币分钟线会被反复扫） |
| `Show Buy/Sell Labels` | true | 显隐 Buy/Sell 文字标签 |
| `Highlight State` | true | 显隐状态色带 |

**输出**

- 图上的线：`Long Stop`（绿）/ `Short Stop`（红）两条阶梯线；
- 翻转当根：小圆点 + `Buy` / `Sell` 标签；
- 状态色带（85% 透明）；
- 状态行 / Data Window：`Long Stop`、`Short Stop` 两列；
- 告警：`NRTR Direction Change`、`NRTR Buy`、`NRTR Sell`。

**注意（含一个已知的冷启动问题）**

- **初始状态自相矛盾**：`trend` 初值为 `0`，而计算分支是 `if trend >= 0`（走多头逻辑、跟踪最高收盘），绘图却是 `trend == 1 ? nrtr : na`（把 `0` 当空头）。后果是：**如果行情从图表最左端就一路上涨、从未触发 `close <= nrtr`，`trend` 会一直停在 `0` 而不变成 `1`**，于是这一段既不出 `Buy` 标签，也不会画绿线，反而按空头（红线 + 红色填充）渲染。这是原脚本的移植遗留，要修就两行：把 `var int trend = 0` 改成 `1`，或把所有 `trend == 1` 改成 `trend >= 0`。
- 百分比是**固定**的：换品种、换周期后必须重新标定，否则等价于换了一个指标。想让它自适应，把 `percentage` 换成 `k * ta.atr(14) / close` 即可（那就是"波动率归一的 NRTR"）。
- 只用收盘价 → 在盘中不会因影线而翻转，但也因此比 ST 慢半拍。

---

## 4. `halftrend.pine` — HalfTrend

**做什么**：一条比 SuperTrend"少一半"的趋势线——只在趋势段的一半位置跟随（用最近 `Amplitude` 根的极值经均线过滤后确认反转），并在趋势线上下再挂一对 ATR 通道（`ATR High` / `ATR Low`），用于判断"当前是趋势中还是震荡中"。

**原理**：

```pine
atr2 = ta.atr(100) / 2                                        // 通道半宽单位（ATR 周期固定 100）
dev  = channelDeviation * atr2                                // 默认 2 → 上下各 1×ATR(100)

highPrice = high[math.abs(ta.highestbars(amplitude))]         // 最近 amplitude 根内的最高价
lowPrice  = low[math.abs(ta.lowestbars(amplitude))]           // 最近 amplitude 根内的最低价
highma    = ta.sma(high, amplitude)
lowma     = ta.sma(low,  amplitude)

// ① 反转判定：两套对称的"支撑被跌破 / 压力被突破"
if nextTrend == 1                                             // 上涨段：不断抬升支撑 maxLowPrice
    maxLowPrice := math.max(lowPrice, maxLowPrice)
    if highma < maxLowPrice and close < nz(low[1], low)       // 均高点跌破支撑 + 收盘破前低
        trend := 1                                            // → 转空
        nextTrend := 0
        minHighPrice := highPrice
else                                                          // 下跌段：不断下压压力 minHighPrice
    minHighPrice := math.min(highPrice, minHighPrice)
    if lowma > minHighPrice and close > nz(high[1], high)     // 均低点突破压力 + 收盘破前高
        trend := 0                                            // → 转多
        nextTrend := 1
        maxLowPrice := lowPrice

// ② 趋势线：棘轮式跟随，翻转当根承接上一段线以保持连续
if trend == 0
    if not na(trend[1]) and trend[1] != 0                     // 刚由空转多
        up := na(down[1]) ? down : down[1]                    // 承接上一段空头线位置
        arrowUp := up - atr2                                  // 箭头位置
    else
        up := na(up[1]) ? maxLowPrice : math.max(maxLowPrice, up[1])
    atrHigh := up + dev
    atrLow  := up - dev
else
    ...（对称：down 取 math.min(minHighPrice, down[1])，atrHigh/Low = down ± dev）

ht = trend == 0 ? up : down                                   // 最终显示的趋势线
```

要点：

1. **双变量状态机**：`trend`（0 = 上涨段，1 = 下跌段）决定"画哪条线、什么颜色"；`nextTrend`（1 = 正在管理上涨段，0 = 正在管理下跌段）决定"这一根该更新支撑还是压力"。两者在正常运转时互为镜像，保留两个变量是 MQL 移植的写法，可以合并。
2. **反转需要两个条件同时成立**：`highma < maxLowPrice`（短均高点已经失守支撑）**且** `close < low[1]`（收盘创出前一根新低）。比 ST 的"价格刺穿止损线"更严格，因此信号更少、更不容易在震荡里被洗。
3. **`Amplitude` 是双层含义**：既决定极值回看窗口（`highestbars/lowestbars`），又决定均线平滑长度（`sma`）。默认 2 意味着"用 2 根内的极值，再取 2 根均线"，已经很灵敏。
4. **通道固定用 `ta.atr(100)`**：ATR 周期没有暴露成参数（写死在代码里），`atr2` 是它的一半，所以 `Channel Deviation = 2` 等于"上下各 1×ATR(100)"，`= 3` 等于 1.5×ATR(100)。
5. **翻转当根的连续性处理**：`up := down[1]` 让新趋势线直接取上一根旧趋势线的值，使 `ht` 在翻转处平滑衔接、不出现跳空——否则新线会突然出现在极值处的远处，造成视觉上的假突破。

**参数**

| 参数 | 默认 | 说明 |
| --- | --- | --- |
| `Amplitude` | 2 | 极值窗口 + 均线长度。调大 → 信号更少、更滞后 |
| `Channel Deviation` | 2 | 通道宽度倍数（相对 `ATR(100)/2`）。只影响 `ATR High/Low` 的显示宽度，**不影响信号** |
| `Show Arrows` | true | 是否显示三角箭头（↑ 在 `ATR Low`，↓ 在 `ATR High`） |
| `Show Channels` | true | 是否显示 `ATR High` / `ATR Low` 圆点通道 |
| `Show Buy/Sell Labels` | true | 是否显示 Buy/Sell 文字标签 |

**输出**

- **趋势线**：`HalfTrend` 一条实线（`linewidth = 2`），上涨段蓝色 `color.blue`、下跌段红色 `color.red`——注意这是 5 个脚本里唯一用蓝色表示多头的；
- **ATR 通道**：`ATR High`（红点）/ `ATR Low`（蓝点）两条 `style_circles` 曲线，以及趋势线与它们之间的两条填充带（红/蓝，85% 透明）；
- **翻转标记**：`Arrow Up`（蓝三角）在 `ATR Low`、`Arrow Down`（红三角）在 `ATR High`，以及 `Buy` / `Sell` 标签；
- **状态行 / Data Window**：`HalfTrend`、`ATR High`、`ATR Low` 三列；
- **告警**：只有 `HalfTrend Buy`、`HalfTrend Sell` 两条（**没有** Direction Change 告警）。

**注意**

- **信号语义与其它脚本不同**：`trend == 0` 表示**上涨**，`trend == 1` 表示**下跌**（其余脚本多为 `+1/-1`）。跨脚本拼逻辑时容易读反。
- **冷启动**：`trend`（=0）与 `nextTrend`（=0）初值不一致，所以第一段行情会走"管理压力位"的分支但画多头线，`maxLowPrice` 也还没被抬升过；表现为图表最左端可能是一条位于低位的横线，直到第一次 `lowma > minHighPrice and close > high[1]` 成立才进入正常跟踪。实盘影响可以忽略（只影响最早的一段历史），但看图时要知道它。
- `Channel Deviation` 只改画图宽度、不改信号——想改信号要动 `Amplitude`。
- 代码中若干形如 `minHighPrice`、`arrowUp`、`atrLow` 的**裸表达式行**（写在 `if` 分支末尾、没有任何赋值）是从 MQL 移植时留下的空语句，不影响结果，可以直接删掉。

---

## 5. `parabolic_sar.pine` — Parabolic SAR

**做什么**：Wilder 的抛物线止损转向系统。止损点沿抛物线加速逼近价格：趋势持续越久，加速因子 AF 越大，止损线收得越快；一旦被价格穿透就翻转，SAR 跳到另一侧并重置加速因子。适合判断"趋势是否已经衰竭"。

**原理**：

```pine
psar = ta.sar(start, increment, maximum)      // 内置实现：SAR、EP、AF 三个状态由函数内部维护
dir  = psar < close ? 1 : -1                  // 方向由"Sar 在价格下方还是上方"决定
```

内置 `ta.sar` 的迭代式（Pine 不暴露内部状态，这里是其数学定义）：

```
SAR_t = SAR_{t-1} + AF × (EP_{t-1} − SAR_{t-1})
AF    = min(start + n × increment, maximum)    // n = 创新极值的次数
EP    = 本轮趋势内的最高价（多头）/ 最低价（空头）
翻转条件：价格穿越 SAR → SAR 置为 EP，AF 重置为 start，方向反转
```

要点：

1. **加速因子**：每创一次新极值 `AF` 增加 `increment`（默认 0.02），上限 `maximum`（0.2）。所以 SAR 与价格的距离是"先慢后快"地收缩——趋势拖得越久，止损越紧。
2. **本脚本只用 `psar < close` 反推方向**，不直接读 `ta.sar` 的内部状态。正常情况下两者一致（翻转后 SAR 必然在价格另一侧），但边界上略有差别：`psar == close` 时方向被记为 `-1`（空头）；历史开头若 `ta.sar` 尚未产生有效值（`na`），`na < close` 结果为 `na`（三元条件按 false 处理）同样会落到 `-1`。
3. **翻转当根重新计数**：SAR 跳变使 `dir` 翻转，`buySignal/sellSignal` 只在翻转那根成立，之后 `dir[1]` 与 `dir` 相同，不再重复。

**参数**

| 参数 | 默认 | 说明 |
| --- | --- | --- |
| `Start` | 0.02 | 初始加速因子（翻转后重置到这个值） |
| `Increment` | 0.02 | 每次创新极值的 AF 增量 |
| `Maximum` | 0.2 | AF 上限。越小 → 止损扩张慢、趋势中持仓更久；越大 → 追赶更快、震荡中更易被扫 |
| `Point Width` | 2 | 圆点线宽（至少 1） |
| `Show Buy/Sell Labels` | true | 显隐 Buy/Sell 标签 |
| `Highlight Start Points` | true | 显隐翻转当根的起始圆点 |
| `Highlight State` | true | 显隐状态色带 |

**输出**

- **图上的点**：`PSAR` 一条圆点曲线（`style_circles`），多头 `#3388bb`（蓝）、空头 `#fdcc02`（黄），线宽由 `Point Width` 控制；
- **翻转标记**：`Long Start` / `Short Start` 圆点（绿/红）与 `Buy` / `Sell` 标签；
- **状态色带**：价格与 SAR 点之间的淡绿/淡红填充（`highlightState` 打开时）；
- **状态行 / Data Window**：`PSAR` 一列（这是 5 个脚本里唯一只输出一条序列的）；
- **告警**：`Direction Change`、`Long`、`Short` 三条。

**注意**

- 只有它把"起点标记"和"状态色带"拆成两个独立开关，其它脚本只有一个 `Highlight State`。
- PSAR 在**震荡市里以"连续翻向"著称**：加速因子升到 0.2 后被反复穿越，会出现密集的 Buy/Sell 标签。日线以下周期建议调大 `Start`/`Increment` 或改用 ST/CE。
- 与其它 4 个脚本不同，PSAR 的点**不构成连续的止损台阶**（翻转时跳变到价格另一侧），因此不适合当作"移动止损单"的挂单位置，更适合当趋势状态的判据。

---

## 横向对比：五个脚本的止损线到底差在哪

| 维度 | Chandelier Exit | SuperTrend | HalfTrend | NRTR | Parabolic SAR |
| --- | --- | --- | --- | --- | --- |
| 距离量度 | `k × ATR(n)` | `k × ATR(n)` | 极值跟踪（无显式距离）+ `ATR(100)` 通道 | 固定百分比 | 抛物线加速 AF |
| 基准 | 最近 n 根极值（close 或 high/low） | `hl2`（可改 source） | 最近 `Amplitude` 根极值 + 均线过滤 | 本轮极值收盘 | SAR 自身迭代 |
| 穿透判定 | close | high/low（可切 close） | `close` 破前根 + 均线失守 | close | close |
| 棘轮实现 | 显式 `math.max/min` + `close[1]` 条件 | 显式 `math.max/min` + `low[1]` 条件 + doji/负值保护 | `maxLowPrice` / `minHighPrice` 单调更新 | 极值变量天然单调 | AF 驱动，翻转即重置 |
| 状态变量 | `dir`（±1） | `dir`（±1） | `trend`（0/1）+ `nextTrend` | `trend`（−1/0/1） | `dir`（±1，由 `psar<close` 推出） |
| 可调强度 | ATR Period、Multiplier、极值口径 | ATR Period、Multiplier、Source、Wicks | Amplitude（Deviation 只影响画图） | 百分比 | Start/Increment/Maximum |
| 信号数量（同参数下相对） | 中 | 中～多（wicks 开启时更多） | 少 | 少（只看收盘、固定百分比） | 多（震荡市里最多） |
| 视觉形态 | 绿/红阶梯线 + 圆点 + 标签 + 色带 | 同 CE | 蓝/红实线 + 双色 ATR 通道 + 三角箭头 | 同 CE | 蓝/黄圆点 + 色带 |
| 告警条数 | 3 | 3 | **2** | 3 | 3 |

**一句话记住区别**：

- 想要**"止损跟着最近的高点走"** → CE；
- 想要**"止损始终离价格 k 倍 ATR"** → ST；
- 想要**"更少、更晚、更确认的信号"** → HalfTrend（双条件反转）；
- 想要**"用统一百分比控制最大回撤"** → NRTR；
- 想要**"判断趋势是否衰竭 / 找转折点"** → PSAR（而非挂单止损）。

## 选型与组合建议

| 需求 | 推荐 | 理由 |
| --- | --- | --- |
| 单一品种、跨周期通用的移动止损 | `supertrend.pine` | 距离恒为 k×ATR，换周期后行为最可预期；参数只有两个 |
| 想要"止损贴着结构性高点/低点" | `chandelier_exit.pine` | 唯一以极值（可选收盘极值）为基准的 ATR 型 |
| 震荡市里不想被反复扫 | `halftrend.pine` 或 `nrtr_nick_rypock_trailing_reverse.pine` | 前者双条件确认，后者只看收盘 |
| 按"最大回撤预算"设止损 | `nrtr_nick_rypock_trailing_reverse.pine` | 百分比直接等于可承受回撤 |
| 判断趋势阶段（趋势 / 衰竭） | `parabolic_sar.pine` | SAR 与价格的距离反映趋势"年龄" |
| 多指标交叉验证 | ST（趋势）+ PSAR（衰竭）+ HalfTrend（确认） | 三者算法不同源，同时翻向时可信度更高；但要注意全部滞后，且都在同一根收盘价上取信号，不能当成独立证据 |

**组合时的两个实操提醒**

1. 5 个脚本都 `overlay = true`，同时加载会叠成 3 条以上的止损线和多层 85% 透明填充，视觉上很难分辨——建议按 On/Off 逐个开关，或只保留 1 条止损线 + 1 个趋势判据。
2. 它们的信号全部在**同一根 bar 收盘价**上产生（除 ST 的 wicks 模式用的是 high/low），所以"三个指标同时翻向"并不等于三次独立确认，本质上还是同一个价格动作。真正想增加独立性，应该换**不同周期的数据**（如 `request.security` 取日线方向 + 小时线入场）。

---

## 执行完会得到什么（5 个脚本的产出清单）

### 会得到

| 产出 | 具体内容 |
| --- | --- |
| **图表上的止损线/点** | CE/ST/NRTR 各两条阶梯线（多头绿、空头红，非当前方向不画）；HalfTrend 一条趋势线 + 上下一对 ATR 圆点通道；PSAR 一条圆点曲线（蓝/黄） |
| **多空信号** | 每个脚本在方向翻转的那一根打圆点 + `Buy`/`Sell` 标签（HalfTrend 额外有三角箭头） |
| **状态色带** | 价格与止损线之间 85% 透明的绿/红色块，用来目视当前所处方向 |
| **Data Window / 状态行数值** | CE/ST/NRTR：`Long Stop`、`Short Stop`；HalfTrend：`HalfTrend`、`ATR High`、`ATR Low`；PSAR：`PSAR`。非当前方向的列为 `n/a` |
| **可挂的告警** | 每个脚本 2～3 条 `alertcondition`（方向变化 / Buy / Sell），可用于 TradingView 弹窗、邮件、webhook |
| **一条可复用的止损参考价** | 最重要的一项：图上的止损线数值 = "如果现在要挂移动止损单，挂在哪里" |

### 不会得到

- **回测报告 / 收益曲线 / 胜率**：5 个都是 `indicator`，没有 `strategy()` 入口、没有 `strategy.entry/exit`，TradingView 的 Strategy Tester 不会给出任何统计。要回测必须自己把计算段复制进 `strategy` 脚本（见下）。
- **仓位、加仓、止盈逻辑**：全部只输出"方向 + 止损位"，没有手数、没有金字塔、没有目标价。
- **方向数值的 Data Window 列**：`dir` / `trend` 没有 `plot` 出来，只体现为图形显隐。想在 Data Window 看方向，加一行 `plot(dir, 'Direction', display = display.data_window)` 即可。
- **多品种/多周期的自动适配**：参数是常量，换品种要手动调（ATR 型相对好一些，NRTR 必须重标定）。
- **未来函数安全性**：脚本本身不含 `lookahead`，但若被 `request.security` 以 `lookahead_on` 调用，会引入前视偏差。

### 想回测怎么办（最小骨架，以 Chandelier Exit 为例）

```pine
//@version=6
strategy('CE Strategy Demo', overlay = true, process_orders_on_close = true,
     default_qty_type = strategy.percent_of_equity, default_qty_value = 100)

// ↓↓ 把 chandelier_exit.pine 里"计算段"原样复制过来（atr / longStop / shortStop / dir）
// ↑↑ 然后接上下单逻辑：

if buySignal
    strategy.entry('Long', strategy.long)
if sellSignal
    strategy.entry('Short', strategy.short)   // 只做多则改成 strategy.close('Long')
```

两点必须注意：

- `process_orders_on_close = true` 让订单在当前 bar 收盘价成交（与指标信号同价），否则默认在**下一根 bar 开盘**成交，结果会与看图直觉不一致；
- 指标脚本里 `dir` 初值等状态用 `var` 保存，复制到 `strategy` 后行为一致，但切换品种/周期会整段重算，历史首段的多空归属也随之改变，不要用它做绝对收益比较。

---

## 重绘与告警：最容易踩的坑

1. **盘中重绘（repaint）**：除 CE 之外，其余 4 个脚本的 `alertcondition` 都没有做收盘确认。实时行情里，当前未收盘 bar 的 high/low/close 还在变，止损线跟着变，可能"先触发 Sell 告警、收盘时又不触发了"。硬性要求收盘确认的话：
   - CE：保持 `Await Bar Confirmation = true`；
   - 其它 4 个：给 `alertcondition` 加上 `and barstate.isconfirmed`，或用 `if barstate.isconfirmed` 包住 `alert()`。
2. **告警只在"翻转那根"成立**：`buySignal = dir == 1 and dir[1] == -1` 是一个**事件**而不是**状态**。想持续接收"当前处于多头"的信息，应该用 `alertcondition(dir == 1, ...)`（状态型），或者用 `alert()` 配合 `freq = alert.freq_once_per_bar_close`。
3. **首根 bar 的默认方向**：CE 的 `dir` 初值 `1`、ST 的 `dir` 初值 `1`、NRTR 的 `trend` 初值 `0`、HalfTrend 的 `trend` 初值 `0`、PSAR 的 `dir` 全靠 `ta.sar` 是否有值——**历史长度不够时，开头几段的多空归属是任意的**。比较跨品种结果时，请从同一段足够长的历史之后开始看。
4. **告警数量上限**：TradingView 免费/付费方案对同时运行的告警数有上限，5 个脚本合计 14 条条件一起挂很容易打满。
5. **`n/a` 不等于"没有信号"**：Data Window 里 `Long Stop = n/a` 只表示"当前不在多头段"，多头段的历史值不会保留在最新一行里。

## 参数标定的通用做法

这 5 个脚本都在同一条 "灵敏度 ↔ 抗噪性" 曲线上：

- **提高灵敏度**（信号更早、更多）：调小 ATR Period / Multiplier、减小 NRTR 百分比、调大 PSAR 的 Increment、调小 HalfTrend 的 Amplitude；
- **提高抗噪性**（信号更少、更迟）：反向调，并把穿透判定从 high/low 改成 close（ST 的 `wicks`、CE 的 `useClose`）；
- **判断是否调过头**：看图上**连续反向翻转的间隔**。如果频繁出现"Buy 之后 3～5 根就 Sell"，说明止损线比该周期的噪声幅度还窄，应放宽距离（或改用 HalfTrend 这种双条件确认的脚本）；
- **跨品种**：ATR 型（CE/ST）参数可跨品种通用；百分比型（NRTR）必须按品种波动率重新标定；PSAR 对周期最敏感，日内周期需要明显放大 `Start`/`Increment`。

## 已知瑕疵清单

| 脚本 | 问题 | 影响 | 修法 |
| --- | --- | --- | --- |
| `nrtr_nick_rypock_trailing_reverse.pine` | `trend` 初值 `0`：计算按多头、绘图按空头；若开局一路上涨则永远停在 `0` | 初始段无 Buy 标签、绿线不画，反而按空头渲染 | `var int trend = 1`，或绘图改用 `trend >= 0` |
| `halftrend.pine` | `trend` / `nextTrend` 初值不一致（均 0） | 最左一段趋势线可能是一条低位横线 | 把 `nextTrend` 初值设为 `1` |
| `halftrend.pine` | `ta.atr(100)` 的周期写死 | 无法从界面调整通道宽度的基准 | 抽成 `input.int` |
| `halftrend.pine` | 分支末尾的裸表达式语句（`minHighPrice`、`arrowUp`、`atrLow` 等） | `if` 块内的空操作，无副作用 | 直接删除 |
| `parabolic_sar.pine` | `dir = psar < close ? 1 : -1` 在 `psar` 为 `na` 或 `== close` 时判为 `-1` | 历史开头可能先出现一小段空头状态 | 加 `nz()` 或改用显式状态判断 |
| `supertrend.pine` | 只在 `longStop > 0` 时更新止损线 | 极低价品种（< k×ATR）会冻结止损线 | 属有意防护，一般无需改 |

---

## 附录：指标的历史出处

- **Chandelier Exit**：Chuck LeBeau 提出（收录于《Computerized Trading》与其 MATA 论文），本脚本默认 `ATR(22) × 3` 即其推荐值；`Use Close Price for Extremums` 是本实现额外提供的口径开关。
- **Parabolic SAR**：J. Welles Wilder Jr.，1978《New Concepts in Technical Trading Systems》，与 RSI、ATR、ADX 同书。
- **SuperTrend / HalfTrend / NRTR**：均属"趋势跟随止损"家族的民间经典——SuperTrend 常归功于 Olivier Seban；HalfTrend 与 NRTR（Nick Rypock Trailing Reverse）从 MetaTrader/MQL 社区流传而来，本目录是 Alex Orekhov（everget）的 Pine 移植版。
