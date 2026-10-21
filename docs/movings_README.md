# Movings — 移动平均与平滑滤波器

本目录收录 **70 个** Pine Script 脚本，全部 `overlay = true`，直接画在价格图上。

它们回答的是同一个问题：**把 OHLC 序列变成一条平滑曲线，这条曲线怎么算、滞后多少、在什么行情下该快该慢？**
换句话说，这里没有振荡器、没有通道、没有形态识别——**每个脚本都只是一条（或几条）价格的低通滤波器**，区别只在"滤波器系数怎么定"。

三个例外／注意点先说清楚：

- `parabolic_sar.pine` 不是均线，是抛物线转向点（跟踪止损），但因为输出是一条跟随价格的曲线，仍收录在此；
- `ehlers_mesa_adaptive_moving_averages.pine` 输出 **2 条**线（MAMA & FAMA），`multiple_hull_moving_average.pine` 输出 **3 条**线（HMA 20/50/100），`corrected_moving_average.pine` 输出 **2 条**线（CMA + SMA）外加一条差值填充带；
- 其余 67 个脚本都只 `plot()` 一条主线。

**没有任何一个脚本是 `strategy`**：不能在策略测试器里回测，不产生买卖指令。少数脚本提供 `alertcondition()`（颜色翻转告警），详见下文。

---

## 一、先读这一节：70 个脚本共用的骨架

### 1.1 代码骨架

几乎所有脚本都是同一个模板，只是中间那段计算不同：

```pine
study("XXX", shorttitle="XXX", overlay=true)     // v3/v4；v6 用 indicator(...)

length = input(title="Length", type=integer, defval=14)
src    = input(title="Source", type=source, defval=close)   // 少数默认 hl2
highlight = input(...)                                       // 是否按方向着色

xxx = ...                                                   // ← 唯一有差异的部分

xxxColor = highlight ? (xxx > xxx[1] ? green : red) : #6d1e7f
plot(xxx, title="XXX", linewidth=2, color=xxxColor, transp=0)
```

所以对任何一个脚本，"做什么 + 原理"就等于"中间那段计算"，"输出"就等于"一条线 + 两种配色"。

### 1.2 三种输出／配色约定

| 形态 | 触发条件 | 输出 | 涉及脚本 |
| --- | --- | --- | --- |
| **单色线**（最朴素） | 没有 `Highlight` 输入 | 一条固定颜色 2px 线（多为 `#6d1e7f` 紫） | 基础实现类：SMA/EMA/WMA/LWMA/WMMA、MGM/MHM/RMS、HMA、ALMA、LSMA、PSAR 等 |
| **方向双色线**（最常见） | `Highlight = true`（多数默认 true） | `value > value[1]` → `green`（上行），否则 `red`（下行） | 绝大多数脚本 |
| **v6 全家桶** | 7 个重写版脚本 | 线 + 可选 `barcolor` + 可选 `bgcolor` + 告警（6 个脚本各 3 个变色告警，`corrected_*` 是 2 个交叉告警） | `corrected_*`、`fibonacci_*`、`henderson_*`、`inverse_distance_*`、`quick_*`、`sharp_modified_*`、`sine_*` |

v6 系列的配色是另一套：上行 `#09b71e`（绿）、下行 `#e91e63`（粉），K 线 `color.new(..., 50)`，背景 `color.new(..., 88)`，关闭高亮时线色 `#512da8`。

### 1.3 有 `alertcondition()` 的 8 个脚本

只有这 8 个能在 TradingView 里挂告警（其余脚本纯画线）：

| 文件 | 告警条件 |
| --- | --- |
| `adaptive_laguerre_filter.pine` | 颜色翻转（且等待 K 线确认） |
| `corrected_moving_average.pine` | `sma` 上穿 / 下穿 `cma`（Long / Short Signal） |
| `fibonacci_weighted_moving_average.pine` | 变绿 / 变粉 / 任意变色 |
| `henderson_weighted_moving_average.pine` | 同上 |
| `inverse_distance_weighted_moving_average.pine` | 同上 |
| `quick_moving_average.pine` | 同上 |
| `sharp_modified_moving_average.pine` | 同上 |
| `sine_weighted_moving_average.pine` | 同上 |

### 1.4 Pine 版本分布

| 版本 | 数量 | 特征 |
| --- | --- | --- |
| v6 | 70 | 全部脚本已迁移到 v6：`indicator()`；`ta.*` / `math.*` 命名空间；`input.int/input.source/input.bool`；`const string` 常量与 `group` 分组；`color.new()` 取代 `transp=` |

实现风格的差异（单色线 / 方向双色线 / 全家桶）见上文 1.1–1.3，与 Pine 版本无关。

下面的替换对照表用于理解旧脚本的原始写法（v3/v4），本仓库当前已无 `study()` 用法：

同一个概念在不同版本里的写法要留意：

| 概念 | v3 | v4 | v6 |
| --- | --- | --- | --- |
| 声明 | `study("X", "Y", overlay=true)` | `study("X", "Y", overlay=true)` | `indicator('X', shorttitle='Y', overlay=true)` |
| 输入 | `input(title="L", type=integer, defval=14)` | `input(title="L", type=input.integer, defval=14)` | `input.int(14, title='L')` |
| 中位数 | `percentile_nearest_rank(src, L, 50)` | 同 | 同 |
| 峰值/谷值 | `highest(L)` / `lowest(L)` | 同 | `ta.highest(high, L)` |

> v3/v4 里 `highest(source, length)` 省略 `source` 时默认取 `high`，`lowest` 默认取 `low`——`middle_high_low_moving_average.pine` 和 `fractal_adaptive_moving_average.pine` 都依赖这个默认值。

---

## 二、按算法族分类（总表）

70 个脚本按"系数如何确定"分成 7 族。想看某个脚本的详解，按文件名字母序在本章之后找到对应小节即可（每族内部按文件名排序）。

| # | 文件 | 图表名（简称） | 版本 | 默认参数 | 输出 |
| --- | --- | --- | --- | --- | --- |
| **1** | `exponential_moving_average.pine` | Exponential MA (EMA) | 6 | `Length=14` | 单色 `#6d1e7f` |
| **1** | `linear_weighted_moving_average.pine` | Linear Weighted MA (LWMA) | 6 | `Length=14` | 单色 `#e69138` |
| **1** | `simple_moving_average.pine` | Simple MA (SMA) | 6 | `Length=14` | 单色 `#6d1e7f` |
| **1** | `weighted_moving_average.pine` | Weighted MA (WMA) | 6 | `Length=14` | 单色 `#6d1e7f` |
| **1** | `wilder_modified_moving_average.pine` | Wilder Modified MA (WMMA) | 6 | `Length=14` | 单色 `#6d1e7f` |
| **2** | `arnaud_legoux_moving_average.pine` | Arnaud Legoux MA (ALMA) | 6 | `Length=9`、`Offset=0.85`、`Sigma=6` | 单色 `#e69138` |
| **2** | `double_weighted_moving_average.pine` | Double Weighted MA (DWMA) | 6 | `Length=10` | 双色 |
| **2** | `elastic_volume_weighted_moving_average.pine` | Elastic Volume Weighted MA (EVWMA) | 6 | `Length=14` | 双色 |
| **2** | `farey_sequence_weighted_moving_average.pine` | Farey Sequence Weighted MA (FSWMA) | 6 | `Order=5` | 双色 |
| **2** | `fibonacci_weighted_moving_average.pine` | Fibonacci Weighted MA (FWMA) | 6 | `Length=14` | 双色 + K 线 + 3 告警 |
| **2** | `henderson_weighted_moving_average.pine` | Henderson Weighted MA (HWMA) | 6 | `Length=7` | 双色 + K 线 + 3 告警 |
| **2** | `least_squares_moving_average.pine` | Least Squares MA (LSMA) | 6 | `Length=14`、`Offset=0` | 单色 `#6d1e7f` |
| **2** | `middle_high_low_moving_average.pine` | Middle-High-Low MA (MHLMA) | 6 | `Length=14`、`Range=10`、`EMA` | 双色 |
| **2** | `parabolic_weighted_moving_average.pine` | Parabolic Weighted MA (PWMA) | 6 | `Length=14`、`Power=2` | 双色 |
| **2** | `quick_moving_average.pine` | Quick MA (QMA) | 6 | `Length=14` | 双色 + K 线 + 3 告警 |
| **2** | `self_weighted_moving_average.pine` | Self-Weighted MA (SEWMA) | 6 | `Length=14` | 单色（Highlight 默认 false） |
| **2** | `sine_weighted_moving_average.pine` | Sine Weighted MA (SWMA) | 6 | `Length=14` | 双色 + K 线 + 3 告警 |
| **2** | `spencer_15_point_moving_average.pine` | Spencer 15-Point MA (SPMA 15) | 6 | 仅 `Source` | 双色 |
| **2** | `spencer_21_point_moving_average.pine` | Spencer 21-Point MA (SPMA 21) | 6 | 仅 `Source` | 双色 |
| **2** | `triangular_moving_average.pine` | Triangular MA (TMA) | 6 | `Length=20` | 双色 |
| **3** | `distance_coefficient_filter.pine` | Ehlers Distance Coefficient Filter (EDCF) | 6 | `Length=14`、`src=hl2` | 单色 `#6d1e7f` |
| **3** | `distance_weighted_moving_average.pine` | Distance Weighted MA (DWMA) | 6 | `Length=14` | 双色 |
| **3** | `ehlers_distance_coefficient_filter.pine` | Ehlers Distance Coefficient Filter (EDCF) | 6 | `Length=14`、`src=hl2` | 单色 `#6d1e7f` |
| **3** | `hampel_filter.pine` | Hampel Filter (HF) | 6 | `Length=14`、`Scaling=3` | 单色 `#6d1e7f` |
| **3** | `inverse_distance_weighted_moving_average.pine` | Inverse Distance Weighted MA (IDWMA) | 6 | `Length=14` | 双色 + K 线 + 3 告警 |
| **3** | `moving_geometric_mean.pine` | Moving Geometric Mean (MGM) | 6 | `Length=14` | 单色 `#6d1e7f` |
| **3** | `moving_harmonic_mean.pine` | Moving Harmonic Mean (MHM) | 6 | `Length=14` | 单色 `#6d1e7f` |
| **3** | `moving_median.pine` | Moving Median (MM) | 6 | `Length=14` | 双色 |
| **3** | `moving_quadratic_mean.pine` | Moving Quadratic Mean / RMS | 6 | `Length=14` | 单色 `#6d1e7f` |
| **3** | `recursive_median_filter.pine` | Recursive Median Filter (RMF) | 6 | `Median=5`、`Lowpass=12` | 双色 |
| **3** | `recursive_median_oscillator.pine` | Recursive Median Filter (RMF) | 6 | `Median=5`、`Lowpass=12` | 单色 `#6d1e7f` |
| **3** | `trimean.pine` | Trimean | 6 | `Length=14` | 双色 |
| **4** | `ahrens_moving_average.pine` | Ahrens MA (AHMA) | 6 | `Length=9` | 双色 |
| **4** | `alpha_decreasing_exponential_moving_average.pine` | Alpha-Decreasing EMA (ADEMA) | 6 | 无参数 | 单色 `orange` |
| **4** | `corrected_moving_average.pine` | Corrected MA (CMA) | 6 | `Length=35`、`Highlight=false` | 双线（CMA + SMA）+ 可选填充 + 2 告警 |
| **4** | `double_exponential_moving_average.pine` | Double EMA (DEMA) | 6 | `Length=14` | 单色 `#6d1e7f` |
| **4** | `holt_exponential_moving_average.pine` | Holt EMA (HEMA) | 6 | `Alpha=20`、`Gamma=20` | 双色 |
| **4** | `hull_moving_average.pine` | Hull MA (HMA) | 6 | `Length=14` | 单色 `#6d1e7f` |
| **4** | `infinite_impulse_response_filter.pine` | IIR Filter (IIRF) | 6 | `Length=14` | 双色 |
| **4** | `jurik_moving_average.pine` | Jurik MA (JMA) | 6 | `Length=7`、`Phase=50`、`Power=2` | 双色 |
| **4** | `laguerre_filter.pine` | Laguerre Filter (LF) | 6 | `Alpha=0.7` | 双色 |
| **4** | `leo_moving_average.pine` | Leo MA (LMA) | 6 | `Length=14` | 双色 |
| **4** | `mcginley_dynamic.pine` | McGinley Dynamic (MD) | 6 | `Length=14`、`K=0.6` | 单色 `#6d1e7f` |
| **4** | `mcnicholl_moving_average.pine` | McNicholl MA (MNMA) | 6 | `Length=20` | 双色 |
| **4** | `multiple_hull_moving_average.pine` | Multiple Hull MA | 6 | `20 / 50 / 100` | 三条线 `#85200c`/`#1155cc`/`#f1c232` |
| **4** | `pentuple_exponential_moving_average.pine` | Pentuple EMA (PEMA) | 6 | `Length=20` | 双色 |
| **4** | `quadruple_exponential_moving_average.pine` | Quadruple EMA (QEMA) | 6 | `Length=20` | 双色 |
| **4** | `recursive_moving_trend_average.pine` | Recursive Moving Trend Average (RMTA) | 6 | `Length=14` | 双色 |
| **4** | `regularized_exponential_moving_average.pine` | Regularized EMA (REMA) | 6 | `Length=14`、`Lambda=0.5` | 双色 |
| **4** | `sharp_modified_moving_average.pine` | Sharp Modified MA (SHMMA) | 6 | `Length=14` | 双色 + K 线 + 3 告警 |
| **4** | `t3_moving_average.pine` | T3 MA (T3) | 6 | `Length=5`、`Factor=0.7` | 双色 |
| **4** | `zero_lag_exponential_moving_average.pine` | Zero Lag EMA (ZLEMA) | 6 | `Length=14` | 双色 |
| **5** | `adaptive_laguerre_filter.pine` | Adaptive Laguerre Filter (ALF) | 6 | `Length=14`、`Median=5`、`Await=true` | 双色 + 1 告警 |
| **5** | `adaptive_rsi_moving_average.pine` | Adaptive RSI MA (ARSIMA) | 6 | `Length=14` | 双色 |
| **5** | `apirine_adaptive_exponential_moving_average.pine` | Adaptive EMA (AEMA) | 6 | `Length=10`、`H/L Lookback=10` | 单色 `#512da8`（Highlight 默认 false） |
| **5** | `apirine_adaptive_moving_average.pine` | Apirine Adaptive MA (AAMA) | 6 | `Length=14`、`Fast=2`、`Slow=30` | 双色 |
| **5** | `bryant_adaptive_moving_average.pine` | Bryant Adaptive MA (BAMA) | 6 | `Length=14`、`Trend=-1`、`MaxLength=100` | 单色 `#6d1e7f`（Highlight 默认 false） |
| **5** | `ehlers_deviation_scaled_moving_average.pine` | Ehlers Deviation-Scaled MA (EDSMA) | 6 | `Length=40`、`SSF=20`、`Poles=2` | 单色 `#6d1e7f` |
| **5** | `ehlers_mesa_adaptive_moving_averages.pine` | MESA Adaptive MAs (MAMA/FAMA) | 6 | `Fast=0.5`、`Slow=0.05`、`src=hl2` | 两条线 `#674ea7`/`#f6b26b` |
| **5** | `fractal_adaptive_moving_average.pine` | Fractal Adaptive MA (FRAMA) | 6 | `Length=14` | 双色 |
| **5** | `kaufman_adaptive_moving_average.pine` | Kaufman Adaptive MA (KAMA) | 6 | `Length=14`、`Fast=2`、`Slow=30` | 双色 |
| **5** | `vidya_variable_index_dynamic_average.pine` | Variable Index Dynamic Average (VIDYA) | 6 | `Length=14` | 双色 |
| **6** | `butterworth_filter.pine` | Butterworth Filter (BF) | 6 | `Length=14`、`Poles=2` | 双色 |
| **6** | `ehlers_leading_indicator.pine` | Ehlers Leading Indicator (ELI) | 6 | `Alpha1=0.25`、`Alpha2=0.33`、`src=hl2` | 单色 `#741b47` |
| **6** | `ehlers_modified_optimum_elliptic_filter.pine` | Modified Optimum Elliptic Filter (MOEF) | 6 | 仅 `Source`（`hl2`） | 单色 `#6d1e7f`（Highlight 默认 false） |
| **6** | `ehlers_optimum_elliptic_filter.pine` | Optimum Elliptic Filter (OEF) | 6 | 仅 `Source`（`hl2`） | 单色 `#6d1e7f`（Highlight 默认 false） |
| **6** | `ehlers_super_smoother_filter.pine` | Ehlers Super Smoother (ESSF) | 6 | `Length=15`、`Poles=2` | 单色 `#741b47` |
| **6** | `finite_impulse_response_filter.pine` | FIR Filter (FIRF) | 6 | 7 个系数 `1/3.5/4.5/3/0.5/-0.5/-1.5` | 双色 |
| **6** | `gaussian_filter.pine` | Gaussian Filter (GF) | 6 | `CyclePeriod=14`、`Poles=4` | 双色 |
| **7** | `parabolic_sar.pine` | Parabolic SAR (PSAR) | 6 | `Start=0.02`、`Increment=0.02`、`Max=0.2` | 十字点（`blue`） |

