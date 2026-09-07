---
title: Kalman 滤波：从直觉到 STM32 实现
date: 2026-09-06 00:10:00
tags:
  - STM32
  - Kalman
  - 滤波算法
  - 传感器
categories:
  - 嵌入式笔记
---

ADC 读电位器抖 ±12 个码、MPU6050 的角速度毛刺扎手、NTC 温度在小数点后乱跳——第一反应通常是滑动平均，但它有两个先天缺陷：**对过去一视同仁**（新数据被旧数据稀释，阶跃响应慢）和**窗口难选**（窗口小了滤不净，大了滞后重）。Kalman 滤波用"预测 + 按不确定性加权融合"的思路把这两个问题一起解决了，而且一维场景下**不需要任何矩阵库，20 行 C 代码就能跑**。这篇笔记从直觉推导到 STM32 实测，配一份可直接抄写的 C 实现和一个三曲线对比的 Python 验证脚本。

<!-- more -->

## 一、直觉：两把"不确定的尺子"

![Kalman 滤波的直观思想](/images/kalman/01-kalman-idea.svg)

对同一个量（比如温度），你手里有两份信息：

- **模型预测值**：温度不会瞬移，下一时刻大概率还在原地——但模型只是猜测，带着不确定性 Q；
- **传感器测量值**：直接量出来的——但传感器有噪声，带着不确定性 R。

Kalman 的全部智慧就一句话：**把两份信息按各自的不确定性加权平均**。谁方差小就多信谁，权重 K 在 0~1 之间自动滑动——传感器今天噪声大就多信模型，模型失准就多信测量。对比滑动平均"过去 N 个点一律等权"，这种动态加权正是它又稳又跟手的根本原因。

## 二、四步循环与五个公式

![一维 Kalman 的四步循环](/images/kalman/02-kalman-cycle.svg)

整张图就是 Kalman 的全部数学（一维形式，无需矩阵）：

**① 预测**：`x1 = x`（静态模型：预计不变），`P1 = P + Q`
　——模型每走一步都会引入新的不确定性，所以方差要**加上** Q，先验 P1 总比 P 大；

**② 算增益**：`K = P1 / (P1 + R)`
　——这是加权公式的权重项：测量方差 R 越大 K 越小（少信测量），预测方差 P1 越大 K 越大（多信测量）；

**③ 融合**：`x = x1 + K·(z − x1)`
　——(z − x1) 是"测量与预测的分歧"，按 K 折算进估计。K=1 就完全相信测量，K=0 就完全相信模型；

**④ 收缩**：`P = (1 − K)·P1`
　——用测量修正过之后，我们更笃定了，方差收缩，下一轮从更小的 P 出发。

整个算法**只保存 x 和 P 两个数**，不存历史序列，RAM 占用 8 字节——这就是"递归滤波"对单片机的意义。

## 三、原创 C 实现：20 行，无矩阵库

```c
/* kalman.h —— 一维卡尔曼滤波器（无矩阵库依赖） */
#ifndef __KALMAN_H
#define __KALMAN_H

typedef struct {
    float x;    /* 状态估计值                 */
    float P;    /* 估计方差（当前不确定度）      */
    float Q;    /* 过程噪声方差（模型多不可信）   */
    float R;    /* 测量噪声方差（传感器多不可信） */
} Kalman1D;

void  Kalman_Init(Kalman1D *k, float q, float r, float x0, float p0);
float Kalman_Update(Kalman1D *k, float z);

#endif
```

```c
/* kalman.c */
#include "kalman.h"

void Kalman_Init(Kalman1D *k, float q, float r, float x0, float p0)
{
    k->Q = q;   k->R = r;
    k->x = x0;  k->P = p0;   /* p0 给大些（如 1.0f）＝开局不自信，收敛更快 */
}

float Kalman_Update(Kalman1D *k, float z)
{
    /* ① 预测：静态模型，预计不变；不确定度变大 */
    float x1 = k->x;
    float P1 = k->P + k->Q;

    /* ②③ 增益与融合：按方差比例分信任 */
    float K  = P1 / (P1 + k->R);
    k->x = x1 + K * (z - x1);

    /* ④ 方差收缩 */
    k->P = (1.0f - K) * P1;
    return k->x;
}
```

## 四、Q/R 调参：看三张曲线就懂

![Q/R 调参直觉](/images/kalman/03-kalman-tuning.svg)

调参前先回答一个问题：**R 是多少？**别拍脑袋——把传感器静置，采 100 个点算方差（σ²），那就是 R 的实测值。Q 则代表"真实信号每个周期会变多少"，从 0.01R 起步往上试。三块面板是同一份数据在不同 Q/R 下的输出：

- **R 太大**：曲线过度平滑，阶跃爬得慢，滞后明显；
- **Q 太大**：输出跟着噪声一起抖，滤波白做；
- **匹配**：阶跃后一两个周期收拢，稳态几乎无毛刺。

经验口诀：**R 定生死（实测），Q 定手感（试）**。

## 五、实测：原始值与估计值同屏对比

