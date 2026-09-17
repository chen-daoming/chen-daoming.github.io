---
title: 嵌入式 Linux C 代码规范：从 STM32 到 Linux 的编码习惯重塑
date: 2026-09-17 10:00:00
tags:
  - C语言
  - 代码规范
  - Linux
  - 嵌入式开发
categories:
  - 嵌入式开发
description: 整理自正点原子左忠凯的嵌入式 Linux C 代码规范化文档，涵盖排版、命名、函数、变量、宏等核心规范，适合从单片机转向 Linux 开发的工程师参考。
---

## 写在前面

做 STM32 开发的时候，代码规范往往被忽略——变量命名随心所欲，`//` 和 `/* */` 混用，大小写不分。但转向 Linux 开发后，这些问题会被放大：Linux 内核源码数千万行，没有统一的代码规范根本无法维护。

本文整理自正点原子左忠凯的《嵌入式 Linux C 代码规范化》文档，结合个人开发经验，总结出一份实用的编码规范速查手册。

<!-- more -->

---

## 一、核心原则

| 原则 | 说明 |
|------|------|
| **简单、明了、清晰** | 代码可读性高于性能，重点是给人看的 |
| **精简** | 函数尽量不超过 100 行，消除重复代码 |
| **与现有代码风格一致** | 维护他人代码时，遵循原有风格 |
| **减少封装** | 教学场景下，不过度封装第三方库 API |

> 套用 Linux CodingStyle 的一句话：「代码风格是因人而异的，但我希望在绝大多数事上保持这种态度。」

---

## 二、排版格式

### 2.1 缩进

- **使用 TAB 缩进，不要用空格**
- TAB 宽度建议 4 个字符（Linux 内核用 8，但 4 更常见）
- `switch` 和 `case` 对齐，`case` 不缩进

```c
switch (suffix) {
case 'G':
case 'g':
    mem <<= 30;
    break;
case 'M':
case 'm':
    mem <<= 20;
    break;
default:
    break;
}
```

### 2.2 代码行

- 每行不超过 **80 列**（高分屏可适当放宽）
- 一行只写一条语句
- `if`、`for`、`while`、`case` 等语句独占一行

```c
// ❌ 不规范
a = x+y; b = x-y;

// ✅ 规范
a = x + y;
b = x - y;
```

### 2.3 括号

**非函数程序块**（if/switch/for/while）：起始 `{` 放行尾，结束 `}` 放行首

```c
if (x is true) {
    we do y
}
```

**函数定义**：起始 `{` 放下一行行首

```c
int function(int x)
{
    body of function
}
```

**单行语句可省略大括号**：

```c
if (condition)
    action();
```

**但只要有一个分支超过一行，必须加括号**：

```c
if (condition) {
    do_this();
    do_that();
} else {
    otherwise();
}
```

### 2.4 空格

| 位置 | 规则 |
|------|------|
| `if`、`switch`、`for`、`while` 后 | 加空格 |
| `sizeof`、`typeof`、`__attribute__` 后 | 不加空格 |
| 指针 `*` | 靠近变量名：`char *linux_banner` |
| 二元/三元操作符两侧 | 加空格 |
| 一元操作符后 | 不加空格 |
| 逗号、分号后 | 加空格 |
| 注释 `/* */` 与内容间 | 加空格 |

---

## 三、注释

### 3.1 注释风格

- 使用 `/* ... */` 而非 `//`
- 注释告诉别人**代码做了什么**，而非**怎么做的**
- 不要过度注释

**多行注释风格**：

```c
/*
 * This is the preferred style for multi-line
 * comments in the Linux kernel source code.
 * Please use it consistently.
 */
```

### 3.2 文件信息注释

```c
/*************************************************
Copyright © company 1998-2024. All rights reserved.
File name:   文件名
Author:      作者
Version:     版本号
Description: 程序功能描述
Others:      其它说明
Log:         修改日志
*************************************************/
```

### 3.3 函数注释

```c
/*
 * @Description: 函数功能描述
 * @param 1     - 参数1说明
 * @param 2     - 参数2说明
 * @return      - 返回值说明
 */
```

---

## 四、标识符命名

### 4.1 命名规则

使用 **Unix 风格**：小写单词 + 下划线连接

```c
// ✅ 规范
int number_of_book;
void read_adc1_value(void);

// ❌ 不规范
int NumberOfBook;
void ReadADC1Value(void);
```