族名对照：

1. **基础线性滤波** —— 教科书级 MA，作为其它所有脚本的基线；
2. **固定窗口权重核** —— 窗口长度固定，但权重由某个核函数决定（三角、正弦、斐波那契、多项式、回归…）；
3. **稳健统计与距离加权** —— 用中位数/分位数/几何均值/样本间距离等非算术手段抗噪、抗离群；
4. **递归滤波与滞后补偿** —— 无限冲激响应（IIR）结构，核心议题是"减滞后"；
5. **自适应** —— 用市场状态（效率比、波动、分形维数、相位、RSI）动态调 α；
6. **数字信号处理滤波器** —— 直接搬信号处理里的经典滤波器设计（Butterworth、Gaussian、椭圆、超平滑、FIR）；
7. **其它** —— 不是均线的跟随曲线。

---

## 第 1 族 · 基础线性滤波（5 个）

教科书级的移动平均。**这 5 个脚本本身几乎没有"指标价值"**——它们是给其它脚本当基线用的：任何"改进型 MA"的价值，都要和同长度的 SMA / EMA 比才有意义。

### `simple_moving_average.pine` — Simple Moving Average (SMA)

**做什么**：窗口内 L 个收盘价的等权算术平均，业界最原始的平滑线。

**原理**：一个矩形窗 FIR 滤波器。

```
SMA_t = (1/L) · Σ(i=0..L-1) src[t-i]
```

所有样本权重都是 `1/L`。频域上对应 sinc 响应，旁瓣衰减慢，所以对"周期接近 L 的噪声"抑制很差，而且价格一旦滑出窗口会"整根掉出去"（drop-off 效应），在图上表现为折角。

**输出**：一条 `#6d1e7f` 紫色 2px 线。没有 `Highlight` 输入，不变色、不告警。

**参数**：`Length=14`、`Source=close`。

**等效滞后**：`(L-1)/2` 根 bar（L=14 → 6.5 根）。

### `exponential_moving_average.pine` — Exponential Moving Average (EMA)

**做什么**：无限记忆的指数衰减平均，交易软件里最默认的"均线"。

**原理**：一阶 IIR（单极点低通）。

```
α = 2/(L+1)
EMA_t = α·src + (1-α)·EMA[t-1]
```

权重按 `(1-α)^i` 几何衰减，越老的样本权重越小但永不归零，所以没有 SMA 的 drop-off 折角。**平均滞后恰好也是 `(L-1)/2`，与同长度 SMA 相同**，但方差更小（更平滑）。

**输出**：一条 `#6d1e7f` 线，无高亮、无告警。

**参数**：`Length=14`、`Source=close`。

### `weighted_moving_average.pine` — Weighted Moving Average (WMA)

**做什么**：线性加权平均，最新一根权重最大。

**原理**：

```
w_i = L - i        (i = 0 是最新 bar)
WMA_t = Σ w_i·src[t-i] / Σ w_i        其中 Σ w_i = L(L+1)/2
```

权重核是三角形（斜坡），比矩形窗更"看近期"，因此**滞后比同长度 SMA/EMA 小**：平均滞后 `(L-1)/3`（L=14 → 4.33 根），代价是噪声通过率更高。

**输出**：一条 `#6d1e7f` 线。

**参数**：`Length=14`、`Source=close`。

### `linear_weighted_moving_average.pine` — Linear Weighted Moving Average (LWMA)

**做什么**：和上一个**数学上完全等价**（权重 `length, length-1, …, 1`），区别只在于这里是**手写 for 循环**而不是调用内置 `wma()`。

**原理**：

```pine
for i = 0 to length - 1
    weight = length - i
    weightSum += weight
    sum += nz(src[i]) * weight
sum / weightSum
```

**输出**：一条 `#e69138` 橙色线（与 `weighted_moving_average.pine` 的紫色区分开）。

**参数**：`Length=14`、`Source=close`。

**备注**：`nz(src[i])` 未给第二参数，历史不足时按 **0** 计入分子——所以图表最左侧前 `Length-1` 根 bar 的输出会明显偏低，与内置 `wma()` 的早期值不同。这是"手写版"和"内置版"唯一的实际差异。

### `wilder_modified_moving_average.pine` — Wilder Modified Moving Average (WMMA)

**做什么**：Wilder 平滑（也叫 RMA / SMMA），RSI、ATR、ADX 里用的那种平均。

**原理**：

```
WMMA = rma(src, L)        α = 1/L        （等价于 EMA 取 α = 1/L）
WMMA_t = WMMA[t-1] + (src - WMMA[t-1]) / L
```

和 EMA 同一个递推式，但 `α = 1/L` 而不是 `2/(L+1)`；对 L>1 而言 `1/L < 2/(L+1)`，所以**更慢更平滑**，平均滞后约 `L-1`（L=14 → 13 根）。

**输出**：一条 `#6d1e7f` 线。

**参数**：`Length=14`、`Source=close`。

**备注**：想复现 TradingView 内置 RSI/ATR 的平滑手感时用这个；想"灵敏"就别用它。

---

## 第 2 族 · 固定窗口权重核（15 个）

这一族的共同结构是：**窗口长度固定（或半固定），但每个样本的权重由某个核函数 / 系数表决定**。它们都属于 FIR（有限冲激响应）——只看最近 L 根，不递推。

一族之内真正的差异是**权重曲线的形状**：矩形（SMA，第 1 族）、三角形（TMA、QMA）、正弦（SWMA）、指数（EMA）、幂函数（PWMA）、斐波那契/法里分数（FWMA、FSWMA）、多项式（HWMA、Spencer）、高斯（ALMA）、回归核（LSMA）、数据驱动（SEWMA、EVWMA）。

### `triangular_moving_average.pine` — Triangular Moving Average (TMA)

**做什么**：平滑两次的 SMA，权重呈三角形。

**原理**：

```pine
tma = sma(sma(src, ceil(length/2)), floor(length/2) + 1)
```

两个矩形窗卷积 = 一个三角形窗。L=20 时实际核是 `1,2,3,…,10,10,9,…,1`（20 项），**总宽度仍为 L**。所以 TMA 与同长度 SMA **平均滞后完全相同**（9.5 根），但方差更小、更平滑——这是"同样滞后、更平滑"的少数几个干净例子。

**输出**：双色线（默认 `Highlight=true`）。

**参数**：`Length=20`、`Source=close`、`Highlight=true`。

### `parabolic_weighted_moving_average.pine` — Parabolic Weighted MA (PWMA)

**做什么**：权重按幂函数递增的加权平均，`Power` 就是"把近期样本放大几倍"的旋钮。

**原理**：

```pine
weight = pow(length - i, power)      // power 默认 2 → 二次权重
pwma = Σ src[i]·weight / Σ weight
```

`Power=1` 退化为 WMA；`Power=2` 时最新样本权重 `(L)²` 远大于最旧样本 `1`，滞后比 WMA 还小，但噪声和尖刺也更容易穿透。

**输出**：双色线。

**参数**：`Length=14`、`Power=2`（`minval=0`）、`Source=close`、`Highlight=true`。

### `double_weighted_moving_average.pine` — Double Weighted Moving Average (DWMA)

**做什么**：连续做两次 WMA，把线性权重核再"压"一次。

**原理**：

```pine
dwma = wma(wma(src, length), length)
```

一次 WMA 的核是斜坡 `1..L`，两次卷积后核变成"斜坡与斜坡的卷积"（近似钟形，峰值仍在近期），**滞后比单次 WMA 大**（约 `2(L-1)/3`，L=10 → 6 根），但平滑度显著提高。

**输出**：双色线。

**参数**：`Length=10`、`Source=close`、`Highlight=true`。

**注意**：文件名叫 `double_weighted_moving_average`，缩写 `DWMA` 与第 3 族的 `distance_weighted_moving_average.pine` **撞名**（那个也是 `DWMA`）。看图表上的 shorttitle 无法区分，只能看脚本名。

### `sine_weighted_moving_average.pine` — Sine Weighted MA (SWMA)

**做什么**：用正弦窗（Hann 窗的半个周期）给窗口加权，中段权重最大、两端平缓收敛到 0。

**原理**：

```pine
weight = sin((i + 1) * π / (L + 1))       // i = 0 是最新 bar
```

取 `i+1` 而不是 `i`，是因为 `sin(0)=0` 会让最新一根权重为 0。权重和恰好等于 `cot(π/(2(L+1)))`（L=14 → 9.5144），脚本用累加得到的 `weightSum` 归一化，所以不需要闭式公式。

权重曲线是"钟形"的，比三角窗更平滑（二阶连续），频谱旁瓣比矩形窗低得多 → **价格噪声抑制好**。但核是**对称**的，重心就在窗口正中，所以等效滞后 = `(L-1)/2` = **6.5，和同长度 SMA/EMA 一样**（比 WMA 的 4.33 慢）——它的收益全在平滑度上，不在速度上。

**输出**：双色线 + 可选 K 线着色 + 可选背景着色 + 3 个告警（变绿 / 变粉 / 任意变色）。

**参数**：`Length=14`、`Highlight Direction=true`、`Apply Bar Colors=true`、`Apply Background Colors=false`。

### `fibonacci_weighted_moving_average.pine` — Fibonacci Weighted MA (FWMA)

**做什么**：权重取斐波那契数列，最新一根拿最大的 `F(L)`。

**原理**：

```pine
prev = 0, curr = 1
for i = 1 to length:
    push(curr)          // F(1)..F(L)
    curr = prev + curr
reverse(weights)        // index 0（最新 bar）对应 F(L)
ma = Σ nz(src[i], src) · weights[i] / weightSum
```

刻意**用整数递推生成**而不是 Binet 公式（`(φⁿ-ψⁿ)/√5`），避免浮点误差；核只依赖 `length`，所以用 `var array<float>` 只算一次（不用每根 bar 重算 14 次加法）。

**输出**：双色线 + 可选 K 线着色 + 可选背景 + 3 个告警。

**参数**：`Length=14`、`Highlight Direction=true`、`Apply Bar Colors=true`、`Apply Background Colors=false`、`Source=close`。

**备注**：L=14 时权重比 `F(14)/F(1) = 377`，核极度右偏——等效滞后只有 **1.60 根 bar**（SMA/EMA 是 6.5），**响应极快但抗噪很差**，接近一块"只比原始价格稍微钝一点"的布。

### `farey_sequence_weighted_moving_average.pine` — Farey Sequence Weighted MA (FSWMA)

**做什么**：用 Farey（法里）数列作为权重，`Order` 同时决定**窗口长度和权重形状**。

**原理**：Farey 序 `F_n` 是分母 ≤ n 的全部最简分数（0 到 1 之间）。脚本取其中位于 `[1/n, 1]` 的一批分数，按降序排列作为权重，最旧到最新一一对应：

| Order | 用到的分数（降序） | 项数 | 分母（权重和） |
| --- | --- | --- | --- |
| 2 | 1, 1/2 | 2 | 1.5 |
| 3 | 1, 2/3, 1/2, 1/3 | 4 | 2.5 |
| 5 | 1, 4/5, 3/4, 2/3, 3/5, 1/2, 2/5, 1/3, 1/4, 1/5 | 10 | 5.5 |
| 14 | … 64 项（`src` 到 `src63`） … | 64 | 32.5 |

