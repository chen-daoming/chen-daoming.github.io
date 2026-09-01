---
title: STM32 Keil在线调试：点三次全速运行才能进main
date: 2026-09-01 14:23:00
tags:
  - STM32
  - Keil
  - printf
  - 调试
categories:
  - 嵌入式
---

## 现象

使用 Keil 开发 STM32，**下载完程序后不能运行**。

在 `main()` 的入口处加打印，啥也没打出来——说明程序根本没跑到 `main()`。

在线调试（Debug）时发现了一个有意思的现象：**每次都是点击三次全速运行才能跑起来**，而且运行起来后可以正常进入 `main()`。

<!-- more -->

## 原因

一番折腾之后发现，原有工程中有一处用到了 `printf()` 函数，而串口打印我使用的是 HAL 库的函数，并没有使用 `printf()` 的方式。

> 关键点：`printf()` 在不做任何重定向、也不勾选 MicroLIB 的情况下，会被链接到**半主机（Semihosting）模式**。程序一旦执行到 printf 相关的代码，就会触发断点等待调试器响应，表现就是"反复点运行才能跑过去"。这也是为什么"每次都要点三次才能进 main"。

## 解决

把所有的 `printf()` 函数删掉即可。

如果确实要使用 `printf()`，推荐用 **MicroLIB + fputc** 的方式实现串口打印，三步走：

### 1. 包含头文件

在 `main.c` 文件中包含 `stdio.h`：

```c
#include "stdio.h"
```

### 2. 重定义 fputc 函数

在 `main.c` 中重定义函数，把 `printf()` 的输出重定向到串口：

```c
int fputc(int ch, FILE *f)
{
    USART_SendData(USART1, (unsigned char) ch);
    while (!(USART1->SR & USART_FLAG_TXE));
    return (ch);
}
```

这样在使用 `printf()` 时就会调用自定义的 `fputc` 函数来发送字符。

### 3. 勾选 Use MicroLIB

在工程属性的 **"Target" -> "Code Generation"** 选项中勾选 **"Use MicroLIB"**，重新编译即可。

## 补充说明

1. **为什么"点三次"**：未重定向的 `printf()` 走半主机模式，程序执行到打印处会等待调试器；勾选 MicroLIB 后 `printf` 不再依赖调试器，全速运行即可正常打印。

2. **示例代码用的是标准外设库（SPL）风格的 API**（`USART_SendData` / `USART1->SR` / `USART_FLAG_TXE`）。如果你的工程实际用的是 **HAL 库**，`fputc` 需要改成：

```c
int fputc(int ch, FILE *f)
{
    HAL_UART_Transmit(&huart1, (uint8_t *)&ch, 1, 0xFFFF);
    return ch;
}
```

其中 `huart1` 替换为你实际使用的串口句柄。
