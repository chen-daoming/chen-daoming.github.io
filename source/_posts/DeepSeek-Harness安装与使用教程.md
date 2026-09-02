---
title: DeepSeek Harness 安装与使用教程（含 Node.js/npm 与 API Key 前置教程）
date: 2026-09-02 14:00:00
tags:
  - AI Agent
  - DeepSeek
  - 效率工具
categories:
  - 工具教程
---

DeepSeek Harness 是 DeepSeek 推出的首款 Agent 产品，它把大模型变成能够管理项目、编排多 Agent、执行长任务的智能体。本文从零基础讲起：先补齐 **Node.js/npm 安装**和 **DeepSeek API Key 获取**两个前置环节，再讲 Harness 的安装、模型配置、日常使用与高级功能，跟着做一遍即可在本地搭好一个开源 AI Agent 工作台。

<!-- more -->

## 0. 它是什么：Model + Harness = Agent

DeepSeek Harness 于 2026 年 8 月发布开发者预览版（v0.1），源代码以 MIT 协议在 GitHub 开源，并提供 npm 包 `@deepseek-ai/dsh`。它与 Claude Code、OpenAI Codex 属于同类产品，区别在于：**可自由接入近 40 家模型厂商，且完全开源**。

Harness 原意为"马具、缰绳"，在 AI 架构中指连接模型与真实执行环境的工程层：模型是负责推理的"大脑"，Harness 是负责调度上下文、工具、任务状态与反馈的"执行层"。缺少 Harness 的模型只能对话回答问题，加上 Harness 才能读写文件、执行命令、完成多步骤真实任务。

![Model + Harness = Agent 架构图](/image/deepseek-harness/01-agent-arch.svg)

核心功能一览：

| 功能 | 说明 |
|---|---|
| 管理项目 | 以项目为单位组织任务与文件，支持会话恢复、分支、检索与回放 |
| 长任务协作 | 通过计划、目标与工作流机制处理跨时长的复杂任务 |
| 多 Agent 编排 | 支持子 Agent 调度，多个 Agent 分工协作 |
| 上下文管理 | 管理上下文注入与压缩，记录系统提示词、思维链、工具调用与结果 |
| 联网搜索 | 内置文件搜索与联网搜索工具 |
| Skill 技能 | 以插件形式组合技能，可自定义 Agent 预设 |

内置四种 Agent 预设模式：

| 模式 | 标识 | 适用场景 |
|---|---|---|
| 极简模式 | `minimal` | 简单修改与测试，仅保留 shell 与文本编辑，用于基准测试 |
| 标准模式 | `standard` | 日常写代码、修 Bug、分析项目，工具集完整 |
| 代码模式 | `code` | 大批量搜索、并行读取与多步骤自动处理 |
| 创造模式 | `cordis` | 试验、组合 Cordis 插件，创建新预设 |

## 1. 全流程总览

完整链路如下，本文按此顺序展开：**安装 Node.js/npm → 获取 API Key → 安装启动 dsh → 配置模型 → 选择工作区 → 运行任务 → 会话复盘**。

![DeepSeek Harness 完整使用流程图](/image/deepseek-harness/02-full-workflow.svg)

环境要求：

| 环境 | 要求 | 说明 |
|---|---|---|
| Node.js | 建议最新 LTS（v20/v22） | Harness 基于 Node.js 运行 |
| npm | 随 Node.js 自动安装 | 用于安装 `@deepseek-ai/dsh`，npx 也由它提供 |
| API Key | DeepSeek 开放平台 | 以 `sk-` 开头，也可换成其他厂商的 Key |
| 硬件 | 普通电脑即可 | 模型推理在云端，本地不需要独立显卡 |

---

## 2. 前置教程一：安装 Node.js 与 npm

### 2.1 三者关系先理清

- **Node.js**：让 JavaScript 能在电脑本地运行的运行时；
- **npm**：Node.js 自带的包管理器，负责从网络下载安装工具包；
- **npx**：npm 自带的运行器，可以**不全局安装**直接临时运行一个包（Harness 推荐的快速启动方式就用到它）。