项数与分母都跟脚本里硬编码的常数对得上（Python 穷举 `F_n` 验证过），所以 `order == "5"` 分支的 `/ 5.5` 不是拍脑袋的归一化系数，而是权重之和。

实现上脚本用 `src1 … src63` 把 64 个历史样本展开成 63 变量，再用一条 13 分支的嵌套三元表达式查表——**文件开头的 `// Generated by script, not manually` 就是这个意思**，它是代码生成器的产物。

**输出**：双色线。

**参数**：`Order`（字符串，默认 `"5"`，可选 2–14）、`Source=close`、`Highlight=true`。

**备注**：`Order` 越大窗口越长、越平滑；`Order=14` 时窗口 64 根，图表最左侧会有 63 根 bar 的空白（样本不足）。

### `henderson_weighted_moving_average.pine` — Henderson Weighted MA (HWMA)

**做什么**：Henderson 加权移动平均——保险精算里用来从"含趋势序列"中提取平滑趋势的经典工具，**权重两端为负**，这是它和其它所有 MA 最不一样的地方。

**原理**：

```
w(j) = 315(m+1)²-j² · (m+2)²-j² · (m+3)²-j² · 3(m+2)²-11j²-16
       ────────────────────────────────────────────────────────
       8(m+2) · ((m+2)²-1)(4(m+2)²-1)(4(m+2)²-9)(4(m+2)²-25)
```

其中 `m = (L-1)/2`，`j` 取 `-(m) … +m`。这组系数是"在窗口内用三次多项式做最小二乘拟合、取拟合曲线中点值"的等价核，因此对**有曲率的趋势**（加速上涨/下跌）不像 SMA 那样系统性地偏低/偏高。

实测权重（L=7）：`-0.0587, 0.0587, 0.2937, 0.4126, 0.2937, 0.0587, -0.0587`，和为 1。

实现上用 `var array<float>` 只算一次核，避免每根 bar 重跑 8 次权重公式。

**输出**：双色线 + 可选 K 线着色 + 可选背景 + 3 个告警。

**参数**：`Length=7`、`Source=close`、`Highlight Direction=true`、`Apply Bar Colors=true`、`Apply Background Colors=false`。

**备注**

- 标准 Henderson 定义在**奇数**窗口上（`2m+1`）。脚本允许偶数：`m` 变成半整数，窗口中心落在两根 bar 中间，权重仍对称、和仍为 1，只是不再对应"某个点"的拟合值。
- 负权重意味着**过冲**：价格走出尖峰时 HWMA 会反向甩过头。这是用零相位特性换来的代价。

### `spencer_15_point_moving_average.pine` — Spencer 15-Point MA (SPMA 15)

**做什么**：Spencer 15 点加权平均，19 世纪精算师为消除死亡率的周期性而设计的固定权重核。

**原理**：完全硬编码的 15 个整数权重，除以它们之和 320：

```pine
w = (-3, -6, -5, 3, 21, 46, 67, 74, 67, 46, 21, 3, -5, -6, -3)
spma15 = Σ src[i]·w[i] / 320
```

权重和 320，中间 7 个为正、两端各 4 个为负。把它当成离散测度验证一下矩：`Σw=1`、`Σw·i=0`、`Σw·i²=0`、`Σw·i³=0`（i 取相对中心的位置，均已数值核实）——
也就是说这组权重**同时消掉常数、一次、二次、三次项**，只保留四次以上的曲率信息。这是它两端必然出现负权重的根本原因：约束越多，核的矩越"紧"，就只能在两端取负来补偿。

**输出**：双色线。

**参数**：只有 `Source`（无 Length）、`Highlight=true`。

**备注**：窗口固定 15 根，不可调；两端负权重 → 有轻微过冲。

### `spencer_21_point_moving_average.pine` — Spencer 21-Point MA (SPMA 21)

**做什么**：同上的 21 点版本，权重和为 350。

**原理**：

```pine
w = (-1, -3, -5, -5, -2, 6, 18, 33, 47, 57, 60, 57, 47, 33, 18, 6, -2, -5, -5, -3, -1)
spma21 = Σ src[i]·w[i] / 350
```

更长、更平滑，负权重占比更小（过冲比 15 点版轻），代价是滞后更大。矩约束与 15 点版相同（`Σw=1`，一/二/三阶矩均为 0）。

**输出**：双色线。

**参数**：只有 `Source`、`Highlight=true`。

### `quick_moving_average.pine` — Quick Moving Average (QMA)

**做什么**：把三角权重核的**峰位从窗口中心挪到 1/3 处**，做出一条"又快又平滑"的 MA。

**原理**：

```pine
peak = length / 3
i <= peak        → mult = i / peak                      // 线性上升段
i >  peak        → mult = (length + 1 - i) / (length + 1 - peak)   // 线性下降段
ma = Σ src[i-1]·mult / Σ mult
```

普通三角核（TMA）的峰在中心，重心滞后 = `(L-1)/2`；把峰右移到 `L/3` 后重心前移，等效滞后从 6.5 降到 **5.57**（L=14）——比 SMA/TMA 快约 1 根，但仍比 WMA 的 4.33 慢。**准确的说法是"同等平滑度下略快"，而不是"低滞后 MA"**。`Length=14` 时峰值权重出现在第 5 根（`peak=4.67`）。

**输出**：双色线 + 可选 K 线着色 + 可选背景 + 3 个告警。

**参数**：`Length=14`、`Source=close`、`Highlight Direction=true`、`Apply Bar Colors=true`、`Apply Background Colors=false`。

**备注**：`Length` 很小时（如 2、3）`peak < 2`，核几乎退化为"只看最新 1–2 根"，会非常抖。

### `self_weighted_moving_average.pine` — Self-Weighted MA (SEWMA)

**做什么**：用一个非常规的加权方式——**把窗口之前更早的那批价格值，当作当前窗口的权重**。

**原理**：

```pine
for i = 0 to length - 1
    weight = src[length + i]          // ← 权重来自窗口之外的"更老"样本
    sum += weight * src[i]
    weightSum += weight
sewma = sum / weightSum
```

权重不是无量纲系数，而是**价格本身**。含义有点绕：如果早期价格高（`src[L+i]` 大），那么对应的近期样本 `src[i]` 就被放大权重；反之被压低。所以它是一条"被早期价格水平调制"的均线——在单边上涨行情里，早期低价 → 近期样本权重小 → 输出偏低；下跌行情里反过来。

**输出**：单色 `#6d1e7f` 线（`Highlight` 默认 `false`，需要手动打开才会红绿变色）。

**参数**：`Length=14`、`Source=close`、`Highlight=false`。

**备注**：`weightSum` 为 0（早期价格为 0 或数据缺失）时输出 `na`。这个指标没有明确的经济含义，更接近一个"数学玩具"，实战价值存疑。

### `middle_high_low_moving_average.pine` — Middle-High-Low MA (MHLMA)

**做什么**：先取"区间中点"序列 `(最高价 + 最低价)/2`，再对它做 EMA / SMA。

**原理**：

```pine
mhl   = avg(highest(mhlRange), lowest(mhlRange))     // avg = (a+b)/2
mhlma = maType == "EMA" ? ema(mhl, length) : sma(mhl, length)
```

`highest(mhlRange)` / `lowest(mhlRange)` 在 v3 省略 `source` 时默认取 **high / low**，所以 `mhl` 就是 Donchian 通道的中轨；再平滑一次得到 MHLMA。

**输出**：双色线。

**参数**：`Length=14`（平滑长度）、`High-Low Range=10`（Donchian 窗口）、`MA Type="EMA"`（可选 `"SMA"`）、`Source` 参数**存在但未被使用**（脚本用的是 high/low）。

**备注**：因为它以 `(HH+LL)/2` 为输入，所以**只看区间上下界的均值，忽略收盘价在区间内的位置**——本质上比用 close 的均线更"钝"，但不会因单根长影线而剧烈跳动（长影线只影响 HH/LL 各一次）。

### `least_squares_moving_average.pine` — Least Squares MA (LSMA)

**做什么**：在窗口内做最小二乘直线拟合，输出**回归线在当前 bar 上的值**（`offset=0`）。

**原理**：

```pine
plot(linreg(src, length, offset))
```

`linreg` = 线性回归端点值。它的权重核长这样（L=14，最旧 → 最新）：

```
-0.1143, -0.0857, -0.0571, -0.0286, 0, 0.0286, 0.0571, 0.0857,
 0.1143,  0.1429,  0.1714,  0.2000, 0.2286, 0.2571       （和为 1）
```

注意两点：**权重正负混合**（最旧的 4 个是负的），且**质心滞后为 0**——回归端点在理论上不滞后于趋势。这是 LSMA 被称作"零滞后均线"的原因，代价是对噪声的增益很高（相当于在窗口内做了一次差分）。

`offset > 0` 会把回归线**外推**到未来（TradingView 上常见的"预测线"就是这么来的），`offset` 为负则是向内回看。

**输出**：单色 `#6d1e7f` 线（无 Highlight）。

**参数**：`Length=14`、`Offset=0`、`Source=close`。

### `arnaud_legoux_moving_average.pine` — Arnaud Legoux MA (ALMA)

**做什么**：高斯权重 + 两个可调参数（`Offset` 控制峰位、`Sigma` 控制宽度）的移动平均，目标是"平滑度高而滞后可控"。

**原理**：调用内置 `alma(src, length, offset, sigma)`：

```
m = offset·(L-1)                    权重峰的位置
s = L / sigma                       高斯标准差
w_i = exp( -(i - m)² / (2s²) )      i = 0 是最新 bar
```

`Offset=0.85` 表示峰落在窗口 85% 处（靠近最新端），`Sigma=6` 越小核越窄、越灵敏。默认组合大致相当于"比 EMA 更平滑、滞后接近 WMA"。

**输出**：单色 `#e69138` 橙色线。

**参数**：`Length=9`、`Offset=0.85`、`Sigma=6`、`Source=close`。

**备注**：核在两端不严格截断（高斯拖尾），所以实际上会略用超过名义窗口的样本；`Sigma` 很小时效果接近"只取最近几根"。

### `elastic_volume_weighted_moving_average.pine` — Elastic Volume Weighted MA (EVWMA)

**做什么**：用成交量当"弹性"权重的 VWMA 变体——成交量越大，均线越"贴"价格。

**原理**：

```pine
volumeSum = sum(volume, length)
evwma = ((volumeSum - volume) · evwma[1] + volume · src) / volumeSum
```

结构上是对 `src` 的一阶递归：当前 bar 的成交量占总量的比例 `volume/volumeSum` 就是"新息权重"，其余部分留给历史。这与标准 VWMA（`Σ p·v / Σ v`，FIR）不同：**EVWMA 是 IIR**，历史权重由滚动总量隐式决定，避免了一次性窗口切换的跳变。

**输出**：双色线。

**参数**：`Length=14`、`Source=close`、`Highlight=true`。

**备注**：`volume` 为 0 的品种（部分外汇/指数）会退化成"完全不动"的直线；`volumeSum=0` 时输出 `na`。

---

## 第 3 族 · 稳健统计与距离加权（12 个）

这一族不追求"更平滑"或"更少滞后"，而是换掉**平均**这个动作本身：用中位数、分位数、几何/调和/均方根平均，或用**样本之间的距离**来定权重。目标是抗离群点（尖刺、跳空、坏 tick）或刻画分布形状。

### `moving_median.pine` — Moving Median (MM)

**做什么**：窗口内价格的中位数，最直接的"抗尖刺均线"。

**原理**：

```pine
median = percentile_nearest_rank(src, length, 50)
```

`percentile_nearest_rank` 取窗口内排序后第 `50%` 位的实际样本值（不是插值）。单个异常 tick 只会让排序位置移动一格，**永远不会像 SMA 那样被一根坏数据整根拽走**。

**输出**：双色线。

**参数**：`Length=14`、`Source=close`、`Highlight=true`。

**备注**：中位数没有固定的"等效滞后"（它取决于价格分布：横盘时几乎不滞后，单边趋势里约 `(L-1)/2`），响应形状像"台阶"；输出是离散的样本值，趋势里会有明显的阶梯感。偶数 `Length` 时中位数取的是两个中间值之一（nearest rank），不是平均。

### `trimean.pine` — Trimean

**做什么**：四分位数的加权平均，一个兼顾"中心位置 + 偏度"的稳健统计量。

**原理**：

```pine
q1 = percentile_nearest_rank(src, length, 25)
q2 = percentile_nearest_rank(src, length, 50)
q3 = percentile_nearest_rank(src, length, 75)
trimean = (q1 + 2*q2 + q3) / 4
```

Tukey 提出的 trimean：中位数拿双倍权重，两个四分位数各一份。相比中位数，它对分布的不对称更敏感（偏高的分布 trimean 更靠近 Q3）；相比均值，它完全不受极端值影响（权重只落在三个分位点上）。

**输出**：双色线。

**参数**：`Length=14`、`Source=close`、`Highlight=true`。

### `hampel_filter.pine` — Hampel Filter (HF)

**做什么**：先把离群点"换掉"，再对清洗后的序列做 EMA——**移动平均 + 异常值免疫**。

**原理**：

```pine
sampleMedian = median(src, length)
mad          = median(|src - sampleMedian|, length)        // 中位数绝对偏差
hampel       = |src - sampleMedian| <= scalingFactor·mad ? src : sampleMedian
plot(ema(hampel, length))
```

判定逻辑和中位数绝对偏差（MAD）是 Hampel identifier 的标准形式：偏离中位数超过 `Scaling × MAD` 就认为这一根是异常值，用中位数替代。注意 `Scaling=3` 的实际带宽**比 3σ 更紧**：正态下 `σ ≈ 1.4826·MAD`，所以 `3·MAD ≈ 2.02σ`，只覆盖约 95.7% 的样本——这个默认值比看上去敏感得多。

**输出**：单色 `#6d1e7f` 线——注意**画的是 `ema(hampel, length)`**，不是清洗后的 `hampel` 本身；滤波结果被再平滑了一次，所以看起来比中位数线顺滑。

**参数**：`Length=14`、`Scaling Factor=3`、`Source=close`。

