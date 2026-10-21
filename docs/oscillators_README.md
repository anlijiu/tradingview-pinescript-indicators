# Oscillators（振荡器指标）

本目录收录 73 个 TradingView Pine Script 振荡器类指标（67 个 `//@version=3`、6 个 `//@version=4`），
全部是**副图指标**（`study()`），不含策略下单逻辑。每个脚本都在 TradingView 上渲染成一条或多条
振荡曲线 + 参考线 + 着色/标记，用来判断超买超卖、动量强弱、趋势方向或周期循环位置。

下面按指标家族分组，逐个说明：**做什么（用途）→ 怎么算（原理）→ 跑出来得到什么（输出）**。

目录：

1. [通用约定（所有脚本共有的输出形态）](#通用约定)
2. [一、价格与基准的离差（Deviation）](#一价格与基准的离差deviation) — 11 个
3. [二、RSI 与相对强弱家族](#二rsi-与相对强弱家族) — 18 个（其中 2.7 覆盖两个重复文件）
4. [三、随机指标与区间归一化](#三随机指标与区间归一化) — 6 个
5. [四、动量、变化率与 MACD 家族](#四动量变化率与-macd-家族) — 12 个
6. [五、成交量类](#五成交量类) — 4 个
7. [六、Ehlers 数字信号处理（DSP）家族](#六ehlers-数字信号处理dsp家族) — 10 个
8. [七、趋势强度与方向](#七趋势强度与方向) — 10 个
9. [八、波动率与区间](#八波动率与区间) — 2 个
10. [附录 A：输出刻度速查](#附录-a输出刻度速查)
11. [附录 B：重复实现、瑕疵与注意事项](#附录-b重复实现瑕疵与注意事项)
12. [附录 C：使用与移植提示](#附录-c使用与移植提示)

---

## 文件清单

| 文件 | 指标（简称） | Pine 版本 | 默认参数 | 输出刻度 |
| --- | --- | --- | --- | --- |
| `accelerator_oscillator.pine` | Accelerator Oscillator (AC) | 6 | fast=5, slow=34, smooth=5 | 四色柱，围绕 0 |
| `accumulative_swing_index.pine` | Accumulative Swing Index (ASI) | 6 | limit=10000 | 累计值，无界 |
| `asymmetrical_rsi.pine` | Asymmetrical RSI (ARSI) | 6 | length=14, OB/OS=70/30 | 0–100 |
| `awesome_oscillator.pine` | Awesome Oscillator (AO) | 6 | fast=5, slow=34 | 四色柱，围绕 0 |
| `cci_commodity_channel_index.pine` | Commodity Channel Index (CCI) | 6 | length=20, OB/OS=±100 | 无界（多数在 ±100） |
| `center_of_gravity_oscillator.pine` | CG Oscillator (CGO) | 6 | length=10, src=hl2 | 围绕 0 |
| `chande_momentum_oscillator.pine` | Chande Momentum Oscillator (CMO) | 6 | length=9, ±50 | −100…100 |
| `coppock_curve.pine` | Coppock Curve | 6 | ROC 14/11, WMA 10 | 围绕 0 |
| `demarker.pine` | DeMarker | 6 | length=14, MA=SMA, OB/OS=0.7/0.3 | 0–1 |
| `derivative_oscillator.pine` | Derivative Oscillator (DOSC) | 6 | RSI 14, EMA 5/3, SMA 9, signal 9 | 围绕 0，柱状 |
| `didi_index.pine` | Didi Index | 6 | SMA 3/8/20 | 围绕 1 |
| `disparity_index.pine` | Disparity Index (DI) | 6 | length=14 | 围绕 0（%） |
| `dorsey_inertia.pine` | Dorsey Inertia | 6 | stdev 21, RVI 14, linreg 14 | 0–100（50 中轴） |
| `dynamic_momentum_index.pine` | Dynamic Momentum Index (DMI) | 6 | RSI 14, stdev 5, MA 10, 5–30 | 0–100 |
| `ehlers_cyber_cycle.pine` | Ehlers Cyber Cycle (ECC) | 6 | length=14, alpha=0.7 | 围绕 0 |
| `ehlers_decycler_oscillator.pine` | Ehlers Decycler Oscillator | 6 | 125/k=1 与 100/k=1.2 | 围绕 0 |
| `ehlers_fisherized_deviation_scaled_oscillator.pine` | EFDSO | 6 | length=40, SSF 20, 2 poles, OB/OS=±2 | Fisher 值（±4 内） |
| `ehlers_roofing_filter.pine` | Ehlers Roofing Filter (ERF) | 6 | highpass 48, SSF 10, 2 poles | 围绕 0 |
| `ehlers_smoothed_adaptive_momentum.pine` | ESAM | 6 | cutoff=8, alpha=0.7, 3 poles | 围绕 0 |
| `ehlers_stochastic_cyber_cycle.pine` | Ehlers Stochastic Cyber Cycle (ESCC) | 6 | length=14, alpha=0.7 | 约 ±1 |
| `ehlers_stochastic.pine` | Ehlers Stochastic (EStoch) | 6 | hp 48, SSF 10, stoch 20, OB/OS=0.8/0.2 | 0–1 |
| `ehlers_super_passband_filter.pine` | Ehlers Super Passband Filter (ESPF) | 6 | fast=40, slow=60, RMS 50 | 围绕 0 + 包络 |
| `ehlers_triple_delay_line_detrender.pine` | Ehlers Triple Delay-Line Detrender (TDLD) | 6 | 固定系数（0.088 / 1.2 / 0.7） | 围绕 0 |
| `ergodic_oscillator.pine` | Ergodic Oscillator (EOSC) | 6 | 8/5/5/5/14, OB/OS=±20 | 围绕 0 |
| `forecast_oscillator.pine` | Forecast Oscillator (FOSC) | 6 | length=14 | 围绕 0（%） |
| `gator_oscillator.pine` | Gator Oscillator | 6 | 13/8/5, offset 8/5/3 | 上下双柱，0 起 |
| `intraday_momentum_index.pine` | Intraday Momentum Index (IMI) | 6 | length=14, OB/OS=70/30 | 0–100 |
| `jurik_rsx.pine` | Jurik RSX | 6 | length=14, src=hlc3, OB/OS=70/30 | 0–100 |
| `kairi_relative_index.pine` | Kairi Relative Index (KRI) | 6 | length=14 | 围绕 0（%） |
| `laguerre_rsi.pine` | Laguerre RSI (LRSI) | 6 | alpha=0.7, OB/OS=0.8/0.2 | 0–1 |
| `laguerre_rsi_self_adjusting_alpha_with_fractals_energy.pine` | Laguerre RSI + 分形能量 | 6 | alpha=0.7, FE length=13 | 0–1 |
| `mass_index.pine` | Mass Index (MI) | 6 | EMA 9, sum 25, breakout 27 | 0–30+ |
| `mfi_money_flow_index.pine` | Money Flow Index (MFI) | 6 | length=14, signal 9, OB/OS=80/20 | 0–100 |
| `normalized_relative_vigor_index.pine` | Normalized RVI (NRVGI) | 6 | length=10, SWMA | 围绕 0（±10） |
| `percentage_price_oscillator.pine` | Percentage Price Oscillator (PPO) | 6 | 12/26/9 | 围绕 0（%） |
| `percentage_volume_oscillator.pine` | Percentage Volume Oscillator (PVO) | 6 | 12/26/9（作用于 volume） | 围绕 0（%） |
| `phase_change_index.pine` | Phase Change Index (PCI) | 6 | length=35, 80/20 | 0–100 |
| `pretty_good_oscillator.pine` | Pretty Good Oscillator (PGO) | 6 | length=14, ±2 / ±4 | ATR 倍数 |
| `price_momentum_oscillator.pine` | Price Momentum Oscillator (PMO) | 6 | 35/20/10 | 围绕 0 |
| `pring_special_k.pine` | Pring Special K (PSK) | 6 | 12 组周期/权重硬编码 | 围绕 0 |
| `rainbow_oscillator.pine` | Rainbow Oscillator (ROSC) | 6 | MA 2 × 10 层, lookback 10 | 柱状 + ±band |
| `random_walk_index.pine` | Random Walk Index (RWI) | 6 | length=14 | 比值（> 1 有趋势） |
| `range_action_verification_index.pine` | RAVI | 6 | fast 7 / slow 65, lookback 1 | ≥ 0 |
| `rapid_rsi.pine` | Rapid RSI (RRSI) | 6 | length=14, OB/OS=70/30 | 0–100 |
| `recursive_median_oscillator.pine` | Recursive Median Oscillator (RMO) | 6 | median 5, LPF 12, HPF 30 | 围绕 0 |
| `regression_oscillator.pine` | Regression Oscillator (ROSC) | 6 | length=63, offset 0 | 围绕 0（%） |
| `relative_momentum_index.pine` | Relative Momentum Index (RMI) | 6 | length=14, momentum 3, 70/30 | 0–100 |
| `relative_spread_strength.pine` | Relative Spread Strength (RSS) | 6 | RSI 14, 10/40, smooth 5, 80/20 | 0–100 |
| `relative_volatility_index.pine` | Relative Volatility Index (RVI) | 6 | stdev 10, smooth 14, OB/OS=60/40 | 0–100 |
| `repulse.pine` | Repulse | 6 | length=5, EMA 25 | 围绕 0 |
| `rocket_rsi.pine` | Rocket RSI | 6 | RSI 10, smooth 8, OB/OS=±2 | Fisher 值（±3） |
| `rsi_relative_strength_index.pine` | Relative Strength Index (RSI) | 6 | length=14, OB/OS=70/30 | 0–100 |
| `rsx.pine` | Jurik RSX | 6 | length=14, src=hlc3, OB/OS=70/30 | 0–100 |
| `self_adjusting_rsi.pine` | Self-Adjusting RSI (SARSI) | 6 | RSI 14, mult 2, smooth 21 | 0–100（动态阈值） |
| `smoothed_rate_of_change.pine` | Smoothed Rate of Change (S-RoC) | 6 | ROC 21, EMA 13 | 围绕 0（%） |
| `stc_schaff_trend_cycle.pine` | Schaff Trend Cycle (STC) | 6 | MACD 23/50, cycle 10, %D 3/3, 75/25 | 0–100 |
| `stiffness_indicator.pine` | Stiffness Indicator | 6 | MA 100, stiff 60, smooth 3, threshold 90 | 0–100 |
| `stochastic_connors_rsi.pine` | Stochastic Connors RSI | 6 | stoch 3, K/D 3/3, RSI 3, updown 2, roc 100 | 0–100 |
| `stochastic_momentum_index.pine` | Stochastic Momentum Index (SMI) | 6 | 13/25/2, signal 12, OB/OS=±40 | 约 ±100 |
| `stochastic.pine` | Stochastic | 6 | K 14, D 3, smooth 3, OB/OS=80/20 | 0–100 |
| `tick_line_momentum_oscillator.pine` | Tick Line Momentum Oscillator (TLMO) | 6 | cumo 10, ROC 5, smooth 5, ±5 | 围绕 0 |
| `total_power_indicator.pine` | Total Power Indicator (TPI) | 6 | length 45, Elder 10, level 75 | 0–100 |
| `trend_continuation_factor.pine` | Trend Continuation Factor (TCF) | 6 | length=35 | 围绕 0 |
| `trend_detection_index.pine` | Trend Detection Index (TDI) | 6 | length=20 | 围绕 0（强度 ≥ 0） |
| `trend_intensity_index.pine` | Trend Intensity Index (TII) | 6 | major 60 / minor 30, 80/20 | 0–100 |
| `trendscore.pine` | Trendscore | 6 | from=11, to=20 | −10…+10 |
| `trend_trigger_factor.pine` | Trend Trigger Factor (TTF) | 6 | length=15, ±100 | ±200 |
| `trix.pine` | TRIX | 6 | length=14, signal 9, mult 1 | 围绕 0（%） |
| `true_range_double.pine` | True Range Double (TRD) | 6 | 无参数 | 价格量纲 |
| `tsi_true_strength_index.pine` | True Strength Index (TSI) | 6 | 25/13/13, OB/OS=±40 | 约 ±100 |
| `vertical_horizontal_filter.pine` | Vertical Horizontal Filter (VHF) | 6 | length=28 | 比值（0–1+） |
| `v_macd.pine` | V-MACD | 6 | 12/26/9（作用于 volume） | 围绕 0 |
| `vw_macd.pine` | VW-MACD | 6 | 12/26/9（VWMA） | 围绕 0 |

> 全部脚本版权声明为 `Alex Orekhov (everget)`，GPL-3.0。

---

## 通用约定

**运行方式**：把 `.pine` 内容粘到 TradingView 的 Pine Editor，保存后 *Add to chart*。因为都是
`study()`，会自动落在副图（separate pane），不会覆盖主图价格。

**“执行完会得到什么”的总答案**：一条随每根 K 线更新的数值序列（画成线或柱）、若干水平参考线、
可选的区域填充/背景色、可选的事件圆点，以及部分脚本声明的告警条件。**没有**买卖指令、没有回测
收益，也不输出表格数据；需要做量化时得自己把公式移植到脚本/回测框架里。

**图元类型**：

| 图元 | 含义 |
|---|---|
| `plot()` 线 | 指标主值（柱线默认 1–2px） |
| `plot(style=columns/histogram)` | 柱状值，通常用于 MACD 柱、AO/AC、四色柱 |
| `hline()` | 0 轴、50 中轴、超买/超卖阈值、最大/最小值边界 |
| `fill()` | 两线之间的色带：超买超卖区、主线与信号线之间的 ribbon、突破区高亮 |
| `plotshape()` | K 线交叉事件的小圆点 |
| `bgcolor()` | 背景色，表示零轴/中轴之上或之下 |
| `alertcondition()` | 可添加到 TradingView 告警的条件（仅 CCI、SARSI、STC 有） |

**配色语义**（作者 Alex Orekhov 的固定习惯，读图时可以按颜色理解状态）：

| 颜色 | 含义 |
|---|---|
| `#0ebb23` green | 多头 / 超买 / 上升 |
| `#ff0000`、`red`、`#cc0000` | 空头 / 超卖 / 下降 |
| `#f4b77d` | 中性区（既不超买也不超卖） |
| `#26A69A` / `#B2DFDB` | 柱值为正且上升 / 为正但下降（收缩） |
| `#FFCDD2` / `#EF5350` | 柱值为负但上升（收缩）/ 为负且下降 |
| `fill(..., color.purple, transp=95)` | 超买与超卖之间的“中性区” |

**参数习惯**：`Fast/Slow Length` 控制快慢均线，`Length` 是回看窗口，`OB/OS Level` 是阈值，
`highlight*` / `applyFilling` 等布尔开关只影响可视化，不影响数值。

---

## 一、价格与基准的离差（Deviation）

这一类把价格和某个基准（均线、线性回归、均线束）相减并标准化，输出围绕 0 或 1 振荡的曲线，
衡量“价格相对基准偏离了多少”。

### 1.1 `cci_commodity_channel_index.pine` — Commodity Channel Index (CCI)

- **用途**：衡量典型价偏离其均值的程度，是超买超卖 + 趋势启动的经典指标。
- **原理**：`CCI = (src − SMA(src, 20)) / (0.015 × dev(src, 20))`，其中 `dev()` 是平均绝对偏差
  （mean absolute deviation）。常数 `0.015` 让约 70%–80% 的数值落在 ±100 内。`src` 可在 20 种
  基础序列里选：OHLC 及其两两均价（`oo2`…`cc2`）、`hl2`/`hlc3`/`ohlc4`/`wc`、`volume`、
  `accdist`、`on balance volume`、`price volume trend`。
- **输出**：CCI 曲线（> 100 绿、< −100 红、其余橙色）+ ±100 虚线 + 0 轴 + ±300 透明边界 +
  ±100 之间紫色中性填充 + 突破 ±100 时的绿/红高亮 + 两个 `alertcondition`（上穿 100 / 下穿 −100）。

### 1.2 `center_of_gravity_oscillator.pine` — CG Oscillator (CGO, Ehlers)

- **用途**：用“价格的重心位置”代替普通均线偏离，得到更平滑、更少滞后的零轴振荡器。
- **原理**：对窗口内价格做线性加权（越新权重越大）后取比值并取负：
  `CGO = −Σ_{i=0..n−1} (i+1)·src[i] / Σ src[i]`，`src` 默认 `hl2`，`length` 默认 10。
  取负号是让上升趋势为正。
- **输出**：CGO 主线 + `CGO[1]`（上一根值）构成的“触发线”；可选填充（CGO > CGO[1] 绿，否则红）。

### 1.3 `disparity_index.pine` — Disparity Index (DI)

- **用途**：最朴素的均线乖离率指标。
- **原理**：`DI = 100 × (close − SMA(close, 14)) / SMA(close, 14)`，单位是百分比。
- **输出**：单线（≥ 0 绿、< 0 红）+ 0 轴虚线。

### 1.4 `kairi_relative_index.pine` — Kairi Relative Index (KRI)

- **用途**：日式乖离率，效果与 Disparity Index 等价。
- **原理**：`KRI = 100 × (src − SMA(src, 14)) / SMA(src, 14)`（与 1.3 公式相同，仅默认配色/线型不同）。
- **输出**：单线（深青色 `#134f5c`，不做涨跌着色）+ 0 轴虚线。

### 1.5 `forecast_oscillator.pine` — Forecast Oscillator (FOSC)

- **用途**：衡量现价相对“线性回归预测值”的偏离（Tushar Chande）。
- **原理**：`FOSC = 100 × (src − linreg(src, 14, 0)) / src`，`linreg` 是窗口末端的线性回归值。
- **输出**：单线（≥ 0 绿、< 0 红）+ 0 轴虚线。

### 1.6 `regression_oscillator.pine` — Regression Oscillator (ROSC)

- **用途**：与 FOSC 同类，但用回归值作分母、默认窗口更长（63），更像“价格相对回归通道的位置”。
- **原理**：`ROSC = 100 × (src / linreg(src, 63, offset) − 1)`，`offset` 可调回归取点位置。
- **输出**：单线（≥ 0 绿、< 0 红）+ 0 轴虚线。

### 1.7 `didi_index.pine` — Didi Index

- **用途**：巴西交易者常用的“短/中/长均线聚合”指标，看短均线是否上穿长均线。
- **原理**：`media = SMA(close, 8)`，`curta = SMA(close, 3) / media`，`longa = SMA(close, 20) / media`。
  短/长均线都换算成中均线的倍数，于是围绕 1 轴比较，消除价格量纲。
- **输出**：`curta` 绿线、`1` 黑实线（中轴）、`longa` 红线；两线之间 ribbon 填充（`curta > longa` 绿，
  否则红）；可选交叉处绿色/红色小圆点。

### 1.8 `gator_oscillator.pine` — Gator Oscillator

- **用途**：把 Bill Williams 的 Alligator（三条不同周期的 RMA）之间的“张口”程度画成直方图。
- **原理**：`jaw = rma(hl2, 13)`、`teeth = rma(hl2, 8)`、`lips = rma(hl2, 5)`，并各自向前位移 8/5/3 根
  （`offset`）；`upper = |jaw − teeth|`，`lower = −|teeth − lips|`（负值画在零轴下方）。
- **输出**：上半部分柱（`upper` 上升绿、下降红）+ 下半部分负柱（`lower` 上升绿、下降红）。
  柱高表示“张嘴”幅度，颜色变化表示张嘴/闭嘴/休眠。

### 1.9 `rainbow_oscillator.pine` — Rainbow Oscillator

- **用途**：用 10 层递进均线的“束”作为基准，衡量价格相对均线束的位置和均线束的宽度。
- **原理**：对 `src` 连续做 10 次均线（默认 SMA、长度 2）得到 `ma1…ma10`；
  `average = (ma1+…+ma10)/10`；`hlDiff = highest(src,10) − lowest(src,10)`；
  `ROSC = 100 × (src − average) / hlDiff`；`band = 100 × (maMax − maMin) / hlDiff`。
- **输出**：ROSC 柱状（正绿负红）+ `+band`/`−band` 两条边界线 + 中间 0 线；可选把带内区域填成
  绿/红（带宽 = 均线发散程度 = 趋势一致性）。
- **注意**：文件内 `if` 块的缩进混用了制表符与空格，粘进编辑器若报缩进错误，统一成同一种缩进即可。

### 1.10 `pretty_good_oscillator.pine` — Pretty Good Oscillator (PGO)

- **用途**：用 ATR 归一化的均线偏离，跨品种可比（Mark Johnson）。
- **原理**：`PGO = (src − SMA(src, 14)) / atr(14)`，单位是“多少个 ATR”。
- **输出**：单线（> 2 绿、< −2 红、其余橙）+ ±2 与 ±4 虚线 + 紫色中性区 + 突破区高亮。

### 1.11 `relative_spread_strength.pine` — Relative Spread Strength (RSS)

- **用途**：对“快慢均线价差”本身再求 RSI，判断趋势是加速还是衰竭。
- **原理**：`spread = EMA(10) − EMA(40)`（`smoothingType` 可选 SMA），`rs = rsi(spread, 14)`，
  `rss = sma(rs, 5)`。
- **输出**：单线（> 80 绿、< 20 红、其余橙）+ 80/50/20 虚线 + 中性区 + 突破区高亮。

---

## 二、RSI 与相对强弱家族

这一组都是“上涨力度 / 总力度”的某种变形，输出多为 0–100（或 0–1），可横向比较超买超卖。

### 2.1 `rsi_relative_strength_index.pine` — Relative Strength Index (RSI)

- **用途**：标准 Wilder RSI，但支持把源换成资金流/量能序列。
- **原理**：`RSI = 100 − 100 / (1 + RMA(up) / RMA(down))`，`length` 默认 14；`up = max(Δ,0)`、
  `down = max(−Δ,0)`。`src` 可选 19 种（含 `accdist`、`on balance volume`、`price volume trend`），
  把 RSI 用在资金流上可得到“资金流 RSI”。
- **输出**：RSI 曲线（> 70 绿、< 30 红、其余橙）+ 70/50/30 虚线 + 紫色中性区 + 突破区高亮。

### 2.2 `rapid_rsi.pine` — Rapid RSI (RRSI / Cutler RSI)

- **用途**：去掉 Wilder 平滑的 RSI 变体，反应更快、噪声更大。
- **原理**：`upSum = Σ max(Δ, 0)`、`downSum = Σ max(−Δ, 0)`（简单求和，非 RMA），
  `RRSI = 100 − 100 / (1 + upSum / downSum)`，并处理了分母为 0 的 100/0 边界。
- **输出**：曲线 + 70/50/30 虚线 + 中性区 + 突破区高亮（阈值写死 70/30）。

### 2.3 `relative_momentum_index.pine` — Relative Momentum Index (RMI)

- **用途**：把 RSI 的“1 期变化”换成“N 期变化”，过滤小波动、保留中期动量。
- **原理**：`up = RMA(max(change(src, 3), 0), 14)`、`down = RMA(−min(change(src, 3), 0), 14)`，
  `RMI = 100 − 100 / (1 + up / down)`。
- **输出**：曲线 + 70/50/30 虚线 + 中性区 + 突破区高亮。

### 2.4 `asymmetrical_rsi.pine` — Asymmetrical RSI (ARSI, v4)

- **用途**：解决“单边趋势中 RSI 长期钝化”的问题——上涨/下跌越一边倒，平滑越少、越灵敏。
- **原理**：窗口内统计上涨 bar 数 `upCount` 与下跌 bar 数 `downCount`
  （`roc >= 0` 计入上涨），令 `α_up = 1 / upCount`、`α_down = 1 / downCount`，分别做单极点平滑：
  `upSum := α_up·roc⁺ + (1 − α_up)·upSum[1]`，`downSum` 对称；`ARSI = 100·upSum / (upSum + downSum)`。
  最后用 `fixnan` 兜底。
- **输出**：ARSI 曲线 + 70/50/30 虚线 + 中性区 + 突破区高亮。

### 2.5 `self_adjusting_rsi.pine` — Self-Adjusting RSI (SARSI)

- **用途**：让超买/超卖阈值随 RSI 自身波动率自动伸缩，避免固定 70/30 在高波动品种上失效。
- **原理**：`rsi = rsi(src, 14)`；调整量 `adjust = mult × stdev(rsi, 14)`
  （`Method of Adjusting = SMA` 时用 `mult × SMA(|rsi − SMA(rsi,14)|, 14)`）；`mult` 默认 2；
  `OB = 50 + adjust`、`OS = 50 − adjust`。
- **输出**：RSI 曲线（按动态阈值着色）+ 可选平滑线（SMA/EMA 21）+ 动态 OB/OS 之间的黄色填充带 +
  RSI 突破动态阈值的绿/红填充 + 两个 `alertcondition`。

### 2.6 `dynamic_momentum_index.pine` — Dynamic Momentum Index (DMI)

- **用途**：RSI 的周期随波动率自动变长/变短：波动大时缩短周期提高灵敏度，波动小时拉长周期降噪。
- **原理**：`V = stdev(src, 5) / SMA(stdev(src, 5), 10)`，`T = round(14 / V)` 并夹在 `[5, 30]`，
  然后用 `T` 作为 RSI 的窗口（脚本内置 Wilder 平滑 `_rma` 的 `_rsi`）。
- **输出**：DMI 曲线 + 70/50/30 虚线 + 中性区 + 突破区高亮。

### 2.7 `jurik_rsx.pine` / `rsx.pine` — Jurik RSX

- **用途**：Mark Jurik 的 RSX，用多级低通滤波构造出“几乎无滞后、噪声很低”的 RSI 替代品。
- **原理**：取 `v8 = 100·src − 100·src[1]`（放大后的 1 期动量），用 `α = 3/(n+2)` 的一阶 EMA 多级级联
  （每级再用 `1.5·f28 − 0.5·f30` 的低通去噪组合），分别对“动量”和“动量的绝对值”反复平滑得到 `v14`
  与 `v20`；`RSX = (v14 / v20 + 1) × 50`，用 `f88/f90` 计数器做预热保护，最后限幅到 0–100。
- **输出**：RSX 曲线 + 70/50/30 虚线 + 中性区 + 突破区高亮。
- **注意**：两个文件实现完全一致（都是 `study("Jurik RSX")`），属于重复文件。

### 2.8 `rocket_rsi.pine` — Rocket RSI

- **用途**：把平滑后的动量做 Fisher 变换，得到类似“火箭发射”的尖锐超买超卖读数。
- **原理**：`mom = change(src, rsiLength − 1)`；用 Ehlers 2 极 Super Smoother
  （`a1 = exp(−1.414π/n)`、`b1 = 2a1·cos(1.414·180/n)`、`c2 = b1`、`c3 = −a1²`、`c1 = 1 − c2 − c3`）
  平滑后累加涨跌和：`tmpRSI = (upSum − downSum) / (upSum + downSum)`，限幅 ±0.999 后
  `0.5·ln((1 + x)/(1 − x))`；可选 ×25 归一化到约 ±100。
- **输出**：Rocket RSI 曲线（> OB 绿、< OS 红）+ OB/OS 圆点状参考 + 0 轴。默认阈值 ±2（Fisher 域）。

### 2.9 `laguerre_rsi.pine` — Laguerre RSI (LRSI)

- **用途**：用 4 级 Laguerre 滤波器做 RSI，一个 α 参数就能连续调节灵敏度/平滑度。
- **原理**：`L0 := α·src + (1−α)·L0[1]`，`L1 := −(1−α)·L0 + L0[1] + (1−α)·L1[1]`，`L2`、`L3` 同样递推；
  `CU = Σ max(Li − Li+1, 0)`、`CD = Σ max(Li+1 − Li, 0)`，`LRSI = CU / (CU + CD) ∈ [0, 1]`。
- **输出**：LRSI 曲线（0–1）+ 0.8 / 0.2 虚线。

### 2.10 `laguerre_rsi_self_adjusting_alpha_with_fractals_energy.pine` — LRSI + 分形能量

- **用途**：把 LRSI 的 α 改成由“分形能量”自适应决定，并用 Heikin 化的综合价作输入。
- **原理**：`feAlpha = ln(Σ (max(H, C₁) − min(L, C₁)) / (HH − LL)，13) / ln(13)`（13 窗口内真实波幅占比的
  几何平均式），作为 α；输入价 `feSrc = (OC + HC + LC + C) / 4`，其中
  `OC = (O + C₁)/2`、`HC = max(H, C₁)`、`LC = min(L, C₁)`；其余递推与 2.9 相同。
- **输出**：LRSI 曲线 + 当启用分形能量时额外的 `feAlpha` 曲线（可当“市场能量/噪声”参考）+ 0.8/0.2 线。

### 2.11 `intraday_momentum_index.pine` — Intraday Momentum Index (IMI)

- **用途**：用每根 K 线“实体”（开→收）的涨跌力度替代收盘价变化，得到日内动量版 RSI。
- **原理**：`gain = close > open ? close − open : 0`，`loss = close > open ? 0 : open − close`，
  14 窗口求和后 `IMI = 100 × Σgain / (Σgain + Σloss)`。
- **输出**：IMI 曲线 + 70/50/30 虚线 + 中性区 + 突破区高亮（阈值写死 70/30）。

### 2.12 `demarker.pine` — DeMarker

- **用途**：用最高价/最低价的“需求扩张/收缩”衡量买卖压力，取值 0–1。
- **原理**：`demax = max(change(high), 0)`、`demin = max(−change(low), 0)`，各自做 MA（SMA/EMA/VWMA/WMA，
  默认 14）后 `DeMarker = MA(demax) / (MA(demax) + MA(demin))`。
- **输出**：0–1 曲线（> 0.7 绿、< 0.3 红、其余橙）+ 0.7/0.5/0.3 线 + 中性区 + 突破区高亮。

### 2.13 `derivative_oscillator.pine` — Derivative Oscillator (DOSC, v4)

- **用途**：把 RSI 当作价格序列再做一次 MACD，得到“RSI 的动量”。
- **原理**：`smoothedRSI = EMA(EMA(RSI(14), 5), 3)`，`DOSC = smoothedRSI − SMA(smoothedRSI, 9)`，
  `signal = SMA(DOSC, 9)`。
- **输出**：DOSC 四色柱（正且升 / 正但降 / 负但升 / 负且降）+ 黑色 signal 线。

### 2.14 `stochastic_connors_rsi.pine` — Stochastic Connors RSI (Stoch CRSI)

- **用途**：把 Larry Connors 的三合 RSI 再做随机归一化，用于极短周期均值回归。
- **原理**：`CRSI = avg( RSI(src,3), RSI(updown,2), percentrank(ROC(src,1), 100) )`；
  `updown` 是连续同向涨跌的连击计数（涨 +1、跌 −1、平 0）。然后
  `stochcalc = stoch(crsi, crsi, crsi, 3)`，`K = SMA(stochcalc, 3)`，`D = SMA(K, 3)`。
- **输出**：K 蓝线 + D 橙线 + 80/70/30/20 四条虚线 + 两个填充带（70–30 深色、80–20 浅色）。

### 2.15 `relative_volatility_index.pine` — Relative Volatility Index (RVI)

- **用途**：Donald Dorsey 的 RVI：用“波动率（标准差）而非价格变化”来度量涨跌方向的强度。
- **原理**：`stdev(src, 10)`（可选总体/样本），上涨时把 stdev 记入 `up`、下跌时记入 `down`，
  各做 `ema(·, 14)`，`RVI = 100 × up/(up + down)`。1995 改良版（默认启用）=
  `avg(RVI(high), RVI(low))`，比只看收盘价稳定。
- **输出**：0–100 曲线（> 60 绿、< 40 红、其余橙）+ 60/50/40 线 + 中性区 + 突破区高亮。

### 2.16 `dorsey_inertia.pine` — Dorsey Inertia

- **用途**：对 RVI 再做线性回归平滑，得到趋势惯性（RVI 的“趋势版”）。
- **原理**：先算 1993 版 RVI（`stdev` 21、`ema` 14，分别作用于 high 和 low 后平均），
  再取 `inertia = linreg(rvi, 14, 0)`。
- **输出**：惯性曲线（> 50 绿、< 50 红）+ 50 中轴虚线。

### 2.17 `normalized_relative_vigor_index.pine` — Normalized Relative Vigor Index (NRVGI)

- **用途**：用“收盘在开盘之上”的相对幅度衡量买盘活力，并放大 100 倍便于读图。
- **原理**：`RVGI = Σ swma(close − open) / Σ swma(high − low)`（`swma` 为 4 期对称加权均线，
  求和窗口 10），`signal = swma(RVGI)`；输出为 `100 × RVGI`。
- **输出**：NRVGI 绿线 + signal 红线（都在 0 附近小幅振荡）+ 可选 ribbon 填充（RVGI > signal 绿）。
- **备注**：它本质是 K 线实体强度的动量，也常被归入动量类（本目录按“相对强弱”归类）。

---

## 三、随机指标与区间归一化

这一组把数值映射到“区间内的相对位置”，输出通常限制在 0–100（或 −100…100），用来判断当前价在
近期区间中的位置。

### 3.1 `stochastic.pine` — Stochastic (v4)

- **用途**：经典随机指标，并补齐了直方图、ribbon、突破高亮等可视化。
- **原理**：`k = SMA(stoch(src, high, low, 14), 3)`（`stoch = 100 × (src − LL)/(HH − LL)`），
  `d = SMA(k, 3)`。
- **输出**：%K / %D 两条线（ribbon 填充看 K 是否在 D 上）+ 可选 `k − d` 四色柱 +
  80/50/20 虚线 + 中性区 + 突破区高亮 + 中轴上下背景色（`bgcolor`）+ K/D 交叉圆点。

### 3.2 `stochastic_momentum_index.pine` — Stochastic Momentum Index (SMI)

- **用途**：双重平滑的随机指标，比 Stochastic 更平滑、对噪声更不敏感。
- **原理**：`num = EMA(EMA(src − 0.5(HH_q + LL_q), r), s)`，`den = 0.5 × EMA(EMA(HH_q − LL_q, r), s)`，
  `SMI = 100 × num / den`，`signal = EMA(SMI, 12)`；默认 `q = 13`、`r = 25`、`s = 2`。
  以区间中点为 0 轴、以半个区间宽度为尺度。
- **输出**：SMI 线 + signal 线 + 可选 ribbon 填充 + ±40 虚线 + 0 轴虚线。

### 3.3 `stc_schaff_trend_cycle.pine` — Schaff Trend Cycle (STC, v4)

- **用途**：把 MACD 做两次“随机归一化 + 平滑”，压缩成 0–100 的周期指标，用于择时。
- **原理**：`macd = EMA(23) − EMA(50)` → `k = nz(fixnan(stoch(macd, macd, macd, 10)))`
  （MACD 在自身 10 周期区间中的百分位）→ `d = EMA(k, 3)` → 再对 `d` 做同样的随机归一化 →
  `stc = EMA(•, 3)`，限幅 0–100。
- **输出**：STC 曲线（> 75 绿、< 25 红、其余橙；或按斜率着色）+ 75/50/25 参考 + 突破区填充 +
  “下穿 75 / 上穿 25”的红/绿圆点 + 6 个 `alertcondition`（Buy、Sell、上轨穿越、下轨穿越等）。

### 3.4 `phase_change_index.pine` — Phase Change Index (PCI)

- **用途**：衡量价格相对“自身趋势直线”的相位偏离，判断趋势是否整齐推进。
- **原理**：以 `src[length]`（默认 35 根前）为起点、当前价为终点构造直线
  `gradient = src[35] + Δ × (n − i)/(n − 1)`，逐根计算 `deviation = src[n−i] − gradient`；
  累计正、负偏差，`PCI = 100 × pos / (pos + neg)`；可选再做一次 MA 平滑。
- **输出**：0–100 曲线 + 80/50/20 虚线 + 中性区 + 突破区高亮。

### 3.5 `trend_intensity_index.pine` — Trend Intensity Index (TII)

- **用途**：统计最近 N 根价格相对长均线的正距离占比，衡量趋势“推进强度”。
- **原理**：`sma = SMA(src, 60)`；在最近 30 根里累计 `price > avg` 的正距离与负距离，
  `TII = 100 × pos / (pos + neg)`。
- **输出**：0–100 曲线 + 80/50/20 虚线 + 中性区 + 突破区高亮。

### 3.6 `chande_momentum_oscillator.pine` — Chande Momentum Oscillator (CMO)

- **用途**：无平滑、无窗口长度偏置的对称动量指标，直接把涨跌和映射到 −100…100。
- **原理**：`upSum = Σ max(Δ, 0)`、`downSum = Σ −min(Δ, 0)`，
  `CMO = 100 × (upSum − downSum) / (upSum + downSum)`，`length` 默认 9。
- **输出**：曲线 + ±50 虚线 + 0 轴虚线 + 中性区 + 突破区高亮。

---

## 四、动量、变化率与 MACD 家族

这一组输出无界（或以价格为量纲），主要通过**零轴上下**、**快慢线交叉**和**柱体收缩/扩张**判断动量。

### 4.1 `accelerator_oscillator.pine` — Accelerator Oscillator (AC)

- **用途**：Bill Williams 系统里的“动量加速度”，是 AO 的变化率。
- **原理**：`AO = SMA(hl2, 5) − SMA(hl2, 34)`，`AC = AO − SMA(AO, 5)`。
- **输出**：AC 四色柱（正值且上升 `#26A69A`、正值但下降 `#B2DFDB`、负值但上升 `#FFCDD2`、
  负值且下降 `#EF5350`）。

### 4.2 `awesome_oscillator.pine` — Awesome Oscillator (AO)

- **用途**：Bill Williams 的动量指标，用中价快速/慢速均线差判断动量方向。
- **原理**：`AO = SMA(hl2, 5) − SMA(hl2, 34)`。
- **输出**：AO 四色柱（配色同上）。

### 4.3 `coppock_curve.pine` — Coppock Curve

- **用途**：为长周期（尤其月线）设计的底部抄底指标，平滑后穿越零轴视为长期买点。
- **原理**：`Coppock = WMA(ROC(close, 14) + ROC(close, 11), 10)`。
- **输出**：单线（> 0 绿、≤ 0 红）+ 0 轴虚线。

### 4.4 `ergodic_oscillator.pine` — Ergodic Oscillator (EOSC)

- **用途**：TSI 的“多级 EMA”版本，把动量指标平滑到很慢，适合做中长周期拐点。
- **原理**：`mom = change(src)`；`ema1 = EMA(mom, 8)`、`ema1Abs = EMA(|mom|, 8)`；
  `ema2 = EMA(ema1, 5)`、`ema2Abs = EMA(|ema1Abs|, 5)`；`tsi = 100 × ema2/ema2Abs`；
  `signal = EMA(tsi, 5)`；`ergodic = EMA(signal, 5)`；`ergodicSignal = EMA(ergodic, 14)`。
- **输出**：ergodic 蓝线 + ergodicSignal 红线 + ±20 虚线。

### 4.5 `percentage_price_oscillator.pine` — Percentage Price Oscillator (PPO)

- **用途**：MACD 的百分比归一化版本，便于跨品种/跨价位比较动量。
- **原理**：`PPO = 100 × (EMA(12) − EMA(26)) / EMA(26)`，`signal = EMA(PPO, 9)`，
  `hist = PPO − signal`。
- **输出**：PPO 蓝线 + signal 橙线 + 粉色 hist 柱 + 可选 ribbon 填充（PPO > signal 绿）。

### 4.6 `price_momentum_oscillator.pine` — Price Momentum Oscillator (PMO)

- **用途**：把 ROC 做两级 EMA 平滑并放大 10 倍，得到平滑的动量振荡器。
- **原理**：`rocMA` 用 `α = 2/35` 的 EMA 平滑 `ROC(src,1)`；`pmo` 用 `α = 2/20` 再平滑并 ×10；
  `signal` 用 `α = 2/10` 平滑。
- **输出**：PMO 蓝线 + signal 红线 + 0 轴虚线 + 可选 ribbon 填充。

### 4.7 `pring_special_k.pine` — Pring Special K

- **用途**：Martin Pring 的 KST 变体，把 12 组不同周期的 ROC 平滑值加权求和，覆盖短到超长周期。
- **原理**（全部常数写死在代码里）：
  `SMA(ROC10,10) + 2·SMA(ROC10,15) + 3·SMA(ROC10,20) + 4·SMA(ROC15,30) + SMA(ROC50,40)
  + 2·SMA(ROC65,65) + 3·SMA(ROC75,75) + 4·SMA(ROC100,100) + SMA(ROC130,195)
  + 2·SMA(ROC130,265) + 3·SMA(ROC130,390) + 4·SMA(ROC195,530)`。
- **输出**：单线（≥ 0 绿、< 0 红）+ 0 轴虚线。

### 4.8 `repulse.pine` — Repulse

- **用途**：用“价格相对近期极值”的多空力量差衡量动能反转。
- **原理**：`bullPower = 100 × (3·close − 2·lowest(5) − open[4]) / close`，
  `bearPower = 100 × (open[4] + 2·highest(5) − 3·close) / close`，
  `repulse = EMA(bullPower, 25) − EMA(bearPower, 25)`。
- **输出**：单线（比前值高绿、否则红）+ 0 轴虚线。

### 4.9 `smoothed_rate_of_change.pine` — Smoothed Rate of Change (S-RoC)

- **用途**：先均线平滑再算变化率，滤掉短期噪声的 ROC。
- **原理**：`ma = EMA(src, 13)`，`S-RoC = 100 × (ma − ma[21]) / ma[21]`。
- **输出**：单线（≥ 0 绿、< 0 红）+ 0 轴虚线。

### 4.10 `tick_line_momentum_oscillator.pine` — Tick Line Momentum Oscillator (TLMO)

- **用途**：把“价格相对均线的方向累计”做成动量振荡器，衡量上涨/下跌笔数的动量。
- **原理**：`cumo = +1 / −1 / 0`（价格高于 / 低于 `MA[1]` / 持平）的累计序列（`cum`），
  取其 `ROC(5)`，再用 EMA/SMA/WMA/VWMA（默认 EMA 5）平滑。
- **输出**：单线（> 5 绿、< −5 红、其余紫）+ ±5 虚线与 0 实线。

### 4.11 `trix.pine` — TRIX

- **用途**：三重指数平滑均线的变化率，过滤掉短中期波动，只留长期动量。
- **原理**：`triple = EMA(EMA(EMA(src, 14), 14), 14)`，`TRIX = mult × ROC(triple, 1)`
  （`mult` 可放大数值）；`signal = SMA/EMA(TRIX, 9)`，`hist = TRIX − signal`。
- **输出**：TRIX 线 + signal 线（可选 ribbon）+ 可选 hist 四色柱 + 0 轴 + 交叉圆点 +
  零轴上下背景色。

### 4.12 `tsi_true_strength_index.pine` — True Strength Index (TSI, v4)

- **用途**：用双重 EMA 平滑后的动量与绝对动量之比，衡量趋势强度，兼顾平滑与灵敏。
- **原理**：`num = EMA(EMA(change(src), 25), 13)`，`den = EMA(EMA(|change(src)|, 25), 13)`，
  `TSI = 100 × num / den`，`signal = EMA(TSI, 13)`。
- **输出**：TSI/signal 两条线（ribbon 填充）+ 可选 hist 四色柱 + ±40 与 ±75 参考线 + 中性区 +
  突破区高亮 + 零轴上下背景色 + 交叉圆点。

---

## 五、成交量类

这一组把成交量（或资金流）纳入计算，判断“价格变化是否有量能支撑”。

### 5.1 `mfi_money_flow_index.pine` — Money Flow Index (MFI, v4)

- **用途**：成交量加权版的 RSI，衡量资金流入/流出的强弱。
- **原理**：直接调用平台内置 `mfi(hlc3, 14)`（用典型价 × 成交量得到资金流，按涨跌分组求比值，
  映射到 0–100），`signal = EMA(MFI, 9)`。
- **输出**：MFI/signal 两条线（ribbon 填充）+ 可选 hist 柱 + 80/50/20 参考线 + 中性区 +
  突破区高亮 + 中轴上下背景色 + 交叉圆点。

### 5.2 `percentage_volume_oscillator.pine` — Percentage Volume Oscillator (PVO)

- **用途**：成交量的 MACD 百分比版，看量能扩张/收缩。
- **原理**：`PVO = 100 × (EMA(volume, 12) − EMA(volume, 26)) / EMA(volume, 26)`，
  `signal = EMA(PVO, 9)`，`hist = PVO − signal`。
- **输出**：四色 hist 柱 + PVO/signal 线 + ribbon 填充 + 0 轴虚线 + 交叉圆点 + 零轴背景色。

### 5.3 `v_macd.pine` — V-MACD

- **用途**：把 MACD 直接作用在成交量上，衡量量能动量的动量。
- **原理**：`[vmacd, signal, hist] = macd(volume, 12, 26, 9)`（内置函数）。
- **输出**：vmacd/signal 线 + 可选四色 hist 柱 + 0 轴 + 交叉圆点 + 零轴背景色。

### 5.4 `vw_macd.pine` — VW-MACD

- **用途**：用成交量加权均线（VWMA）替代 EMA 构造 MACD，使权重偏向放量 K 线。
- **原理**：`vwmacd = vwma(close, 12) − vwma(close, 26)`，`signal = EMA(vwmacd, 9)`，
  `hist = vwmacd − signal`。
- **输出**：同 5.3（四色 hist 柱 + 两条线 + cross 圆点 + 零轴背景色）。

---

## 六、Ehlers 数字信号处理（DSP）家族

John Ehlers 用数字滤波器（高通、低通、带通、Hilbert 变换）把价格分解成“趋势 + 周期”两部分，
这一组都是滤波后的循环分量振荡器。公式较长，参数含义统一为：
`length`/`Cutoff` = 滤波器截止周期，`poles` = 极点数量（2 或 3，越大越平滑），`alpha` = 平滑系数。

### 6.1 `ehlers_cyber_cycle.pine` — Ehlers Cyber Cycle (ECC)

- **用途**：提取价格中的短周期循环分量（约 8–30 根 K 线），比普通振荡器更“波形化”。
- **原理**：先 4 点加权平滑 `smooth = (src + 2C₁ + 2C₂ + C₃)/6`，再用二阶 IIR 带通递推：
  `ECC = (1 − α/2)²·(smooth − 2·smooth[1] + smooth[2]) + 2(1 − α)·ECC[1] − (1 − α)²·ECC[2]`，
  `α` 默认 0.7。
- **输出**：ECC 主线 + ECC[1] 触发线（两者可视为快慢线）+ 可选填充（ECC > ECC[1] 绿，否则红）。

### 6.2 `ehlers_stochastic_cyber_cycle.pine` — Ehlers Stochastic Cyber Cycle (ESCC)

- **用途**：把 Cyber Cycle 再做随机归一化，得到 ±1 区间、更易读的循环位置。
- **原理**：先按 6.1 算 `cycle`，然后 `stoch = (cycle − lowest(cycle,n))/(highest − lowest)`，
  用 `(4·stoch + 3·stoch[1] + 2·stoch[2] + stoch[3])/10` 加权平滑，
  `escc = 2 × (该值 − 0.5)` 映射到约 ±1；触发线 `esccPrev = 0.96 × (escc[1] + 0.02)`。
- **输出**：ESCC 蓝线 + 粉色触发线 + 0 轴虚线 + 可选填充。

### 6.3 `ehlers_decycler_oscillator.pine` — Ehlers Decycler Oscillator

- **用途**：用两个不同截止周期的高通滤波器对消，得到“去周期趋势 + 短周期振荡”的双线结构。
- **原理**：高通滤波 `hp`：`α = (cosθ + sinθ − 1)/cosθ`（`θ = 2π/(mult·period·√2)`），
  `hp := (1 − α/2)²·(src − 2src[1] + src[2]) + 2(1 − α)·hp[1] − (1 − α)²·hp[2]`；
  去周期趋势 `decycler = src − hp`；振荡值
  `decosc = 100 × k × HP(decycler, period, mult=0.5) / src`（对 decycler 再用 `mult=0.5` 的高通）。
  脚本同时算两组参数（period=125 / k=1 与 period=100 / k=1.2）形成快慢双线。
- **输出**：两条曲线（快/慢 decycler 振荡）+ 0 轴虚线 + 交叉圆点 + 可选零轴上下背景色 +
  可选 ribbon 填充。

### 6.4 `ehlers_fisherized_deviation_scaled_oscillator.pine` — EFDSO

- **用途**：把带通滤波后的偏差用标准差标准化再做 Fisher 变换，使输出近似正态，阈值可解释为 σ。
- **原理**：`zeros = src − src[2]`，取 `(zeros + zeros[1])/2` → Super Smoother（20，2/3 极）→
  除以 `stdev(·, 40)` 做标准化 → `EFDSO = 0.5·ln((1 + x/2)/(1 − x/2))`（仅当 `|x| < 2`，否则保持前值）。
- **输出**：EFDSO 曲线 + ±2（约 2σ）与 ±4 虚线 + 中性区 + 突破区高亮。

### 6.5 `ehlers_roofing_filter.pine` — Ehlers Roofing Filter (ERF)

- **用途**：“屋顶”形状的带通：先去长周期漂移（高通），再去高频噪声（Super Smoother），
  留下中间频段的循环分量（Ehlers 推荐的通用前置滤波）。
- **原理**：高通（48）→ `(hp + hp[1])/2` → 2 极或 3 极 Super Smoother（10）。
- **输出**：单线（≥ 0 绿、< 0 红）+ 0 轴虚线。

### 6.6 `ehlers_stochastic.pine` — Ehlers Stochastic (EStoch)

- **用途**：在 Roofing Filter 输出上做随机归一化，得到 0–1 的“滤波后随机指标”。
- **原理**：`erf = roofingFilter(...)`，`stoch = (erf − LL)/(HH − LL)`（窗口 20），
  再对 `(stoch + stoch[1])/2` 做 Super Smoother（10）。
- **输出**：0–1 曲线（> 0.8 绿、< 0.2 红、其余紫）+ 0.8/0.5/0.2 参考线。

### 6.7 `ehlers_smoothed_adaptive_momentum.pine` — ESAM

- **用途**：用 Hilbert 变换估测主导周期，让动量回看长度自适应，再用 Super Smoother 平滑。
- **原理**：`cycle` 用 6.1 的带通算法；用 Hilbert 变换近似
  `0.0962·x + 0.5769·x[2] − 0.5769·x[4] − 0.0962·x[6]` 求 I/Q 分量，
  推 `deltaPhase`（限幅 0.1–1.1）→ `medianDelta`（5 窗口中位数）→ 主导周期 `dc = 2π/medianDelta + 0.5`
  → `instPeriod`、`lookback`（EMA 平滑）；`mom = src − src[floor(lookback − 1)]`，
  最后用 2/3 极 Super Smoother（Cutoff 默认 8）平滑。
- **输出**：ESAM 曲线（正绿负红）+ 0 轴虚线。

### 6.8 `ehlers_super_passband_filter.pine` — Ehlers Super Passband Filter (ESPF)

- **用途**：只放行快慢两个截止周期之间的频段，并用 RMS 包络判断振荡是否“显著”。
- **原理**：二阶带通
  `espf := (a1 − a2)·src + (a2(1−a1) − a1(1−a2))·src[1] + ((1−a1)+(1−a2))·espf[1] − (1−a1)(1−a2)·espf[2]`，
  `a1 = 5/40`、`a2 = 5/60`；`rms = √(Σespf²/50)`（50 根硬编码）。
- **输出**：ESPF 曲线（突破 ±RMS 绿/红，否则紫）+ `+RMS` / `−RMS` 包络线 + 0 轴虚线。

### 6.9 `ehlers_triple_delay_line_detrender.pine` — TDLD

- **用途**：用三段延迟线构造去趋势器，再做两次“动量 + EMA 组合”平滑，得到快慢双线振荡器。
- **原理**：`tmp1 = src + 0.088·tmp1[6]`；`tmp2 = tmp1 − tmp1[6] + 1.2·tmp2[6] − 0.7·tmp2[12]`；
  `detrender = tmp2[12] − 2·tmp2[6] + tmp2`；再两次 `_moef` 递归
  （系数 `0.13785 / 0.0007 / 0.13785 / 1.2103 / −0.4867`）分别得到 TDLD 和 signal。
- **输出**：TDLD 线 + signal 线 + 可选 hist 四色柱 + 交叉圆点 + 零轴背景色 + 可选 ribbon 填充。

### 6.10 `recursive_median_oscillator.pine` — Recursive Median Oscillator (RMO)

- **用途**：中值滤波 + 递归低通/高通，得到抗脉冲噪声的平滑零轴振荡器。
- **原理**：`median(src, 5)`（`percentile_nearest_rank` 取 50 分位）→ 一阶低通（12）
  `rm := α₁·median + (1 − α₁)·rm[1]` → Ehlers 二阶高通（30，`α₂` 用 `2π/(√2·30)`）→ `rmo`。
- **输出**：单线（≥ 0 绿、< 0 红）+ 0 轴虚线。

---

## 七、趋势强度与方向

这一组不追求超买超卖，而是回答“现在有没有趋势、趋势多强、方向如何”。

### 7.1 `accumulative_swing_index.pine` — Accumulative Swing Index (ASI)

- **用途**：Wilder 的 Swing Index 累积曲线，用 K 线实体、影线与跳空综合衡量趋势的累计推进。
- **原理**：`K = max(|H − C₁|, |L − C₁|)`，`R` 按三种情形组合 `H−C₁`、`L−C₁`、`H−L`、`C₁−O₁`；
  `SI = 50 × (C − C₁ + 0.5(C − O) + 0.25(C₁ − O₁)) / R × K / limit`（`limit` 默认 10000）；
  `ASI = cum(SI)`，即从数据起点一直累加（注意：绝对值取决于图表起始位置）。
- **输出**：ASI 曲线（比前值高绿、否则红），无参考线。

### 7.2 `range_action_verification_index.pine` — RAVI

- **用途**：用快慢均线的百分比距离（取绝对值）判断趋势是否“有效”，常用于过滤震荡。
- **原理**：`RAVI = |100 × (fastMA − slowMA) / slowMA|`，默认 SMA 7 / SMA 65（可选 EMA/WMA/VWMA）。
- **输出**：单线（比 `comparisonLookback` 根前高 → 绿，否则红）。

### 7.3 `stiffness_indicator.pine` — Stiffness Indicator

- **用途**：衡量价格“贴着上轨走”的刚性程度，用于识别强趋势与其衰竭。
- **原理**：`bound = SMA(close, 100) − 0.2·stdev(close, 100)`；统计最近 60 根中 `close > bound`
  的比例 ×100，再用 `EMA(·, 3)` 平滑。
- **输出**：柱状图 + 90 阈值红线 + 可选阈值上方的绿/红背景。

### 7.4 `total_power_indicator.pine` — Total Power Indicator (TPI)

- **用途**：统计多空力量“占优的 K 线比例”，把 Elder Bull/Bear Power 变成 0–100 的占比指标。
- **原理**：`bullPower = high − EMA(close, 10)`、`bearPower = low − EMA(close, 10)`；
  统计最近 45 根中各自 > 0 / < 0 的根数，`bull = 100·bullCount/45`、`bear = 100·bearCount/45`、
  `total = 100·|bullCount − bearCount|/45`。
- **输出**：Bull 绿线 + Bear 红线 + Total 蓝线 + 75 入场线与 50 中轴虚线 + 可选 ribbon 填充。

### 7.5 `trend_continuation_factor.pine` — Trend Continuation Factor (TCF)

- **用途**：衡量趋势“连续推进”的能力：同向推进累加，反向变化清零。
- **原理**：`plusCF` 连续累加正向变化（遇到非正变化清零），`minusCF` 对称；
  `+TCF = Σ(plusChange − minusCF, 35)`、`−TCF = Σ(minusChange − plusCF, 35)`。
- **输出**：+TCF 绿线、−TCF 红线 + 0 实线。

### 7.6 `trend_detection_index.pine` — Trend Detection Index (TDI)

- **用途**：区分“趋势市”与“震荡市”，并给出方向。
- **原理**：`mom = change(src, 20)`；
  `TDI = |Σmom(n)| − (Σ|mom|(2n) − Σ|mom|(n))`（趋势越单边、TDI 越大）；
  方向线 `direction = Σmom(n)`。
- **输出**：TDI 红线（强度）+ Direction 绿线（正负=方向）+ 0 轴虚线。

### 7.7 `trendscore.pine` — Trendscore

- **用途**：最直观的趋势计数：和过去 10 个不同回看期的价格逐一比较打分。
- **原理**：对 `i = 11…20`（默认 `from = 11, to = 20`）逐一比较，`src ≥ src[i]` 记 +1、
  否则 −1，求和 → 取值 −10…+10。
- **输出**：单线（0 以上为多头、以下为空头），无参考线。

### 7.8 `trend_trigger_factor.pine` — Trend Trigger Factor (TTF)

- **用途**：用“买方推进幅度 vs 卖方推进幅度”的比值判断趋势触发器。
- **原理**：`buyPower = HH(15) − LL[30]`、`sellPower = HH[30] − LL(15)`，
  `TTF = 200 × (buyPower − sellPower) / (buyPower + sellPower)`，范围约 ±200。
- **输出**：曲线（> 100 绿、< −100 红、其余橙）+ ±100 触发线 + 中性区 + 突破区高亮。

### 7.9 `vertical_horizontal_filter.pine` — Vertical Horizontal Filter (VHF)

- **用途**：趋势/横盘的经典判别器：价格净位移与路径长度之比。
- **原理**：`VHF = |highest(src, 28) − lowest(src, 28)| / Σ|change(src)|`。趋势单边时分母小、
  分子大 → VHF 高；横盘震荡时接近 0。
- **输出**：单线（紫色），无参考线（常用经验阈值 0.3/0.5）。

### 7.10 `random_walk_index.pine` — Random Walk Index (RWI)

- **用途**：把实际位移与“随机游走的期望位移”比较，判断趋势是否强于随机。
- **原理**：`RWI_high = (high − low[14]) / (atr(14) × √14)`，
  `RWI_low = (high[14] − low) / (atr(14) × √14)`。
- **输出**：两条线（High 紫、Low 红），无参考线；> 1 通常意味着趋势强于随机波动。

---

## 八、波动率与区间

### 8.1 `mass_index.pine` — Mass Index

- **用途**：通过高低价区间的扩张/收缩识别“区间突然放大后的反转”（Dorsey）。
- **原理**：`MI = Σ_{25}( EMA(high − low, 9) / EMA(EMA(high − low, 9), 9) )`。
  区间持续放大时比值上升，MI 冲高后回落常预示反转。
- **输出**：MI 曲线（> 26.5 橙色、否则蓝）+ 27 突破线 + 26.5 确认线 + 突破区填充。

### 8.2 `true_range_double.pine` — True Range Double (TRD)

- **用途**：跨 2 根 K 线的真实波幅（比 ATR 的单根 TR 更宽松），常作波动率基线。
- **原理**：`TRD = max(high, high[1], close[2]) − min(low, low[1], close[2])`。
- **输出**：单线（价格量纲，未归一化），无参数、无参考线。

---

## 附录 A：输出刻度速查

| 输出刻度 | 脚本 |
|---|---|
| 0–100（超买超卖带） | CCI（绝对值可超过 100）、RSI、Rapid RSI、RMI、ARSI、SARSI、DMI、Jurik RSX/RSX、IMI、RVI、TII、PCI、CMO（±100）、Stoch、STC、MFI、TSI（±75 内）、TPI、Mass Index（可超 100） |
| 0–1 | DeMarker、Laguerre RSI、Laguerre FE、Ehlers Stochastic（0–1） |
| −1…+1 附近 | Ehlers Stochastic Cyber Cycle |
| 围绕 0（无界） | CGO、DI、KRI、FOSC、ROSC、AO、AC、Coppock、PPO、PMO、PSK、Repulse、S-RoC、TRIX、TLMO、Ergodic、NRVGI、SMI（±100）、TTF（±200）、Trendscore（±10）、+TCF/−TCF、Didi（围绕 1）、Ehlers 系列、RMO、BSI/ASI（累计）、RWI、VHF、RAVI、PGO（ATR 单位）、Fisher 类（Rocket RSI、EFDSO）、TDLD、Gator（绝对值柱） |
| 价格量纲（非振荡） | ASI（累计值）、TRD |

## 附录 B：重复实现、瑕疵与注意事项

1. **完全重复**：`jurik_rsx.pine` 与 `rsx.pine` 代码一致（标题都是 `Jurik RSX`）；
   `disparity_index.pine` 与 `kairi_relative_index.pine` 公式一致（仅配色/线型不同）。
2. **直接调用平台内置函数**：`rsi_relative_strength_index.pine`（`rsi()`）、
   `mfi_money_flow_index.pine`（`mfi()`）、`stochastic.pine`（`stoch()`）、
   `v_macd.pine`（`macd()`）。这些脚本的“增值”主要在源选择与可视化。
3. **`intraday_momentum_index.pine` 的 `src` 输入未被使用**（计算固定用 `close`/`open`）。
4. **`asymmetrical_rsi.pine`**：`upAlpha = 1/upCount`，若窗口内没有任何上涨 bar（`upCount = 0`）
   会出现除以零，极端单边行情下读数可能异常。
5. **`rainbow_oscillator.pine`**：`getMA` 内 `if` 块缩进混用制表符和空格，若编辑器报缩进错误，
   统一缩进即可。
6. **`trendscore.pine`**：局部变量名用了 `sum`，与内置函数同名（Pine v3 可运行，升版需改名）。
7. **`cci_commodity_channel_index.pine`** 的 `price volume trend` 源写成 `cum(roc(src,1)*volume)`，
   其中 `src` 是函数内局部变量，与标准 PVT（`cum(change(close)/close[1]*volume)`，见
   `rsi_relative_strength_index.pine`）不完全等价。
8. **`pring_special_k.pine`** 的 12 组周期/权重全部硬编码，无法通过输入调整。
9. **`accumulative_swing_index.pine`** 用 `cum()` 从图表第一根 K 线累加，数值随图表起始位置变化，
   跨品种/跨区间不可直接比较。
10. **`true_range_double.pine`** 输出是价格量纲，不同品种不可直接比较。
11. **`mass_index.pine` 的 27 / 26.5 阈值**和 **`stiffness_indicator.pine` 的 90 阈值** 来自原始文献，
    在不同市场需要重新校准。
12. **`ehlers_cyber_cycle.pine` 的 `length` 输入未被使用**（Cyber Cycle 的周期由 `alpha` 间接控制），
    调整该参数不会改变输出；`ehlers_stochastic_cyber_cycle.pine` 的 `length` 才用于随机归一化窗口。

## 附录 C：使用与移植提示

- **运行**：所有脚本都是 `indicator()`（副图），且均已迁移到 v6。下表的 `study()` →
  `indicator()`、`input(title=..., type=...)` → `input.int()` 等对照保留为旧写法的参考。
- **升级到 v5/v6 时的常见改动**：
  - 序列函数加命名空间：`sma → ta.sma`、`ema → ta.ema`、`rsi → ta.rsi`、`atr → ta.atr`、
    `stdev → ta.stdev`、`linreg → ta.linreg`、`change → ta.change`、`roc → ta.roc`、
    `cum → ta.cum`、`highest/lowest → ta.highest/ta.lowest`、`swma → ta.swma`、
    `percentile_nearest_rank → ta.percentile_nearest_rank`、`cross* → ta.cross*`、`fixnan → ta.fixnan`；
  - `iff(c, a, b)` 改为三元 `c ? a : b`；
  - `transp=` 参数改为 `color.new(color, transp)`；
  - 颜色常量改为 `color.green` 等，`hline`/`plot` 参数名有调整；
  - v5 `ta.` 函数默认带 `series` 语义，`nz()`/`na` 逻辑基本不变。
- **参数调优顺序建议**：先定用途（超买超卖 / 动量 / 趋势过滤）→ 选对应家族 →
  只调 `Length`（灵敏度）与阈值（OB/OS）→ 最后再开 `highlight*` / `applyFilling` 等可视化开关。
- **当作信号用**：可直接把 `cross*`/阈值穿越写成 `alertcondition`（参考 CCI、SARSI、STC 三个脚本的
  写法），或把公式移植到 `strategy()` 里做回测；注意本目录所有指标都是**单指标**，
  没有仓位管理或止损逻辑（止损类脚本在 `../trailing_stops/`）。
