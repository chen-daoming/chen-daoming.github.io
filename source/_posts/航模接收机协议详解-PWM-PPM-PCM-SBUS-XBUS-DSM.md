---
title: 一文打尽 PWM、PPM、PCM、SBUS、XBUS、DSM 接收机协议（附 STM32 TIM3 实现 PPM 输出）
date: 2026-09-02 10:00:00
tags:
  - 航模
  - STM32
  - 通信协议
categories:
  - 航模
---

> 本文整理自博客园文章《一文打尽PWM协议、PPM协议、PCM协议、SBUS协议、XBUS协议、DSM协议 | STM32的通用定时器TIM3实现PPM信号输出》（作者：蔡子CaiZi，2020-12-10），在原文基础上补充了协议对比表、帧结构表格与代码讲解。
> 原文链接：<https://www.cnblogs.com/cai-zi/p/14110204.html>

<!-- more -->

## 0. 先厘清一个概念边界

PWM、PPM、PCM、SBUS、XBUS、DSM 都是**接收机与其他设备（舵机、电调、飞控）之间通信的协议**。

注意不要和**遥控器 ↔ 接收机之间的协议**混淆：遥控器和接收机之间采用什么协议沟通，往往各厂牌自有一套、互不兼容；而**接收机输出给后端设备的信号是有通行标准的**，本文讨论的正是后者。

![航模接收机实物](/images/01-receiver-protocols.svg)

六种协议总览对比：

| 协议 | 调制/编码方式 | 物理连线 | 通道数 | 刷新周期 | 硬件接口 | 典型场景 |
|---|---|---|---|---|---|---|
| PWM | 脉宽调制 | 每通道 1 根线 | 1 路/线 | 20 ms（50 Hz） | GPIO/定时器 | 舵机、固定翼电调 |
| PPM(CPPM) | 脉冲位置调制 | 1 根线 | 最多 10 路 | 约 20 ms/帧 | 定时器输入捕获 | 教练线、模拟器、多轴飞控 |
| PCM | 脉冲编码调制（A/D、D/A） | — | — | — | 数字编解码 | 抗干扰要求高的遥控链路 |
| SBUS | 11 bit 串行总线 | 1 根线（负逻辑） | 16+2 路 | 4 ms / 14 ms | 串口 100 kbps 8E2 | 穿越机、航拍机飞控 |
| XBUS | 串行总线 | 1 根线 | 18 路 | 两种串口模式 | 串口 | JR/Spektrum 体系 |
| IBUS | 串行总线 | 1 根线（不取反） | — | — | 串口 115200 | FlySky 体系 |
| DSM2/DSMX | 数字扩频调制 | 串行 | — | — | 标准串口 | JR/Spektrum、卫星接收机 |

---

## 1. PWM 协议

### 1.1 基本原理

PWM（Pulse Width Modulation，脉宽调制）在航模中主要用于**舵机控制**，是一种古老而通用的工业信号。原理是通过周期性跳变的高低电平组成方波，进行连续数据输出。

![PWM 波形示意图：周期约 2ms 的方波，高电平宽度即信号量](/images/02-pwm-waveform.svg)

航模常用的 PWM 信号只使用了 PWM 的一部分功能：**只用高电平宽度承载信息，周期固定，忽略占空比参数**。

### 1.2 信号特征

1. PWM 是周期性方波，**周期 20 ms，即 50 Hz 刷新频率**；
2. 每周期内**高电平持续 1~2 ms（1000 µs~2000 µs）**，代表控制量。四旋翼中一般 **1100 µs 对应 0 油门，1900 µs 对应满油门**；
3. 舵机中位对应 1500 µs，两端极限对应 1000 µs / 2000 µs。

| 高电平脉宽 | 舵机位置 | 油门（多轴示例） |
|---|---|---|
| 1000 µs | 一端极限 | — |
| 1100 µs | — | 0 油门 |
| 1500 µs | 中位 | 中点 |
| 1900 µs | — | 满油门 |
| 2000 µs | 另一端极限 | — |

### 1.3 PWM 的优势

