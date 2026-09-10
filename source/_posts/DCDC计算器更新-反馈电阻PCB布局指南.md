---
title: DCDC计算器更新——反馈电阻PCB布局指南：Buck与Boost的接地差异
date: 2026-09-10 22:30:00
tags:
  - DCDC
  - PCB设计
  - 布局
  - 反馈电阻
  - Bambu Lab
categories:
  - 硬件设计
---

## 前言

在 DC-DC 电源设计中，计算出正确的分压电阻值只是第一步。如何在 PCB 上正确布局这些电阻，往往才是决定电源稳定性的关键。

我们的 [DC-DC 升降压计算器](/dcdc-calculator/) 新增了 **反馈电阻 PCB 布局指南**，帮助你理解 Buck 和 Boost 拓扑中 FB 接地的本质差异。

---

## 一句话核心

> **找到"平滑端"，FB 地就接在那里。**

---

## Buck 与 Boost 的 FB 布局对比

### Buck 降压

```
FB 采样点：输出小电容处
FB 接地：单点接输出电容地
平滑端：输出端（电感串联输出，电流连续平滑）
重点关注：输入电容、SW 节点、功率地
环路面积：输入环路最小化
```

**为什么？**

Buck 电路中，电感在输出侧，输出电流经电感后连续平滑。输出电容的地是干净的，噪声小。所以 FB 的参考地应该接到输出电容地，而非输入电容地（输入地有高频开关噪声）。

### Boost 升压

```
FB 采样点：输出小电容处
FB 接地：单点接输入电容地
平滑端：输入端（电感串联输入，电流连续平滑）
重点关注：输出电容、SW 节点、功率地
环路面积：输出环路最小化
```

**为什么？**

Boost 电路中，电感在输入侧，输入电流经电感后连续平滑。输入电容的地是干净的。所以 FB 的参考地应该接到输入电容地，而非输出电容地。

---

## 核心原理图解

```
BUCK:                        BOOST:
                            
Vin──[Cin]──[IC]──L──[Cout]──Vout    Vin──L──[IC]──[Cout]──Vout
         │    │                              │    │
        GND  GND                            GND  GND
         ↑    ↑                              ↑    ↑
      输入地(噪声) 输出地(干净)            输入地(干净) 输出地(噪声)
                            ↓                ↓
                    FB地接输出电容地     FB地接输入电容地
```

---

## 通用布局规则

| 规则 | 要求 |
|------|------|
| **FB 走线长度** | < 10mm，越短越好 |
| **分压电阻位置** | 靠近芯片 FB 引脚 |
| **远离噪声源** | 电感、SW 节点、续流二极管 |
| **包地屏蔽** | FB 走线两侧铺地铜皮 |
| **采样点** | 输出电容正端（而非电感引脚） |
| **接地隔离** | 信号地（AGND）与功率地（PGND）单点连接 |

---

## 常见错误

1. **FB 采样点在电感之前** — 采样到的是含开关噪声的电压
2. **FB 地接到功率地（PGND）** — 功率地噪声会耦合到反馈信号
3. **分压电阻远离 FB 引脚** — 走线过长像天线，容易拾取噪声
4. **FB 走线靠近 SW 节点** — 容性耦合把 SW 噪声传到反馈路径
5. **Buck 的 FB 地接输入电容地** — 输入地噪声大，输出地才是干净的
6. **Boost 的 FB 地接输出电容地** — 输出地噪声大，输入地才是干净的

---

## 实际效果

某客户项目遇到输出电压振荡（纹波 200mV），排查发现：
- R1/R2 放在输出端，FB 走线 15mm，从电感旁边绕过

改进后：
- R1/R2 移到芯片 FB 引脚附近
- FB 走线从芯片底下过孔到背面，单独走线从输出电容取样
- 走线两侧铺地屏蔽

**结果：纹波从 200mV 降到 30mV**

---

## 使用计算器

打开 [DC-DC 升降压计算器](/dcdc-calculator/)，切换到「分压电阻计算」页面，即可查看完整的布局指南。

计算器支持：
- E96/E24 电阻系列选择
- 多组电阻组合推荐
- 固定电阻反算
- Buck/Boost 布局对比

---

## 参考资料

1. [TI - Layout Optimization of 4-Switch Buck-Boost Converters](https://www.ti.com/lit/pdf/slvafj3)
2. [面包板社区 - BUCK DCDC电路PCB layout](https://mbb.eet-china.com/tech/t2/181917.html)
3. [凡亿课堂 - 一文将DCDC的Layout讲的明明白白](https://www.fanyedu.com/content/114640.html)
4. [CSDN - DCDC电源设计必看：FB反馈电阻布局的3个常见错误](https://blog.csdn.net/l3m4n/article/details/155403905)
5. [CSDN - DC-DC PCB布线参考](https://blog.csdn.net/icifan/article/details/141062340)
6. [ROHM - Feedback Path Wiring](https://techweb.rohm.com/product/power-ic/dcdc/4682/)
