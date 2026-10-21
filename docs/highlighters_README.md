# Highlighters — 时间轴高亮 / 标记类脚本

本目录收录 5 个脚本，它们**不是技术指标**：不计算任何价格衍生量（均线、波动率、振荡器都没有），只做一件事——**按日历时间或 K 线结构给图表"上色 / 贴标签 / 重绘"**。

共同点：

- 全部 `overlay=true`，直接画在价格图上；
- 只调用 `bgcolor()` / `plotshape()` / `plotcandle()` 三类绘图函数；
- 不产生买卖信号、不产生 `alertcondition`、不能当 `strategy` 回测；
- 判定条件几乎全部来自时间内置函数（`year` / `month` / `hour` / `dayofweek`）。

按作用对象可以分成两组：

| 分组 | 脚本 | 作用对象 |
| --- | --- | --- |
| 时间日历类（4 个，本质是"背景色 + 标签"模板） | `bitmex_withdrawals_cutoff_time`、`leap_years`、`quarters`、`weekdays_gaps` | 每根 bar 的**时间属性** |
| 价格图形类（1 个） | `range_candles` | 每根 bar 的**OHLC 数值**（重新画一遍蜡烛） |

## 文件清单

| 文件 | 脚本名（图表上显示） | Pine 版本 | 判定依据 | 视觉输出 | 输入参数 |
| --- | --- | --- | --- | --- | --- |
| `bitmex_withdrawals_cutoff_time.pine` | BitMEX Withdrawals Cutoff Time | 6 | `hour(time) == 13` | 整点背景色 `#f57f17` | Transparency = 70 |
| `leap_years.pine` | Leap Years | 6 | 年份是否为闰年（格里高利历规则） | 全年背景色 `#ffa726` + 年首 `Leap Year` 标签 | Transparency = 88，Show Labels = true |
| `quarters.pine` | Quarters | 6 | `month(time)` 落在哪个季度 | 季度背景色（四色）+ 季首 `Q1..Q4` 标签 | Transparency = 88，Show Labels = true |
| `range_candles.pine` | Range Candles | 6 | `close >= open` | 覆盖重绘蜡烛：涨 `#26a69a` / 跌 `#ef5350` | Bull Color、Bear Color |
| `weekdays_gaps.pine` | Weekdays Gaps | 6 | 相邻两根 bar 的 `dayofweek` 是否连续 | 缺口后首根 bar 背景色 `#ffcc80` | 无（颜色硬编码） |

> 版权声明均为 `Alex Orekhov (everget)`，GPL-3.0。`leap_years` / `quarters` 是同一套代码模板，只是判定条件和配色不同。

---

## 1. `bitmex_withdrawals_cutoff_time.pine` — 提现截止时刻高亮

**做什么**：把每天 **13 点那一小时**的 K 线背景涂成琥珀色，用来在图上标出 BitMEX 的每日提现截止（cutoff）时刻。

**原理**：

```pine
cutoff = hour(time) == 13
bgcolor(cutoff ? #f57f17 : transparent, title="Cutoff Time", transp=transparency)
```

只有一行判定：**当根 bar 的小时数等于 13** 就着色。13:00 UTC 曾是 BitMEX 长期沿用的每日提现批次时间——截止前提交的请求在当日批次广播，之后提交的顺延到下一个批次（2022 年后小额提现改为更高频的加速通道，13:00 批次逐渐变成大额/风控提现的兜底方案）。因此这个脚本的实际用途是：**提醒"今天的提现窗口已经/即将关闭"**，属于交易节奏管理，与价格判断无关。

**参数**

| 参数 | 默认 | 说明 |
| --- | --- | --- |
| `Transparency` | 70 | 背景透明度（0 = 实色，100 = 全透明） |

**输出**：仅在 `hour(time) == 13` 的 bar 上有背景色，其余 bar 完全透明。1 小时周期上正好一格；5 分钟周期上会连续涂 12 格（13:00–13:59）。

**注意**