- **抗干扰**：全程满电压传输，非 0 即 1，类似数字信号，拥有数字信号的抗干扰能力；
- **可传模拟量**：脉宽连续可调，能传输连续的模拟控制量；
- **实现简单**：信号的发生和采集都很简单，现代数字电路用计数器即可产生/采集；
- **与电压无关**：电池电压下降、DCDC 纹波等因素不会干扰信号传输。

### 1.4 局限

PWM **每条物理连线只传 1 路信号**——需要几个通道就要几组线。直接驱动多个独立设备（固定翼各舵面、电调）没问题，但要把多通道集中送给飞控时接线就非常繁琐，于是有了 PPM。

---

## 2. PPM（CPPM）协议与 PCM 协议

### 2.1 PPM 原理

PPM（Pulse Position Modulation，脉冲位置调制，又称脉位调制）。航模 PWM 信号的高电平在整个时间轴上占比很小，绝大部分时间是空白；PPM 简单地把**多个通道的数值一个接一个合并进一根线，用两个相邻上升沿（高电平脉冲）之间的宽度表示一个通道的值**。

![PPM 帧波形与解码：一帧内 t1~t6 为 6 个通道脉宽，尾部 Synchro Blank Time 为同步空白](/images/03-ppm-frame.svg)

### 2.2 帧结构

- 一帧周期约 **20 ms**；
- 帧内依次排列各通道脉冲，通道间间隔宽度（1000~2000 µs）即该通道值；
- 每帧尾部必须插入一个**足够长的同步空白（Synchro Blank Time，显著超过正常 PWM 宽度）**分隔前后帧；
- 受 20 ms 帧长限制，**单帧最多传输 10 个通道**；
- 一个定时器即可完成解码/输出，是通行标准，绝大多数厂牌遥控/接收都支持。

典型应用：教练模式（两个遥控器相连）、遥控器接电脑玩模拟器、多轴接收机信号送飞控。

### 2.3 PCM 与 PPM 的区别；编码方式 ≠ 调制方式

PCM（Pulse-Code Modulation，脉冲编码调制，又称脉码调制）。常有人误把 PPM 编码说成 FM，这是两个层面的概念：

- **PPM / PCM 是信号脉冲的编码方式**；
- **AM / FM 是高频电路的调制方式（调幅/调频）**。

航模遥控器发射链路：

```
手柄电位器阻值变化 → 编码电路生成脉冲编码(PPM 或 PCM)
                  → 高频调制电路调制(AM 或 FM)
                  → 功放 → 天线发射
```

| 对比项 | PPM | PCM |
|---|---|---|
| 编解码实现 | 积分电路 | A/D 与 D/A 转换技术 |
| 常见解码芯片 | CD4013、CD4015 等通用数字 IC | 数字编解码 |
| 抗干扰 | 窄干扰可滤波，宽干扰无能为力 | 数字化处理，抗干扰更强 |

最常见组合：**PPM/AM（脉位调制/调幅）、PPM/FM（脉位调制/调频）、PCM/FM（脉冲编码/调频）** 三种。

### 2.4 PPM 的"抖舵"问题

普通 PPM 接收解码电路（如 CD4013/CD4015）只要输入脉冲上升沿达到一定幅度就会翻转：

- 信号中混入随机干扰脉冲 → 输出混乱 → **抖舵**；
- 电位器接触不良造成编码波形畸变 → 同样抖舵；
- 窄干扰脉冲可用滤波消除，**宽干扰脉冲滤波无效**；
- 强干扰环境、超出控制距离、同频干扰时，模型可能完全失控。

---

## 3. SBUS（S-BUS / S.BUS）协议

### 3.1 概述

SBUS（Serial Bus，串行总线）最早由日本厂商 **FUTABA** 引入，随后 FrSky 等厂商的接收机也广泛支持。它是全数字化接口总线：

- **数字化**：以现有数字通信接口（串口）为硬件基础，配专用软件协议，非常适合单片机/飞控；
- **总线化**：多个设备可通过一个 Hub 挂在同一总线上，各取所需控制信息；
- **一根线即可获取全部通道数据**，效率高、省硬件资源。

### 3.2 串口硬件参数（重点）