安装 Node.js 后，npm 和 npx 自动就绪，无需单独安装。

### 2.2 方式 A：官网安装包（新手推荐）

1. 打开 Node.js 官网 `https://nodejs.org/`，下载 **LTS（长期支持版）** 的 Windows Installer（`.msi`，64 位）；
2. 双击安装，安装路径默认即可，**务必保持 "Add to PATH" 勾选**；
3. 一路 Next 完成安装（其中 "Tools for Native Modules" 可勾选也可跳过）；
4. **关闭并重新打开**一个终端（PowerShell 或 CMD），让 PATH 环境变量生效。

### 2.3 方式 B：nvm-windows 多版本管理（开发者推荐）

如果以后要在多个 Node 版本间切换，用 nvm-windows 更方便：

1. 在 GitHub 搜索 `nvm-windows`，下载 `nvm-setup.exe` 安装；
2. 以**管理员身份**打开终端，执行：

```powershell
nvm install lts        # 安装最新 LTS 版 Node.js
nvm list               # 查看本机已安装的所有版本
nvm use 22.23.2        # 切换并启用指定版本（换成你实际装的版本号）
```

### 2.4 验证安装与国内镜像配置

重新打开终端，执行以下两条命令，能各自输出版本号即说明环境就绪：

```powershell
node --version         # 例如 v22.23.2
npm --version          # 例如 10.9.8
```

![Node.js 与 npm 安装路径图](/image/deepseek-harness/03-nodejs-npm-install.svg)

**国内网络强烈建议先切换 npm 镜像源**，否则后续安装 dsh 可能很慢甚至超时：

```powershell
# 切到国内镜像（npmmirror，原淘宝镜像）
npm config set registry https://registry.npmmirror.com

# 验证当前源
npm config get registry

# 以后想恢复官方源
npm config delete registry
```

常见问题：

- **输入 node 提示"不是内部或外部命令"**：安装时没勾 Add to PATH，或没重开终端；重装并勾选、重开终端即可。
- **权限报错（Windows）**：用管理员身份运行终端，或把 npm 全局目录改到用户目录。
- **安装卡住不动**：基本都是网络问题，先按上面切换镜像源再试。

---

## 3. 前置教程二：获取 DeepSeek API Key

Harness 本身免费开源，但它调用云端模型时需要一个模型服务的 API Key，Key 里的余额按 token 计费。

### 3.1 获取步骤

1. **注册登录**：打开 DeepSeek 开放平台 `https://platform.deepseek.com/`，用手机号注册登录；
2. **实名认证**：按页面提示完成个人实名认证（调用 API 的前提）；
3. **账户充值**：进入充值页面充入少量金额（5~10 元足够体验很久，DeepSeek 单价很低）；
4. **创建密钥**：左侧菜单进入「API keys」，点击创建，给它起个备注名（如 `harness-local`）；
5. **立即复制保存**：生成的 Key 以 `sk-` 开头，**完整内容只显示这一次**，关闭弹窗后无法再查看，丢了只能重新创建。

![DeepSeek API Key 获取流程图](/image/deepseek-harness/04-apikey-flow.svg)

### 3.2 安全红线

- 绝不要把 Key 硬编码进会上传 Git 的代码或配置文件；推荐用环境变量保存：

```powershell
# Windows PowerShell 永久设置用户环境变量
[Environment]::SetEnvironmentVariable("DEEPSEEK_API_KEY","sk-你的key","User")
# 设置后重开终端生效，读取：$env:DEEPSEEK_API_KEY
```

- 怀疑 Key 泄露时，立刻在开放平台删除该 Key 并重新创建；
- Harness 会把 Key 保存在仅当前用户可读写的凭据文件中（见 5.2 节）。

### 3.3 计费说明

