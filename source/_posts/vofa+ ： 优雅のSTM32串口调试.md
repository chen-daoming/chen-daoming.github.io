---
title: VOFA+：优雅の STM32 串口调试
date: 2026-09-05 22:00:00
tags:
  - STM32
  - VOFA+
  - 串口调试
  - HAL库
  - PID
categories:
  - 嵌入式笔记
---

调电机的都懂：Keil 在线 Watch 半秒才刷一次，看寄存器还行，看波形纯纯折磨；换串口 printf，速度是上来了，可满屏滚动的数字根本看不出趋势。直到用上 **VOFA+**——一款把串口数据实时画成曲线的上位机，调 PID、看采样、对频谱就像看示波器一样直观。这篇笔记把我从零搭起来的 **STM32 + VOFA+ 调试环境**完整整理一遍：协议原理、原创驱动代码、硬件接线、实测效果和踩坑记录，全文代码可直接抄走编译。

<!-- more -->

## 一、先看疗效：调试方式的代差

![printf 调试之痛 vs VOFA+ 波形调试](/images/vofa-plus/01-debug-pain-compare.svg)

同样是每 10ms 打三个数，串口助手给你的是"数据长龙"，VOFA+ 给你的是三条实时曲线——超调多少、振荡几次、有没有丢步，一眼定位。它还有两个杀手锏：

- **波形控件自带 FFT 与直方统计**：看谐波、看噪声分布不用再导 MATLAB；
- **控件可绑定命令**：拖个滑块就能在线改目标值 / Kp，边调边看，闭环调参的体验直接起飞。

## 二、VOFA+ 是什么，两种协议怎么选