| 参数 | SBUS 取值 | 备注 |
|---|---|---|
| 硬件基础 | RS232C 串口 | TTL 电平 3.3 V |
| 逻辑 | **负逻辑**：低电平=1，高电平=0 | 必须硬件取反 |
| 波特率 | **100000 bps（100 K）** | **非标准波特率，不兼容 115200** |
| 数据格式 | 8 数据位、偶校验、2 停止位（**8E2**） | — |
| 高速模式 | 每 **4 ms** 一帧 | — |
| 低速模式 | 每 **14 ms** 一帧 | — |

> 100 K 不是标准波特率，普通电脑串口工具一般无法直接读取，需用单片机（或支持自定义波特率的设备）解析。

### 3.3 一帧 25 字节的结构

SBUS 一帧固定 **25 字节**：

```
[数据头 0x0F] [22 字节通道数据] [标志字节] [结束字节 0x00]
```

| 字节位置 | 内容 | 说明 |
|---|---|---|
| 字节 [0] | 0x0F | SBUS 帧头，固定 |
| 字节 [1..22] | 16 个伺服通道 | **每通道 11 bit**，共 16×11 = 176 bit = 22 字节 |
| 字节 [23] | 标志位 | 见下表 |
| 字节 [24] | 0x00 | 结束字节，固定 |

字节 [23] 标志位定义：

| 位 | 掩码 | 含义 |
|---|---|---|
| bit7 | 0x80 | 数字通道 17（开关量） |
| bit6 | 0x40 | 数字通道 18（开关量） |
| bit5 | 0x20 | 丢帧标志（为 1 时接收机红灯亮，本帧异常） |
| bit4 | 0x10 | 故障保护（failsafe）激活 |
| bit3~bit0 | — | 保留未用（n/a） |

### 3.4 通道值映射

每个通道 11 bit，范围 0~2047（2¹¹ = 2048 级精度），与 PWM 脉宽对应关系：

| 摇杆位置 | SBUS 值 | 等效 PWM 脉宽 |
|---|---|---|
| −100% | 173 | 1000 µs |
| 0%（中位） | 992 | 1500 µs |
| +100% | 1811 | 2000 µs |

### 3.5 编码原理

1. 每个通道是 11 bit 二进制数（如 `1111 1111 111`）；
2. 16 个通道按顺序紧密排成 176 bit：16 × 11 = 176 bit = **22 字节**；
3. 前面加帧头 0x0F、后面加标志字节和帧尾 0x00，共 25 字节；
4. 每 4 ms（高速）或 14 ms（低速）发送一帧。

> 通道 17/18 是只有高低电平的数字通道，一般用于通断控制，例如 1=启动电机、0=停止电机。

### 3.6 负逻辑与硬件取反（重点）

SBUS 采用负逻辑，**无论接收还是发送都必须硬件取反**（注意：要硬件取反，不是软件取反）。经典三极管取反电路：SBUS 信号经 1 kΩ 基极电阻驱动 NPN 三极管（Q1），集电极 10 kΩ 上拉至 3.3 V 后接 MCU 的 RXD。

![SBUS 硬件取反电路：RC_SBUS 经 1K 接三极管基极，集电极 10K 上拉 VCC3V3 后输出 RXD](/images/04-sbus-inverter.svg)

多数支持 SBUS 的飞控已板载反相器，接收机直接接飞控即可；**STM32F0 系列芯片内置反相电路，外围不用再加**。

### 3.7 XBUS 与 IBUS 补充

- **XBUS**：常规串行通信协议，支持 **18 个通道**，数据包较大；串口有两种模式，可在遥控器配置项中选择，接收机无需特殊配置。
- **IBUS**：不需要信号取反，波特率 **115200**（FlySky 富斯体系常用）。

---

## 4. DSM2 / DSMX 协议

DSM（Digital Spread Spectrum Modulation，数字扩频调制）共三代：**DSM → DSM2 → DSMX**，国内最常见 DSM2，JR 和 Spektrum 遥控器支持。

