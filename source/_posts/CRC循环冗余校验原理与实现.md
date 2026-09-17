---
title: CRC循环冗余校验：从原理到实现的完整指南
date: 2026-09-15 08:00:00
categories:
  - 嵌入式开发
  - 通信协议
tags:
  - CRC
  - 校验算法
  - 数据完整性
  - 嵌入式
---

# CRC循环冗余校验：从原理到实现的完整指南

## 什么是CRC？

**循环冗余校验**（Cyclic Redundancy Check，CRC）是一种根据数据产生简短固定位数校验码的信道编码技术，主要用来检测或校验数据传输或保存后可能出现的错误。

<!-- more -->

CRC是数据通信领域中最常用的一种差错校验码，其特征是信息字段和校验字段的长度可以任意选定。

### 为什么需要CRC？

在数据传输过程中，无论系统设计再完美，差错总会存在。这些差错可能导致：

- 比特差错：0变为1，或1变为0
- 突发错误：连续多个比特发生错误
- 数据损坏：整个数据包被破坏

CRC通过数学运算建立数据位和校验位的约定关系，能够高效检测传输错误。

## 模二运算基础

CRC的核心是**模二运算**，它与普通四则运算的关键区别是**不考虑进位和借位**。

![模二运算](/images/crc/xor-operation.svg)

### 模二运算特点

1. **模二加法 = 模二减法 = XOR运算**
2. 奇数个1相加得1，偶数个1相加得0
3. 模二乘除法与普通乘除法类似，但部分积/余数相加时用模二加

### 重要性质

对于模二除法，有三个关键性质：

1. 当余数位数小于除数位数时，除法停止
2. 当被除数位数小于除数位数时，商为0，被除数就是余数
3. 只要被除数或部分余数的位数与除数相同，且最高位为1，皆可商1

## CRC算法原理

CRC的基本思想是将传输的数据当做一个位数很长的数，将这个数模二除以另一个数，得到的余数作为校验数据附加到原数据后面。

![CRC校验原理](/images/crc/crc-principle.svg)

### 算法流程

1. **发送方**：
   - 在原始数据后添加r个0（r = 除数位数-1）
   - 对扩展后的数据进行模二除法
   - 余数即为CRC校验码
   - 发送：原始数据 + CRC码

2. **接收方**：
   - 收到数据后进行模二除法
   - 余数为0 → 无差错
   - 余数不为0 → 有差错，丢弃或请求重传

### 数学描述

设：
- 原始数据：D(x)
- 生成多项式：P(x)（最高次数为r）
- CRC码：R(x) = [2^r × D(x)] mod P(x)

则实际发送的数据：T(x) = 2^r × D(x) + R(x)

可以证明：T(x) mod P(x) = 0

## CRC计算示例

以CRC-8为例，演示完整的计算过程：

![CRC计算示例](/images/crc/crc-calculation.svg)

### 计算步骤

**原始数据**：1101011011
**生成多项式**：10011（对应 x⁴+x+1）

1. **添加4个0**：11010110110000
2. **模二除法**：
   - 每次取与除数相同位数的部分余数
   - 最高位为1则商1，进行XOR运算
   - 最高位为0则商0，继续下一位
3. **得到余数**：1110
4. **实际发送**：11010110111110

## 二进制系数多项式

CRC算法中，二进制数用**二进制系数多项式**描述：

例如：10011B → P(x) = x⁴ + x + 1

多项式的每一位对应一个系数：
- x⁴ 系数 = 1
- x³ 系数 = 0
- x² 系数 = 0
- x¹ 系数 = 1
- x⁰ 系数 = 1

## 常用CRC版本

![常用CRC版本对比](/images/crc/crc-versions.svg)

### 特殊说明

- **CRC-1** 就是奇偶校验
- 多项式均省略了最高位（1）
- 不同版本适用于不同应用场景

### 选择建议

| 应用场景 | 推荐版本 | 原因 |
|---------|---------|------|
| 短帧通信 | CRC-8 | 计算简单，开销小 |
| 工业总线 | CRC-16 | 平衡检错能力和开销 |
| 网络传输 | CRC-32 | 强检错能力 |
| 高速存储 | CRC-32C | 支持硬件加速 |

## CRC算法参数

![CRC算法参数](/images/crc/crc-parameters.svg)

### 参数详解

| 参数 | 说明 | 示例值 |
|------|------|--------|
| width | CRC码位宽 | 32 |
| poly | 生成多项式 | 0x04C11DB7 |
| init | 初始值 | 0xFFFFFFFF |
| refin | 输入反射 | true |
| refout | 输出反射 | true |
| xorout | 输出异或值 | 0xFFFFFFFF |

### 反射（Reflection）

反射是指将数据的位序反转：
- 输入反射：计算前将每个字节的位序反转
- 输出反射：计算后将CRC码的位序反转

例如：11010010 反射后为 01001011

## C语言实现

### CRC-32/ISO-HDLC 实现

```c
#include <stdio.h>

#define POLY 0x04c11db7
#define INIT 0xffffffff
#define XOROUT 0xffffffff

// 位反转函数
unsigned int reverse(unsigned int input) {
    unsigned int output = 0;
    for (unsigned int i = 1; i != 0; i <<= 1) {
        output <<= 1;
        if (input & 1) {
            output |= 1;
        }
        input >>= 1;
    }
    return output;
}

// CRC-32 计算
unsigned int crc32(unsigned char* addr, unsigned int num) {
    unsigned int crc = INIT;
    while (num-- > 0) {
        crc ^= reverse(*addr++);
        for (int i = 0; i < 8; i++) {
            if (crc & 0x80000000) {
                crc = (crc << 1) ^ POLY;
            } else {
                crc <<= 1;
            }
        }
        crc &= 0xffffffff;
    }
    return reverse(crc ^ XOROUT);
}

int main() {
    unsigned char input[] = {'1', '2', '3', '4', '5', '6', '7', '8', '9'};
    printf("check=0x%08x\n", crc32(input, sizeof(input)));
    return 0;
}
```

### 运行结果

```
check=0xcbf43926
```

与标准模型完全一致。

## 实际应用中的优化

### 查表法

直接计算法效率较低，实际应用中常用**查表法**：

1. 预先计算0-255所有字节的CRC值
2. 运算时按字节查表
3. 空间换时间，效率大幅提升

### 硬件加速

现代处理器通常提供CRC硬件指令：
- Intel SSE4.2
- ARM CRC32指令
- 显著提升计算速度

## 总结

CRC校验是数据通信中不可或缺的差错检测技术：

1. **原理清晰**：基于模二除法的数学原理
2. **检错能力强**：能检测所有奇数个错误、所有双比特错误、所有突发错误
3. **实现灵活**：有多种标准版本，适应不同场景
4. **性能可控**：可通过查表法、硬件加速等方式优化

掌握CRC原理，对于嵌入式开发、通信协议设计、数据存储等领域都具有重要意义。

## 参考资料

1. [CRC（循环冗余校验）原理](https://blog.csdn.net/weixin_44256803/article/details/105805628)
2. [CRC Catalog](http://reveng.sourceforge.net/crc-catalogue/)
3. [STC8G1K08 Datasheet](https://www.stcmicro.com/datasheet/stc8g1k08.pdf)