**备注**：`mad = 0`（窗口内价格几乎完全不动，或多数样本相同）时，任何偏离中位数的 bar 都会被判为异常并替换成中位数 → 长期横盘后突然启动时，HF 会"压住"第一根突破 bar。这是 Hampel 类滤波器共有的边界行为。

### `moving_geometric_mean.pine` — Moving Geometric Mean (MGM)

**做什么**：窗口内价格的几何平均。

**原理**：

```pine
mgm = exp(sum(log(src), length) / length)        // = (Π src)^(1/L)
```

几何平均适合**乘法/比率**型数据（收益率、价格倍数）。性质：`MGM ≤ SMA`（算术 ≥ 几何），对低价样本更敏感、对高价样本不那么敏感，所以在同一段行情里 MGM 通常略低于 SMA。

**输出**：单色 `#6d1e7f` 线。

**参数**：`Length=14`、`Source=close`。

**备注**：`src ≤ 0` 时 `log` 给 `na`，整条线断开；用于价差/负值序列（如某些 spread）会失效。

### `moving_harmonic_mean.pine` — Moving Harmonic Mean (MHM)

**做什么**：窗口内价格的调和平均。

**原理**：

```pine
sum = Σ(i=0..L-1) 1 / (na(src[i]) ? 1 : src[i])
mhm = length / sum
```

调和平均对**小值**最敏感（`1/x` 在 x 小时很大），因此 `MHM ≤ MGM ≤ SMA`。它在金融里常用于"定投成本"（每期投入固定金额时的平均买入价）。

**输出**：单色 `#6d1e7f` 线。

**参数**：`Length=14`、`Source=close`。

**备注**：`src = 0` 会除零（`na` 被替换成 1，但 0 没有保护）；价格序列正常不会触发。

### `moving_quadratic_mean.pine` — Moving Quadratic Mean / RMS

**做什么**：窗口内价格的均方根（Root Mean Square）。

**原理**：

```pine
rms = sqrt(sum(src*src, length) / length)
```

RMS 对大值最敏感（先平方再开方），所以 `RMS ≥ SMA`。在金融里它的意义更接近"波动幅度"而不是"中心价格"：**它衡量的是量级，不是均值位置**。价格恒定时 RMS = 价格本身，所以图上看仍是一条跟随线。

**输出**：单色 `#6d1e7f` 线。

**参数**：`Length=14`、`Source=close`。

### `distance_weighted_moving_average.pine` — Distance Weighted MA (DWMA)

**做什么**：权重与"该样本到窗口内其它样本的距离和"成**反比**——越"像大家"的样本权重越大，越离群的样本权重越小。

**原理**：

```pine
calcWeight(i):  d_i = Σ(j=0..L-1) |src[i] - src[j]|
                w_i = 1 / d_i
dwma = Σ src[i]·w_i / Σ w_i
```

和普通的"按年龄加权"不同，这里的权重**完全由价格分布决定**：横盘时所有样本距离都小 → 权重趋同，结果接近 SMA；出现尖刺时，尖刺本身到所有样本的距离都大 → 权重被压到最低，**输出几乎不受尖刺影响**。这也是它不需要额外异常值检测的原因。

**输出**：双色线。

**参数**：`Length=14`、`Source=close`、`Highlight=true`。

**备注**

- 计算量 **O(L²)**（双重 for 循环），`Length` 提到 100 时每根 bar 要做 1 万次距离计算，图表会明显变慢。
- 窗口内价格完全相同时 `d_i = 0`，`1/0` → `na`，整条线断开。脚本没有做除零保护（对比 v6 重写版 `inverse_distance_weighted_moving_average.pine` 里的 `weightSum != 0 ? ... : src`，那里补上了这个洞）。
- 与 `inverse_distance_weighted_moving_average.pine` 名字相近但语义**相反**（那边是正比），见下一条。

### `inverse_distance_weighted_moving_average.pine` — Inverse Distance Weighted MA (IDWMA)

**做什么**：同一族思路的 **v6 重写版**，但权重方向反了：距离越大（越离群）权重越大。

**原理**：

```pine
// 距离矩阵对称，只遍历一半 (i<j)，一次把 d 同时记到 i 和 j 上
for i = 0 to length - 2
    for j = i + 1 to length - 1
        d = |src[i] - src[j]|
        w[i] += d ;  w[j] += d
ma = Σ src[i]·w[i] / Σ w[i]
```

`|src[i]-src[j]|` 对 `(i,j)` 对称，所以内层只跑 `j > i`，把距离同时累加到两端——计算量减半。名义上叫 "inverse distance"，但代码里权重是 `w_i = Σ_j d_ij`（**正比**），效果是放大离群样本，与 `distance_weighted_moving_average.pine` 的 `1/d` 相反。

另外补了三个边界处理：`length=1` 时跳过双重循环（否则 Pine 的自动反向 for 会越界）、`nz(src[i], src)` 处理历史不足、`weightSum != 0 ? sum/weightSum : src` 处理全平窗口。

**输出**：双色线 + 可选 K 线着色 + 可选背景 + 3 个告警。

**参数**：`Length=14`、`Source=close`、`Highlight Direction=true`、`Apply Bar Colors=true`、`Apply Background Colors=false`。

**备注**：这两个脚本的差异很值得对照看——同一个"距离加权"的想法，一个抗离群、一个放大离群；而且 v6 版顺手修掉了 v3 版的除零缺陷。

### `distance_coefficient_filter.pine` — Ehlers Distance Coefficient Filter (EDCF)

**做什么**：Ehlers 的"距离系数"滤波：用**距离平方和**做权重，输出窗口内的加权中心。

**原理**：

```pine
for count = 0 to length - 1
    distance = Σ(lookback = 1..L-1) (src[count] - src[count+lookback])²
    srcSum  += distance · src[count]
    coefSum += distance
dcf = coefSum != 0 ? srcSum / coefSum : 0
```

与 DWMA 同一个家族（权重由样本间距离决定），但两点不同：距离用**平方**（放大大偏差）、权重与距离**正比**（离群样本被放大而不是压制）。Ehlers 用它的本意是让滤波器"感知局部离散度"——离散度高的区域权重更大，从而在转折处更灵敏。

**输出**：单色 `#6d1e7f` 线。

**参数**：`Length=14`、`Source=hl2`（默认是 `hl2`，不是 `close`）。

**备注**：又一处 O(L²)；`coefSum=0`（窗口全平）时被硬编码为 0，图上会突然掉到 0——**这是一个明显的边界 bug**，不是价格信号。

### `ehlers_distance_coefficient_filter.pine` — Ehlers Distance Coefficient Filter (EDCF)

**做什么**：与上一条**完全同一个脚本**，图表名、简称（`EDCF`）、算法、参数默认值都一致。

**原理**：见上。

**输出**：单色 `#6d1e7f` 线。

**参数**：`Length=14`、`Source=hl2`。

**备注**：两个文件 `diff` 之后唯一的差别是版权注释里的一个 `the`（`under the terms of the GPL-3.0` vs `under the terms of GPL-3.0`）。属于历史遗留的重复文件，**任选一个用即可**；`distance_coefficient_filter.pine` 看起来是较新的命名。

### `recursive_median_filter.pine` — Recursive Median Filter (RMF)

**做什么**：**中位数 + 一阶低通**的两级级联：先用中位数除掉尖刺，再用低通把台阶抹平。

**原理**：

```pine
alphaArg = 2π / lowpassLength
alpha    = (cos(alphaArg) + sin(alphaArg) - 1) / cos(alphaArg)     // L=12 → 0.4226
rmf      = alpha · median(src, medianLength) + (1-alpha) · rmf[1]
```

`alpha` 不是拍脑袋的：在双线性变换下，截止周期为 `lowpassLength` 的一阶低通，其系数正好是这个组合（`cos+sin-1` 是 Ehlers 常用的"低通 α"参数化）。所以第一级去尖刺，第二级给出平滑曲线。

**输出**：双色线。

**参数**：`Median Length=5`、`Lowpass Length=12`、`Source=close`、`Highlight=true`。

### `recursive_median_oscillator.pine` — Recursive Median Filter (RMF)

**做什么**：上一条的 **v3 旧版实现**，算法完全相同。

**原理**：同 `recursive_median_filter.pine`（`cos(alphaArg)` 直接在判断与算式里各算一次，而不是先存进局部变量）。

**输出**：单色 `#6d1e7f` 线（v3 版没有 `Highlight` 输入）。

**参数**：`Length=5`（对应 v4 版的 `Median Length`）、`Lowpass Length=12`、`Source=close`。

**备注**：**文件名叫 `oscillator`，但它不是振荡器**——脚本内 `study()` 的标题仍是 `"Recursive Median Filter"`，shorttitle 也是 `RMF`，输出是价格量纲的均线而不是 0 轴振荡。这大概是最初命名失误，别被文件名误导。

---

## 第 4 族 · 递归滤波与滞后补偿（20 个）

这一族是**全部 70 个脚本的技术核心**。它们都是 IIR 结构（有反馈、无限记忆），共同议题只有一个：**怎样在不把噪声放大的前提下减少滞后**。

`EMA` 的滞后来自它只有一阶；这一族的做法可以归成四类：

| 手段 | 原理 | 脚本 |
| --- | --- | --- |
| **多级 EMA 线性组合**（相消法） | `n` 级 EMA 加权相减，抵消低阶滞后项 | DEMA、T3、QEMA、PEMA、MNMA |
| **预补偿输入**（提前量） | 把 `src + (src - src[lag])` 喂给普通 EMA | ZLEMA、IIRF、REMA |
| **换一种平均核** | WMA/SMA 组合、Laguerre 级联、非线性步长 | LMA、SHMMA、HMA、Laguerre、McGinley |
| **状态空间 / 双平滑** | 维护"水平 + 趋势"两个状态 | Holt、RMTA、JMA、AHMA |

### `double_exponential_moving_average.pine` — Double EMA (DEMA)

**做什么**：最经典的"减滞后"MA：两次 EMA 相减。

**原理**：

```pine
ema1 = ema(src, length)
ema2 = ema(ema1, length)
dema = 2·ema1 - ema2
```

一眼看不出为什么能减滞后，但从冲激响应看：`ema2` 相当于"滞后 2 倍"的版本，`2·ema1 - ema2` 是两者的线性组合，把一阶滞后项抵消掉，剩下的高阶项变小。代价是**相位超前于趋势时会过冲**。

**输出**：单色 `#6d1e7f` 线。

**参数**：`Length=14`、`Source=close`。

### `t3_moving_average.pine` — T3 Moving Average (T3)

**做什么**：Tim Tillson 的 T3，用"广义 DEMA（GD）"串联三次，是 DEMA 的平滑化推广。

**原理**：

```pine
gd(x) = ema(x, L)·(1+factor) - ema(ema(x, L), L)·factor
t3    = gd(gd(gd(src, L), L), L)
```

`factor=0` 时 `gd(x) = ema(x)`，T3 退化成三重 EMA（很平滑很滞后）；`factor=0.7`（默认）时每一级都带"减滞后"的修正。展开后 T3 是 8 项 EMA（`EMA1…EMA8`）的加权和，**滞后小于 DEMA、平滑度接近三重 EMA**——这是它在实战里受欢迎的原因。

**输出**：双色线。

**参数**：`Length=5`、`Factor=0.7`、`Source=close`、`Highlight=true`。

**备注**：`Length=5` 这个默认值比其它脚本小得多，跟 T3 的"三级串联"有关——每级 EMA(5) 的合成记忆已接近 EMA(15)。

### `quadruple_exponential_moving_average.pine` — Quadruple EMA (QEMA)

**做什么**：四重指数平滑的相消组合（Tillson/Bruno Pio 体系）。

**原理**：

```pine
ema1..ema5 逐级 EMA
qema = 5·ema1 - 10·ema2 + 10·ema3 - 5·ema4 + ema5
```

系数 `5, 10, 10, 5, 1` 依次是二项式系数 `C(5,k)`（`k=1..5`），符号交替——本质是把 5 个不同滞后程度的 EMA 线性组合，抵消到 4 阶滞后项（Tillson 的"高阶 MA"通式：`QEMA = Σ(-1)^(k+1)·C(5,k)·EMA_k`）。

**输出**：双色线。

**参数**：`Length=20`、`Source=close`、`Highlight=true`。

**备注**：系数和为 1（`5-10+10-5+1=1`），所以对常数输入输出恒定，不会漂移；但系数绝对值大（最大 10），**噪声增益很高**，Length 太小时线会非常毛躁。

### `pentuple_exponential_moving_average.pine` — Pentuple EMA (PEMA)

**做什么**：同上的 8 级版本（Pio 的实现）。

**原理**：

```pine
pema = 8·e1 - 28·e2 + 56·e3 - 70·e4 + 56·e5 - 28·e6 + 8·e7 - e8
```

系数 `8, 28, 56, 70, 56, 28, 8, 1` 是二项式系数 `C(8,k)`（`k=1..8`），符号交替；和为 1。级数越多，理论滞后越小、噪声增益越大。

**输出**：双色线。

**参数**：`Length=20`、`Source=close`、`Highlight=true`。

### `mcnicholl_moving_average.pine` — McNicholl MA (MNMA)

**做什么**：用两级 EMA 的另一种代数组合来减滞后（Dennis McNicholl 的形式）。

**原理**：

```pine
alpha = 2/(length+1)
ema1  = ema(src, length)
ema2  = ema(ema1, length)
mnma  = ((2 - alpha)·ema1 - ema2) / (1 - alpha)
```

和 DEMA 一样是 `ema1` 与 `ema2` 的线性组合，但权重由 `alpha` 决定而不是固定的 `2/-1`。`alpha → 1`（超短周期）时系数退化为 `ema1` 本身；`alpha → 0`（超长周期）时接近 `2·ema1 - ema2`，即 DEMA。

**输出**：双色线。

**参数**：`Length=20`、`Source=close`、`Highlight=true`。

### `leo_moving_average.pine` — Leo MA (LMA)

**做什么**：`2·WMA - SMA` 的组合，用两种不同"重心"的平均相减来提前。

**原理**：

```pine
lma = 2·wma(src, length) - sma(src, length)
```

WMA 的重心（`(L-1)/3`）比 SMA（`(L-1)/2`）更靠近当前，这个组合相当于把 WMA 的"提前量"再放大一倍，同时保留 SMA 的平滑性。**滞后比 WMA 更小**，但过冲也更明显。