### 4.2 注意事项

1. **命名要清晰**，使用完整单词或通用缩写
2. **不要用汉语拼音**
3. **互斥操作用互斥词组命名**：

```
add/remove    begin/end      create/destroy
insert/delete first/last     get/release
lock/unlock   open/close     min/max
start/stop    next/previous  show/hide
```

4. **文件名全小写**
5. **变量名全小写**，避免匈牙利命名法
6. **宏名全大写**：`#define PI_ROUNDED 3.14`
7. **不要用单字母变量**，但允许 `i`、`j`、`k` 作循环变量

---

## 五、函数

### 5.1 核心原则

| 原则 | 说明 |
|------|------|
| 一个函数只做一件事 | 职责单一，便于理解和维护 |
| 消除重复代码 | 重复代码提炼成函数 |
| 函数间用空行隔开 | 提高可读性 |
| 嵌套不超过 4 层 | 嵌套太深不利于阅读 |
| 参数做合法性检查 | 防止野指针、越界等 |
| 错误返回要处理 | 不要忽略错误码 |
| 内部函数用 `static` | 限制作用域，避免命名冲突 |

### 5.2 goto 集中退出

Linux 内核大量使用 `goto` 实现函数集中退出，这在需要多处清理资源时非常方便：

```c
int fun(int a)
{
    int result = 0;
    char *buffer;

    buffer = kmalloc(SIZE, GFP_KERNEL);
    if (!buffer)
        return -ENOMEM;

    if (condition1) {
        while (loop1) {
            ...
        }
        result = 1;
        goto out_buffer;
    }
    ...

out_buffer:
    kfree(buffer);
    return result;
}
```

### 5.3 EXPORT_SYMBOL

函数的导出宏紧贴结束大括号：

```c
int system_is_up(void)
{
    return system_state == SYSTEM_RUNNING;
}
EXPORT_SYMBOL(system_is_up);
```

---

## 六、变量

### 6.1 核心原则

| 原则 | 说明 |
|------|------|
| 一个变量只做一件事 | 不要复用变量 |
| 少用全局变量 | 用 `static` 限定文件作用域 |
| 局部变量不要和全局变量重名 | 避免误解 |
| 使用前必须初始化 | 禁止未初始化变量作为右值 |
| 明确全局变量初始化顺序 | 分析启动时序 |
| 减少不必要的类型转换 | 转换可能改变数据意义 |

### 6.2 初始化示例

```c
// ❌ 不好的初始化
int num;
if (a)
    num = 3;
else
    num = 4;

// ✅ 较好的初始化
int num = a ? 4 : 3;
```

---

## 七、宏和常量

### 7.1 命名

- 常量宏：**全大写**，下划线连接
- 函数宏：可用小写，但能写成内联函数就不要写宏

```c
#define CONSTANT 0x12345

#define macrofun(a, b, c) \
    do { \
        if (a == 5) \
            do_this(b, c); \
    } while (0)
```

### 7.2 使用注意事项

1. **不要写影响控制流的宏**（如宏中包含 `return`）
2. **多条语句用 `do { ... } while (0)` 包裹**：

```c
// ❌ 错误写法
#define FOO(x) \
    printf("arg is %d\n", x); \
    do_something_useful(x);

// ✅ 正确写法
#define FOO(x) do { \
    printf("arg is %d\n", x); \
    do_something_useful(x); \
} while (0)
```

3. **表达式宏要加括号**：

```c
#define CONSTANT 0x4000
#define CONSTEXP (CONSTANT | 3)
```

4. **常量优先用 `const` 而非宏**

---

## 八、总结

| 维度 | 要点 |
|------|------|
| 排版 | TAB 缩进、80 列限制、括号位置、空格规则 |
| 注释 | `/* */` 风格、文件头注释、函数注释 |
| 命名 | Unix 风格、小写+下划线、宏全大写 |
| 函数 | 单一职责、goto 集中退出、static 限定 |
| 变量 | 一变量一用途、少用全局、必须初始化 |
| 宏 | do-while(0)、加括号、优先用 const |

---

## 参考资料

1. Linux 内核源码《CodingStyle》文档
2. 《代码整洁之道》
3. 《GNU 编码规范》
4. 《华为 C 语言编程规范》
5. 正点原子 左忠凯《嵌入式 Linux C 代码规范化》

---

> 本文整理自网络公开资料，结合个人开发经验总结，仅供学习参考。