- Harness 软件：MIT 开源，**免费**；
- 模型调用：按所选模型的 token 用量从 API 账户余额扣费；
- 也可以不使用 DeepSeek，改用 Kimi、OpenAI、Anthropic、Google 等其他厂商的 Key，配置方法相同。

---

## 4. 安装与启动 DeepSeek Harness

### 4.1 方式 A：npx 免安装启动（最快体验）

无需全局安装，终端任意目录执行：

```powershell
npx @deepseek-ai/dsh web
```

首次运行 npx 会自动从 npm 拉取 `@deepseek-ai/dsh` 包并启动 Web UI，之后再启动会直接使用本地缓存。

### 4.2 方式 B：全局安装（长期使用推荐）

```powershell
# 全局安装
npm install -g @deepseek-ai/dsh

# 以后任意位置直接用 dsh 命令启动
dsh web
```

### 4.3 方式 C：从源码运行（开发者/贡献者）

想体验最新开发版或参与开发，可以克隆源码：

```powershell
git clone https://github.com/deepseek-ai/deepseek-harness.git
cd deepseek-harness
pnpm install
pnpm run build
pnpm dsh web
```

> 开发者预览版迭代很快，可能出现破坏性变更，正式使用建议固定版本号。

### 4.4 启动成功的标志

终端会输出本地访问地址，默认：

```
http://127.0.0.1:3080
```

用浏览器打开即可看到 Harness 的 Web 界面。若 3080 端口被占用，按终端提示换端口访问即可；本地地址只有你自己的电脑能打开，无需公网。

---

## 5. 配置模型

### 5.1 在 Web 界面配置（推荐）

打开 `http://127.0.0.1:3080`，进入 **Settings → Models**：

1. Provider（模型提供方）选择 **DeepSeek**（也可选其他近 40 家厂商，或自定义 OpenAI 兼容接口）；
2. 粘贴第 3 节获取的、以 `sk-` 开头的 API Key；
3. Base URL 默认 `https://api.deepseek.com`，用官方服务不用改，只有用第三方中转时才改；
4. 选择默认模型（如 `deepseek-chat`）；
5. 点击 **Save** 保存，**即时生效，无需重启服务器**。

![Web 界面配置模型线框图](/image/deepseek-harness/05-webui-models.svg)

### 5.2 凭据文件

API Key 和可选的自定义接口地址保存在：

```
$DSH_HOME/.credentials.yaml
```

该文件权限为仅当前用户可读写（owner-only）。熟悉配置的话也可以直接编辑它。Windows 上 `$DSH_HOME` 默认为用户目录下的 `.dsh`（或 `.config/dsh`，以实际生成位置为准）。

### 5.3 命令行检查最终配置

不用启动界面，也可以在终端查看"多层配置叠加后的完整配置树"，用于排查配置是否生效：

```powershell
dsh --profile web --dump-config
```

---

## 6. 开始使用

### 6.1 Web 模式（主要用法）

1. 首页点击 **Choose workspace**，选择一个目录作为工作区（就是让 Agent 操作的项目文件夹）；
2. 新建会话，在输入框用自然语言下达任务；
3. Agent 会在工作区内读写文件、执行命令，并把过程与结果反馈给你。

示例提示词：

```text
请读取当前目录下的 requirements.txt，帮我安装依赖并运行 main.py，最后总结输出结果。
```

```text
分析这个工程的目录结构，说明主程序入口在哪里，并找出 TODO 注释列出待办清单。
```

### 6.2 Headless 命令行模式（脚本/CI 场景）

不开 Web 界面，直接在终端跑一次性任务，输出最终答案后退出，适合写进脚本：

```powershell
dsh --profile headless "运行当前项目的测试并总结失败项"
```

### 6.3 常用 CLI 命令速查

| 命令 | 作用 |
|---|---|
| `dsh web` | 启动 Web 界面（`--profile web` 的别名） |
| `dsh --profile headless "任务"` | headless 模式运行一次性任务后退出 |
| `dsh --profile <名称>` | 启动指定的自定义 profile |
| `dsh plugin --profile <名称> <pnpm 参数>` | 管理指定 profile 的插件 |
| `dsh --profile web --dump-config` | 查看叠加后的完整配置树 |