**输出**：双色线。

**参数**：`Length=14`、`Source=close`、`Highlight=true`。

### `sharp_modified_moving_average.pine` — Sharp Modified MA (SHMMA)

**做什么**：`3·WMA - 2·SMA`，比 LMA 更"锋利"的同类组合（v6 重写版）。

**原理**：

```pine
ma = 3·ta.wma(src, length) - 2·ta.sma(src, length)
```

权重相消后，核在近期被显著抬升、远端被压成负值 → **响应更快、过冲更大**。参数只有 `Length`，没有 `factor` 之类可调项。

**输出**：双色线 + 可选 K 线着色 + 可选背景 + 3 个告警。

**参数**：`Length=14`、`Source=close`、`Highlight Direction=true`、`Apply Bar Colors=true`、`Apply Background Colors=false`。

### `hull_moving_average.pine` — Hull MA (HMA)

**做什么**：Alan Hull 的经典"低滞后"均线，实战中最常用的改良 MA 之一。

**原理**：

```pine
hma = wma( 2·wma(src, length/2) - wma(src, length), round(sqrt(length)) )
```

分三步：① 用"半个窗口的 WMA"减去"整窗口的 WMA"，得到一条滞后被抵消但很毛躁的线；② 再套一层 `sqrt(L)` 长度的 WMA 去毛刺。`sqrt(L)` 的窗口比 `L` 小得多，所以第二步几乎不增加滞后。

**输出**：单色 `#6d1e7f` 线。

**参数**：`Length=14`、`Source=close`（`length/2` 与 `sqrt` 都会被取整）。

**备注**：HMA 在震荡市里会因为内部 WMA 相减而出现**明显的失真拐点**（"台阶"），这是它唯一的著名缺陷。

### `multiple_hull_moving_average.pine` — Multiple Hull MA

**做什么**：把 HMA 同时画三条（20 / 50 / 100），用于多周期趋势排列判断。

**原理**：三段完全相同的 HMA 计算，只是长度和颜色不同：

```pine
hma1 = wma(2·wma(src1, len1/2) - wma(src1, len1), round(sqrt(len1)))   // 20 → #85200c
hma2 = 同上，len2 = 50                                                // → #1155cc
hma3 = 同上，len3 = 100                                               // → #f1c232
```

**输出**：三条线（红褐 / 蓝 / 黄），无高亮、无告警。三条线的相对位置就是"多空排列"：短线在上、长线在下 = 多头排列。

**参数**：三组独立 `1st/2nd/3rd Length` + `1st/2nd/3rd Source`（默认全部 `close`）。

**备注**：脚本内 `shorttitle` 也是 `HMA`，与单条版 `hull_moving_average.pine` 在图表上同名；三条线的 `title` 只有 `"1"/"2"/"3"`，样式图例里不好辨认。

### `zero_lag_exponential_moving_average.pine` — Zero Lag EMA (ZLEMA)

**做什么**：先把输入"往前挪"半个滞后量再喂给 EMA，抵消 EMA 的滞后。

**原理**：

```pine
lag   = floor((length - 1) / 2)
zlema = ema(src + (src - src[lag]), length)
```

`src + (src - src[lag]) = 2·src - src[lag]` 就是把最新价格沿当前斜率外推 `lag` 根 bar，相当于"补偿掉 EMA 平均滞后的一半"（EMA 的平均滞后是 `(L-1)/2`，正好等于 `lag`）。所以 **ZLEMA 的相位滞后接近 0**，代价是趋势反转时提前掉头（假信号）。

**输出**：双色线。

**参数**：`Length=14`（`lag=6`）、`Source=close`、`Highlight=true`。

### `infinite_impulse_response_filter.pine` — IIR Filter (IIRF)

**做什么**：带延迟补偿的一阶 IIR 低通。

**原理**：

```pine
alpha  = 2/(length+1)
lag    = round(1/alpha - 1)                       // ≈ (L-1)/2，L=14 → 6.5
iirf   = alpha·(src + change(src, lag)) + (1-alpha)·iirf[1]
```

`change(src, lag) = src - src[lag]`，所以 `src + change = 2·src - src[lag]`——与 ZLEMA **同一个预补偿技巧**，唯一区别是 `lag` 用 `1/α - 1`（EMA 的平均滞后）而不是 `(L-1)/2`（两者在 L=14 时都是 6.5，对其它 L 略有差异）。

**输出**：双色线。

**参数**：`Length=14`、`Source=close`、`Highlight=true`。

### `regularized_exponential_moving_average.pine` — Regularized EMA (REMA)

**做什么**：在 EMA 递推里加一个"向线性外推靠拢"的正则项。

**原理**：

```pine
alpha = 2/(length+1)
rema  = (rema1 + alpha·(src - rema1) + lambda·(2·rema1 - rema2)) / (lambda + 1)
```

括号里第一项是标准 EMA 更新，第二项 `2·rema1 - rema2` 是"沿当前速度继续走"的线性外推。`lambda=0` 时完全退化成 EMA。分母 `lambda+1` 保证权重和为 1（不会整体抬升/压低）。

需要澄清一个常见误解：**`lambda` 并不减少平均滞后**（实测 λ=0/0.5/1/2/4 时等效滞后恒为 6.5），它改变的是**响应形状**——λ 越大，冲激响应越平缓、并开始在尾部出现负值，表现为对趋势转折的反应更锐利、以及轻微振铃。

**输出**：双色线。

**参数**：`Length=14`、`Regularization Constant=0.5`、`Source=close`、`Highlight=true`。

**备注**：`lambda > 1` 时可能进入欠阻尼状态，价格横盘时输出会自己"荡"起来。

### `holt_exponential_moving_average.pine` — Holt EMA (HEMA)

**做什么**：Holt 双指数平滑（水平 + 趋势两个状态），把"趋势"显式建模。

**原理**：

```pine
alpha = 2/(alphaLength+1)      // 水平平滑系数
gamma = 2/(gammaLength+1)      // 趋势平滑系数
hema = (1-alpha)·(hema[1] + b[1]) + alpha·src
b    = (1-gamma)·b[1] + gamma·(hema - hema[1])
```

先预测 `hema[1] + b[1]`（上一期水平 + 趋势），再用 `alpha` 修正到实际价格；`b` 是对"水平变化量"的 EMA。**趋势项让它在单边行情里自动提前**，因此比同参数 EMA 滞后小；横盘时 `b` 趋近 0，退化为 EMA。

**输出**：双色线。

**参数**：`Alpha Length=20`、`Gamma Length=20`、`Source=close`、`Highlight=true`。

**备注**：`gamma` 越大（`Gamma Length` 越小）趋势项越灵敏，也越容易在震荡里被"甩"出假趋势。

### `recursive_moving_trend_average.pine` — Recursive Moving Trend Average (RMTA)

**做什么**：另一种"水平 + 趋势"的双状态递推，用累积和 `b` 代替显式趋势项。

**原理**：

```pine
b    = (1-alpha)·b[1] + src                  // 对 src 的加权累积（无 alpha 缩放）
rmta = (1-alpha)·rmta[1] + alpha·(src + b - b[1])
```

`b - b[1]` 是一阶差分（速度），把它加回 `src` 后做 EMA——**结构与 Holt/ZLEMA 同类**：把"趋势"补偿进输入。`b` 的归一化方式与 Holt 的 `b` 不同（这里是"累积水平的 EMWA"），所以 RMTA 在强趋势里的提前量比 Holt 保守一些。

**输出**：双色线。

**参数**：`Length=14`、`Source=close`、`Highlight=true`。

### `ahrens_moving_average.pine` — Ahrens MA (AHMA)

**做什么**：用"新旧值的中点"做参考点的简化递推。

**原理**：

```pine
ahma = ahma1 + (src - (ahma1 + ahma[length]) / 2) / length
```

每一步只把 `(当前值 - 参考点)/L` 加到上一期。参考点取"上期输出"与"L 根之前的输出"的中点，相当于用**输出历史上的一个点**判断趋势方向，而不是像 EMA 那样只看输入。

**输出**：双色线。

**参数**：`Length=9`、`Source=close`、`Highlight=true`。

**备注**：`ahma[length]` 在图表最左侧不足 L 根时是 `na`，`nz()` 只保护了 `ahma[1]` 没保护 `ahma[length]` → **最左端 L 根会出现 `na` 断线**。这是脚本的小瑕疵。

### `laguerre_filter.pine` — Laguerre Filter (LF)

**做什么**：Ehlers 的 Laguerre 级联，用 4 个一阶环节+加权和做平滑，特点是**用单个 `alpha` 控制整个频率响应曲线**。

**原理**：

```pine
L0 = alpha·src + (1-alpha)·L0[1]
L1 = -(1-alpha)·L0 + L0[1] + (1-alpha)·L1[1]
L2 = -(1-alpha)·L1 + L1[1] + (1-alpha)·L2[1]
L3 = -(1-alpha)·L2 + L2[1] + (1-alpha)·L3[1]
lf = (L0 + 2·L1 + 2·L2 + L3) / 6
```

四个环节构成一个 4 阶 Laguerre 网络（`L1..L3` 的更新里带负反馈，所以它们输出的相位依次滞后 90°），最后用 `(1,2,2,1)/6` 的二项式权重合成。`alpha` 越小越平滑。相比 EMA，它在"平滑度—滞后"曲线上更靠前，且**对趋势转折的响应更圆滑**（不会像 DEMA 那样过冲）。

**输出**：双色线。

**参数**：`Alpha=0.7`（0–1）、`Source=close`、`Highlight=true`。

### `jurik_moving_average.pine` — Jurik MA (JMA)

**做什么**：Jurik Research 的 JMA 简化实现——三级递推 + 可调相位，是"低滞后、低过冲"的代表作。

**原理**：

```pine
phaseRatio = phase < -100 ? 0.5 : phase > 100 ? 2.5 : phase/100 + 1.5
beta  = 0.45·(length-1) / (0.45·(length-1) + 2)
alpha = beta^power

e0 = (1-alpha)·src + alpha·e0[1]                                  // 一阶平滑
e1 = (src - e0)·(1-beta) + beta·e1[1]                             // 残差平滑
e2 = (e0 + phaseRatio·e1 - jma[1])·(1-alpha)² + alpha²·e2[1]      // 相位修正
jma = e2 + jma[1]
```

`e0/e1` 是一对快慢状态，`phaseRatio` 决定把残差 `e1` 放大多少（`phase` 从 -100 到 100 对应 0.5–2.5 倍），`e2` 再把它积分回主输出。`power` 通过 `alpha = beta^power` 改变有效记忆长度。

**输出**：双色线。

**参数**：`Length=7`、`Phase=50`、`Power=2`、`Source=close`、`Highlight=true`。

**备注**：原版 JMA 的完整实现还有自适应速度和"噪声抑制"层，这里是公开的简化版；`phase` 调大 → 更超前（也可能过冲），`phase` 调小 → 更平滑。

### `mcginley_dynamic.pine` — McGinley Dynamic (MD)

**做什么**：不需要调参、能自动"跟上"价格速度的均线——价格偏离越远，它移动得越快。

**原理**：

```pine
md = na(md[1]) ? src
              : md[1] + (src - md[1]) / (k · length · (src/md[1])^4)
```

关键在分母里的 `(src/md[1])^4`：价格**高于**均线时该因子 > 1，步长被压小（慢慢跟）；价格**低于**均线时该因子 < 1，步长被放大（快速跟）。所以它在下跌时比上涨时更灵敏——这是设计者为了让均线"不落后于暴跌"故意做的非对称。

**输出**：单色 `#6d1e7f` 线（无 Highlight）。

**参数**：`Length=14`、`K=0.6`、`Source=close`。

**备注**：`src/md[1]` 的四次方在价格剧烈跳变时会极大/极小，`Length` 很小时步长可能大到"直接贴住价格"（失去平滑意义）。这是本目录里唯一**非线性**的均线。

### `corrected_moving_average.pine` — Corrected MA (CMA)

**做什么**：以 SMA 为基准，用一次不动点迭代求出"带惯性的修正均线"，并画出 SMA 与 CMA 的差值填充。

**原理**：

```pine
sma = ta.sma(src, length)
cmaPrev = cma[1]（初始为 sma）
v1 = ta.variance(src, length)          // 窗口方差
dev = cmaPrev - sma
v2 = dev²
v3 = (v1 == 0 or v2 == 0) ? 1 : v2/(v1 + v2)
k  = max(0, 2 - 1/v3)                  // 不动点迭代 k ← v3·k·(2-k) 的闭式解
cma = cmaPrev + k·(sma - cmaPrev)
```

原版算法要跑 1001 次定点迭代 `k ← v3·k·(2-k)`，这里用它的**非零不动点** `k = 2 - 1/v3` 一次算完（`k = v3·k·(2-k) ⇒ 1 = v3(2-k) ⇒ k = 2 - 1/v3`，已数值验证）。

- `v3 ∈ (0.5, 1]`：`k ∈ (0, 1]`，CMA 在 SMA 与本身上期值之间插值，**偏离越大追得越快**；
- `v3 ≤ 0.5`（`cma` 与 `sma` 已经接近）：迭代会**塌陷到 k=0**，即 CMA 停止更新——所以脚本用 `max(0, ·)` 把它钳到 0，行为与原迭代一致；
- `v1=0`（窗口内价格不变）或 `v2=0`（`cma == sma`）：直接取 `v3=1`、`k=1`，CMA 跳到 SMA。

**输出**：**两条线** + 填充：CMA（默认 `color.orange`，高亮时绿/红）和 SMA（蓝）。填充色：`sma > cma` 时绿（含 75% 透明），否则红。另有 2 个告警：`sma` 上穿 `cma`（Long）、下穿（Short）。

**参数**：`Length=35`、`Source=close`、`Highlight CMA=false`、`Apply Ribbon Filling=true`。

**备注**：`Length` 默认 35，明显大于其它脚本的 14——CMA 的"修正"是个慢过程，长度太小时 `v1` 太小、`v3` 频繁落在 0.5 附近，会出现"黏住不动再猛跳"的观感。

### `alpha_decreasing_exponential_moving_average.pine` — Alpha-Decreasing EMA (ADEMA)

**做什么**：让 EMA 的 `alpha` 随图表历史长度递减——越早的 bar 权重越"新"，越晚越平滑。

