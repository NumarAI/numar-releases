<!-- 语言切换 -->
[English](./README.md) · **中文**

---

# Numar

基于 VS Code 内核构建的 AI-native 桌面 IDE，**BYOK（自备 Key）**：你在 Numar 里发起的所有 **AI/模型请求** 都由你的电脑直接发送到你配置的**模型服务（provider）**（OpenAI / Anthropic / DeepSeek / GLM（智谱）/ Qwen / Gemini / OpenRouter / 本地 Ollama，或任何 OpenAI-compatible 端点），**不会先发到 Numar 的服务器再转发**。

本仓库存放的是 Numar 的**签名二进制版本**，产品本身默认闭源；企业客户可以在签订 NDA 后申请源码审阅（见 [FAQ](#faq)）。

---

## 目录

- [设计](#设计)
- [架构概览](#架构概览)
- [快速开始](#快速开始)
- [功能详解](#功能详解)
- [Settings 速览](#settings-速览)
- [自动升级](#自动升级)
- [安全与隐私](#安全与隐私)
- [FAQ](#faq)
- [关于本仓库](#关于本仓库)
- [许可与联系方式](#许可与联系方式)

---

## 设计

Numar 由以下几部分构成：

**1. BYOK + 本地 Sidecar**
Numar 包含一个本地后端服务（`127.0.0.1:3901`），所有 AI/LLM 调用都从它发起。你配置一次模型服务（provider：OpenAI、Anthropic、DeepSeek、GLM（智谱）、Qwen、Gemini、OpenRouter、本地 Ollama，或任何 OpenAI-compatible 端点），之后每个请求都由你的机器直接发到该模型服务（provider）。Numar 不提供云端中转，服务器也不接收请求内容。

**2. 网络出向**
Numar 会进行两类对外网络访问：

- **Numar 官方服务**：定期向 `updates.numar.ai` 发起**签名升级检查**（返回签名清单）。
- **你配置的模型服务（provider）**：当你使用 AI 功能时，请求由本机直接发送到该模型服务（provider）。

诊断日志写到本地 NDJSON 文件（`~/.numar/telemetry.ndjson`），从不上传；对话记忆和代码索引存储在本机的 SQLite 数据库里。

**3. Agent Workflow（Standard / Phased）**
新建 Agent 或 SO 会话时，Numar 会询问使用哪种工作流——该选择在本会话内锁定。

- **Standard** —— 连续编码 Agent：调查 → 编辑 → 校验 → 完成（一条工具环；Free / Pro 均可）。
- **Phased** —— 分阶段编码工作流：THINKING → PLAN → GENERATE → APPLY → 主机侧 Parse / Compile / Test（Pro）。可在 **Settings ▸ Agent Workflow** 为各阶段配置模型与 Effort。

**4. SO Mode（自运行）**
SO Mode 让 Numar 在清晰目标下端到端负责——调查、实现、校验、在限额内修复——不必逐步批准。它跟随当前会话的 Standard / Phased，一般只在凭证、权限或无法从仓库推断的业务事实上停下。

**5. 本地对话历史与记忆**
在 Pro 中，新对话会记录到本地 SQLite。Conversation History 开关控制 AI 能否搜索和使用这些记录；关闭 AI 访问不会上传、删除记录，也不会停止记录新的 Pro 对话。Project Memory 保留绑定到当前工作区（workspace）的条目，Global Memory 保留跨工作区（workspace）适用的条目，两层记忆均为可选开启（opt-in）。

**6. AI 自动维护的工程 Wiki**
Numar 可以在项目里增量生成和维护 markdown 形态的工程 Wiki，独立于对话历史，落在仓库（repo）里随 Git 版本管理。

**7. Business Knowledge（业务知识）**
Numar 可以维护项目级的业务规则、决策与关联关系。你可以从当前工程初始化知识库，把它只保存在本机，或通过 `.numar/business/` 随 Git 共享；还可以用只读关系图查看整体业务结构。检索会结合业务文本与当前实现引用，并把已有知识当作证据，而不是要求 Agent 无条件相信。

---

## 架构概览

```mermaid
graph LR
    subgraph YourMachine["你的机器"]
        Editor[Numar 编辑器<br/>基于 VS Code 内核]
        Sidecar[本地 Sidecar<br/>127.0.0.1:3901]
        SQLite[(SQLite<br/>记忆 + 代码索引)]
        Wiki[(项目 Wiki<br/>markdown 文件)]
        BusinessKnowledge[(Business Knowledge<br/>规则 + 关联关系)]
        Editor <--> Sidecar
        Sidecar <--> SQLite
        Sidecar <--> Wiki
        Sidecar <--> BusinessKnowledge
    end

    subgraph YourProvider["你配置的模型服务（provider）"]
        LLM[OpenAI / Anthropic /<br/>DeepSeek / Ollama / ...]
    end

    Sidecar -.BYOK 直连.-> LLM

    subgraph NumarServers["Numar 服务器"]
        UpdateServer[updates.numar.ai<br/>仅返回签名升级清单]
    end

    Editor -.签名升级检查.-> UpdateServer

    classDef yours fill:#e8f4f8,stroke:#1f6f8b
    classDef provider fill:#fff4e6,stroke:#b56500
    classDef ours fill:#f0f0f0,stroke:#666
    class YourMachine yours
    class YourProvider provider
    class NumarServers ours
```

**哪些东西在哪儿：**

- 蓝色框（你的机器）装的是编辑器、本地 Sidecar、你的代码、对话历史、记忆、Wiki、Business Knowledge、API key
- 橙色框（模型服务/provider）是你配置的模型服务（provider），每次模型调用都由你的机器直接发到这里
- 灰色框（Numar 服务器）只接收周期性的签名升级清单请求

---

## 快速开始

### 系统要求

- **macOS 11（Big Sur）或更新**，Apple Silicon（M1/M2/M3/M4）
- 约 500 MB 磁盘空间装应用，加上几百 MB 的记忆/索引数据库（随使用增长）
- 至少一家模型服务（provider）的 API Key（或本地 Ollama）

> Windows 和 Linux 已在路线图，目前尚未发布。

### 1. 下载

到 [Releases 页](https://github.com/NumarAI/numar-releases/releases) 拿最新的 macOS 包（当前最新：**[v0.1.29](https://github.com/NumarAI/numar-releases/releases/tag/v0.1.29)**）。

### 2. 安装

```bash
# 解压下载下来的 zip
unzip ~/Downloads/Numar-darwin-arm64.zip -d ~/Downloads/

# 拖进 Applications
mv ~/Downloads/Numar.app /Applications/
```

> **Gatekeeper 提醒。** Numar 用 Apple Developer ID 签名并通过 Apple notarize，正常情况打开不会有警告；如果 macOS 第一次启动还是拦了，右键点应用 → 选**打开**，确认即可。

### 3. 校验下载（推荐）

每个版本发布包都附带公开的 SHA-256，首次启动前校验一下：

```bash
# 算一下你下载的文件的 hash
shasum -a 256 ~/Downloads/Numar-darwin-arm64.zip

# 和 Releases 页上贴的值对比
# 必须一致。不一致就别运行，告诉我们。
```

### 4. 首次启动 —— 配你自己的 Key

第一次启动时，Numar 会引导你配置至少一家模型服务（provider）：

1. 打开 **Settings ▸ Numar**
2. 选择模型服务（provider：OpenAI / Anthropic / DeepSeek / GLM / Qwen / Gemini / OpenRouter / Ollama / OpenAI-compatible 自定义）
3. 粘贴你的 API Key
4. 如果想给 **向量嵌入（Embedding）/ 视觉（Vision）/ 搜索（Search）/ Wiki / Business Knowledge** 配独立模型，也可以单独设置

Key 存在系统 keychain，不会被发送到 Numar 的服务器，只会在本地 Sidecar 直连你配置的模型服务（provider）时用于发起请求。

### 5. 打开第一个项目

`文件 ▸ 打开文件夹…` 选任意目录，打开聊天面板（macOS 默认快捷键：⌘L），选模式：

- **Ask** —— 跟模型聊天，不动文件
- **Agent** —— 编辑文件、调用工具；新建会话时还需选择 **Standard** 或 **Phased** 工作流（见下文）
- **Plan** —— 先在 `.numar/plans/` 写出可审阅计划，确认后再执行
- **SO**（可选）—— Self-Operating：Numar 在限额内自主推进任务，仍使用你选的 Standard / Phased

也可打开 **Settings ▸ Numar ▸ Docs** 查看应用内 Quick Start、Agent Workflow、SO Mode 与排查说明。

---

## 功能详解

### 聊天与模式

Numar 的聊天面板是主要的工作界面，提供这些模式：

| 模式 | 行为 | 适合场景 |
|---|---|---|
| **Ask** | 纯对话，不改文件、不调工具 | 学习代码库、求解释、探讨方案 |
| **Agent** | 带工具的编码 Agent；每个新会话选择一次 **Standard** 或 **Phased** | 日常编码、功能开发、重构 |
| **Plan** | 在 `.numar/plans/` 写下带 TODO 的计划文档，你审阅后点 Execute | 跨多文件重构、破坏性操作、想先看再改的任务 |
| **SO** | 叠在 Agent 上的自运行：调查 → 实现 → 校验 → 修复，中途少打断 | 目标清晰、希望少插手交付 |

### 模型、推理强度与免费模型

可以注册多家 provider 下的多个模型，并从聊天的模型选择器里随时切换。

- **推理强度（Reasoning effort）。** 对支持推理/思考链的模型，可以直接在模型旁选择强度档位——Low / Medium / High / Extra High。统一档位会映射到各 provider 的原生参数（如 OpenAI、Anthropic）。对"仅可选推理"的模型，当 Agent 推理开关关闭时不显示强度选项。
- **浏览免费模型。** Models 设置页提供入口，列出精选的免费模型（当前经由 OpenRouter），均支持工具调用与编程。每个条目会显示其厂商（带官网链接）和获取 API Key 的链接，可一键添加。免费档有限流，重度 Agent 任务可能很快触达 provider 上限。

### Agent Workflow —— Standard 与 Phased

新建 **Agent** 或 **SO** 会话时，Numar 会询问使用哪种工作流。该选择在**本会话内锁定**（要换就开新会话）。对比说明、推荐场景与 Phased 阶段模型在 **Settings ▸ Agent Workflow**；应用内 Docs 同步维护。

| | **Standard** | **Phased** |
|---|---|---|
| **是什么** | 连续编码 **Agent**（工具环） | 分阶段编码 **工作流** |
| **形态** | 调查 → 编辑 → 校验 → 完成 | THINKING → PLAN → GENERATE → APPLY → 主机侧检查 |
| **可用性** | Free 与 Pro | Pro（阶段模型 / Effort 为 Pro） |
| **更适合** | 修 bug、小改动、问答、最快闭环 | 多文件功能、高风险改动、希望先有明确计划再改 |
| **校验** | 诊断 + 可选 Numar Test / Post-Edit Validation | APPLY 后 Parse check → Compile → Test；**失败**可回到 THINKING 修复；**跳过**不会进入修复环 |

#### Phased 流水线（可见阶段）

会话使用 **Phased** 时，会走可见阶段：

1. **THINKING** —— 高层分析，带超时、可取消；可选推理 / 阶段模型。
2. **PLAN** —— 只读工具收集上下文后定计划（最大轮次 / 超时**仅作用于本阶段**，Standard 不用）。
3. **GENERATE** —— 产出编辑。
4. **APPLY** —— 写盘。
5. **主机侧检查** —— **Parse check**，再 **Compile**（若开启 Post-Edit Validation），再 **Numar Test**（若开启）。失败可进入自修复；命令解析不出则提示配置并跳过，不堵对话。
6. **SUMMARY** —— 简洁回顾改了什么。

每个阶段在 UI 上都看得到，随时可以中途停。

```mermaid
stateDiagram-v2
    [*] --> THINKING
    THINKING --> PLAN: continue
    THINKING --> [*]: cancel
    PLAN --> GENERATE: collect info
    PLAN --> [*]: cancel
    GENERATE --> APPLY
    APPLY --> TEST: 编辑提交
    TEST --> SUMMARY: 通过或跳过
    TEST --> THINKING: 检查失败 (重试预算内)
    SUMMARY --> [*]
```

#### Standard 工作流

**Standard** 保持一条连续 Agent 环：模型调工具、改文件，任务完成后收尾。近期版本会把完成结算绑到新鲜的校验证据与工作区真实变更上，文件又改过之后，旧的“已完成”声明更不容易继续生效。对高风险回合仍可出现 TODO / 快照卡（与 Plan Mode 自动升级共用阈值）。没有独立的 PLAN 阶段超时——沿用 Agents 的「模型回合超时」与最大步数。

### SO Mode

**SO Mode**（Self-Operating）面向目标清晰、希望少插手的交付。Numar 在步数/时间限额内调查、实现、校验并修复，并跟随会话的 **Standard** 或 **Phased**。建议配强编码模型；Phased 会话仍使用 Agent Workflow 上的阶段模型与 Effort。通常只会在凭证、权限或仓库里推不出来的业务事实上询问。详见 **Settings ▸ SO Mode**。

### Plan Mode

Plan Mode 包含两层相关能力：

1. **聊天底部的 Plan 模式** —— 大改前在 `.numar/plans/` 写出可审阅计划，你读完再点 Execute。
2. **Agent 自动升级** —— Agent 模式下，高风险回合可弹出快照 / TODO 卡（阈值在 Settings ▸ Plan Mode）。Standard 的 TODO 卡共用同一套阈值。

自动升级触发条件可配置：

- 影响超过 N 个文件
- 总编辑数超过 N
- 至少新建 N 个文件
- 任何破坏性操作（移动、重命名）
- 任何敏感文件（`package.json`、`tsconfig*`、`.env*`、CI 配置、Dockerfile、DB migration……）

**删除文件永远需要明确确认**，不受这些开关影响。

### Numar Test —— 自修复闭环

Agent 改完之后（尤其在 **Phased**、编译通过之后），Numar Test 可以：

1. 解析测试命令（**Auto** = 项目发现 + 必要时模型选型，或锁定预设 / Custom）
2. 跑测试（可用 `{files}` 做文件级范围）
3. 把**新失败**喂回 Agent，在重试预算内自修复
4. 若解析不出命令则**提示配置并跳过**——不会堵在等待用户输入

默认值按典型 Node / Python 项目调好，可按工作区覆盖。与 Agents → Post-Edit Validation 下的 Compile command 交互对齐。

```mermaid
graph LR
    A["🔧 Agent 编辑完成"] --> B["🧪 解析 / 运行测试"]
    B --> C{测试通过?}
    C -->|是 / 跳过| E["✅ 继续"]
    C -->|失败| F{重试预算<br/>未耗尽?}
    F -->|是| G["📋 收集失败信息"]
    G --> H["🤔 Agent 自修复"]
    H --> I["📝 生成新编辑"]
    I --> B
    F -->|否| J["❌ 停止"]
```

### Post-Edit Validation 与 Compile command

在 **Settings ▸ Agents → Post-Edit Validation**：

- 需要类型化编辑后的主机侧检查时打开自动校验（对 **Phased** 尤其有用）。
- **Compile command** —— **Auto**（从项目发现 / 必要时问模型）、框架**预设**，或 **Custom**（支持 `{files}`，例如 `python -m py_compile {files}`）。思路与 Numar Test 相同。
- 命令定不下来时 Numar **跳过**并给短提示（打开含 `package.json` / `go.mod` 的文件夹、保持 Auto，或锁定命令），不堵对话。
- Compile / Test **失败**可进入修复；**跳过**不会。

### Memory —— 两层

Numar 有两层**可选开启（opt-in）**的持久化记忆，**都默认关闭**：

- **Global Memory** —— 跨工作区（workspace）的个人偏好：交互风格、语气、不绑定到具体项目的偏好
- **Project Memory** —— 绑定到当前工作区（workspace）的事实和决策：库选型、截止日期、代码里看不出来的约束

记忆以 markdown 文件存储，Agent 可以读、搜、更新，你也一样。

### 对话历史与搜索

在 Pro 中，新对话会写入本地 SQLite。启用 AI 访问后，Agent 可以通过内置工具跨会话搜索过去的对话。

你可以控制：
- AI 能否搜索和使用 Conversation History
- 本地对话记录的保留与存储阈值
- 触达阈值后是否允许自动执行永久清理

关闭 AI 访问不会删除已有记录，也不会停止记录新的 Pro 对话。活跃上下文的轮次数与 token 限额在 **Context Window** 中单独配置。

### Sessions（会话）

Sessions 视图提供精简的本地会话历史和 **New Session** 操作。它复用 Numar 的会话生命周期，因此打开某个 Session 会恢复原会话，而不是创建一份重复副本。Session 历史保留在本机，不是云同步功能。

### Business Knowledge（业务知识）

Business Knowledge 是按项目维护的 AI 业务知识库，用来保存业务规则、决策与关联关系。它用于补充源代码、Conversation History、Memory 和工程 Wiki，而不是替代它们。

- **从工程初始化。** 前台扫描从代码、测试与文档中提取候选事实；失败批次可以 Retry，用户主动停止的扫描可以 Continue。
- **持续更新。** 成功完成的任务可以增量更新匹配事实；Relationships 会随扫描刷新，也可以在已有知识变化后单独刷新。
- **本地或共享。** 可以只保存在当前设备，也可以把 `.numar/business/` 通过 Git 分享给团队。
- **证据感知检索。** 通过关键词/文本与实现引用匹配召回相关事实和 Relationships；Agent 行动前仍会检查当前代码。
- **只读预览。** 可以打开力导向图查看规则及其直接关联，不会编辑文件，也不会离开 Settings。

### 项目 Wiki

Numar 可以为你的项目增量生成和维护一份工程 Wiki，独立于对话历史，以 markdown 落到仓库（repo）里（带版本管理、可审阅）。如果你想用更便宜或更大上下文的模型专门跑 Wiki 生成，可以单独给 Wiki 配模型服务（provider）。

### 代码索引

Numar 在本地 SQLite 维护一份项目代码的索引，支持关键字搜索和（可选）语义搜索，语义搜索需要配置向量嵌入模型服务（embedding provider）。两种搜索都对 Agent 作为工具开放，也对你通过搜索 UI 开放。

### Cross-Stack Agents（多窗口协同）

你可以把同一台机器上的多个 Numar 窗口连成一个协同工作区（workspace），不同窗口里的 Agent 可以跨工作区（workspace）提问和回答——全程不经过云。适合单仓库（monorepo）这类想保持编辑器实例聚焦、但又希望它们之间能对话的场景。

### 聊天内 Git（Changes 面板）

不离开聊天就能审阅并提交 Agent 的改动。**Changes 面板**按当前会话跟踪 AI 修改过的文件——带增/删行数和 A/M/D/R 状态标记——并随对话推进与源代码管理对账。提供三个动作：

- **Review** —— 打开这些文件的多文件工作区 diff。
- **Commit** —— 暂存并提交这些文件。
- **Commit & Push** —— 提交后把当前分支推到上游。

提交由 Agent 通过其 git 工具执行；受保护分支遵循确认规则（推送到 `master` 一律需确认），空提交或已是最新的 push 会被自动跳过。

### Code Review（代码审查，建议性）

Numar 内置一个按 commit 的 AI 代码审查器，在活动栏有独立的 **Code Review** 容器。它审查单个 git commit 的 diff，产出结构化的 markdown 报告——摘要、按 **Must-fix / Suggestions / Nits** 分组的发现项，以及安全性、正确性、测试相关的说明——最后给出结论：`clean`、`needs-attention` 或 `blocking-advisory`。

- **仅供参考。** Code Review 从不阻断 commit 或 push，只产出报告。
- **手动或自动。** 在视图里运行 *Review Latest Commit*（或审查指定 commit），也可开启每次提交后自动审查。自动审查受可配置阈值门控（最少改动文件数 / 行数）；合并提交、已审查过的提交、被 git 忽略的文件、以及排除路径都会跳过。大 diff 会自动分块。
- **报告存在仓库之外。** 每份报告写到项目**同级目录** `<项目>_CodeReview/`，按分支分文件夹、每个 commit 一个 markdown 文件，另有 `_meta.json` 索引——审查内容不会污染你的项目树或 git 历史。
- **可用独立模型（可选）。** 在 *Settings ▸ Code Review* 给 Code Review 配独立的模型 / endpoint / key，或让它回退到你的主模型。

### Content Language（内容语言）

Numar 会识别你主要用中文还是英文聊天，并把该语言套用到 AI 生成的正文——Wiki 页面、Code Review 报告、摘要——同时结构性标题保持英文。它全自动、按工作区（workspace）粘性记忆，无需手动开关。

### Trusted Commands

配一份白名单，列出 Agent 可以不弹窗直接执行的终端命令，其它都会先问你，也可以全局开启自动批准。

---

## Settings 速览

所有设置都在 **Settings ▸ Numar** 下，每个面板在 UI 里都有详细说明、默认值和可选范围——这里只给你一张地图，让你知道**哪些方面可以自己调**。

**General（通用）** —— 系统通知；本地诊断（开关、保留天数、大小上限）；交互日志保留天数

**Cross-Stack Agents（多窗口协同）** —— 是否允许本窗口与同机器其它 Numar 窗口通讯；对等（peer）任务记录保留数

**Memory（记忆）** —— 全局记忆（Global）与项目记忆（Project）的总开关（**两者默认都关**）

**Business Knowledge（业务知识）** —— 模型选择；本地或 Git 共享存储；工程初始化、Retry/Continue、Relationships 刷新与只读关系图预览

**Indexing & Embeddings（索引与向量嵌入）** —— 工作区（workspace）向量嵌入总开关；嵌入模型、端点（endpoint）、API Key

**Vision（视觉）** —— 主模型不支持视觉时，用于图像描述的独立模型

**Search（搜索）** —— 网页搜索的模型服务（provider）、端点（endpoint）、API Key

**Wiki（工程 Wiki）** —— Wiki 生成可单独配置的模型、端点（endpoint）、API Key

**Code Review（代码审查）** —— 自动审查开关；触发时机（post-commit / post-push）；阈值（最少改动文件数、行数）；排除模式；可选的独立 CR 模型、端点（endpoint）、API Key

**Commands & Git（命令与 Git）** —— 终端命令自动批准总开关；可信命令白名单；可跳过确认的可信 git 操作；推送到 `master` 强制确认

**Agents（Agent）** —— 单次最大步数；模型回合超时；混合模型推理；**Post-Edit Validation**（开关 + **Compile command**：Auto / 预设 / Custom，支持 `{files}`）

**Agent Workflow** —— Standard / Phased 对比；Phased 阶段模型与 Effort（Pro）；Phased PLAN 最大轮次 / 超时

**SO Mode** —— 自运行说明与模型建议；跟随会话的 Standard / Phased

**Plan Mode（计划模式）** —— 聊天 Plan 模式 + Agent 自动升级阈值（文件数、编辑数、创建数、破坏性操作、敏感文件）；可选用 THINKING 流生成计划

**Numar Test（测试）** —— 总开关；测试命令 **Auto / 预设 / Custom**（`{files}`）；超时；自修复重试上限

**Context Window（上下文窗口）** —— 最近轮次数；历史 token 预算；超预算策略（截断或 LLM 摘要）

**Conversation History（对话历史）** —— AI 能否搜索和使用本地记录的 Pro 对话；保留、存储与清理控制（关闭 AI 访问不会停止记录，也不会删除已有数据）

**Code Index（代码索引）** —— 是否在本地索引项目代码（**默认开**，可关，已有数据保留）

**Remote Configuration（远程配置）** *（在 `settings.json` 里）* —— 是否从升级服务器拉取签名运行时配置；可覆盖端点

**Server（服务）** *（高级）* —— 本地 Sidecar 地址和请求超时

---

## 自动升级

Numar 会自动向 `updates.numar.ai` 检查更新，检查机制严控、签名、保守：

- **签名清单**：每份升级清单都用 ed25519 签名，公钥在打包时嵌入应用，验签失败的清单直接拒绝。
- **基于 cohort 的灰度**：每个客户端计算一个稳定的 cohort hash，新版本先放给一部分 cohort，跑稳了再 ramp 到 100%。
- **Kill Switch**：发现问题可以远端急刹，还没升级的客户端会被告知"没有新版本"。
- **双通道**："Binary update" 通过 Squirrel.Mac 替换整个 `.app`，"Core update" 可以发热修复（JS / 运行时层）不需要全量重装；两者独立签名。

**手动操作：**

- 检查更新：`Numar ▸ Check for Updates…`
- 关闭远端运行时配置：在 Settings 里把它关掉
- 审计 Numar 向哪发请求：**Numar 官方服务侧**只有一个端点（`updates.numar.ai`），返回的只是签名清单；AI 功能请求会直连你配置的模型服务（provider），任何 HTTP 抓包工具都能验证

---

## 安全与隐私

这一节列出 Numar 在本地存什么、通过网络发什么。

**留在你机器上的：**

- 你打开的所有源码
- 你发给模型的所有提示词（prompt）
- 模型的所有回复
- API key（存系统 keychain）
- 对话历史（本地 SQLite）
- Project Memory 和 Global Memory（本地 markdown）
- 代码索引（本地 SQLite）
- 项目 Wiki（落到项目里的 markdown）
- Business Knowledge（本机知识库文件或随 Git 共享的项目文件，包含规则与关联关系）
- 诊断日志（`~/.numar/telemetry.ndjson`，永不自动上传）

**Numar 主动发的，发给谁：**

- **你配置的模型服务（provider）** —— 每个模型调用直接发到这里，适用该 provider 的数据策略（data policy，例如 OpenAI 的保留策略）。
- **`updates.numar.ai`** —— 周期性升级检查，带平台、版本、一个稳定的不透明 cohort hash，返回签名清单，**不含使用数据、埋点、代码或提示词（prompt）**。

**从目的地来看，对外主要就是这两类：你配置的模型服务（provider） + `updates.numar.ai`。** 抓包时，你会看到 AI 调用直连模型服务（provider），以及周期性的更新检查请求。若你启用了网页搜索等可选能力，还会出现你所配置的搜索服务（Search provider）目的地。

**给企业 / 合规团队：**

- 签 NDA 后可申请源码审阅
- 支持自建升级服务器（在 Settings 里覆盖端点）
- 离线 / 隔离网络部署：在 Settings 里关掉远程配置（remote configuration）和自动升级，Numar 可以无限期跑在本地配置的模型服务（provider）上（包括本地 Ollama）

如果你需要正式的安全问卷回复，联系我们。

---

## FAQ

### 源码开放吗？

Numar 目前**默认闭源**，我们是个聚焦的小团队，目前以商业产品形态发售。

**企业客户**可在签订 NDA 后申请源码审阅，用于安全审计、合规审查、平台/安全团队的内部验证，需要的话联系我们。

### Numar 能看到我的代码或提示词（prompt）吗？

**不能，** 每个 LLM 调用都是本地 Sidecar 在你机器上直接发给你配置的模型服务（provider），**不会先发到 Numar 服务器再转发**，我们服务器看到的只是签名升级检查（平台 + 版本 + 不透明 cohort hash）。

### Numar 能看到我的 API key 吗？

**不能，** Key 存在系统 keychain，只在本地 Sidecar 调你的模型服务（provider）时使用，从不发给我们。

### 和 Cursor / Cline / GitHub Copilot 有什么不同？

- **Cursor** 是托管型 IDE，LLM 路由、团队管理、计费都在它的服务器上运行。
- **Cline** 是 VS Code 扩展，运行在你自己提供的宿主编辑器里。
- **GitHub Copilot** 集成 GitHub 托管的模型和 Microsoft 生态。
- **Numar** 是桌面 IDE，每一次 LLM 调用都由机器上的本地 Sidecar 直接发到你配置的模型服务（provider），自身服务器只接收周期性的签名升级清单请求。

### 能用本地模型代替托管的吗？

可以，把 Ollama（或任何 OpenAI-compatible 的本地服务）配为模型服务（provider），Numar 的本地 Sidecar 会通过 `localhost` 调用它。端到端可以做到**除升级检查外零出向流量**（升级检查也能关）。

### 能让 Numar 调公司自建的 LLM 网关（gateway）吗？

可以，把模型服务（provider）的 URL 设成你内部网关（gateway）的端点，按网关要求的鉴权头（auth header）走就行。

### 会有 Windows / Linux 版本吗？

会，已在路线图。macOS / Apple Silicon 是第一个支持的平台，因为我们最早的用户在那儿。订阅版本更新通知或关注（watch）本仓库，可以第一时间看到平台扩展。

### Bug 和功能请求去哪反馈？

二进制相关 bug 请在本仓库提 Issue，私密的安全问题请发邮件到下方联系方式。

### Numar 服务器挂了会怎样？

你的编辑器照常工作。Numar 这边唯一的依赖是升级检查，失败会优雅降级——已经装上的版本可以无限期继续用。模型调用直接打你的模型服务（provider），跟我们服务器状态无关。

### 为什么要 fork VS Code 而不是做个插件？

插件无法完全掌控聊天面板布局、Agent 状态、Plan 模式 UX、多窗口协同、升级通道，fork 是把这些做成一等公民的方式。Numar 跟随 VS Code 上游，自己加的代码集中在新增模块和少量集成点上。

---

## 关于本仓库

本仓库（`NumarAI/numar-releases`）是 Numar 签名二进制的**公开分发点**，不含源码——只有 GitHub Releases 上的 `.app.zip`（按平台）、对应的 SHA-256 清单、以及这份 README。

> **关于命名。** 从 **v0.0.5** 起，产品、Settings 和 UI 全部使用 **Numar** 品牌。本机数据落在 `~/.numar/`（例如 `~/.numar/telemetry.ndjson`、记忆与交互日志）；工作区内的计划与 skills 等资产落在项目的 `.numar/`。极早期构建曾使用 `~/.newma/` 前缀，当前版本已不再写入该路径。

产品源码私有托管，这种拆分的原因：

- 分发应该是**公开、免费、CDN 兜底**的——GitHub Releases 给我们零成本拿到这一切。
- Numar 升级服务器（`updates.numar.ai`）从本仓库读版本发布元数据，公开本仓库就意味着任何人都能审计我们发布了什么；
- 源码保持私有，原因见上面 FAQ。

如果你是点了下载链接到这里的，最新 macOS 包在 [Releases 页](https://github.com/NumarAI/numar-releases/releases)。

---

## 许可与联系方式

**本仓库中的 Numar 二进制**是专有软件，© 2026 Numar，保留所有权利；你可以在自己的机器上下载、安装、用于个人或公司内部用途，不允许再分发、反编译、逆向工程。

**本仓库的元数据**（README、发布说明/release notes）采用 MIT License。

**联系方式：** `evenzhi@gmail.com` —— 一般咨询、企业源码审阅申请、安全披露、Bug 反馈都用这个，非敏感的 Bug 也可以直接在本仓库提 Issue。

---

<p align="center">
  <em>Numar —— AI-native 桌面 IDE</em>
</p>