[VOFA+](https://www.vofa.plus/)（Very Optimal Data Analyser）是一款免费的上位机，Windows / macOS / Linux 全平台，核心特点是**协议极简 + 插件化**：协议和控件都以插件形式组织，官方协议只有两条，几分钟就能在自己单片机上跑通。

![VOFA+ 两种数据协议的帧格式](/images/vofa-plus/02-protocol-formats.svg)

| 协议 | 帧内容 | 优点 | 代价 | 适用 |
|---|---|---|---|---|
| **FireWater** | ASCII 文本，逗号分隔，`\r\n` 结尾 | printf 直接用，肉眼可读 | 浮点转字符串很吃 CPU，字节多 | 低速、少通道、调试早期 |
| **JustFloat** | float 数组按内存原样发送 + 4 字节帧尾 | 零格式化开销，带宽最小 | 不可读，必须用 VOFA+ 看 | 高频采样、多通道、FFT |

**我的习惯**：先把传感器数据用 FireWater 打通"能看"，确认数值合理后切 JustFloat 拉采样率做精细分析。下面把两种协议一次讲透。

## 三、JustFloat 原理：小端序与 +Inf 帧尾

![JustFloat 原理：小端序与帧尾](/images/vofa-plus/04-justfloat-memory.svg)

JustFloat 的设计非常聪明，三句话说完：

1. **小端序零转换**：STM32 是小端机，`float f = 3.1416f` 在内存里就是 `DA 0F 49 40` 四个字节，`memcpy` 出去就是协议要求的字节序，一次转换都不用做；
2. **帧尾用 +Inf**：帧尾固定 4 字节 `00 00 80 7F`，按 float 解释就是 `0x7F800000`——IEEE754 里的正无穷。正常采样数据永远算不出 +Inf，所以 VOFA+ 在字节流里一"闻"到它就知道一帧结束，**天然抗误码、自动切帧**；
3. **收发两端零解析**：发送端不格式化字符串，接收端按 4 字节重组成 float 直接画线。同样的数据，JustFloat 的字节数通常只有 FireWater 的一半，CPU 占用更是差一个数量级。

> 注意帧尾是 `00 00 80 7F`（小端的 0x7F800000），网上有些文章把字节顺序写反成 `7F 80 00 00`，那样 VOFA+ 是识别不出来的。

## 四、硬件清单与接线

![调试硬件与接线图](/images/vofa-plus/03-hardware-wiring.svg)

我的调试台配置（都是常见货，总成本不到 30 元）：

| 硬件 | 数量 | 作用 | 备注 |
|---|---|---|---|
| STM32F103C8T6 最小系统板（Blue Pill 小蓝板） | 1 | 被调试目标 | F4 系列带 FPU，高频采样更从容 |
| ST-Link V2 | 1 | SWD 下载调试 | 不占用串口，可同时在线仿真 |
| CH340 / CP2102 USB 转 TTL 模块 | 1 | 串口监看通道 | 接 VOFA+ 的"眼睛" |
| 杜邦线 | 若干 | 接线 | 公对母 4 根足够 |

接线只有三条有效规则：

| STM32 | USB 转 TTL | 说明 |
|---|---|---|
| PA9（USART1_TX） | RXD | **必接**，数据从这走 |
| PA10（USART1_RX） | TXD | 本例可不接，需要接收命令时再接 |
| GND | GND | **必须共地**，否则全是乱码 |

供电二选一：ST-Link 的 3.3V 或 USB-TTL 的 3.3V，不要两边同时供电。串口参数 **115200-8-N-1**。

## 五、CubeMX 配置与驱动代码

先在 CubeMX 里配三处：**SYS** 选 Serial Wire；**RCC** HSE 选晶振、主频拉到 72MHz；**USART1** 选 Asynchronous（115200/8/N/1），并在 DMA Settings 里给 USART1_TX 添加一条 DMA 通道（Mode: Normal，Priority: Medium），对应的中断保持默认使能。生成 HAL 工程后，把下面两个文件加进工程即可。

**vofa.h**

```c
#ifndef __VOFA_H
#define __VOFA_H

#include "stm32f1xx_hal.h"   /* F4 工程改成 stm32f4xx_hal.h */
#include <stdint.h>
#include <stdarg.h>

#define VOFA_UART     huart1       /* 使用的串口句柄，换串口只改这里 */
#define VOFA_MAX_LEN  128          /* 单帧最大字节数 */

void Vofa_FireWater(const char *fmt, ...);              /* 文本协议 */
void Vofa_JustFloat(const float *data, uint8_t num);    /* 二进制协议 */

#endif
```

**vofa.c** —— 这是我按官方协议重新实现的版本，相比网上流传的写法补了四个工程细节：自动补 `\r\n`、vsnprintf 截断保护、去掉 VLA（部分编译器不支持）、DMA 占线时丢帧计数而不是硬等：

```c
#include "vofa.h"
#include <stdio.h>
#include <string.h>

/* 帧尾：小端的 0x7F800000（+Inf），正常数据不会出现 */
static const uint8_t VOFA_TAIL[4] = {0x00, 0x00, 0x80, 0x7F};

uint32_t vofa_drop_cnt = 0;   /* 丢弃帧计数，调试时可在 Watch 里看 */

/* 交给 DMA 发送：占线则丢弃本帧，绝不阻塞控制环 */
static uint8_t Vofa_SendDMA(const uint8_t *buf, uint16_t len)
{
    if (HAL_UART_GetState(&VOFA_UART) == HAL_UART_STATE_BUSY_TX ||
        HAL_UART_GetState(&VOFA_UART) == HAL_UART_STATE_BUSY_TX_RX) {
        vofa_drop_cnt++;
        return 0;
    }
    return (HAL_UART_Transmit_DMA(&VOFA_UART, (uint8_t *)buf, len) == HAL_OK);
}

/* FireWater：printf 风格，函数内部自动补 \r\n，调用者不用关心帧尾 */
void Vofa_FireWater(const char *fmt, ...)
{
    char     buf[VOFA_MAX_LEN];
    va_list  args;
    int      n;

    va_start(args, fmt);
    n = vsnprintf(buf, sizeof(buf), fmt, args);
    va_end(args);

    if (n <= 0) return;                          /* 格式化失败 */
    if (n > (int)sizeof(buf) - 2)                /* 截断保护 */
        n = (int)sizeof(buf) - 2;
    buf[n++] = '\r';
    buf[n++] = '\n';

    Vofa_SendDMA((uint8_t *)buf, (uint16_t)n);
}

/* JustFloat：float 数组原样发送 + 帧尾，一帧 = num*4 + 4 字节 */
void Vofa_JustFloat(const float *data, uint8_t num)
{
    uint8_t  buf[VOFA_MAX_LEN];
    uint16_t bytes;

    if (num == 0 || num > (VOFA_MAX_LEN - 4) / 4) /* 通道数越界保护 */
        return;
    bytes = num * 4;

    memcpy(buf, data, bytes);       /* STM32 小端机，内存即协议 */
    memcpy(buf + bytes, VOFA_TAIL, 4);

    Vofa_SendDMA(buf, bytes + 4);
}
```

发送链路是"非阻塞"的：`HAL_UART_Transmit_DMA` 只是把缓冲区地址交给 DMA 就返回，剩下的搬运由硬件完成，CPU 继续跑控制环——这就是它敢塞进 1kHz 控制中断里的底气。

![非阻塞发送与数据分析](/images/vofa-plus/05-dma-flow-fft.svg)

带宽账也要会算：串口 8-N-1 每字节 10 位，**115200 bps ≈ 11.5 KB/s**。3 通道 JustFloat 一帧 16 字节 → 最多 **720 帧/s**，每通道 720Hz；想看更高频的信号就拉波特率到 921600（≈5760 帧/s），同时记得采样定理：采样率 ≥ 2×信号最高频率。

## 六、跑起来：两个最小演示

**演示 1：FireWater 画正弦 / 余弦**（main.c 的 while 循环里）：

```c
#include <math.h>

while (1)
{
    float t = HAL_GetTick() * 0.001f;
    Vofa_FireWater("%.3f,%.3f", sinf(t * 6.2832f), cosf(t * 6.2832f));
    HAL_Delay(10);                 /* 100 Hz 刷新 */
}
```

**演示 2：JustFloat + FFT 看谐波**——合成了一个"1Hz 基波 + 3 次谐波"的信号，在 VOFA+ 波形控件里点 FFT，能看到 1Hz 和 3Hz 两根谱线，和上图完全对应：

```c
float ch[3];
while (1)
{
    float t = HAL_GetTick() * 0.001f;
    ch[0] = sinf(t * 6.2832f);              /* 1Hz 基波        */
    ch[1] = 0.5f * sinf(t * 6.2832f * 3);   /* 3Hz 谐波        */
    ch[2] = ch[0] + ch[1];                  /* 合成：时域看失真 */
    Vofa_JustFloat(ch, 3);
    HAL_Delay(10);
}
```

VOFA+ 端的操作三步：**协议选 JustFloat → 端口选 CH340、波特率 115200 → 连接**，然后把通道拖进波形控件即可；演示 2 里对 ch2 开 FFT 就能看到频谱。（FireWater 演示则把协议切回 FireWater，其余不变。）

## 七、实战：PID 调参"看得见"

串口调试最大的价值在闭环控制。下面是我在 10ms 定时器中断里跑的增量式 PI——目标值是 ±2 的方波，被控对象用一阶惯性环节代替（换成编码器转速、ADC 温度同理），三个通道全部丢给 VOFA+：

```c
/* tim.c：10ms 周期中断回调 */
#define TS  0.01f
#define KP  1.2f
#define KI  0.8f

void HAL_TIM_PeriodElapsedCallback(TIM_HandleTypeDef *htim)
{
    static float y = 0.0f, u = 0.0f, e_prev = 0.0f;
    float r = ((HAL_GetTick() % 4000) < 2000) ? 2.0f : -2.0f;  /* 方波目标 */
    float e = r - y;

    u += KP * (e - e_prev) + KI * TS * e;      /* 增量式 PI */
    if (u >  3.0f) u =  3.0f;                  /* 执行器限幅 */
    if (u < -3.0f) u = -3.0f;
    e_prev = e;

    y += TS * (2.0f * u - y) / 0.5f;           /* 对象：K=2, T=0.5s */

    float ch[3] = {r, y, u};                   /* 目标 / 反馈 / 输出 */
    Vofa_JustFloat(ch, 3);
}
```

在 VOFA+ 里把 ch0（目标）和 ch1（反馈）叠在同一个波形控件，方波与响应曲线的偏差就是误差；觉得爬坡太慢就拖大 KI、振荡就减 KP，**每次修改立即反映在下一拍的曲线上**，比"改代码-编译-下载-再猜"的循环快了一个数量级。这也是我推荐每个嵌入式新人搭一套 VOFA+ 环境的原因。

## 八、常见坑速查

| 现象 | 原因 | 解决 |
|---|---|---|
| 终端乱码 / 曲线是锯齿噪声 | 波特率不匹配、晶振配置错（8M vs 12M） | 核对 115200 与 HSE 值 |
| JustFloat 模式满屏乱码 | 用文本思维看二进制流 | 协议必须选 JustFloat |
| 波形周期性断线 / 卡顿 | DMA 占线丢帧（`vofa_drop_cnt` 在涨） | 降采样率或提波特率 |
| 曲线全是 0 或平线 | memcpy 长度错、数组越界 | 检查 `num * 4` 与缓冲区大小 |
| 间隔性乱字符 | 没共地、TX/RX 接反 | 共地 + 交叉相接 |
| printf 浮点打印空白 | GCC 默认裁剪 `%f` | 链接选项加 `-u _printf_float`（Keil 勾选 MicroLIB） |
| 大端芯片画图错乱 | 字节序与协议不符 | 发送前手动反转每通道 4 字节 |

## 九、原创说明与参考

本文是学习 VOFA+ 后的原创整理：**硬件方案、CubeMX 配置流程、PID 实战与带宽分析为本人独立完成；`vofa.c/vofa.h` 参考官方协议文档与社区实现思路重新编写**，并补上了自动帧尾、截断保护、去 VLA、DMA 占线丢弃计数等工程化改进（感谢 [LiZhiY0u/vofa_for_stm32](https://github.com/LiZhiY0u/vofa_for_stm32) 的实现给出的启发）。数值口径以 [VOFA+ 官网](https://www.vofa.plus/) 协议说明为准。

1. [VOFA+ 官网（下载与协议说明）](https://www.vofa.plus/)
2. [知乎：vofa+ ： 优雅のSTM32串口调试](https://zhuanlan.zhihu.com/p/690064154)
3. [GitHub：LiZhiY0u/vofa_for_stm32（HAL 库驱动参考）](https://github.com/LiZhiY0u/vofa_for_stm32)

> 说明：文中五张示意图均为本人按原理重新绘制的 SVG，统一存放于仓库 `source/images/vofa-plus/` 目录。
