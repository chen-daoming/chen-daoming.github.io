---
title: PCB布线 Intel 规则
date: 2026-09-04 23:40:00
tags:
  - PCB
  - 硬件设计
  - PCB布线
  - Intel
  - 高速PCB
categories:
  - 硬件笔记
---

上一篇[《常见PCB布局困扰分析及PCB美学分享》](/2026/09/03/%E5%B8%B8%E8%A7%81PCB%E5%B8%83%E5%B1%80%E5%9B%B0%E6%89%B0%E5%88%86%E6%9E%90%E5%8F%8APCB%E7%BE%8E%E5%AD%A6%E5%88%86%E4%BA%AB/)聊完了布局，布局定稿之后就到了**布线**环节。如果说布局决定"连得好不好"，那布线解决的就是"连得对不对"：阻抗、等长、回流路径、过孔数量……每一条规则背后都是信号完整性（SI）、电源完整性（PI）或电磁兼容（EMC）的血泪教训。

这篇笔记以电子发烧友的 [Intel PCB 设计规则专题](https://www.elecfans.com/zt/393154/)为主线，结合 Intel 官方文档（[AN 315《高速 FPGA PCB 设计指南》](https://www.intel.com/content/www/us/en/content-details/655015/an-315-guidelines-for-designing-high-speed-fpga-pcbs.html)、[AN 766 Stratix 10 高速信号接口布局指南](https://docs.altera.com/r/docs/683132/current/an-766-intel-stratix-10-devices-high-speed-signal-interface-layout-design-guideline/recommendations-for-high-speed-signal-pcb-routing)、《Intel® Core™ Processor Datasheet》中的 Layout Guidelines 章节以及 [Resource & Design Center](https://www.intel.com/content/www/us/en/design/resource-design-center.html) 的公开资料），整理一份可直接落地的布线规则清单。

<!-- more -->

## 一、布线的三条主线：SI、PI、EMC

动手连线之前先明确：所有布线规则都可以归到这三条主线上——

1. **信号完整性（SI）**：阻抗控制、等长匹配、控制过孔数量、保证回流路径连续；
2. **电源完整性（PI）**：低阻抗电源回路、去耦电容就近、平面完整；
3. **电磁兼容（EMC）**：3W 间距、20H 内缩、时钟包地、减小回路面积。

正确的流程是"**先约束、后布线、再仿真**"：在 EDA 工具的约束管理器里先把阻抗目标、线宽线距、等长公差写成规则，让 DRC 全程兜底，而不是靠肉眼一条条对照。

## 二、阻抗控制：一切高速规则的起点

| 信号类型 | 目标阻抗 | 公差 |
|---|---|---|
| 单端信号（普通高速线） | 50 Ω | ±10% |
| 差分对（USB） | 90 Ω | ±10% |
| 差分对（PCIe / 以太网） | 100 Ω | ±10% |

阻抗由**线宽、介质厚度、铜厚、介电常数**共同决定：先用 Polar Si9000 等阻抗计算工具根据板厂叠层参数定出线宽，首板回来后再做 TDR 抽测确认。**没有受控阻抗，后面所有等长、过孔规则都失去了意义。**

## 三、3W 与 20H：两条最经典的经验法则

![3W 与 20H 规则示意](/images/pcb-routing-intel/01-3w-20h-rule.svg)

- **3W 规则**：信号线间距（边到边）≥ 3 倍线宽，可显著降低近端/远端串扰。时钟、复位、模拟小信号以及差分对之间**优先满足**；噪声源与敏感线之间可进一步加大到 4W~5W。
- **20H 规则**：电源平面相对相邻地平面边缘内缩 ≥ 20 倍层间介质厚度（H），抑制电源平面边缘的场泄漏，降低板边 EMI 辐射。四边对称内缩，配合屏蔽地过孔效果更佳。

## 四、差分对规则

![差分对布线规则示意](/images/pcb-routing-intel/02-diff-pair-rules.svg)

| 项目 | 要求 | 原因 |
|---|---|---|
| 对内长度偏差 | ≤ 5 mil | 保证两线同步翻转，抑制共模噪声，眼图对称 |
| 对间长度偏差 | ≤ 15 mil | 同组信号同步切换，组内相位一致 |
| 对内间距 S | ≥ 2W（W 为单根线宽） | 稳定差分阻抗 |
| 参考平面 | 必须连续完整 | 跨分割会导致阻抗突变、回流路径断裂、EMI 恶化 |

蛇形绕线补偿对内偏差时，绕线幅值至少 3W，避免蛇形自身引入新的耦合。

## 五、DDR 布线规则

![DDR 等长匹配与拓扑选择](/images/pcb-routing-intel/03-ddr-matching.svg)

| 项目 | 要求 |
|---|---|
| DQ / DM 组内等长 | 互差 ≤ 25 mil |
| DQS 与所属 DQ 组 | 长度偏差 ≤ 10 mil |
| DQS 差分对 | 对内 ≤ 5 mil，且尽量镜像对称 |
| 地址 / 控制线 | 以时钟为基准，偏差 ≤ 50 mil |
| 拓扑 | 数据组用 T 型 / 菊花链；地址、命令、时钟用 Fly-by |

Fly-by 拓扑下时钟沿链条依次到达各颗粒，靠 DDR 控制器的 Write Leveling / 读训练来补偿传播延迟——这不是误差，而是 Intel 等厂商推荐的标准做法。

## 六、过孔：高速信号的"楼梯"，越少越短越好

![叠层设计与过孔背钻示意](/images/pcb-routing-intel/04-stackup-backdrill.svg)

- **≥ 1 GHz 的高速信号，换层过孔不超过 2 个**；每个过孔都是阻抗不连续点；
- 长插孔的**残桩（stub）会形成谐振腔**，在特定频率吞噬信号能量——10 Gbps 以上接口必须背钻（backdrill）或改用盲埋孔；
- 过孔在平面层要有足够的**反焊盘（Anti-pad）隔离**；
- 制造下限：孔径 ≥ 8 mil，焊盘 ≥ 16 mil（常规工艺）。

叠层方面，Intel 官方指南反复强调两点：**每个信号层紧邻一个完整参考平面**；相邻信号层的走线方向相互垂直，减小层间串扰。阻抗公差按 ±10% 控制，与板厂确认叠层后首板 TDR 抽测。

## 七、时钟与时钟包地

![时钟包地与去耦电容](/images/pcb-routing-intel/05-clock-guard-decoupling.svg)

- 关键时钟（系统时钟、参考时钟）**优先走内层带状线**，上下有完整平面屏蔽；
- 时钟两侧**包地（GND 护线）**，地过孔间距 ≤ λ/10；工程上的经验值是**每 1000 mil 至少打 1 个地过孔缝合**；
- 整条时钟线**少打过孔**，远离板边、远离接口和电源边界。

## 八、电源完整性与去耦

- **每个电源引脚至少 1 个去耦电容**，距离引脚 ≤ 150 mil，电容的接地端就近落地过孔；
- 大小电容并联使用（0.1 μF ~ 10 μF），配合电源平面构成低阻抗 PDN；
- 芯片底部热焊盘（thermal pad）必须打**散热过孔阵列**：孔径 8~10 mil、间距约 1 mm，盘内孔要塞孔 + 电镀填平，防止焊接时锡膏流失；
- **上电时序**：先 Vcore 后 VCCIO（不同器件要求不同，以具体 Datasheet 的 Power Sequence 章节为准——这是 Intel 处理器/FPGA 手册里被引用最多的表格之一）。

## 九、可制造性（DFM）红线

| 项目 | 常规工艺下限 |
|---|---|
| 线宽 | ≥ 4 mil（BGA 扇出区域可到 3 mil） |
| 孔径 / 焊盘 | ≥ 8 mil / ≥ 16 mil |
| 阻焊桥 | ≥ 2 mil |

另外注意：铜箔分布尽量平衡（避免某一层大面积铺铜而对应层空旷导致翘板）；清理孤立铜皮（孤岛）；补泪滴（teardrop）提高 pad-孔连接可靠性；导线拐角用 45° 或圆弧，避免直角。

## 十、仿真验证与出厂检查清单

布线完成后，先用 Intel 提供的 IBIS/AMI 模型跑一轮后仿真：串扰、眼图、SSN、PDN 阻抗扫描，全部收敛后再出 Gerber。最后过一遍这份 5 项清单：

- [ ] 所有高速信号阻抗符合目标值 ±10%；
- [ ] 差分对内偏差 ≤ 5 mil，DDR 组内 ≤ 25 mil；
- [ ] 无信号跨分割，回流路径完整；
- [ ] 去耦电容距引脚 ≤ 150 mil，PDN 阻抗达标；
- [ ] DRC 全绿：线宽 / 间距 / 孔径满足板厂制造能力。

## 参考资料

1. [电子发烧友：pcb 布线 intel 规则专题](https://www.elecfans.com/zt/393154/)
2. [Intel Resource & Design Center（官方设计资料中心）](https://www.intel.com/content/www/us/en/design/resource-design-center.html)
3. [Intel AN 315：Guidelines for Designing High-Speed FPGA PCBs](https://www.intel.com/content/www/us/en/content-details/655015/an-315-guidelines-for-designing-high-speed-fpga-pcbs.html)
4. [Intel AN 766：Stratix 10 高速信号接口布局设计指南](https://docs.altera.com/r/docs/683132/current/an-766-intel-stratix-10-devices-high-speed-signal-interface-layout-design-guideline/recommendations-for-high-speed-signal-pcb-routing)
5. [Intel/Altera Board Developer Center（高速 PCB 设计指南合集）](https://www.altera.com/design/guidance/board-developer)
6. 《Intel® Core™ Processor Datasheet》Layout Guidelines 章节（需在 Resource & Design Center 登录获取）

> 说明：文中五张示意图均为按上述规则重新绘制的 SVG，统一存放于仓库 `source/images/pcb-routing-intel/` 目录。文中数值为行业通用/Intel 指南的典型推荐值，具体项目请以所用器件的官方手册为最终依据。
