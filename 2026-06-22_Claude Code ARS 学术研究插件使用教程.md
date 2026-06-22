# Claude Code ARS 学术研究插件使用教程

> 版本日期：2026-06-22  
> 适用对象：已在 Claude Code 中安装 `Imbad0202/academic-research-skills` 插件的用户，希望系统性地使用 ARS 完成学术论文从选题到定稿的全流程。  
> 前置条件：Claude Code 已配置好 API（DeepSeek V4 Pro 或 Anthropic 官方），且已通过 `/plugin install academic-research-skills` 完成安装。

---

## 资料依据

本教程基于以下资料交叉整理：

1. ARS 官方仓库 README 及架构文档：https://github.com/Imbad0202/academic-research-skills
2. ARS 官方文档（ARCHITECTURE.md、PERFORMANCE.md、SETUP.md）
3. 社区使用经验与案例分享
4. 插件内置 slash command 定义文件

核心设计哲学：**"AI 是你的副驾驶，不是机长"**——ARS 帮你处理文献检索、引用格式、数据校验、逻辑一致性检查等机械性工作，你则专注于定义问题、选择方法、解读数据和撰写核心论点。

---

## 1. ARS 是什么

ARS（Academic Research Skills）是一套面向 Claude Code 的学术研究技能包，覆盖学术写作全流程：

```text
选题探索 → 文献检索 → 论文撰写 → 同行评审 → 修订回复 → 终稿定版
```

它由 **4 个技能组、27 种模式** 组成，通过 **10 阶段流水线** 串联。每个阶段都需要人工确认，强制插入学术诚信检查点，防止 AI 幻觉污染论文。

### 1.1 四大技能组一览

| 技能组 | 定位 | 典型场景 |
|--------|------|----------|
| **Deep Research**（深度研究） | 文献检索、系统综述、事实核查 | "帮我系统检索少子化对劳动力市场影响的文献" |
| **Academic Paper**（学术论文） | 论文撰写、大纲规划、格式转换、引用检查 | "帮我把这篇 Markdown 转成 APA 7.0 格式的 LaTeX" |
| **Academic Paper Reviewer**（学术评审） | 多视角模拟同行评审 | "以 NeurIPS 审稿人视角审阅我的论文" |
| **Academic Pipeline**（学术流水线） | 10 阶段全流程编排器，自动串联以上三者 | "从零开始，完成一篇关于 XX 的完整论文" |

---

## 2. 安装与验证

### 2.1 安装

在 Claude Code 会话中依次执行：

```text
/plugin marketplace add Imbad0202/academic-research-skills
/plugin install academic-research-skills
```

安装完成后，重启 Claude Code 会话以加载所有 skill 和 slash command。

### 2.2 验证安装

输入以下命令，如果能进入 Socratic 对话模式，说明安装成功：

```text
/ars-plan
```

然后简单描述你的论文主题（例如"我想写一篇关于大语言模型在医学影像诊断中应用的综述"），ARS 会开始通过一系列提问帮你梳理论文结构。

### 2.3 可选依赖

| 工具 | 用途 | 是否必需 |
|------|------|----------|
| Pandoc | 导出 DOCX | 可选 |
| tectonic + Source Han Serif TC 字体 | 编译 APA 7.0 PDF | 可选 |
| Python（python.org 安装版） | PreToolUse 写入守卫、缓存命令 | 可选（Windows 上建议安装） |

---

## 3. 核心 Slash Command 速查

ARS 提供了 12 个常用 slash command，按使用频率排列：