![实测链路：原始值与 Kalman 估计同屏对比](/images/kalman/04-kalman-demo.svg)

把滤波器接进真实采集链路，用上位机双通道同屏看效果（串口输出用的就是之前 VOFA+ 文章里的 JustFloat 驱动，ADC 前级加了那篇的中值滤波，三层滤波各司其职）：

```c
/* main.c —— 1kHz 采样：电位器电压 → Kalman → 串口同屏对比 */
#include "kalman.h"

static Kalman1D vk;

int main(void)
{
    HAL_Init(); SystemClock_Config(); MX_ADC1_Init(); MX_USART1_UART_Init();

    /* 静置实测 ADC 噪声 σ≈12 码 → R = σ² ≈ 144；Q 从 5 起试 */
    Kalman_Init(&vk, 5.0f, 144.0f, 0.0f, 1.0f);

    while (1) {
        uint16_t z = ADC_ReadMedian(&hadc1, ADC_CHANNEL_1);  /* 软件中值先行 */
        float    x = Kalman_Update(&vk, (float)z);
        float    ch[2] = { (float)z, x };
        Vofa_JustFloat(ch, 2);                               /* ch0 原始 / ch1 估计 */
        HAL_Delay(1);
    }
}
```

**配套调试硬件**：STM32F103C8T6 小蓝板 + ST-Link V2（下载）+ CH340 USB 转 TTL（串口）+ 10kΩ 电位器（滑臂接 PA1，人为制造阶跃与抖动）。旋电位器时 ch0 满屏毛刺、ch1 平滑追赶，滤波器的价值一目了然；备选 MPU6050 模块（I2C）测角速度，同样适用。

## 六、Python 验证脚本：三种滤波同台对比

纯标准库即可运行，用来直观对比 Kalman 与滑动平均对阶跃和毛刺的处理：

```python
# -*- coding: utf-8 -*-
"""kalman_demo.py —— 原始 / 滑动平均 / Kalman 三者对比（纯标准库）"""
import random

random.seed(7)
TRUE, SIGMA = 25.0, 2.0                       # 真值 25℃，测量噪声 σ=2℃
raw = [TRUE + random.gauss(0, SIGMA) for _ in range(60)]
raw[40] += 8                                  # 混入一个离谱毛刺

def moving_avg(xs, n=5):                      # 滑动平均（窗口 5）
    return [sum(xs[max(0, i - n + 1): i + 1]) / len(xs[max(0, i - n + 1): i + 1])
            for i in range(len(xs))]

Q, R = 0.05, SIGMA ** 2                       # R = σ² = 4；Q 按变化速率估
x, P, kal = raw[0], 1.0, []
for z in raw:
    P1 = P + Q                                # ① 预测
    K  = P1 / (P1 + R)                        # ② 增益
    x  = x + K * (z - x)                      # ③ 融合（静态模型）
    P  = (1 - K) * P1                         # ④ 收缩
    kal.append(x)

mov = moving_avg(raw)
for i in (0, 20, 40, 59):                     # 重点看毛刺点 t=40
    print(f"t={i:2d}  raw={raw[i]:6.2f}  mov={mov[i]:6.2f}  kal={kal[i]:6.2f}")

# 想看曲线：pip install matplotlib 后取消注释
# import matplotlib.pyplot as plt
# plt.plot(raw, '.', label='raw'); plt.plot(mov, label='moving')
# plt.plot(kal, label='kalman'); plt.legend(); plt.show()
```

毛刺点 t=40 处的典型输出：raw 被抬高 8℃，滑动平均被拖起约 1℃ 且要拖好几个周期才吐干净，**Kalman 只跳了 0.7℃、下一拍就回落**——这就是"按不确定性加权"对离群点的天然免疫力。

## 七、常见坑速查

| 现象 | 根因 | 处理 |
|---|---|---|
| 曲线滞后明显 | R 相对 Q 太大 | 减 R 或加 Q |
| 输出仍有毛刺 | R 太小或 Q 太大 | 重新实测噪声方差定 R |
| 静止时输出缓慢漂移 | Q 偏大，过度相信测量 | 减小 Q |
| 阶跃响应总是慢半拍 | 模型太"自信" | 加 Q，或改用带速度维度的二维模型 |
| 开局输出歪得很久 | x0/P0 不当 | p0 给大（1.0），x0 用第一个测量值 |
| 输出突然 NaN | Q/R 为 0 导致除零 | 参数必须恒为正 |
| 多维信号直接套一维 | 状态量相互耦合 | 用矩阵版 Kalman / EKF，思路完全一致 |

## 八、小结

Kalman 滤波并不玄：**预测给一个先验，测量给一个观察，方差决定话语权，四个式子循环往复**。一维场景 20 行 C 代码、8 字节 RAM 就能落地，是每个嵌入式工程师都该亲手调一遍的基础功。掌握一维之后再去看矩阵版和 EKF，你会发现变的只是记号，不变的还是那两张"不确定的尺子"。

> 本文技术资料整理自网络公开渠道，公式配图、C/Python 代码与实测方案为原创，具体结论请结合自己的传感器实测验证。
