---
title: 在线工具上线｜DC-DC 升降压（Buck/Boost）电感计算器
date: 2026-09-02 16:30:00
tags:
  - DC-DC
  - 电源设计
  - 在线工具
categories:
  - 工具教程
---

做开关电源设计时，电感选型是绕不开的一步：先按伏秒平衡算电感量区间，再去 E12/E24 标称系列里挑值，最后还要校核峰值电流与饱和电流。以前我把这些公式做成了 Excel，这次把它搬上了网页，手机和电脑打开浏览器就能用。

**工具地址（点击直接使用）：[https://chen-daoming.github.io/dcdc-calculator/](https://chen-daoming.github.io/dcdc-calculator/)**

也可以从本站顶部「工具栏 → DC-DC升降压电感」进入。

<!-- more -->

## 一、工具能算什么

页面顶部一键切换 **BUCK 降压** 和 **BOOST 升压** 两种拓扑，输入输入电压、输出电压、最大负载电流、开关频率和纹波率区间，即时给出：

- **电感量可选区间 Lmin ~ Lmax**（对应纹波率 γmax 与 γmin）；
- **首选标称电感**：按 IEC 60063 的 E12 系列就近推荐，并列出区间内全部可选的 E12/E24 标称值；
- 选定标称值后的**实际纹波电流 ΔIL、实际纹波率 r、峰值电流 Ipeak、谷值电流 Ivalley、电流有效值 Irms**；
- **饱和电流选型下限 Isat ≥ k × Ipeak**（余量系数 k 可调，默认 1.3）；
- Buck 模式额外给出输入电容参考值；
- CCM/DCM 模式判断：谷值电流大于 0 才是连续导通。

所有计算在浏览器本地完成，不联网、不上传数据。

## 二、计算原理

### 1. Buck 降压

![Buck 降压拓扑与公式](/images/dcdc-calculator/buck-topology.svg)

由伏秒平衡，导通期电感两端电压为 Vin−Vout，关断期为 Vout，可得占空比：

```
D = Vout / Vin
```

设纹波电流 ΔIL = r × Iout（r 为纹波率，工程上常取 0.2~0.4），则：

```
L = Vout × (1 − D) / (r × Iout × fsw)
  = Vout × (1 − Vout/Vin) / (r × Iout × fsw)
```

r 取得越小，需要的电感越大、纹波越小；r 越大电感越小但纹波和开关应力越大。因此用纹波率上下限 γmin、γmax 就能圈出一个**电感量区间 [Lmin, Lmax]**，标准标称值落在这个区间内都可用。

### 2. Boost 升压

![Boost 升压拓扑与公式](/images/dcdc-calculator/boost-topology.svg)

Boost 的电感位于输入端，电感平均电流等于输入平均电流。考虑变换效率 η：

```
D = 1 − Vin / Vout
ILavg = Iout × Vout / (Vin × η)
ΔIL = r × ILavg
L = Vin × D / (ΔIL × fsw)
```

当 η = 1（理想）时，化简为：

```
L = Vin² × (Vout − Vin) / (r × Iout × Vout² × fsw)
```

Boost 输入电流通常比输出电流大不少（例如 5 V 升 12 V，电感平均电流约为输出的 2.4 倍），这也是升压电路选电感时最容易踩的坑——**不能拿输出电流直接当电感电流**。

### 3. 峰值电流与饱和电流

![CCM 电感电流三角波](/images/dcdc-calculator/inductor-current.svg)

CCM 下电感电流是以 ILavg 为中心的三角波：

```
Ipeak   = ILavg + ΔIL / 2
Ivalley = ILavg − ΔIL / 2
Irms    = √(ILavg² + ΔIL² / 12)
```

电感饱和后磁导率骤降、感量崩塌，电流会失控，因此规格书的 **Isat 必须大于 Ipeak 并留 20%~50% 余量**；温升额定电流 Irms 则应大于电感电流有效值。

## 三、算例：12 V 降 5 V / 2 A / 400 kHz

切到 BUCK，填入 Vin=12 V、Vout=5 V、Iout=2 A、FSW=400 kHz、γmin=0.2、γmax=0.4：

| 结果项 | 数值 |
|---|---|
| 占空比 D | 41.7% |
| 电感区间 Lmin~Lmax | 9.12 µH ~ 18.23 µH |
| 区间内 E12 标称值 | 10 / 12 / 15 µH |
| 首选 | **12 µH**（最接近区间中心） |
| 12 µH 下实际纹波率 | 30.4% |
| 峰值电流 Ipeak | 2.30 A |
| 建议 Isat ≥（k=1.3） | 3.0 A |
| 输入电容参考 | 10.42 µF |

选型时就在 10/12/15 µH 里挑一颗 Isat ≥ 3 A、DCR 尽量小的屏蔽功率电感即可，例如 12 µH/Isat 3.5 A 档。

## 四、选型经验小结

1. **纹波率 r**：Buck 一般 0.2~0.4，Boost 可放到 0.3~0.5；追求小纹波取小 r，追求小体积取大 r。
2. **饱和电流 Isat**：≥ (1.2~1.5) × Ipeak，瞬态负载重、输入电压波动大的场合余量取上限。
3. **温升电流 Irms**：≥ 电感电流有效值，注意规格书通常标 20℃/40℃ 温升值两个口径。
4. **DCR 直流电阻**：直接决定铜损 I²R 和效率，同感量下优先 DCR 小的。
5. **屏蔽结构**：闭磁路屏蔽电感对外辐射小，高密度 PCB 优先；开磁路功率电感价格低但漏磁大。
6. **最恶劣工况校核**：Buck 在 Vin 最大时占空比最小、纹波最大；Boost 在 Vin 最小时占空比和电感电流最大。要按最恶劣输入条件算一遍。

## 五、工具入口

- 在线使用：[https://chen-daoming.github.io/dcdc-calculator/](https://chen-daoming.github.io/dcdc-calculator/)
- 本站导航：顶部「工具栏 → DC-DC升降压电感」
- 公式已用 15 组 Excel 设计样例逐组核对通过。后续计划补充输出电容纹波、效率估算等功能，欢迎收藏使用。