**原理**：

```pine
alpha = 2 / (int(bar_index) + 1)
ema   = alpha·close + (1-alpha)·nz(ema[1], close)
```

`bar_index` 是当前 bar 在图表上的序号（从 0 开始），所以 `alpha` 从第一根 bar 的 2 一路衰减到接近 0。等价于"用不断变长的窗口做 EMA"，最终结果趋近于一段长期平均。

**输出**：单色 `orange` 线（无 Highlight）。

**参数**：无（`src` 硬编码为 `close`）。

**备注**：**这是本目录里唯一"结果依赖图表加载范围"的脚本**：`bar_index` 由图表左边界决定，向左多加载历史、或换一个时间周期，输出曲线都会整体改变（因为 `alpha` 序列变了）。做研究时不要把它和其它指标直接叠加比较。另外第一根 bar `alpha = 2 > 1`，`(1-alpha)` 为负，首值相当于"2×close − close"，仍是 `close`，但最初几十根 bar 的 `alpha` 都接近 1，几乎完全跟随价格。

---

## 第 5 族 · 自适应（10 个）

前四族的系数都是**常数**：一旦选定 `Length`，滤波器对"趋势"和"震荡"一视同仁。这一族的核心思想是让系数**随市场状态变化**：

- 趋势干净时 → `alpha` 调大（跟紧价格，减少滞后）；
- 震荡/噪声大时 → `alpha` 调小（重平滑，避免假信号）。

各自用来度量"市场状态"的量不同，这是它们唯一的分野：

| 脚本 | 状态度量 | α 的构造 |
| --- | --- | --- |
| KAMA | 效率比 `ER = abs(ΔL) / Σabs(Δ)` | `α = (ER(fast-slow)+slow)²` |
| AAMA | 价格在 HH/LL 区间中的位置 | 同上结构，用 `mltp` 代替 `ER` |
| AEMA | 同上 | `α = 2/(L+1)·(1+mltp)` |
| BAMA | 带 `trendParam` 的变体 ER | 反解出动态长度 `vlength` |
| VIDYA | `abs(CMO)` | `α = (2/(L+1))·abs(CMO)` |
| FRAMA | 分形维数 `D` | `factor = exp(-4.6(D-1))` |
| EDSMA | 高通差分的标准化幅度 | `α = abs(z)·5/L` |
| MAMA/FAMA | Hilbert 变换估出的相位变化率 | `α = fastLimit/Δphase` |
| ARSIMA | RSI 偏离 50 的程度 | `α = 2·abs(RSI/100 - 0.5)` |
| ALF | 价格差分在窗口内的相对位置 | `α = median(归一化距离)` |

### `kaufman_adaptive_moving_average.pine` — Kaufman Adaptive MA (KAMA)

**做什么**：Perry Kaufman 的 KAMA，自适应均线的标杆。

**原理**：

```pine
mom        = |change(src, length)|                  // 净位移
volatility = sum(|change(src)|, length)             // 总路程
er         = volatility != 0 ? mom/volatility : 0   // 效率比：0=纯噪声，1=纯趋势
fastAlpha  = 2/(fastLength+1)                       // 默认 0.667
slowAlpha  = 2/(slowLength+1)                       // 默认 0.0645
alpha      = (er·(fastAlpha - slowAlpha) + slowAlpha)²
kama       = alpha·src + (1-alpha)·kama[1]
```

`ER` 是"净位移 ÷ 总路程"：直线行情 `ER→1`，来回震荡 `ER→0`。`α` 在 `[slowAlpha², fastAlpha²] = [0.0042, 0.4444]` 之间滑动（已核实），且**平方**让两端更极端：震荡时几乎冻结，趋势时接近 2 周期 EMA。

**输出**：双色线。

**参数**：`Length=14`、`Fast EMA Length=2`、`Slow EMA Length=30`、`Source=close`、`Highlight=true`。

### `vidya_variable_index_dynamic_average.pine` — Variable Index Dynamic Average (VIDYA)

**做什么**：用 Chande 动量振荡器（CMO）的绝对值缩放 EMA 的 `alpha`。

**原理**：

```pine
mom     = change(src)
upSum   = sum(max(mom,0), length)
downSum = sum(-min(mom,0), length)
cmo     = |(upSum - downSum) / (upSum + downSum)|        // 0..1
alpha   = 2/(length+1)
vidya   = src·alpha·cmo + vidya[1]·(1 - alpha·cmo)
```

`CMO` 衡量"上涨力度 vs 下跌力度"的净差：单边行情 `|CMO|→1`（滤波器用满 `alpha`、跟得快），多空拉锯 `|CMO|→0`（α→0，输出几乎冻结）。

**输出**：双色线。

**参数**：`Length=14`、`Source=close`、`Highlight=true`。

**备注**：`|CMO|=0` 时 `vidya` 停在上一期不动，会画出一段水平线；`upSum+downSum=0`（完全无波动）时 CMO 为 `na`，整条线断掉。

### `fractal_adaptive_moving_average.pine` — Fractal Adaptive MA (FRAMA)

**做什么**：用**分形维数**估计市场"折叠程度"，据此决定平滑强度。

**原理**：

```pine
length2 = floor(length/2)
N1 = (HH(length2) - LL(length2)) / length2          // 前半窗的价格跨度/长度
N2 = (HH(length2) - LL(length2))[length2] / length2 // 后半窗
N3 = (HH(length) - LL(length)) / length             // 整窗
D  = (log(N1+N2) - log(N3)) / log(2)                // 分形维数（约 1..2）
factor = exp(-4.6·(D - 1))
frama  = factor·src + (1-factor)·frama[1]
```

`D=1`（趋势笔直，前后半窗跨度之和 ≈ 整窗跨度）→ `factor = 1`，完全跟随价格；`D=2`（噪声把跨度"折叠"了）→ `factor ≈ 0.01`，重平滑。`-4.6` 这个常数就是让 `D` 从 1 到 2 时 `factor` 从 1 衰减到约 1%。

**输出**：双色线。

**参数**：`Length=14`、`Source=close`、`Highlight=true`。

**备注**：`highest/lowest` 省略 source → 用 **high/low**（不是 close）；`N1+N2` 或 `N3` 为 0（价格完全不动）时 `log(0)` 给 `na`。

### `ehlers_deviation_scaled_moving_average.pine` — Ehlers Deviation-Scaled MA (EDSMA)

**做什么**：Ehlers 的"偏差缩放"均线：用价格的高通残差幅度决定 `alpha`。

**原理**：

```pine
zeros = src - src[2]                                 // 二阶差分（高通，去掉趋势）
ssf   = SuperSmoother((zeros + zeros[1])/2, ssfLength) // 平滑残差
scaledFilter = stdev(ssf, length) != 0 ? ssf/stdev(ssf, length) : 0
alpha = |scaledFilter| · 5 / length                   // L=40 → 系数 0.125
edsma = alpha·src + (1-alpha)·edsma[1]
```

思路：价格**偏离自身趋势**越剧烈（残差 z 分数越大），`alpha` 越大、跟得越快；趋势平稳时残差小 → 重平滑。这是本族里唯一"先高通再低通"的两段式设计。

**输出**：单色 `#6d1e7f` 线。

**参数**：`Length=40`、`Super Smoother Filter Length=20`、`Number of Poles=2`、`Source=close`。

**备注**：`alpha` 没有上限保护；`|z|` 极端时（如跳空）`alpha` 可能 > 1，导致输出越过价格（`edsma` 会短暂甩出）。`length` 默认 40 也是全目录最大的之一，配合 `5/length` 使常态 `alpha` 很小（≈0.01–0.1）。

### `ehlers_mesa_adaptive_moving_averages.pine` — MESA Adaptive MAs (MAMA & FAMA)

**做什么**：Ehlers 最著名的自适应均线组合——MAMA 快速跟随、FAMA 作为其慢速参考线，两条线的交叉常被当作趋势反转信号。

**原理**：核心不是均线公式（那只是 `mama = α·src + (1-α)·mama[1]`），而是**怎么估 α**：用 Hilbert 变换式的相位/周期估计（homodyne discriminator）。

```pine
smooth    = (4·src + 3·src[1] + 2·src[2] + src[3]) / 10
detrender = 0.962·smooth + 0.5769·smooth[2] - 0.5769·smooth[4] - 0.0962·smooth[6]   // 90° 相移
I1 = detrender[3] ;  Q1 = 同式(detrender)                    // 同相 / 正交分量
jI = 同式(I1) ; jQ = 同式(Q1)                                // 再转 90°
I2 = I1 - jQ ;  Q2 = Q1 + jI                                 // 相位加法
I2 = 0.2·I2 + 0.8·I2[1] ; Q2 = 0.2·Q2 + 0.8·Q2[1]            // 平滑
Re = I2·I2[1] + Q2·Q2[1] ;  Im = I2·Q2[1] - Q2·I2[1]         // 同相鉴别器
mesaPeriod = 360 / atan(Im/Re)                               // 主周期（限制 6..50，且限幅 0.67–1.5×）
smoothPeriod = 0.33·mesaPeriod + 0.67·smoothPeriod[1]
deltaPhase = phase[1] - phase ;  deltaPhase = max(deltaPhase, 1)
alpha = max(fastLimit/deltaPhase, slowLimit)
mama = alpha·src + (1-alpha)·mama[1]
fama = (alpha/2)·mama + (1-alpha/2)·fama[1]
```

要点：`mesaPeriod` 被限制在 6–50 根之间，且每根 bar 的周期变化被限制在 ±50%/±33% 内（防止估计跳变）；`alpha = fastLimit/deltaPhase` 意味着**相位变化越快（趋势启动）α 越大**。`FAMA` 用 `alpha/2`，所以永远比 MAMA 慢。

**输出**：**两条线** —— MAMA `#674ea7`（紫）、FAMA `#f6b26b`（橙）。无高亮、无告警。

**参数**：`Fast Limit=0.5`、`Slow Limit=0.05`、`Source=hl2`（默认 `hl2`）。

**备注**：函数 `computeAlpha()` 在 `mama()`、`fama()` 里被反复调用（`fama` 里还额外调用一次 `mama`），Pine 的函数是内联展开的，所以**同一次相位估计实际被算了多份**——计算量约为单份的 3–4 倍。逻辑上结果一致，但脚本较重。

### `apirine_adaptive_moving_average.pine` — Apirine Adaptive MA (AAMA)

**做什么**：Vitali Apirine 的 AAMA——把 KAMA 的"效率比"换成"价格在区间中的相对位置"。

**原理**：

```pine
hh = highest(length+1) ;  ll = lowest(length+1)      // 默认取 high/low
mltp = (hh - ll != 0) ? |2·src - ll - hh| / (hh - ll) : 0     // 0..1
ssc  = mltp·(fastAlpha - slowAlpha) + slowAlpha
aama = aama[1] + ssc²·(src - aama[1])
```

`mltp` 度量"收盘价偏离区间中点多少"：在区间中点（`src = (HH+LL)/2`）时 `mltp=0` → 最慢（`slowAlpha²`）；在区间边界时 `mltp=1` → 最快（`fastAlpha²`）。直觉是：**价格处在极端位置说明在走趋势，处在中间说明在震荡**。

**输出**：双色线。

**参数**：`Length=14`、`Fast EMA Length=2`、`Slow EMA Length=30`、`Source=close`、`Highlight=true`。

### `apirine_adaptive_exponential_moving_average.pine` — Adaptive EMA (AEMA)

**做什么**：AAMA 的"EMA 变体"：不用平方、不用 slowAlpha 插值，直接在 EMA 的系数上乘一个 1–2 倍的放大因子。

**原理**：

```pine
hh = highest(hlLookback) ;  ll = lowest(hlLookback)
mltp1 = 2/(length+1)
mltp2 = (hh - ll != 0) ? |2·src - ll - hh| / (hh - ll) : 0
rate  = mltp1 · (1 + mltp2)                          // ∈ [2/(L+1), 4/(L+1)]
ma    = n <= length ? sma(src, length)               // 前 L 根用 SMA 播种
                     : ma[1] + rate·(src - ma[1])
```

`rate` 在 `[2/(L+1), 4/(L+1)]` 之间——即**最慢等于普通 EMA，最快等于 2 倍长度减半的 EMA**。前 `length` 根用 SMA 播种，避免递归初值偏差。

**输出**：单色 `#512da8` 线（`Highlight Movements` 默认 **false**）。

**参数**：`Length=10`、`Highest/Lowest Lookback=10`、`Source=close`、`Highlight Movements=false`。

**备注**：`n <= length` 用的是 v3 的 `n`（当前 bar 序号，从 1 开始），所以严格说前 `length` 根是 SMA，之后才切换为递归。

### `bryant_adaptive_moving_average.pine` — Bryant Adaptive MA (BAMA)

**做什么**：Michael Bryant 的 BAMA：由 ER 反解出**动态长度** `vlength`，再用它算 `alpha`。

**原理**：

```pine
er  = |change(src, length)| / sum(|change(src)|, length)
ver = (er - (2·er - 1)/2·(1 - trendParam) + 0.5)²
vlength = (length - ver + 1) / ver          // 上限 maxLength
valpha  = 2 / (vlength + 1)
bama    = valpha·src + (1-valpha)·bama[1]
```

`trendParam=-1`（默认）时括号内化简为 `1.5 - er`，于是 `ver = (1.5-ER)²`：

- `ER=0`（纯震荡）→ `ver=2.25` → `vlength≈5.7` → `α≈0.30`（快）
- `ER=1`（纯趋势）→ `ver=0.25` → `vlength≈59` → `α≈0.03`（慢）

注意这个方向**和 KAMA 相反**（默认参数下 KAMA 在趋势里变快、BAMA 在趋势里变慢）。把 `trendParam` 调向 `+1` 可以反转方向（此时 `ver=(ER+0.5)²`，趋势时 `vlength` 变小、响应变快）。

**输出**：单色 `#6d1e7f` 线（`Highlight Movements` 默认 **false**）。

**参数**：`Length=14`、`Trend Parameter=-1`（步长 0.1）、`Max Length=100`、`Source=close`、`Highlight Movements=false`。

### `adaptive_rsi_moving_average.pine` — Adaptive RSI MA (ARSIMA)