| 命令 | 功能 | 适用时机 |
|------|------|----------|
| `/ars-plan` | Socratic 对话式规划论文结构 | 想法模糊、需要梳理框架时 |
| `/ars-full` | 全流程：研究→撰写→评审→修订→定稿 | 结构已清晰、需要完整初稿时 |
| `/ars-outline` | 生成详细大纲 + 证据映射 | Plan 之后、正式写作之前 |
| `/ars-lit-review` | 文献综述检索 | 需要系统检索某一领域文献时 |
| `/ars-reviewer` | 模拟同行评审（EIC + 3 审稿人 + Devil's Advocate） | 投稿前自查 |
| `/ars-revision` | 根据审稿意见生成修订稿 + 逐条回复 | 收到 Major Revision 后 |
| `/ars-revision-coach` | 将审稿意见解析为修订路线图 | 审稿意见复杂、需要先理清修改策略时 |
| `/ars-abstract` | 生成中英双语摘要 + 关键词 | 论文主体完成后 |
| `/ars-citation-check` | 引用完整性检查（校验每一条参考文献是否真实存在） | 投稿前最终检查 |
| `/ars-format-convert` | 格式转换（Markdown / LaTeX / DOCX / PDF） | 需要适配期刊模板时 |
| `/ars-disclosure` | 生成符合期刊要求的 AI 使用声明 | 投稿前 |
| `/ars-3w` | 三篇论文的 WHY / HOW / WHAT 对比 | 快速比较相关文献时 |

### 3.1 命令路由说明

- **重型命令**（`/ars-full`、`/ars-revision-coach`）：继承当前会话模型（如 DeepSeek V4 Pro），适合需要深度推理的任务
- **轻型命令**（其余 10 个）：路由到 Sonnet 级模型，响应更快、成本更低

---

## 4. 10 阶段流水线详解

ARS 的核心是 10 阶段学术流水线（Academic Pipeline v3.13.0），每个阶段之间设有人工确认检查点：

```text
Stage 1   → RESEARCH        深度研究、文献检索
Stage 2   → WRITE           论文初稿撰写
Stage 2.5 → INTEGRITY       诚信检查（必选，不可跳过）
Stage 3   → REVIEW          同行评审（5 份审稿报告）
Stage 4   → REVISE          根据评审意见修订
Stage 3'  → RE-REVIEW       修订后复审
Stage 4'  → RE-REVISE       二次修订（如需要）
Stage 4.5 → FINAL INTEGRITY 终稿诚信检查（必选，不可跳过）
Stage 5   → FINALIZE        格式转换与定稿
Stage 6   → PROCESS SUMMARY 质量评估（6 维度 1-100 分）
```

### 4.1 中途插入

不需要每次都从零开始：

- **已有初稿** → 从 Stage 2.5 切入（诚信检查后进入评审）
- **已有审稿意见** → 从 Stage 4 切入（直接进入修订流程）

### 4.2 诚信门禁（Integrity Gates）

这是 ARS 最核心的安全机制，**两次检查不可跳过**：

- **Stage 2.5（投稿前）**：检查虚构引用、统计错误、逻辑不一致、AI 幻觉痕迹
- **Stage 4.5（定稿前）**：确认修订后零退化，所有标记问题已解决

---

## 5. 三种典型使用场景

### 5.1 场景一：从零开始写一篇论文

适用情况：只有一个大致方向，没有成文。

推荐流程：

```text
/ars-plan
   ↓  通过 Socratic 对话梳理研究方向、核心论点、章节结构
/ars-outline
   ↓  生成详细大纲 + 证据映射表，人工确认框架
/ars-full
   ↓  执行完整流水线（研究→撰写→评审→修订→定稿）
/ars-citation-check
   ↓  最终引用校验
/ars-abstract
   ↓  生成中英双语摘要
/ars-disclosure
   ↓  生成 AI 使用声明
```

### 5.2 场景二：已有初稿，准备投稿

适用情况：论文已写好，需要自查和格式调整。

推荐流程：

```text
/ars-reviewer
   ↓  多视角模拟评审，发现盲区
/ars-citation-check
   ↓  逐条校验引用真实性
/ars-abstract
   ↓  生成规范摘要
/ars-format-convert
   ↓  转为目标期刊格式（LaTeX / DOCX / PDF）
/ars-disclosure
   ↓  生成 AI 使用声明
```

### 5.3 场景三：收到 Major Revision

适用情况：审稿意见回来了，需要系统性地修改和回复。

推荐流程：

