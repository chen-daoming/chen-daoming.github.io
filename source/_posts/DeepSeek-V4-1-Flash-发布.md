---
title: DeepSeek V4.1 Flash 发布：全新架构、原生多模态、更强更快更普惠
date: 2026-09-10 18:00:00
tags:
  - AI
  - DeepSeek
  - 大模型
  - MoE
categories:
  - AI 资讯
---

2026 年 9 月 10 日，深度求索（DeepSeek）正式发布 **DeepSeek V4.1 Flash** 模型。这是全新模型结构系列中的最小尺寸模型，具备原生多模态视觉理解能力，在基准测试中成功超越了包括 DeepSeek V4 Pro 在内的一众旗舰模型。

<!-- more -->

## 一、全新架构：非对称 Causal-Encoder-Decoder

![DeepSeek V4.1 Flash 架构示意](/images/deepseek-v41-flash/architecture.svg)

V4.1 Flash 采用了全新的 **Causal-Encoder-Decoder** 结构，输入和输出不对称：

| 参数 | 数值 |
|---|---|
| 总参数量 | 552B（MoE） |
| 输入激活 | 8B |
| 输出激活 | 16B |
| 多模态 | 原生支持（文本 + 图像） |
| 上下文窗口 | 1M tokens |
| 最大输出 | 384K tokens |

这种非对称设计的核心优势是：**输入端轻量化，输出端保持高能力**，成本显著低于同尺寸模型。

## 二、KV Cache 大幅压缩：Agent 场景成本骤降

![KV Cache 压缩进展](/images/deepseek-v41-flash/kv-cache.svg)

新一代模型在 KV Cache 缓存方面取得了显著突破：

| 对比项 | V4.1 Flash vs 上一代 |
|---|---|
| HBM 需求 | 减少到 **1/4** |
| SSD 需求 | 减少到 **1/8** |
| 相对初代模型 | KV Cache 缩小 **437 倍** |

在 Agent 使用场景中，缓存命中的费用往往占比较高。KV Cache 的压缩大幅降低了 Agent 类任务的使用成本。

## 三、性能全面超越 V4 Pro

DeepSeek 官方表示，V4.1 Flash 在**性能、费用、速度、总用时**等各项指标上已全面超越 V4 Pro：

| 对比维度 | V4.1 Flash | V4 Pro |
|---|---|---|
| 推理速度 | 社区实测 ~300-500 tok/s | 相对较慢 |
| 多模态 | 原生支持 | 待确认 |
| 成本 | Flash 级定价 | Pro 级定价 |
| 架构 | 全新 Causal-Encoder-Decoder | V4 架构 |

社区测试者报告的生成速度达到 **280-500 tokens/秒**，峰值甚至超过 500 tok/s。

## 四、API 定价调整

得益于模型架构创新，V4.1 Flash 能以更低成本服务更多用户。新价格于 **2026 年 9 月 10 日 12:00** 生效：

| Token 类型 | 闲时价格（¥/百万 token）| 高峰价格（¥/百万 token）|
|---|---|---|
| 缓存命中输入 | 0.05 | 0.10 |
| 未缓存输入 | 1.50 | 3.00 |
| 输出 | 4.50 | 9.00 |

> 高峰时段：周一至周五 09:00-12:00、14:00-18:00（北京时间），闲时价格为高峰时段的一半。

## 五、API 调用方式

V4.1 Flash 已同步上线 DeepSeek API，调用方式非常简单：

```python
from openai import OpenAI

client = OpenAI(
    api_key="your-api-key",
    base_url="https://api.deepseek.com"
)

response = client.chat.completions.create(
    model="deepseek-flash",  # 使用新模型名
    messages=[
        {"role": "user", "content": "你好，请介绍一下自己"}
    ]
)

print(response.choices[0].message.content)
```

**重要变更：**
- 模型名称更改为 `deepseek-flash` 即可调用最新 V4.1 Flash
- 旧版 V4 Flash 与 V4 Flash Vision Exp 已下线
- `deepseek-v4-flash`、`deepseek-v4-flash-vision-exp` 将暂时路由到 V4.1 Flash
- V4 Pro 请求将路由到 V4.1 Flash，按 Flash 单价计费

## 六、模型开源计划

DeepSeek 表示将全力支持开源社区进行新模型的推理适配：

- **HuggingFace：** [deepseek-ai/DeepSeek-V4.1-Flash](https://huggingface.co/deepseek-ai/DeepSeek-V4.1-Flash)
- **技术报告：** [DeepSeek_V41_Tech_Report.pdf](https://huggingface.co/deepseek-ai/DeepSeek-V4.1-Flash/blob/main/DeepSeek_V41_Tech_Report.pdf)

如有大规模部署需求且具备相应资源（2k 卡 GPU、存储集群），可联系 DeepSeek 官方。

## 七、对开发者的影响

![开发者迁移指南](/images/deepseek-v41-flash/migration.svg)

**建议迁移策略：**

1. **新项目**：直接使用 `deepseek-flash` 模型名
2. **现有 V4 Flash 用户**：可保持现有调用，自动路由到 V4.1 Flash
3. **V4 Pro 用户**：无需修改代码，自动享受 Flash 级定价
4. **自部署用户**：等待开源权重发布

**注意事项：**
- 腾讯（WorkBuddy、CodeBuddy）和 OpenCode 已全量接入 V4.1 Flash
- 由于模型重新训练，prompt 格式和输出风格可能有变化，建议重新测试现有应用
- V4.1 Flash 支持原生多模态，可直接处理图像输入

---

> **参考来源：**
> - [DeepSeek 官方公众号](https://mp.weixin.qq.com/s/qg0NU3NNUbp1co2PdkAPAg)
> - [HuggingFace 模型页面](https://huggingface.co/deepseek-ai/DeepSeek-V4.1-Flash)
> - [DeepSeek V4.1 Flash vs V4 Flash 对比分析](https://www.orcarouter.ai/blog/deepseek-v4-1-flash-vs-deepseek-v4-flash)