**做什么**：用 RSI 的相对位置当自适应因子：RSI 越极端（超买/超卖）→ α 越大。

**原理**：

```pine
alpha  = 2·|rsi(src, length)/100 - 0.5|        // ∈ [0, 1]
arsima = alpha·src + (1-alpha)·arsima[1]
```

`RSI=50`（多空平衡）→ `α=0`，输出冻结；`RSI=0` 或 `100` → `α=1`，输出即价格。所以这条线在**单边行情里几乎贴着价格走，在震荡区里变成一条水平线**——是"用 RSI 当开关的 EMA"。

**输出**：双色线。

**参数**：`Length=14`、`Source=close`、`Highlight=true`。

**备注**：`RSI` 偶尔正好等于 50 时 `α=0`，输出会短暂停滞；连续震荡会让曲线出现不自然的"平台"。这是"α 允许降到 0"的通病（VIDYA 同理）。

### `adaptive_laguerre_filter.pine` — Adaptive Laguerre Filter (ALF)

**做什么**：Laguerre 滤波器的自适应版：`alpha` 由"当前差分在窗口极值中的相对位置"决定，并通过中位数平滑。

**原理**：

```pine
diff = |src - nz(alf[1], src)|                  // 价格与上一期输出的偏离
HH/LL = diff 在过去 length 根内的最大/最小值
alpha = HH != LL ? median( (diff - LL)/(HH - LL), medianLength ) : alpha[1]

L0 = alpha·src + alpha2·L0[1]
L1 = L0[1] + alpha2·(L1[1] - L0)
L2 = L1[1] + alpha2·(L2[1] - L1)
L3 = L2[1] + alpha2·(L3[1] - L2)
alf = (L0 + 2·L1 + 2·L2 + L3) / 6
```

偏离处于近期极值上限 → `alpha→1`（重设到价格）；偏离小 → `alpha→0`（重平滑）。用中位数（`percentile_nearest_rank(...,50)`）而不是均值来平滑 α，是为了让 α 不被单根异常 bar 拉动。

**输出**：双色线（`alf > alf[1]` 绿，否则红）；`Highlight=false` 时固定 `#6d1e7f`。**1 个告警**：颜色翻转。

**参数**：`Length=14`、`Median Length=5`、`Source=close`、`Highlight=true`、`Await Bar Confirmation=true`。

**备注**：`Await Bar Confirmation=true` 时颜色只在**收盘确认的 bar** 上更新（`barstate.isconfirmed`），盘中最后一根不会闪色；关掉它颜色会随实时价格抖动，告警也会盘中触发。

---

## 第 6 族 · 数字信号处理滤波器（7 个）

这一族直接把**信号处理教科书里的滤波器设计**搬到价格序列上，基本都来自 John Ehlers 的 *Cyclers* 体系。和前面"MA 家族"的区别在于：它们的参数是**周期/截止频率**，设计目标是**频率响应形状**（通带平坦、阻带陡峭、相位可控），而不是"权重好不好看"。

| 脚本 | 滤波器类型 | 阶数 | 关键参数 | 设计目标 |
| --- | --- | --- | --- | --- |
| ESSF | 超平滑（Super Smoother） | 2 或 3 极点 | `Length` | 通带最平坦、无过冲 |
| BF | Butterworth 低通 | 2 或 3 极点 | `Length` | 最大平坦幅频响应 |
| GF | Gaussian 低通 | 1–4 极点 | `CyclePeriod` | 高斯冲激响应，无过冲 |
| OEF / MOEF | 最优椭圆低通 | 固定二阶 | 无 | 最陡的过渡带（固定系数） |
| FIRF | 任意 FIR | 7 抽头 | 7 个手调系数 | 完全自定义 |
| ELI | Lead-Lag 网络 | 2 级 | `Alpha1/Alpha2` | 相位领先（预测） |

### `ehlers_super_smoother_filter.pine` — Ehlers Super Smoother (ESSF)

**做什么**：Ehlers 设计的二阶/三阶超平滑滤波器，专门解决"均线在趋势转折处的过冲/滞后"。

**原理**：

```pine
// 2 极点
arg = √2·π/length
a1  = exp(-arg) ;  b1 = 2·a1·cos(arg)
ssf = (1 - b1 + a1²)·src + b1·ssf[1] - a1²·ssf[2]

// 3 极点
arg = π/length
a1  = exp(-arg) ;  b1 = 2·a1·cos(1.738·arg) ;  c1 = a1²
ssf = (1 - (b1+c1) - (-(c1+b1·c1)) - c1²)·src + (b1+c1)·ssf[1] - (c1+b1·c1)·ssf[2] + c1²·ssf[3]
```

极点位置由 `exp(-π/L)` 决定（把"周期 L"映射到单位圆的阻尼位置），`cos` 项给出振荡频率。2 极点版本在 L=15 时的系数是 `0.1305, 1.4225, -0.553`（已核实）。

**与 `butterworth_filter.pine` 的关系**：两者的**极点（分母系数）完全一样**（`b1 = 2a1·cos(…)、-a1²`），差别只在**输入侧** ——

- ESSF：输入项直接是当前值乘 `1-b+a²`（2 极点）或 `(1-b+c)(1-c)`（3 极点）；
- Butterworth：输入项前面多一层二项式平滑 `(src+2·src1+src2)/4`（2 极点）或 `(src+3·src1+3·src2+src3)/8`（3 极点）。

所以 ESSF 的输入里**没有额外的短窗平均**，时域响应更"纯"、过冲更小、滞后也略小（实测 L≈14 时 2.43 vs 3.20）。

**输出**：单色 `#741b47` 深紫线。

**参数**：`Length=15`、`Number of Poles=2`（可选 3）、`Source=close`。

### `butterworth_filter.pine` — Butterworth Filter (BF)

**做什么**：Butterworth 低通——"通带内最大平坦"的经典设计，这里取 2 阶或 3 阶。

**原理**：

```pine
PI = 2·asin(1)
// 2 极点
a = exp(-√2·π/L) ;  b = 2·a·cos(√2·π/L)
bf = b·bf[1] - a²·bf[2] + (1-b+a²)/4·(src + 2·src[1] + src[2])

// 3 极点
a = exp(-π/L) ;  b = 2·a·cos(1.738·π/L) ;  c = a²
bf = (b+c)·bf[1] - (c+b·c)·bf[2] + c²·bf[3] + (1-b+c)(1-c)/8·(src + 3·src[1] + 3·src[2] + src[3])
```

输入侧的多项式 `(1,2,1)` / `(1,3,3,1)` 是二项式平滑（相当于先把输入过一遍小窗平均），再加上递归的极点项。直流增益严格为 1（已验证 `(1-b+a²)/(1-b+a²) = 1`）。

**输出**：双色线。

**参数**：`Length=14`、`Poles=2`（可选 3）、`Source=close`、`Highlight=true`。

**备注**：`Length` 这里是**截止周期**（越大越平滑），不是样本数；阶数从 2 提到 3 会更平滑，但相位滞后也更大。

### `gaussian_filter.pine` — Gaussian Filter (GF)

**做什么**：冲激响应近似高斯的低通，极点数可调 1–4。

**原理**：

```pine
beta  = (1 - cos(2π/cyclePeriod)) / (2^(1/poles) - 1)
alpha = -beta + sqrt(beta² + 2·beta)          // 由 beta 解出的一阶极点系数
// 把 alpha 级联 poles 次，用二项式展开成单个递推式，例如 4 极点：
filter = alpha⁴·src + 4(1-alpha)·f1 - 6(1-alpha)²·f2 + 4(1-alpha)³·f3 - (1-alpha)⁴·f4
```

`beta` 由目标周期和阶数共同决定，`alpha` 解的是"一阶环节的极点"方程；`poles` 次级联后用二项式系数展开成一个 4 阶递推（避免维护多个中间变量）。L=14 时：1 极点 `α=0.357`、2 极点 `0.493`、3 极点 `0.572`、4 极点 `0.626`（已核实）。

**输出**：双色线。

**参数**：`Cycle Period=14`、`Number of Poles=4`、`Source=close`、`Highlight=true`。

**备注**：`poles` 越高越平滑（`alpha` 越大 → 递推记忆越长）；`cyclePeriod=1` 时 `cos(2π)=1` → `beta=0` → `alpha=0` 退化，别设那么小。

### `ehlers_optimum_elliptic_filter.pine` — Optimum Elliptic Filter (OEF)

**做什么**：Ehlers 的"最优椭圆"低通，系数全部写死（没有 `Length`），是 2 阶固定截止的滤波器。

**原理**：

```pine
oef = 0.13785·src + 0.0007·src[1] + 0.13785·src[2] + 1.2103·oef[1] - 0.4867·oef[2]
```

系数满足椭圆滤波器的设计约束：分子对称（`0.13785, 0.0007, 0.13785`），分母给出两个极点（`1.2103, -0.4867`），直流增益为 1（`(0.13785+0.0007+0.13785)/(1-1.2103+0.4867) = 0.2764/0.2764 = 1`）。椭圆滤波器在同等阶数下**过渡带最陡**，代价是通带和阻带都有纹波。

**输出**：单色 `#6d1e7f` 线（`Highlight Movements` 默认 **false**）。

**参数**：只有 `Source`（默认 `hl2`），无长度参数。

**备注**：因为系数固定，**"Length"完全不归你控制**——想换截止周期只能用下面的 MOEF 或者自己改系数。这条线的响应相当于"周期约 6–10 根"的极陡低通。

### `ehlers_modified_optimum_elliptic_filter.pine` — Modified Optimum Elliptic Filter (MOEF)

**做什么**：OEF 的"相位补偿版"：把输入从 `src` 换成 `2·src - src[1]` 的形式。

**原理**：

```pine
moef = 0.13785·(2·src - src[1])
     + 0.0007·(2·src1 - src2)
     + 0.13785·(2·src2 - src3)
     + 1.2103·moef[1] - 0.4867·moef[2]
```

每一项都用 `2·x - x[1]` 代替 `x`——这正是前面 ZLEMA/IIRF 用的**预补偿（预加重）**技巧：先给输入加上一阶斜率，抵消滤波器本身的相位滞后。所以 MOEF 与 OEF 幅频响应相同、**相位更超前**，代价是过冲/毛刺更多。

**输出**：单色 `#6d1e7f` 线（高亮默认 `false`）。

**参数**：只有 `Source`（`hl2`）。

### `finite_impulse_response_filter.pine` — FIR Filter (FIRF)

**做什么**：7 抽头的 FIR 滤波器，系数全部开放给你手动调。

**原理**：

```pine
firf = (c1·src + c2·src1 + c3·src2 + c4·src3 + c5·src4 + c6·src5 + c7·src6) / (c1+…+c7)
```

除以系数和保证单位增益（常数输入输出不变）。默认系数 `1, 3.5, 4.5, 3, 0.5, -0.5, -1.5`（和 = 10.5）是一个"中间峰、尾部负"的核——**尾部两个负系数就是在做线性去趋势**，因此默认设置下的响应接近"轻度 LSMA"。

**输出**：双色线。

**参数**：7 个 `Coefficient`（步长 0.1）+ `Source` + `Highlight`。

**备注**：这是本目录里**唯一完全通用**的 FIR：任何 7 抽头以内的线性平滑核都能用它复现（`[1,1,1,1,1,1,1]` 就是 SMA(7)，`[7,6,5,4,3,2,1]` 就是 WMA(7)）。系数和接近 0 时会放大噪声（增益趋于无穷），别乱设。

### `ehlers_leading_indicator.pine` — Ehlers Leading Indicator (ELI)

**做什么**：输出一条**相位领先**于价格的曲线，用于提前预示转折。

**原理**：

```pine
lead = 2·src + (alpha1 - 2)·src[1] + (1 - alpha1)·lead[1]
leadingIndicator = alpha2·lead + (1 - alpha2)·leadingIndicator[1]
```

第一级是"二阶 lead 网络"：`lead` 的 Z 变换可写成 `(2 - (2-alpha1)z⁻¹) / (1 - (1-alpha1)z⁻¹)`，在低频段产生**正相位（超前）**；`alpha1` 越小，超前越多。第二级再用 `alpha2` 做一次低通，压掉一级放大出来的噪声。这正是把"趋势外推"用滤波器语言写出来的形式。

**输出**：单色 `#741b47` 深紫线。

**参数**：`Alpha 1=0.25`、`Alpha 2=0.33`、`Source=hl2`。

**备注**：`alpha1` 调小（超前多）时，横盘里会出现明显的**人为波动**（超前网络把噪声也放大了）；这是超前类指标的固有问题，`alpha2` 只能缓解。

---

## 第 7 族 · 其它（1 个）

### `parabolic_sar.pine` — Parabolic SAR (PSAR)

**做什么**：抛物线转向点（Stop And Reverse），Wilder 的经典跟踪止损指标。它给出的是**一条会"翻面"的止损线**，而不是均线。

**原理**：

```pine
psar = sar(start, increment, maximum)
plot(psar, style=cross, color=blue)
```

`sar()` 是 Pine 内置实现：维护一个"极值点"（EP）和加速因子 AF（从 `start=0.02` 起，每创新高/新低加 `increment=0.02`，上限 `maximum=0.2`），止损线按抛物线向 EP 收敛；价格触及止损线时**翻面**（多空互换、AF 重置）。

**输出**：不是连线，而是 `style=cross` 的**蓝色十字点**，只画在 SAR 值上——TradingView 上表现为沿趋势排列的一串叉号，翻面处会从价格下方跳到上方。

**参数**：`Start=0.02`、`Increment=0.02`、`Maximum=0.2`。

**备注**：`start`/`increment` 越大翻面越频繁；横盘时 PSAR 会反复翻面（经典假信号源）。脚本没有 `Highlight`、没有告警；`plot(style=cross)` 只画十字，`linewidth` 对十字记号不起作用，所以视觉上比其它脚本"碎"。

---

## 三、跨脚本对照

### 3.1 等效滞后（均值滞后）实测表，单位：根 bar

**怎么算的**：把每个滤波器的**冲激响应**（一个单位脉冲输入下的输出序列）当成概率分布，取它的重心到最新 bar 的距离。FIR 脚本可直接由权重算出精确值；IIR 脚本用 4000 项截断做数值积分（误差 < 10⁻³）。

三个读表要点：