```text
/ars-revision-coach
   ↓  解析审稿意见 → 生成修订路线图（人工确认修改策略）
/ars-revision
   ↓  生成修订稿 + 逐条回复信（Response to Reviewers）
/ars-reviewer
   ↓  对修订稿再次模拟评审（re-review 模式）
/ars-citation-check
   ↓  确认修订过程中引用未被破坏
```

---

## 6. 实操示例

以下演示一个完整的研究启动流程。假设研究主题是"远程办公对软件工程师生产力的影响"。

### 6.1 启动规划

```text
/ars-plan
```

向 ARS 描述主题：

> 我想写一篇关于远程办公对软件工程师生产力影响的实证研究论文。我手头有一份来自 3 家科技公司的 survey 数据（N≈500），但目前还没有清晰的论文框架。

ARS 会通过类似这样的 Socratic 对话引导你：

1. "你的核心研究问题是什么？是'远程办公是否提高了生产力'，还是更聚焦于'哪些因素在远程办公中最影响生产力'？"
2. "你打算用什么理论框架？例如 Job Demands-Resources Model 还是 Self-Determination Theory？"
3. "你的方法论倾向定量、定性还是混合方法？"
4. "目标期刊是什么？这会影响论文结构和写作风格。"

经过 5-15 轮对话，ARS 会输出一份包含研究问题、理论框架、方法论建议和初步章节结构的规划文档。

### 6.2 生成大纲与证据映射

```text
/ars-outline
```

ARS 会基于前面的对话产出：

- **章节大纲**：从 Introduction 到 Conclusion 的完整层级结构
- **证据映射表**：每个核心论点对应哪些文献支撑、哪些数据支撑
- **缺口标注**：明确指出哪些论点还缺乏文献或数据支撑

**人工检查这一步非常重要**——确认框架符合预期后再进入正式写作。

### 6.3 文献综述检索

```text
/ars-lit-review
```

ARS 会围绕你的研究问题展开多方向检索，并输出：

- 带注释的文献目录（Annotated Bibliography）
- 文献间关联分析
- 研究空白（Research Gap）标注
- **局限性说明**：明确告知检索到的文献存在哪些局限

> **注意区分**：`/ars-lit-review` 调用的是 deep-research 的 lit-review 模式（检索文献）；而 academic-paper 的 lit-review 模式是把已有文献整理成论文格式的文献综述章节。**先检索，后撰写**。

### 6.4 格式转换示例

```text
/ars-format-convert
```

ARS 支持以下转换方向：

| 源格式 | 目标格式 |
|--------|----------|
| Markdown | LaTeX（APA 7.0 / IEEE / Chicago） |
| Markdown | DOCX（通过 Pandoc） |
| Markdown | PDF（通过 tectonic 编译） |
| LaTeX | Markdown |
| 引用格式 | APA 7.0 / Chicago / MLA / IEEE / Vancouver 互转 |

---

## 7. 高级特性

### 7.1 风格校准（Style Calibration）

ARS 可以学习你的写作风格。如果你有 3 篇以上的已发表论文，ARS 会分析你的句式结构、用词习惯、论证节奏，使生成内容更接近你自己的声音，而非通用的"AI 腔"。

### 7.2 Material Passport（材料护照）

这是跨会话的"真相来源"账本，记录所有产出物、文献语料库、核心主张和关键决策。有了它：

- 可以跨会话恢复进度（`resume_from_passport=<hash>`）
- 设置 `ARS_PASSPORT_RESET=1` 可清空上下文重新开始
- 支持在流水线中途安全地切换会话

### 7.3 引用验证（Citation Verification）

ARS v3.11.0+ 引入了确定性的引用存在性门禁：每条引用都会通过 **Semantic Scholar + OpenAlex + Crossref + arXiv** 四重交叉验证。验证结果缓存在 `~/.cache/ars/verification.db`（90 天有效期）。默认策略是 advisory（警告但不阻止），可配置为 terminal（阻止输出）。

### 7.4 主张保真度审计（Claim-Faithfulness Audit）

