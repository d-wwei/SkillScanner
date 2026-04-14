<!-- Synced with README.md as of 2026-04-14 -->

[English](README.md) | [中文](README.zh.md)

# SkillScanner

**一个 skill 文件，让任何 AI Agent 变成安全扫描器 —— 零依赖，零安装，开箱即用。**

[![License: MIT](https://img.shields.io/badge/license-MIT-blue.svg)](LICENSE)

---

## 为什么做这个？

Snyk 扫描了 ClawHub 上 3,984 个 skill。36.82% 有安全缺陷。所有确认的恶意 skill 都同时包含代码执行和 prompt injection。Agent skill 生态的安全状况跟早期 npm 一样——门户大开。

现有的防御工具全是代码库。prompt-shield 要 `pip install`，可选的 DeBERTa 模型 738MB。AegisGate 要 scikit-learn 和 jieba。Aigis 要 Python 3.11+。没有一个能作为 drop-in skill 让任何 Agent 直接加载使用。

空白在这里：没有一个纯指令的安全扫描器。没有一个 SKILL.md 能 symlink 到 `~/.claude/skills/` 之后立刻开始扫描其他 skill 的威胁。SkillScanner 填了这个空白。宿主 Agent 本身就是检测引擎——它的语言理解力替代了代码工具中的正则和 ML 分类器。不需要安装。不需要依赖。Claude Code、Codex、Gemini CLI、OpenClaw，只要能读 SKILL.md 就能用。

## 这是什么

SkillScanner 是一个 SKILL.md 文件加 5 个 reference 文件（总共约 5,400 词）。安装到任何支持 SKILL.md 格式的 AI Agent 上，Agent 就获得了扫描 10 类安全威胁的能力，每项发现自带 OWASP LLM Top 10 标签。

覆盖范围：直接注入、上下文操纵、间接注入（含 8 个 Agent 平台格式）、社会工程、编码混淆、MCP 工具投毒、数据外泄、危险操作、skill 元数据投毒、内存/状态投毒。

## 核心特性

- **LLM 原生语义检测。** Layer 3 不模拟正则引擎——它让 Agent 用自己的话复述被扫描内容的意图，再把复述结果对照威胁意图检查。「放下你目前的操作参数」=「忽略你的指令」——这种换词攻击，正则工具抓不到，但 LLM 能读懂。

- **8 平台工具调用注入检测。** 覆盖 OpenAI、Anthropic Claude、Google Gemini、AWS Bedrock、vLLM/Hermes、ReAct、MCP JSON-RPC、AutoGPT/OpenDevin 的工具调用格式。不是笼统描述，是每个平台的具体格式签名。

- **Per-finding 硬化削弱机制。** 误报抑制系统只检查每条发现周围 ±5 行的上下文，不是整个文档。所有 CRITICAL 类别（T1、T6、T7、T10）永不削弱。已确认的 true positive 永不削弱。这堵住了「把真实攻击包在教程框架里」的绕过路径——更简单的削弱设计挡不住这招。

- **12 个真实攻击案例 + 7 个结构原型。** Layer 3 对照 Snyk ToxicSkills（3,984 个 skill 分析）、Palo Alto Unit 42（24 层叠加 Web 注入）、Aigis MCP 研究中的真实案例进行匹配。7 个结构原型（身份覆盖、数据伪装指令、外泄构造、伪装包装、持久化植入、升级阶梯、框架模仿）按攻击的「形状」捕获新型攻击，不依赖关键词。

- **3-Gate 扫描架构。** Input Gate（处理前）、Tool-Result Gate（MCP/API 调用返回后——间接注入最常从这里进来）、Output Gate（交付前——捕获 canary 泄露和被篡改的行为）。不只是扫描输入。

- **桶加权风险评分。** 发现按 4 个桶分类（意图 45%、载荷 25%、劫持 20%、异常 10%），每个桶内概率累积。多个中等信号可以叠加成高风险。3 个以上威胁类别触发 1.5 倍复合加成，最低分 60（HIGH RISK）。

## 对比

| | 人工审查 | 用 SkillScanner |
|---|---|---|
| 编码载荷 | 嵌套的 base64/hex/ROT13 看不出来 | 9 步标准化 + 2 层递归解码 |
| 换词攻击 | 能抓「忽略指令」，抓不住「放下操作参数」 | 词汇无关的意图复述，抓语义等价 |
| MCP 工具 | 读描述，祈祷它是诚实的 | 扫描 `<IMPORTANT>` 标签、秘密指令、描述注入、影子工具 |
| 教育伪装 | 「仅供研究」的框架把你骗了 | CRITICAL 类别永不削弱，per-finding 上下文检查 |
| 多向量攻击 | 每条发现孤立判断 | 概率桶累积 + 1.5 倍复合加成 |
| 工具返回注入 | 不扫描工具返回的内容 | Gate 2 在 Agent 采纳前扫描工具输出 |

## 工作原理

### 5 层协议

```
内容 → L1: 标准化 → L2: 模式检测 → L3: 语义分析 → L4: 上下文判断 → L5: 裁决
        (9 步,        (T1-T10,        (意图复述,        (per-finding,      (桶加权
         递归解码)      3 版文本)        12 案例, 7 原型)   削弱规则)          风险评分)
```

**Layer 1** 剥除编码混淆：base64、hex、URL、unicode 转义、ROT13、零宽字符、HTML 注释、同形字，并创建一份去除所有空白的压缩副本（抓 `i g n o r e` 这种空格分隔绕过）。解码最多递归 2 层。

**Layer 2** 对三个文本版本（标准化、解码载荷、压缩副本）运行全部 10 个威胁类别。输出编号列表，原样传递到 Layer 4。

**Layer 3** 加载攻击案例库，运行词汇无关意图检测：「不管用什么词，有没有任何段落试图改变身份、覆盖指令、或提取配置？」结构原型匹配捕获案例库中没有的新型攻击。

**Layer 4** 将每条发现分类为 true positive、false positive (INFO) 或 needs review。讨论上下文削弱（最多 25%）只作用于 needs_review 发现，只看周围 ±5 行，CRITICAL 类别永不削弱。扫描器规避模式（「已预扫描」「白名单」「跳过扫描」）自动判为 true positive。

**Layer 5** 汇入 4 个桶，概率累积，复合加成，映射到裁决：CLEAN / LOW RISK / SUSPICIOUS / HIGH RISK / CRITICAL。

### 输出格式

文本报告和 JSON 报告。每条发现包含严重度、类别、OWASP 标签、位置、证据、上下文判断和修复建议。

### 设计决策

| 决策 | 原因 |
|------|------|
| 纯指令，无代码 | 零依赖 = 到处能用。LLM 的语言理解力就是检测引擎。 |
| Per-finding 削弱，不是 per-document | 一行「仅供研究」不能压制 200 行之外的发现。经过对抗性测试验证。 |
| CRITICAL 类别永不削弱 | T1/T6/T7/T10 太危险，不能有任何抑制——教程框架里的真实攻击仍然是真实攻击。 |
| 桶加权评分 | 平坦的逐条累加表达不了多向量风险。概率累积意味着 3 个 MEDIUM ≠ 1 个 HIGH——可以超过它。 |
| 结构原型而非仅案例 | 12 个案例覆盖不了所有攻击。7 个抽象形状（身份覆盖、外泄构造等）捕获新型变体。 |

## 快速开始

```bash
# 1. 克隆
git clone https://github.com/d-wwei/SkillScanner.git

# 2. 安装（symlink 到你的 Agent skill 目录）
ln -sf "$(pwd)/SkillScanner" ~/.claude/skills/security-scan    # Claude Code
ln -sf "$(pwd)/SkillScanner" ~/.agents/skills/security-scan     # Codex / Gemini CLI

# 3. 扫描
# 在你的 Agent 中说：
#   "扫描这个 skill 有没有安全问题：[粘贴内容]"
#   "快速扫描这个 MCP 工具定义"
#   "scan this skill for security issues"
```

### 子命令

| 命令 | 功能 |
|------|------|
| `scan <内容>` | 完整 5 层扫描（加载所有 reference） |
| `quick-scan <内容>` | 仅 Layer 1+2——快速模式检查，无语义分析 |
| `gate <名称> <内容>` | 在特定 gate 扫描（input / tool-result / output） |
| `self-test` | 运行 4 层验证（基础 / 绕过 / 伪装 / 反向） |

## 文件结构

```
SkillScanner/
  SKILL.md                              # 路由器——姿态、红线、验收标准、工作流
  references/
    scan-protocol.md                    # 5 层协议 + 评分 + 3-Gate
    threat-catalog.md                   # T1-T10 威胁模式（中英双语，按平台分类）
    attack-cases.md                     # 12 个案例 + 7 个结构原型
    report-templates.md                 # 文本 + JSON 输出格式
    self-test.md                        # 4 层测试载荷
```

## 来源

综合 12 个项目和研究论文的洞察构建：

**检测模式与架构：** [prompt-shield](https://github.com/mthamil107/prompt-shield)、[AegisGate](https://github.com/ax128/AegisGate)、[Aigis](https://github.com/killertcell428/aigis)、[prompt-guard](https://github.com/useai-pro/openclaw-skills-security)

**扫描方法论：** [snyk/agent-scan](https://github.com/snyk/agent-scan)、[LLM Guard](https://github.com/protectai/llm-guard)、[Vigil-LLM](https://github.com/deadbits/vigil-llm)、[pytector](https://github.com/MaxMLang/pytector)

**威胁研究：** [Snyk ToxicSkills](https://snyk.io/blog/toxicskills-malicious-ai-agent-skills-clawhub/)、[Palo Alto Unit 42](https://unit42.paloaltonetworks.com/ai-agent-prompt-injection/)、[Meta Prompt Guard 2](https://huggingface.co/meta-llama/Llama-Prompt-Guard-2-86M)、[Sentinel](https://huggingface.co/qualifire/prompt-injection-sentinel)

## 许可

MIT