- 串行协议，但比 SBUS **更通用**：使用标准串口定义，兼容接收机便宜、兼容设备多（如电直三轴陀螺 VBar 可直接接收 DSM2）；
- **非总线化协议**：需要靠接收机把协议转成 PWM 驱动舵机；DSM2 接口只能连接收机和卫星接收机；
- **DSMX 是 DSM2 升级版**，协议基本一致、速率更快，并支持**双向传输**（把飞机上的信息回传遥控器液晶屏显示）。

---

## 5. 到底该选哪种协议？

- **固定翼玩家、不打算加飞控**：直接用 **PWM**，这不是问题；
- **无线教练线 / 无线模拟器 / 休闲多轴（航拍、穿越）**：选支持 **PPM** 输出的接收机，省去一团乱麻的连线，PPM 足够胜任；
- **追求极限穿越机低延迟，或正经航拍机（还要控云台等附加设备）**：**SBUS** 的低延迟和多通道优势明显，但需要配套支持 SBUS 的遥控/接收组合，可能意味着额外投入。

---

## 6. STM32 通用定时器 TIM3 实现 PPM 信号输出

### 6.1 实现思路

利用定时器更新中断不断改变自动重装载值（ARR），每次中断翻转一次 IO 电平：

- 定时器时钟 72 MHz，预分频 `PSC=71` → 计数频率 72 MHz/(71+1) = **1 MHz，即 1 个计数 = 1 µs**；
- ARR 依次装入 PPM 数组中的值，每个值就是一段电平持续的微秒数；
- 中断中先更新 ARR、再翻转引脚电平，数组循环输出即得到连续 PPM 帧。

PPM 数组结构（8 通道示例，共 18 个元素 = 1 + 8×2 + 1）：

```
500, 1000, 500, 1000, 500, 500, ... , 500, 8000
 │     │    │     │
低500  通道1 低500 通道2 ...                 帧尾同步空白 8000µs
```

即：每个通道 = 500 µs 低电平间隔 + 1000~2000 µs 高电平脉宽，最后一个 8000 µs 长空白作为帧同步分隔，整帧凑满约 20 ms。

### 6.2 tim.c

```c
#include "tim.h"
#include "main.h"

#define PPM_NUM 18  // PPM数组中的个数，这里是8通道 1+8*2+1

u16 PPM_Array[PPM_NUM] = {500,1000,500,1000,500,500,500,1000,
                          500,1000,500,1000,500,1000,500,1000,
                          500,8000};
u16 PPM_Index = 0;  // PPM数组索引号

/* TIM3的中断初始化 */
static void NVIC_Configuration(void)
{
    NVIC_InitTypeDef NVIC_InitStructure;
    NVIC_InitStructure.NVIC_IRQChannel = TIM3_IRQn;               // TIM3中断号
    NVIC_InitStructure.NVIC_IRQChannelPreemptionPriority = 0;     // 抢占优先级0
    NVIC_InitStructure.NVIC_IRQChannelSubPriority = 1;            // 子优先级1
    NVIC_InitStructure.NVIC_IRQChannelCmd = ENABLE;               // IRQ通道使能
    NVIC_Init(&NVIC_InitStructure);
}

/* PPM引脚初始化 */
static void PPM_GPIO_Configuration(void)
{
    GPIO_InitTypeDef GPIO_InitStructure;
    RCC_APB2PeriphClockCmd(RCC_APB2Periph_GPIOC, ENABLE);         // 使能GPIOC时钟
    GPIO_InitStructure.GPIO_Pin = PPM_Pin;
    GPIO_InitStructure.GPIO_Speed = GPIO_Speed_50MHz;
    GPIO_InitStructure.GPIO_Mode = GPIO_Mode_Out_PP;              // 推挽输出
    GPIO_Init(PPM_GPIO_Port, &GPIO_InitStructure);
}

/* TIM3初始化 */
void PPM_Init(void)
{
    TIM_TimeBaseInitTypeDef TIM_TimeBaseStructure;
    RCC_APB1PeriphClockCmd(RCC_APB1Periph_TIM3, ENABLE);          // 使能定时器3时钟

    TIM_TimeBaseStructure.TIM_Period = 500;                       // 初始ARR
    TIM_TimeBaseStructure.TIM_Prescaler = 71;                     // 72MHz/72=1MHz，1tick=1us
    TIM_TimeBaseStructure.TIM_ClockDivision = 0;
    TIM_TimeBaseStructure.TIM_CounterMode = TIM_CounterMode_Up;   // 向上计数
    TIM_TimeBaseInit(TIM3, &TIM_TimeBaseStructure);

    TIM_ITConfig(TIM3, TIM_IT_Update, ENABLE);                    // 允许更新中断

    PPM_GPIO_Configuration();
    NVIC_Configuration();
    TIM_Cmd(TIM3, ENABLE);                                        // 使能TIM3
    PPM = 0;                                                      // 初始输出低电平
}

/* TIM3中断服务子程序 */
void TIM3_IRQHandler(void)
{
    if (TIM_GetITStatus(TIM3, TIM_IT_Update) != RESET)
    {
        TIM_ClearITPendingBit(TIM3, TIM_IT_Update);               // 清除中断标志
        // 更新ARR，减1是给后几行程序执行留出时间
        TIM3->ARR = PPM_Array[PPM_Index] - 1;
        PPM = !PPM;                                               // 翻转电平
        PPM_Index++;
        if (PPM_Index >= PPM_NUM)
        {
            PPM_Index = 0;                                        // 一帧结束，从头循环
        }
    }
}
```

