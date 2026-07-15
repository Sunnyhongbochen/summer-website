---
title: "Skill 知识体系整理：Skill / MCP / 工具 / Prompt 四者关系与创建指南"
date: 2026-07-15
description: "从零搭建 Skill 认知框架：Skill 与 MCP/Tool Calling/Prompt 的层级关系、Skill 目录结构与加载机制、6 步创建流程、3 个核心设计原则与发布前自测清单。"
tags: ["AI Agent", "Skill", "MCP", "Tool Calling", "Prompt", "Claude Code", "OpenClaw"]
---

> 📄 完整版飞书文档：https://feishu.cn/docx/Kz7NdC3o6oVR4CxvPZncOVrAnFg  
> 整理日期：2026-07-15  
> 参考来源：
> 1. [xiangzhihong《Skill学习指南》](https://xiangzhihong.blog.csdn.net/article/details/159977443)
> 2. [m0_46464899《skill、tool calling、MCP 区别》](https://blog.csdn.net/m0_46464899/article/details/160688573)

---

## 一、Skill 是什么

**一句话定义**：Skill（技能）是把"通用 AI 不具备、但领域专家天天用"的知识、操作流程和工具调用封装在一起的文件包，让 AI 具备领域专家能力。

**三要素公式**：

`Skill = 专业知识 + 操作流程 + 工具调用`

**最佳类比**：

- MCP = 给 AI 装新手臂（让 AI 能做到新的事）
- Prompt = 给 AI 贴标签（一次性人设）
- Skill = 给 AI 装操作手册（教 AI 怎么把手脚配合好、把活干规范）

**核心价值**：

- 把"反复要写、要查、要试"的内容固化下来
- 让 AI 跨任务、跨会话稳定复现一套行为
- 一次编写，团队/项目、人人都能复用

---

## 二、Skill / MCP / 工具（Tool Calling）/ Prompt 四者关系

### 2.1 LLM 本质

LLM（Claude、GPT、Kimi、DeepSeek）本质只会"输出文字"。让它能读文件、查天气、写代码，**必须**外面套一层机制。这一层就是 Tool Calling、MCP、Skill 等"扩展层"。

### 2.2 四层关系（从底层到上层）

**层级一：Tool Calling（工具调用）—— LLM 层的协议**

- 本质：LLM 厂商提供的 API 特性
- 关键事实：**LLM 不会真执行工具**，只返回一段结构化 JSON，表达"我想调用哪个工具、参数是什么"
- 真正执行的是你写的代码：解析 JSON → 调用函数 → 把结果回填对话 → 让模型生成最终回复
- 价值：解决了"模型如何表达想用工具"

**层级二：MCP（Model Context Protocol）—— 工具层的协议**

- 本质：Anthropic 2024 年底推出的工具"标准化接口"
- 形态：把工具做成可独立运行的 MCP Server（Python / Node / Rust 实现），监听端口或 stdio
- 价值：解决了"工具如何被分发和调用"
- 类比：MCP 之于 LLM 工具，就像 **USB 之于硬件外设**——插上就能用
- 关系：MCP Server 内部仍然用 Tool Calling 把工具暴露给模型
- 配套生态：Claude Desktop、Claude Code、Cursor、OpenClaw 等都支持 MCP

**层级三：Skill —— 行为 / 操作手册**

- 本质：放在 agent 能找到的地方的 Markdown 文件（通常为 SKILL.md）
- 价值：Tool 解决"能做什么"（capability），**Skill 解决"该怎么做"（know-how）**
- 关键事实：一个 Skill 通常会调用**多个工具**——可以是内置工具，也可以是 MCP 工具
- 配套生态：Claude Code（.claude/skills/）、OpenClaw（~/.openclaw/workspace/skills/）

**层级四：Prompt（含 System Prompt）—— 临时指令**

- 本质：一次性角色设定 / 风格调整
- 持久性：**仅当前对话**，关闭即失效
- 适用：临时调整 AI 风格、临时补充上下文

### 2.3 对比表

| 维度 | System Prompt | MCP | Skill | Tool Calling |
|------|--------------|-----|-------|--------------|
| 本质 | 一次性角色设定 | 工具 / 能力扩展协议 | 可复用行为约束 | LLM 调用工具的协议 |
| 解决什么问题 | 临时调整 AI 风格 | AI 连不到外部系统 | AI 不按规范工作 | 模型如何表达"想用工具" |
| 需要写代码 | 否 | 是（Server/Client） | 非必须（纯 Markdown 即可） | 是（解析 JSON + 执行） |
| 持久性 | 仅当前对话 | 持久（服务常驻） | 持久（文件存储） | 由调用方决定 |
| 面向人群 | 所有人 | 开发者 | 所有人 | 开发者 |
| 典型例子 | "你是一个翻译助手" | 连接数据库、调用 API | 强制 TDD、规范 Git 提交 | 模型返回 `{"name":"get_weather","args":{"city":"上海"}}` |

### 2.4 一句话总结

- **Tool Calling**：LLM "开口"说要工具的协议
- **MCP**：工具的"USB 接口"
- **Skill**：怎么用工具的操作手册
- **Prompt**：临时对话里的角色设定

---

## 三、Skill 的结构

### 3.1 目录结构

一个完整的 Skill 是**一个文件夹**，SKILL.md 是核心文件，其他按需扩展：

```
skill-name/
├── SKILL.md          # 核心（必须有）
├── scripts/          # 可选：可执行脚本（Python/Bash 等）
├── references/       # 可选：参考文档（按需加载）
└── assets/           # 可选：静态资源（模板、图片、字体等）
```

**各目录用途**：

- **scripts/**：把反复写的代码固化成脚本，**不需要读入上下文就能执行**，省 token
  - 典型：旋转 PDF（rotate_pdf.py）、分析日志（parse_log.py）
- **references/**：AI 需要参考但不必一直占上下文的知识
  - 典型：数据库表结构、API 文档、公司政策
  - 设计原则：**按领域拆分**文件，问销售时只加载 sales.md
- **assets/**：输出中会用到的文件、模板
  - 典型：前端模板、PPT 模板、品牌 Logo、checklist 模板

### 3.2 SKILL.md 格式

**YAML Front Matter + Markdown 正文**：

```yaml
---
name: skill-unique-name              # 技能的唯一标识符
description: |                       # 触发场景描述（AI 据此判断何时使用）
  Use when doing X.
  Use BEFORE doing Y.
  触发词：xxx、yyy
---

# 技能正文（Markdown）—— 给 AI 的操作指南

## 核心原则
- 原则一
- 原则二

## 执行流程
1. 第一步
2. 第二步
3. 第三步

## 禁止行为
- 禁止做 A
- 禁止跳过 B
```

### 3.3 Skill 五要素

编写 SKILL.md 时建议覆盖以下五个部分：

- **Metadata（元数据）**：`name`（唯一标识符）、`description`（触发场景）
- **Context（适用上下文）**：该技能适用的场景和前提条件
- **Process（执行流程）**：步骤化工作流，可包含 checklist
- **Constraints（约束规则）**：禁止行为、必须遵守的原则
- **Output Format（输出规范）**：结果如何呈现、什么格式

### 3.4 安装位置

```text
# Claude Code 中
~/.claude/skills/      # 全局技能（所有项目可用）
.claude/skills/        # 项目级技能（仅当前项目可用）

# OpenClaw 中
~/.openclaw/workspace/skills/
```

### 3.5 加载机制（按需加载，避免浪费上下文）

| 级别 | 内容 | 何时加载 | 大小 |
|------|------|---------|------|
| 第一级 | name + description | 始终在上下文中 | ~100 词 |
| 第二级 | SKILL.md 正文 | Skill 被触发后 | < 5000 词 |
| 第三级 | scripts/ references/ assets/ | AI 判断需要时 | 无限制 |

**实际效果**：AI 随时知道有哪些 Skill 可用，但只有真正需要时才"打开"它读正文或调脚本。

---

## 四、Skill 创建的经验及注意事项

### 4.1 六步创建流程

**第 1 步：明确使用场景**

动手前先想清楚：

- 用户会说什么话来触发这个 Skill？
- 期望 AI 完成什么具体任务？
- 有哪些典型的输入 / 输出例子？

**第 2 步：规划可复用资源**

对每个用例问自己：

> "完成这个任务，每次都需要重写什么代码 / 重查什么文档？"

把答案整理成 scripts/、references/、assets/ 的规划清单。

**第 3 步：初始化 Skill**

```bash
python scripts/init_skill.py <skill-name> --path <输出目录>
```

自动生成标准目录结构和 SKILL.md 模板。

**第 4 步：编写内容**

- `description` 是触发机制，必须清晰全面
- 正文控制在 500 行以内，细节放 references/
- 用祈使句 / 动词开头："运行脚本…"、"读取文件…"
- 脚本写完必须**实际运行测试**
- 参考文档超过 100 行要加目录
- 不需要的示例文件**全部删掉**

**第 5 步：打包发布**

```bash
python scripts/package_skill.py <skill目录路径>
```

打包前会自动校验：

- YAML frontmatter 格式
- 必填字段完整性
- 目录结构规范

校验通过后生成 `.skill` 文件（本质是 zip 包）。

**第 6 步：迭代优化**

在真实任务中使用 → 观察 AI 表现 → 持续改进 SKILL.md 和资源文件。

### 4.2 核心设计原则

**原则 1：精简优先**

上下文窗口是公共资源。每加一段文字前问自己：

> "这是 AI 不知道的信息吗？删掉会有什么后果？"

- ❌ "在开始之前，你需要理解这个任务的重要性…"
- ✅ 直接给出操作步骤

**原则 2：自由度要匹配任务**

| 任务特性 | 自由度 | 写法 |
|---------|--------|------|
| 步骤固定、易出错 | 低 | 具体脚本 + 严格参数 |
| 有首选方案、允许变化 | 中 | 伪代码 + 可配置参数 |
| 多种方案均可、依赖上下文 | 高 | 文字指导 + 启发式原则 |

**原则 3：渐进式披露**

SKILL.md 只放核心工作流，细节通过引用指向 references/：

```markdown
## 高级功能
- **表单填写**：详见 [FORMS.md](references/FORMS.md)
- **API 参考**：详见 [API.md](references/API.md)

## 日志格式
详见 [references/log-format.md](references/log-format.md)
```

### 4.3 经典实战案例

**案例 1：TDD 技能（强制测试驱动开发）**

问题：AI 总是直接写实现代码，跳过测试。

要点：

- 在 description 里强制声明 "Use this skill when the user asks to implement any feature"
- 强制工作流：先写测试 → 用户确认 → 写实现
- 禁止行为：先写实现再补测试、跳过测试、没有断言的测试
- 提供测试代码模板（pytest / Jest）

**案例 2：规范化 Git 提交技能**

要点：

- description 锁定触发词：commit、write commit messages、git commit
- 强制 Conventional Commits 规范
- 必填 Type（feat / fix / docs / ...）+ 必填 Scope（影响模块）+ Subject（≤50 字符）
- 提供正确示例和反例

### 4.4 常见反模式

- ❌ description 写得很笼统（AI 不知道何时触发）→ ✅ 写清"什么场景用、用户会说什么"
- ❌ 把所有细节塞进 SKILL.md（浪费上下文）→ ✅ 正文 500 行以内，细节放 references/
- ❌ 写不验证的脚本 → ✅ 脚本写完必须实际跑通
- ❌ 一份 Skill 想覆盖 N 个场景 → ✅ 一个 Skill 只做一个事，做透
- ❌ 用陈述句、被动语态 → ✅ 用祈使句、动词开头

### 4.5 验证清单（自测用）

发布前对照下面 7 条逐项打勾：

1. SKILL.md 是否有完整的 YAML frontmatter（name + description）
2. description 是否清晰描述了"何时用、用户会说什么"
3. 正文是否在 500 行以内
4. 脚本是否已经实测通过
5. references 文档是否加了目录（>100 行时）
6. 是否有"禁止行为"清单
7. 是否有明确的"输出格式"约束

---

## 五、关键洞察

- **Skill 是"软件工程化"的 AI 能力封装**。它和 prompt 的根本区别不是"长度"，而是"复用性 + 可执行性 + 团队可传递性"。
- **Skill 不必从零写起**。可以先用 init 脚本生成标准结构，再填业务内容。
- **description 是 Skill 最重要的部分**。AI 不读 SKILL.md 正文就触发不到它，description 写得差，Skill 等于不存在。
- **Skill 和 MCP 是互补关系，不是替代关系**。MCP 提供"新能力"，Skill 教"怎么用好已有能力"。好的 Skill 往往会同时调用多个 MCP 工具。
- **Skill 的价值随使用次数指数级增长**。一次编写，跨项目、跨团队、跨会话复用，是个人 / 团队 AI 资产的真正沉淀形式。