- 数值 **= 0** 表示"平均不滞后"，**< 0** 表示"平均超前"（靠负权重/外推换来的）——它们一定伴随**过冲**和大噪声增益，不是白来的；
- 只对**线性时不变**的脚本成立。自适应类（KAMA/VIDYA/FRAMA/MAMA/…）、排序类（Median/Trimean）、非线性类（McGinley）的系数随行情变，没有单一的滞后值，故不在表内；
- `.pine` 后的括号是计算时用的参数（默认值）。

**固定权重核（FIR，精确值）**

| 脚本（参数） | 等效滞后 | 备注 |
| --- | --- | --- |
| `simple_moving_average` (14) | **6.50** | 基准 `(L-1)/2` |
| `exponential_moving_average` (14) | **6.50** | 与 SMA 同滞后，但更平滑 |
| `triangular_moving_average` (20) | **9.50** | 同滞后、更平滑（核 `1..10,10..1`） |
| `sine_weighted_moving_average` (14) | **6.50** | 核对称 → 和 SMA 一样 |
| `weighted_moving_average` / `linear_weighted_moving_average` (14) | **4.33** | 斜坡核 `(L-1)/3` |
| `quick_moving_average` (14) | **5.57** | 峰在 `L/3`，比 SMA 快 1 根，比 WMA 慢 |
| `parabolic_weighted_moving_average` (14, p=2) | **3.14** | 幂核 |
| `fibonacci_weighted_moving_average` (14) | **1.60** | 权重比 377:1，几乎贴价格 |
| `henderson_weighted_moving_average` (7) | **3.00** | 核对称，但含负权重 |
| `spencer_15_point_moving_average` (15) | **7.00** | 核重心恰在中心 |
| `spencer_21_point_moving_average` (21) | **10.00** | 同上 |
| `least_squares_moving_average` (14) | **0.00** | 端点外推，含负权重 |
| `arnaud_legoux_moving_average` (9, 0.85, 6) | **6.44** | 高斯核宽，名义 9 根但很像 14 根 |
| `wilder_modified_moving_average` (14) | **13.00** | `α=1/L`，最慢的一档 |
| `finite_impulse_response_filter` (默认系数) | **1.14** | 默认核已在做去趋势 |
| `gaussian_filter` (14, 4 极点) | **2.39** | 极点数 ↑ → 滞后 ↑ |
| `ehlers_super_smoother_filter` (15, 2 / 3 极点) | **2.43 / 3.33** | 同等滞后下最平滑的一档 |
| `butterworth_filter` (14, 2 / 3 极点) | **3.20 / 4.52** | 比同长度 ESSF 略慢 |
| `ehlers_optimum_elliptic_filter` (固定系数) | **1.86** | 过渡带最陡 |
| `ehlers_modified_optimum_elliptic_filter` (固定系数) | **0.86** | 输入预补偿后快了 1 根 |

**递归 / 减滞后类（IIR，数值解）**

| 脚本（参数） | 等效滞后 | 备注 |
| --- | --- | --- |
| `double_exponential_moving_average` (14) | **0.00** | 相消后理论零滞后 |
| `quadruple_exponential_moving_average` (20) | **0.00** | 等价 5 级 EMA 组合 |
| `pentuple_exponential_moving_average` (20) | **0.00** | 8 级组合 |
| `mcnicholl_moving_average` (20) | **-1.00** | 平均**超前** 1 根 |
| `holt_exponential_moving_average` (20/20) | **0.00** | 趋势项抵消滞后 |
| `recursive_moving_trend_average` (14) | **-1.00** | 同上，更激进 |
| `sharp_modified_moving_average` (14) | **0.00** | `3WMA-2SMA` |
| `leo_moving_average` (14) | **2.17** | `2WMA-SMA`，较温和 |
| `hull_moving_average` (14) | **0.67** | 实战中最常用的低滞后解 |
| `zero_lag_exponential_moving_average` (14) | **0.50** | `2src - src[6]` 预补偿 |
| `infinite_impulse_response_filter` (14) | **0.50** | 同技巧，`lag = 1/α-1` |
| `t3_moving_average` (5, 0.7) | **1.80** | 默认 `Length=5` 时非常快 |
| `laguerre_filter` (α=0.7) | **3.21** | 4 级级联 |
| `ahrens_moving_average` (9) | **4.00** | 与 `(L-1)/2` 接近 |
| `regularized_exponential_moving_average` (14, λ=0.5) | **6.50** | λ 只改响应形状，不改平均滞后 |
| `recursive_median_filter` (低通 12 那一段) | **1.37** | 不含中位数的滞后 |

**这张表就是"平滑度 vs 滞后"的取舍全貌**：想让线更贴价格，就得接受核里有负权重（过冲）或干脆外推（假突破）；想让线更稳，就得接受滞后。注意 `spencer_15`（7.00）、`triangular_moving_average`（9.50）这类"核对称"的脚本，滞后和同长度 SMA 完全一样，它们的收益全部在**平滑度**而不是速度上。

### 3.2 计算开销

| 脚本 | 每根 bar 的复杂度 | 说明 |
| --- | --- | --- |
| `distance_weighted_moving_average`、`inverse_distance_weighted_moving_average`、`distance_coefficient_filter`、`ehlers_distance_coefficient_filter` | **O(L²)** | 双重 for 循环；L=100 时每根 10⁴ 次距离运算 |
| `ehlers_mesa_adaptive_moving_averages` | 常数但很大 | `computeAlpha()` 被内联 3–4 份，每份含 6 次 4 抽头运算 |
| `farey_sequence_weighted_moving_average` | O(项数)，最多 64 | 窗口最长 64 根，且 63 个 `srcN` 变量常驻 |
| `multiple_hull_moving_average` | 3 × HMA | 三条线各自 3 层 WMA |
| 其余 | O(L) 或 O(1) | 递推类基本都是 O(1) |

`for` 循环在 Pine 里是每根 bar 重跑的，所以 O(L²) 脚本在 `Length` 调大后**不是"慢一点"，而是"图表直接卡住"**。

### 3.3 重复 / 冗余脚本清单

| 文件 | 与谁重复 | 差异 |
| --- | --- | --- |
| `distance_coefficient_filter.pine` | `ehlers_distance_coefficient_filter.pine` | 完全相同（仅版权注释差一个 `the`） |
| `linear_weighted_moving_average.pine` | `weighted_moving_average.pine` | 数学等价；前者手写循环、颜色不同、早期值因 `nz(...,0)` 偏低 |
| `recursive_median_oscillator.pine` | `recursive_median_filter.pine` | 同一算法的 v3 旧实现；无 `Highlight`；文件名误叫 oscillator |
| `middle_high_low_moving_average.pine` | —— | `Source` 输入存在但**未被使用** |
| `multiple_hull_moving_average.pine` | `hull_moving_average.pine` | 三条 HMA 打包版，shorttitle 同为 `HMA` |

---

## 四、已知缺陷与使用注意

### 4.1 会画出"假信号"的边界行为

| 脚本 | 现象 | 原因 |
| --- | --- | --- |
| `distance_coefficient_filter` / `ehlers_distance_coefficient_filter` | 输出突然掉到 **0** | `coefSum == 0`（窗口内价格全同）时硬编码返回 `0` |
| `distance_weighted_moving_average` | 整条线**断开**（`na`） | `1/0`，无除零保护（v6 版 IDWMA 已修） |
| `moving_geometric_mean` | 线断开 | `src ≤ 0` 时 `log` 无效 |
| `moving_harmonic_mean` | 线断开 | `src = 0` 时 `1/0`（`na` 被替换成 1，但 0 没有） |
| `ahrens_moving_average` | 图表最左 L 根断开 | `ahma[length]` 未做 `nz()` |
| `adaptive_rsi_moving_average` / `vidya_variable_index_dynamic_average` | 出现一段**水平直线** | `α` 允许降到 0，输出冻结 |
| `ehlers_deviation_scaled_moving_average` | 跳空时短暂"甩出"价格外 | `α = abs(z)·5/L` 无上限 |
| `mcginley_dynamic` | 极端 bar 上直接贴住价格 | `(src/md)⁴` 非线性放大步长 |
| `self_weighted_moving_average` | 单边行情系统性偏离 | 权重是价格量纲，不是归一化系数 |
| `hampel_filter` | 横盘后第一根突破被"压住" | `mad = 0` 时任何偏离都判为异常 |
| `parabolic_sar` | 横盘反复翻面 | 加速因子机制固有 |

### 4.2 结果依赖图表环境

- **`alpha_decreasing_exponential_moving_average.pine`**：`alpha = 2/(bar_index+1)`，`bar_index` 由图表左边界决定 → 多加载/少加载历史、换周期，输出的曲线形状都会变。**不要用它做跨品种/跨周期的对比。**
- **所有 Ehlers 系脚本默认 `src = hl2`**（`ehlers_leading_indicator`、`ehlers_mesa_adaptive_moving_averages`、`ehlers_optimum_elliptic_filter`、`ehlers_modified_optimum_elliptic_filter`、`distance_coefficient_filter`、`ehlers_distance_coefficient_filter`）；其余脚本默认 `close`。混着叠加时注意输入不一致。
- **`middle_high_low_moving_average` 与 `fractal_adaptive_moving_average`** 用 `highest()/lowest()` 的默认 `high/low`，而不是 `close`。
- 全部 70 个脚本都是 `overlay=true` 的价格量纲输出（PSAR 也是价格）——**除了没有任何一个是振荡器**（包括那个叫 "oscillator" 的文件）。

### 4.3 版本迁移注意

本目录 70 个脚本已全部迁移到 v6，可直接复制使用。迁移时统一做了这些替换：`study()` → `indicator()`、`type=integer` → `input.int()`、`highest()` 单参数 → `ta.highest()`、裸颜色名 `green`/`red` → `color.green`/`color.red`、`transp=` → `color.new()`。

---

## 五、怎么选：按用途速查

| 目标 | 首选 | 理由 |
| --- | --- | --- |
| 最稳、最不容易出错 | `simple_moving_average`、`exponential_moving_average`、`weighted_moving_average` | 基线，任何"改良 MA"都该和它们对比 |
| 趋势跟踪要小滞后 | `hull_moving_average`、`t3_moving_average`、`double_exponential_moving_average`、`zero_lag_exponential_moving_average`、`jurik_moving_average` | 相消/预补偿类，滞后显著小于 EMA |
| 平滑优先、允许慢 | `wilder_modified_moving_average`、`triangular_moving_average`、`spencer_21_point_moving_average`、`henderson_weighted_moving_average` | 核宽、无过冲或有负权重 |
| 抗尖刺/坏 tick | `moving_median`、`hampel_filter`、`trimean`、`recursive_median_filter` | 排序或 MAD 判定，不受单根异常影响 |
| 自适应（少调参） | `kaufman_adaptive_moving_average`、`vidya_variable_index_dynamic_average`、`fractal_adaptive_moving_average`、`ehlers_mesa_adaptive_moving_averages`、`mcginley_dynamic` | 系数随行情自动变 |
| 想看"滤波器设计" | `ehlers_super_smoother_filter`、`butterworth_filter`、`gaussian_filter`、`ehlers_optimum_elliptic_filter` | 参数是截止周期，可按周期分析调 |
| 自己设计权重核 | `finite_impulse_response_filter` | 7 个系数全开放 |
| 要成交量 | `elastic_volume_weighted_moving_average` | 唯一带成交量的脚本 |
| 要挂告警 | 7 个 v6 脚本 + `adaptive_laguerre_filter` + `corrected_moving_average` | 只有这 9 个有 `alertcondition` |
| 要"信号"而不只是线 | `corrected_moving_average`（SMA/CMA 交叉）、`ehlers_mesa_adaptive_moving_averages`（MAMA/FAMA 交叉）、v6 家族的变色告警 | 其余脚本只能自己目视判断 |

### 阅读顺序建议

如果只是想理解这一目录，建议按这个顺序看（每步只引入一个新概念）：

1. `simple_moving_average` → `exponential_moving_average` → `weighted_moving_average`：三种最基础的核；
2. `triangular_moving_average` → `sine_weighted_moving_average`：核形状的作用；
3. `double_exponential_moving_average` → `zero_lag_exponential_moving_average` → `hull_moving_average`：减滞后的三种思路；
4. `kaufman_adaptive_moving_average` → `fractal_adaptive_moving_average` → `ehlers_mesa_adaptive_moving_averages`：自适应的三种信息源；
5. `ehlers_super_smoother_filter` → `butterworth_filter`：把均线当滤波器看。

---

## 六、术语小抄

| 术语 | 含义 | 在本目录的体现 |
| --- | --- | --- |
| **FIR**（有限冲激响应） | 只看最近 L 根，不递推 | SMA、WMA、TMA、FWMA、LSMA、FIRF… |
| **IIR**（无限冲激响应） | 用上一期输出递推，记忆无限 | EMA、DEMA、T3、JMA、Laguerre、BF、GF… |
| **α（alpha）** | 递推式里"新样本占多大比重" | `α = 2/(L+1)`（EMA）、`1/L`（WMA/Wilder）、`f(ER)`（KAMA）… |
| **极点（pole）** | IIR 传递函数的分母根，决定平滑/稳定性 | BF/ESSF/GF 的 `poles` 参数（2/3/4 极点） |
| **等效滞后** | 权重核重心到最新 bar 的距离 | 见 3.1 节 |
| **过冲 / 负权重** | 核里有负系数，趋势转折时冲过目标 | HWMA、Spencer、LSMA、SHMMA、QEMA/PEMA |
| **相位超前** | 输出比输入提前（用斜率外推换来） | ZLEMA、IIRF、MOEF、ELI |
| **增益** | 常数输入下的输出比例，必须为 1 才不抬价 | 所有脚本都做归一化（除以权重和/系数和） |
| **播种（seed）** | 递推第一根 bar 的初值处理 | `nz(x[1], src)`、`nz(..., src)`、`n <= length ? sma(...)` |

---

> 全部公式、默认参数、颜色与边界行为均已对照源码逐条核对；标"已核实/已验证"的数值（Spencer 权重和 320/350、Henderson 权重和 1、Farey 分母 5.5/32.5、正弦核和 `cot(π/2(L+1))`、CMA 闭式解、各滤波器 α）是用脚本独立算过的。