- **时区**：`hour(time)` 这种带 `time` 参数的调用在 v3/v4 中默认取交易所时区（`syminfo.timezone`），BitMEX 的加密交易对就是 UTC，所以默认即对齐 UTC 13:00。改图表的显示时区**不会**影响着色位置；换成其它交易所的品种时这个 13 点会变成当地 13 点，语义就变了。
- **周期**：日线及以上周期里 `time` 是 bar 的开盘时间，`hour` 通常恒为 0（或交易所开盘时刻），此时要么全图不高亮，要么每根 bar 都高亮——该脚本只适合分钟/小时级图表。
- 与 BitMEX 现状无关（该所已于 2026 年 9 月停止交易并进入仅保留提现的清退阶段），脚本本身是历史需求的产物，作为"单时刻背景高亮"的极简范例仍然可用。
  13:00 UTC 批次的历史依据：[BitMEX 提现流程说明](https://www.bitmex.com/blog/safeguarding-the-stability-of-our-withdrawals-process?lang=en_us)、[CoinDesk 报道](https://www.coindesk.com/markets/2020/10/02/bitmex-moves-337m-in-bitcoin-ahead-of-first-user-withdrawals-since-us-charges)。

---

## 2. `leap_years.pine` — 闰年高亮

**做什么**：把**闰年的整段行情**铺一层淡橙色背景，并在进入闰年的第一根 bar 下方打一个 `Leap Year` 标签，用于做跨年/跨十年的时间序列对比。

**原理**：标准格里高利历闰年判定，逐级排除：

```pine
if year % 4 != 0          // 不能被 4 整除 → 平年
    isLeap := false
else
    if year % 100 != 0    // 能被 4 整除但不能被 100 整除 → 闰年
        isLeap := true
    else
        if year % 400 != 0 // 能被 100 整除但不能被 400 整除 → 平年
            isLeap := false
        else               // 能被 400 整除 → 闰年
            isLeap := true
```

等价于一句：`isLeap = (year % 4 == 0 and year % 100 != 0) or (year % 400 == 0)`。
年首标签的判定是 `lyStart = year != year[1] and isLeap`，即"年份发生变化的第一根 bar 且该年是闰年"。

**参数**

| 参数 | 默认 | 说明 |
| --- | --- | --- |
| `Transparency` | 88 | 背景透明度 |
| `Show Labels ?` | true | 是否画年首标签（关掉只保留背景色） |

**输出**

- 闰年区间的背景色 `#ffa726`（琥珀橙），透明度 88 → 视觉上非常淡，只作为"年份边界"的提示；
- 每个闰年第一根 bar 下方一个 `shape.labelup` 小标签，蓝底 `#1e90ff` 白字，文本两行 ` Leap ` / `Year`；
- Data Window 无新增列。

**注意**

- 闰年判定**只看年份**，不看是否已经过了 2 月 29 日，所以标签出现在 1 月 1 日而不是 2 月 29 日。
- 若图表的起始日期晚于某年 1 月 1 日，那一年的标签不会出现（`year != year[1]` 在首根 bar 上恒为 false），但背景色照常生效。
- `year` 用的是无参变量形式，与同目录 `quarters` 的 `month(time)` 取值口径一致——都按**交易所时区**（`syminfo.timezone`）计算，不受图表显示时区影响。
- 对隔夜开盘的市场（外汇、部分期货），新年第一根 bar 可能落在 12 月 31 日的交易时段里，标签会提前一天出现。

---

## 3. `quarters.pine` — 季度高亮

**做什么**：给四个季度分别上不同的背景色，并在每季度第一根 bar 下方标 `Q1` / `Q2` / `Q3` / `Q4`，用来做季节性（seasonality）分析的目视分区。

**原理**：对每个季度重复同一套"判定 + 背景 + 标签"三件套，月份直接用 `month(time)`：

```pine
m = month(time)

q1 = m == 1 or m == 2 or m == 3
q1Start = m[1] == 12 and m == 1        // 上个月是 12 月、本月是 1 月 → 季度起点
bgcolor(q1 ? #e6194b : transparent, title="Q1", transp=transparency)
plotshape(showLabels and q1Start, text=" Q1 ", style=shape.labelup, location=location.belowbar, ...)
```

四季判定 `q1..q4` 互斥且穷尽（`1-3 / 4-6 / 7-9 / 10-12`），所以四次 `bgcolor()` 调用不会互相覆盖。

**配色**

| 季度 | 颜色 | 说明 |
| --- | --- | --- |
| Q1 | `#e6194b` | 红 |
| Q2 | `#3cb44b` | 绿 |
| Q3 | `#ffe119` | 黄（高透明度下最不显眼） |
| Q4 | `#f58231` | 橙 |

四色取自常用的 4 色分类调色板，色相间隔足够大，缩放到全图仍然可区分。

**参数**

| 参数 | 默认 | 说明 |
| --- | --- | --- |
| `Transparency` | 88 | 背景透明度（四个季度共用） |
| `Show Labels ?` | true | 是否画季度首标签 |

**输出**

- 全图按月切换的四季背景色（同一透明度 88）；
- 每季度第一根 bar 下方的 `Q1`–`Q4` 蓝色标签；
- Data Window 无新增列。

**注意**

- 季度标签依赖**相邻两根 bar 的月份比对**（`m[1] == 12 and m == 1` 之类）。在月线及以下周期上工作正常；到**季线/半年线/年线**时，bar 的 `month` 是 1 / 4 / 7 / 10，`m[1]` 对不上，标签就会缺失，但背景色仍然正确。
- 季度边界是**自然季度**（1/4/7/10 月），不支持财政年度偏移（如 4 月起的财年）。
- 若只看某一季度，把对应脚本段落之外的判断删掉即可；也可以把 `Transparency` 调到 92+ 让四季几乎不可见、只留标签。

---

## 4. `range_candles.pine` — 重绘 K 线（强制涨跌配色）

**做什么**：用 `plotcandle()` 在原生 K 线之上**重画一遍同样的 OHLC**，只为了强制使用"涨青绿 / 跌珊瑚红"的二色配色，并把实体、边框、影线统一成同一颜色。

**原理**：

```pine
candleColor = close >= open ? bullColor : bearColor
plotcandle(open, high, low, close, color=candleColor, bordercolor=candleColor, wickcolor=candleColor)
```

数值上没有任何加工：`open/high/low/close` 原样传入。唯一的判断是**当根是阳线还是阴线**（`close >= open`，平盘算阳线）。视觉上得到的是"实心、无空心、无第三色"的蜡烛，覆盖在原生蜡烛的同位置上。

**参数**

| 参数 | 默认 | 说明 |
| --- | --- | --- |
| `Bull Color` | `#26a69a` | `close >= open` 时的颜色（青绿） |
| `Bear Color` | `#ef5350` | 其余情况的颜色（珊瑚红） |

两个输入都是 v4 的**颜色选择器**，可在设置面板里直接改。

**输出**

- 图上多一层蜡烛（位置与原生 K 线完全重合，所以看不出"两根"，只看到配色被替换）；
- Data Window 会多出 4 列 OHLC 数值（`plotcandle` 的 `title=""`，列名是空的）；
- 不改变任何数据、不产生信号。

**注意**

- **不会**把图表变成 Range / Renko 图表。名字里的 Range 指的是"这类非时间型 K 线图"：因为它只读 OHLC 值、不依赖时间轴，所以在 Range、Renko、Kagi、Line Break 等图表上同样生效——这是它存在的主要场景（这些图表类型下部分配色设置不可用，用脚本重绘更可控）。
- 重绘层会**盖住**原生蜡烛的样式设置（空心/填充、上下影线配色、`barcolor` 的蜡烛着色都会被遮住）；把脚本从图表移除即恢复。
- 和原生蜡烛一样，它按 `close >= open` 二分，因此不区分"高开低走的阳线"与"真正的强势阳线"——只是配色工具，不携带更多信息。
- 想让它在普通时间图上不显得多余，最简单的用法是配合 `barcolor` 类脚本（本目录没有）做"蜡烛统一配色"，避免多脚本各自染色互相覆盖。

---

## 5. `weekdays_gaps.pine` — 缺失交易日（数据缺口）标记

**做什么**：检测**相邻两根 bar 的星期几是否连续**，一旦出现跳跃（例如上根是周五、这根是周一），就把缺口之后的第一根 bar 背景涂成淡橙色——用来在 24/7 市场上发现"整天缺失的数据"。

**原理**：

```pine
td = dayofweek(time)        // 当根
yd = dayofweek(time[1])     // 上一根

gaps = td == mo and yd != su or   // 本根周一，上根却不是周日 → 缺周日
       td == tu and yd != mo or   // 本根周二，上根却不是周一 → 缺周一
       td == we and yd != tu or
       td == th and yd != we or
       td == fr and yd != th or
       td == sa and yd != fr or
       td == su and yd != sa
```

（Pine 中 `and` 优先于 `or`，所以这串表达式就是 7 个 `(今天 == X and 昨天 != X-1)` 的并集。）
逻辑本质是：**"今天的星期"与"上一根 bar 的星期"必须严格相差一天**，否则中间就有数据缺失。七个条件覆盖一周七天，缺任何一天都能被相邻的那次比较抓到。

**参数**：无。缺口色 `#ffcc80`（浅橙，透明度 70）和正常色（全透明白）都写在代码里。

**输出**

- 缺口后第一根 bar 上的背景色，宽度正好 1 根 bar（`bgcolor` 是按 bar 着色的，只有跨日的那一根满足条件）；
- 其余 bar 全透明；
- Data Window 无新增列。

**注意**

- **对 24/5 市场（股票、期货）每个周一都会触发**：周五 → 周一本身就是星期跳跃，脚本无法区分"正常休市"和"数据缺失"。所以它的正确定位是**加密货币等 7×24 品种的数据完整性检查**。
- 只比较星期的**相邻性**，因此 `周一 → 下周一`（整周缺失）不会触发，也分不清缺了几天；高亮宽度恒为 1 根 bar，不会按缺口长度延伸。
- 对隔夜开盘的市场（外汇、期货夜盘），`dayofweek` 取的是 bar 的**开盘**星期，周日晚上 22:00 开盘的"周一交易日"会被算成周日，缺口判定随之整体偏移；这类品种不建议使用。
- 高周期（周线、月线）上，`dayofweek` 反映的是 bar 的开盘星期，条件几乎总为真或恒假，失去了区分意义；建议在日线及以下周期使用。

---

## 执行后得到什么？（输出清单）

把这 5 个脚本加载到图表并执行完，产物只有**价格图上的可视化层**；除 `range_candles` 外都不会在 Data Window 留下数值列。

| 脚本 | 图上新增 | 标签 / 着色 | Data Window | 首根有效 bar | 有信号 / 告警？ |
| --- | --- | --- | --- | --- | --- |
| `bitmex_withdrawals_cutoff_time` | 1 个背景色图层（`Cutoff Time`） | 仅 `hour == 13` 的 bar 上色 | 无 | 第 1 根起（只读当根 `time`） | 无 |
| `leap_years` | 1 个背景层（`Leap Year`）+ 1 个 `plotshape` 层 | 闰年整段淡橙；年首蓝色小标签 | 无 | 第 2 根起（年首标签需 `year[1]`） | 无 |
| `quarters` | 4 个背景层（`Q1`–`Q4`）+ 4 个 `plotshape` 层 | 四季四色背景；季首 `Qx` 标签 | 无 | 第 2 根起（`m[1]` 依赖） | 无 |
| `range_candles` | 1 个 `plotcandle` 层 | 全图蜡烛被替换为二色实心蜡烛 | **4 列**（O/H/L/C，列名为空） | 第 1 根起 | 无 |
| `weekdays_gaps` | 1 个背景层（`Gap`） | 缺口后首根 bar 淡橙 | 无 | 第 2 根起（`dayofweek(time[1])`） | 无 |

### 具体在图上看到什么

- **BitMEX 提现截止**：一天里只有一小段竖条有颜色（1h 周期 = 1 格，5m 周期 = 12 格），像日历上的"每日打卡点"。
- **闰年**：图上出现约每 4 年一组的淡橙色区块（遇到 2100 这类世纪平年会断开），区块起点有蓝标签；由于透明度 88，几乎只能看出"有个颜色块"。
- **季度**：全图被切成红 / 绿 / 黄 / 橙四段循环，每段起点有 `Q1`–`Q4` 蓝标签，是最"花"的一个脚本。
- **Range Candles**：图表蜡烛外观整体变为青绿/珊瑚红实心样式，无空心、无边框差异；把脚本移除即还原。
- **Weekdays Gaps**：散落的单根淡橙色竖线，位置对应数据断档（加密品种上=真正的数据缺失；股票上=每周一）。

### 不会得到的东西

- **买卖信号 / 箭头 / 收益曲线**：5 个脚本都是 `study`（indicator），没有 `strategy` 入口，不能回测。
- **内置告警**：没有 `alertcondition`，只能在 TradingView 的「创建警报」里基于已有 plot 手动挂（本目录前 4 个时间类脚本连 plot 都没有，无法直接挂条件）。
- **副图**：全部 `overlay=true`，没有独立子图、没有振荡器式读数。
- **跨脚本共存的稳定外观**：`bgcolor` 之间、`plotcandle` 与原生蜡烛之间会互相覆盖，同时加载多个高亮脚本时建议逐个开关确认效果。
- **可复用的数值序列**：只有 `range_candles` 的 OHLC 可在 Data Window 读到；其余脚本的时间判定结果没有输出到 Data Window，想程序化使用需要自己加 `plot()` 或 `alertcondition()`。

---

## 横向对比：该用哪一个？

| 需求 | 推荐 | 理由 |
| --- | --- | --- |
| 固定时刻提醒（提现/结算/开盘） | `bitmex_withdrawals_cutoff_time.pine` | 单条件背景色，改一下 `hour(time) == 13` 就能改成任意整点 |
| 跨年对比、看长周期节奏 | `leap_years.pine` | 约 4 年一组（世纪平年除外），用来观察"闰年 vs 平年"的季节性假设 |
| 季节性 / 季度效应研究 | `quarters.pine` | 四季异色，季度边界一目了然 |
| 统一 K 线配色、Range/Renko 图上着色 | `range_candles.pine` | 唯一作用在 OHLC 上的脚本，非时间型图表也能用 |
| 检查 24/7 数据是否完整 | `weekdays_gaps.pine` | 用星期连续性反推缺失日，零参数即插即用 |

## 本目录脚本的共同约定

- 统一的绘图风格：`bgcolor()` + `transp` 高透明度（70 / 88），不遮挡价格；标签统一为 `shape.labelup` + `location.belowbar` + `size.small` + 蓝色 `#1e90ff`。
- 参数极简：时间类脚本只有 `Transparency` 与 `Show Labels ?`（`weekdays_gaps` 甚至没有参数）。
- 时间类脚本的判定都基于 bar 的**时间戳**（`time`），因此**不依赖任何品种特性**，可套用到任意 symbol；但同时也意味着**时区与图表周期决定了实际效果**（见下节）。`range_candles` 是唯一的例外——它只读 OHLC，与时间无关。

## 时区与图表周期：最容易踩的三个坑

1. **时区**：Pine 脚本**读不到图表的显示时区**（没有 `chart.timezone` 这种东西）。`hour` / `month` / `dayofweek` / `year` 等时间函数和变量，无论带不带 `time` 参数，默认都按**交易所时区** `syminfo.timezone` 计算。因此：
   - 改图表时区只会改变坐标轴上的时间文字，**不会**让高亮位置跟着移动——"我明明切了时区，色块还差 8 小时"是这类脚本最常见的困惑；
   - BitMEX 的加密交易对 `syminfo.timezone` 就是 UTC，`hour(time) == 13` 天然对齐 13:00 UTC；换成股票/期货品种时同样按该交易所的时区解释，语义就变了；
   - 想彻底锁定某个时区，改成显式传参（v5+：`hour(time, "UTC")`、`month(time, "UTC")`），别依赖交易所默认值。
2. **周期**：时间类脚本都是"逐 bar 判定"，
   - 越小周期，一个"时刻"占用的 bar 越多（1h 的 cutoff 在 5m 图上变成 12 根）；
   - 越大周期，相邻 bar 的月份/星期比对越容易失配（季线上季度标签会消失，周线/月线上缺口判定失去意义）；
   - 日线以上的图表里 `hour` 恒为开盘时刻，`bitmex_withdrawals_cutoff_time` 基本失效。
3. **bar 时间取的是开盘时刻**：`month` / `dayofweek` 等返回的是该 bar **开盘时间**所属的年月日。对隔夜交易时段（外汇周一凌晨开盘、期货夜盘）而言，这个"日期"可能比通常意义的交易日早一天，`quarters` 的季初标签与 `weekdays_gaps` 的缺口判定都会随之偏移。

## 迁移到 Pine v5 / v6 的改动清单

本目录为历史脚本（v3 / v4 语法），在新版编辑器中需要做如下机械替换：

| 旧（v3 / v4） | 新（v5+） |
| --- | --- |
| `study("...", overlay=true)` | `indicator("...", overlay=true)` |
| `input(title=..., type=integer, defval=88, minval=0, maxval=100)` | `input.int(88, "Transparency", minval=0, maxval=100)` |
| `input(title=..., type=bool, defval=true)` | `input.bool(true, "Show Labels ?")` |
| `input(title="Bull Color", defval=#26a69a)` | `input.color(#26a69a, "Bull Color")`（去掉 `var color`） |
| `color(white, 100)` | `color.new(color.white, 100)` |
| `bgcolor(..., transp=transparency)` | `bgcolor(color.new(c, transparency), title=...)`，删除 `transp` 参数 |
| `plotshape(..., transp=0)` | 删除 `transp`，颜色直接用 `color.new(c, 0)` |
| `plotcandle(..., color=..., bordercolor=..., wickcolor=...)` | 参数名不变，`color` 支持 series color，移除 `transp` |
| `hour(time)` / `month(time)` / `dayofweek(time)` | 保留（v6 仍可用），建议显式传时区 `hour(time, "UTC")` |
| `dayofweek.monday` 等常量 | 不变 |
| `year % 4`、`x != x[1]` | 不变 |
| 多行 `if / else` 给 `isLeap` 赋值 | 可内联为 `isLeap = (year % 4 == 0 and year % 100 != 0) or (year % 400 == 0)` |
| `labelColor = #1e90ff` | 保持不变（十六进制颜色字面量在 v5/v6 仍支持） |