> 注意：dsh 启动器只解析它自己的参数，其后的内容全部交给对应 profile 处理，因此**启动器参数必须写在最前面**。

---

## 7. 高级功能速览

### 7.1 "一切皆插件"架构

Harness 采用 everything is a plugin 设计：模型、工具、技能、会话、沙箱、存储、循环、调度、UI 等所有能力都由插件组合而成，可自由替换、灵活重组。插件系统基于 **Cordis** 框架，支持加载/卸载与依赖管理，不改源码即可扩展能力。

### 7.2 Profile 与配置叠加体系

一个 profile 目录包含三类文件：

| 文件 | 作用 |
|---|---|
| `package.json` | 记录插件依赖 |
| `dsh.profile` | profile 元数据清单，按顺序列出组合包 bundles |
| `cordis.patch.yml` | 用户自己的配置补丁层 |

最终生效的配置树从空根开始，按以下顺序逐层叠加，**越靠后的层优先级越高、覆盖前面的同名配置**：

1. 组合包（bundles）自带补丁；
2. profile 自身的 `cordis.patch.yml`；
3. 用户主目录 `$DSH_HOME/cordis.patch.yml`（对所有 profile 生效）；
4. 命令行 `--patch` 指定的临时覆盖层（最高优先级）。

![配置树叠加层次图](/image/deepseek-harness/06-config-layers.svg)

### 7.3 会话日志与回放

会话日志采用**追加写入（append-only）**，完整记录系统提示词、思维链、工具调用与结果、子 Agent 调度和上下文注入。在 Trajectory（轨迹）视图中可以回看完整执行过程，并支持：

- **resume 恢复**：从断点继续任务；
- **fork 分支**：从历史某一步分叉尝试不同方案；
- **检索与 replay 回放**：复盘排错。

---

## 8. 常见问题 FAQ

**Q1：DeepSeek Harness 收费吗？**
软件本身 MIT 开源、完全免费；只有调用模型时按所选模型的 API token 用量计费。

**Q2：对电脑配置有要求吗？**
没有高要求。Harness 只在本地做调度，模型推理在云端完成，不需要显卡，普通办公电脑即可。

**Q3：能接入 OpenAI、Kimi 等其他模型吗？**
可以。Settings → Models 里默认提供近 40 家厂商，也支持自定义 OpenAI 兼容接口，换 Key、换 Provider 即可。

**Q4：启动时报网络错误、拉包失败怎么办？**
先按 2.4 节把 npm 源切到 `https://registry.npmmirror.com`，再重新执行安装/启动命令。

**Q5：和 Claude Code 有什么区别？**
两者定位相同，都是"智能体 + 项目"工作单元。Harness 的差异点是模型选择自由、完全开源、支持多 Agent 编排与插件化重组。

**Q6：API Key 填了却报 401/余额不足？**
检查三点：Key 是否复制完整（`sk-` 开头无空格）、账户是否充值、Base URL 是否被误改。

---

## 9. 一页速查清单

```text
【一次性准备】
1. nodejs.org 装 LTS → node -v / npm -v 验证
2. npm config set registry https://registry.npmmirror.com
3. platform.deepseek.com 实名、充值、创建并复制 sk- Key

【每次使用】
4. npx @deepseek-ai/dsh web   （或全局安装后 dsh web）
5. 浏览器开 http://127.0.0.1:3080
6. Settings → Models 填 Key 保存
7. Choose workspace 选项目目录 → 自然语言下任务
8. Trajectory 里恢复/分支/回放会话
```

## 参考资料

- 参考教程：<https://eogee.com/article/47>
- Node.js 官网：<https://nodejs.org/>
- DeepSeek 开放平台：<https://platform.deepseek.com/>
- Harness 源码仓库：<https://github.com/deepseek-ai/deepseek-harness>