### 6.3 tim.h

```c
#ifndef __TIM_H
#define __TIM_H

#include "main.h"
void PPM_Init(void);

#endif
```

### 6.4 main.c

```c
/*
 * 定时器TIM3实现PPM输出
 */
#include "main.h"

/* 程序开始时运行一次 */
void setup(void)
{
    delay_init();                                                 // 延时函数初始化
    NVIC_PriorityGroupConfig(NVIC_PriorityGroup_2);               // 2位抢占+2位子优先级
    usart_init(115200);                                           // 串口1，115200 8N1
    PPM_Init();
}

/* 不断循环 */
void loop(void)
{
}

int main()
{
    setup();
    while (1) { loop(); }
}
```

### 6.5 main.h（引脚统一定义，便于移植）

```c
#ifndef __MAIN_H
#define __MAIN_H
#ifdef __cplusplus
extern "C" {
#endif

#include "stm32f10x.h"
#include "sys.h"
#include "delay.h"
#include "usart.h"
#include "tim.h"

/* 引脚定义：PPM 输出在 PC13 */
#define PPM_Pin         GPIO_Pin_13
#define PPM_GPIO_Port   GPIOC
#define PPM             PCout(13)

#ifdef __cplusplus
}
#endif
#endif
```

### 6.6 逻辑分析仪验证

用 Saleae Logic 接 PC13 抓取波形：单个高电平脉冲宽度 0.500 ms、周期 1.5 ms 处占空比 66.67%、频率约 666.67 Hz，时序准确，说明动态修改 ARR 的方式输出 PPM 可行。

![Saleae Logic 抓取的 PPM 输出波形](/images/05-ppm-logic-analyzer.svg)

---

## 7. 要点速记

1. **PWM**：20 ms 周期、1~2 ms 高电平、一线一通道，舵机/电调通用语言；
2. **PPM**：把多路 PWM 脉宽塞进一根线，帧尾长空白同步，最多 10 通道，定时器即可编解码；
3. **PCM** 是编码方式（A/D、D/A），**AM/FM** 是高频调制方式，两个维度不要混为一谈；
4. **SBUS**：100 kbps、8E2、负逻辑需硬件取反、25 字节/帧、11 bit×16 通道 + 2 数字通道、4/14 ms 帧周期，通道中位值 992；
5. **IBUS** 不取反、115200；**XBUS** 18 通道；
6. **DSM2/DSMX** 用标准串口、扩频体制，DSMX 速率更快且支持回传；
7. STM32 输出 PPM 的核心技巧：**1 µs 计数基准 + 更新中断里动态改 ARR + 翻转 IO**。

## 参考链接

- 原文：<https://www.cnblogs.com/cai-zi/p/14110204.html>（作者：蔡子CaiZi）
- SBUS 代码实现：<https://www.bilibili.com/read/cv5824207>
- 5iMX 相关资料：<http://www.5imx.com/portal.php?mod=view&aid=1351>