可选功能（`ARS_CLAIM_AUDIT=1`）：获取引用文献原文，逐条判断论文中的主张是否确实被引用文献所支持。五种 HIGH-WARN 类别可以门禁拒绝输出。

### 7.5 反谄媚协议（Anti-Sycophancy Protocol）

这是 ARS v3.0 的重要更新，解决 AI 审稿中三个常见问题：

| 问题 | 表现 | ARS 对策 |
|------|------|----------|
| Frame-lock（框架锁定） | 审稿 AI 不会质疑你设定的前提 | 让步阈值协议：Devil's Advocate 仅在 ≥4/5 时让步，禁止连续让步 |
| Sycophancy（谄媚） | 被反驳后过快妥协 | 让步率追踪 + 偏转检测 + 反谄媚规则 |
| Intent misdetection（意图误判） | 你还在探索，AI 却急于收敛输出 | 意图检测层：探索模式禁用自动收敛，最大回次提升至 60 |

### 7.6 协作深度观察者（Collaboration Depth Observer）

这是纯咨询性质的代理，从 4 个维度评估人机协作质量（Delegation Intensity、Cognitive Vigilance、Cognitive Reallocation、Zone Classification），不计入任何判定，仅供自我反思参考。

---

## 8. 常见坑与注意事项

### 8.1 两种 lit-review 不要混淆

- **deep-research 的 lit-review**：检索文献，输出带注释的文献目录
- **academic-paper 的 lit-review**：将已有文献写成论文格式的综述章节

**正确顺序**：先用 deep-research 检索文献 → 人工筛选 → 再用 academic-paper 撰写综述章节。

### 8.2 revision-coach 和 revision 的区别

- **`/ars-revision-coach`**：解析审稿意见 → 生成修订路线图（**不修改正文**）
- **`/ars-revision`**：根据路线图实际执行修订（**修改正文 + 生成回复信**）

**正确顺序**：先 coach 确认策略 → 再 revision 执行修改。

### 8.3 人工确认不可省略

ARS 在每个阶段之间都会暂停等待人工确认。**不要一路回车跳过**——这是保证论文质量和你对内容负责的关键环节。特别是：

- 大纲阶段的框架确认
- 文献检索结果的人工筛选
- 审稿意见的修改策略确认

### 8.4 成本预期

根据官方 PERFORMANCE.md，一篇约 15,000 字的完整论文（全流水线），API 费用约为 **$4–6 美元**（基于 Opus 4.x 测量）。实际费用取决于模型选择、论文长度和是否启用跨模型验证等可选功能。

### 8.5 ARS 不是自动写论文的机器

ARS **不会**一键生成完整论文。它的设计目标是通过结构化的问答和分阶段检查，帮助你：
- 追问研究问题
- 查证文献真实性
- 复核逻辑一致性
- 发现你自己可能忽略的盲区

**"I think" 语句（核心论点、原创分析、数据解读）始终是研究者的责任。**

---

## 9. 环境变量配置（可选）

以下环境变量可按需设置：

| 变量 | 作用 | 默认值 |
|------|------|--------|
| `ARS_CROSS_MODEL` | 启用跨模型验证（GPT-5.4 / Gemini 3.1 等） | 未启用 |
| `ARS_CLAIM_AUDIT=1` | 启用主张保真度审计 | 未启用 |
| `ARS_PASSPORT_RESET=1` | 重置 Material Passport | 未启用 |

---

## 10. 官方资源链接

- GitHub 仓库（Claude Code 版）：https://github.com/Imbad0202/academic-research-skills
- GitHub 仓库（Codex CLI 版）：https://github.com/Imbad0202/academic-research-skills-codex
- 配套实验代理：https://github.com/Imbad0202/experiment-agent
- 教学技能包（基于 ARS 架构）：https://github.com/YujxZJCN/teaching-skills
- 许可证：CC-BY-NC 4.0（非商业用途）

---

> **使用提醒**：LLM 输出不是字节级可复现的——模型提供商可能在未变更 model ID 的情况下更新权重，外部 API 每天返回的数据也可能不同。所有 AI 辅助生成的内容，最终都需要你作为作者进行核实和确认。
