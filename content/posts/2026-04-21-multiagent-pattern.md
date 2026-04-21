---
title: "MultiAgent 协作模式全解析：从单体 Agent 到分布式 AI 系统"
date: "2026-04-21T16:00:00+08:00"
tags: ["ai", "llm"]
toc: true
---

> 当你的 Agent 开始在复杂任务面前"力不从心"，不是模型不够强，而是架构不够好。这篇文章，我们彻底讲透多智能体系统的六大经典模式。

## 1. 引言：为什么单体 Agent 不够用了？

### 单体 Agent 的边界与天花板

2023 年，大多数人对 AI Agent 的想象是这样的：一个超级 LLM，接收用户输入，调用几个工具，输出结果。这个模型简洁优雅，在简单任务上表现出色。

但当你试图用它完成"分析竞争对手的产品策略、撰写一份 20 页的深度报告"时，问题开始暴露：

- **上下文窗口是有限的**。即便 GPT-4o 支持 128K tokens，一个需要反复迭代、持续积累上下文的长任务，终究会遇到天花板。
- **并行处理能力缺失**。单体 Agent 是串行思考的，它不能同时调研五个竞争对手，只能一个一个来。
- **错误传播不可控**。在一个长链任务中，第三步出错，后续所有步骤都在错误的基础上继续——没有隔离，没有检查点。
- **专业性存在边界**。一个通才 Agent 写代码不如专门调了代码数据集的 Agent，做财务分析不如经过金融语料微调的 Agent。

**单体 Agent 的本质是"一个大脑包揽一切"。** 这在人类组织中早已被证明是低效的，为什么在 AI 系统里还要走这条老路？

### 从"一个大脑"到"分工协作"的架构必然性

人类解决复杂问题的方式从来都不是依赖一个全知全能的个体。我们建立团队、设立角色、划定职责边界、通过沟通协作完成超越个人能力上限的任务。

多智能体系统（Multi-Agent Systems）正是这一思想在 AI 领域的工程化实践：

- **专业化**：每个 Agent 聚焦特定能力域，深而不广
- **并行化**：多个 Agent 可以同时工作，突破串行处理的速度瓶颈
- **可组合性**：不同 Agent 组合成不同的"工作流"，灵活应对多种场景
- **可观测性**：系统的每个节点状态清晰可见，便于调试和优化

这不是趋势，这是复杂系统演化的必然结果。

## 2. 核心概念：Workflow vs Agent

在深入模式之前，必须厘清一个经常被混用的概念对：**Workflow** 和 **Agent**。混淆这两者，会导致架构决策的根本性偏差。

### Workflow（工作流）：预定义代码路径，确定性执行

Workflow 是一种**由代码硬编码的控制流**。执行路径在编写时就已确定，输入 A 经过步骤 1、2、3 得到输出 B，这条路径是固定的、可预期的、可重复的。

```
Input → Step1 → Step2 → Step3 → Output
         ↓
    （LLM 只是其中一个处理节点）
```

Workflow 的核心优势是**确定性**。你知道它会怎么运行，你可以为它写测试，你可以精确地控制它的行为。

### Agent（智能体）：动态规划，基于环境反馈自主决策

Agent 是一种**由 LLM 驱动控制流**的系统。Agent 接收目标，自主决定采取什么行动，观察结果，再决定下一步——这个循环持续到任务完成。

```
Goal → [LLM decides: which tool? what args?] → Execute → Observe → [LLM decides again...]
```

Agent 的核心优势是**灵活性**。面对未预见的情况，它能自主调整策略，处理 Workflow 难以穷举的边缘情况。

### 关键差异：控制流由代码编排 vs. 由 LLM 驱动

| 维度 | Workflow | Agent |
|------|----------|-------|
| 控制流 | 代码决定 | LLM 决定 |
| 确定性 | 高 | 低 |
| 灵活性 | 低 | 高 |
| 可调试性 | 容易 | 困难 |
| 适用场景 | 结构化、可预期的任务 | 开放式、动态的任务 |
| 成本 | 可预测 | 波动较大 |

> **工程哲学**：能用 Workflow 解决的问题，不要过度引入 Agent 的自主性。自主性是把双刃剑——它带来灵活性，也带来不可预测性。**在确定性和灵活性之间找到正确的平衡点，是多智能体架构设计的核心命题。**


## 3. 多智能体系统的三大通信原语

在多个 Agent 需要协作时，它们通过什么方式"说话"？理解通信原语是理解所有协作模式的基础。

### 共享状态 (Shared Session State)：被动式数据传递

最简单的通信方式。Agent A 将数据写入共享状态（如一个 key-value store），Agent B 从中读取。

```python
# Agent A 写入
session_state["analysis_result"] = "竞品A的核心优势在于..."

# Agent B 读取
result = session_state.get("analysis_result")
```

**特点**：解耦、异步、无需直接调用。适合传递中间结果，但需要明确定义 key 的命名规范，避免写冲突。

### LLM 驱动委派 (LLM-Driven Delegation)：动态路由与 Agent Transfer

由 LLM 在运行时决定"把这个任务交给谁"。Orchestrator Agent 分析输入，动态路由到合适的 Sub-Agent。

```
User: "帮我分析这段 Python 代码是否有安全漏洞"
Orchestrator LLM: [分析] → "这是代码安全问题" → Transfer to SecurityCodeAgent
```

**特点**：高度灵活，可以处理多分类场景。但依赖 LLM 的路由判断，存在误判风险，需要良好的 Prompt 设计来提升路由准确率。

### 显式调用 (Explicit Invocation / AgentTool)：同步工具化调用

将一个 Agent 封装成另一个 Agent 可以调用的"工具"。调用者等待被调用 Agent 完成并返回结果。

```python
# Sub-agent 被包装为 Tool
class DataAnalysisAgentTool(Tool):
    def run(self, data: str) -> str:
        return data_analysis_agent.run(data)

# Orchestrator 像调用普通工具一样调用它
orchestrator.tools = [DataAnalysisAgentTool(), ...]
```

**特点**：同步、可预期、结果清晰。相比 LLM-Driven Delegation，更适合需要确定性调用关系的场景。

## 4. 六大经典协作模式详解

### 4.1 Prompt Chaining

#### 模式定义

串行处理流水线。任务被分解为若干有序步骤，前一步骤的输出作为后一步骤的输入。整个流程像工厂流水线一样顺序推进。

```
[Input] → [Step1: 提取关键信息] → [Step2: 翻译] → [Step3: 格式化] → [Output]
              output_key="extracted"    input=extracted   input=translated
```

#### 适用场景

- **翻译流水线**：原文 → 直译 → 润色 → 文化本地化
- **内容校验**：生成内容 → 事实核查 → 合规检查 → 最终输出
- **分步生成**：提纲 → 章节草稿 → 润色 → 排版

**核心判断标准**：任务能否被明确分解为有清晰先后依赖关系的子步骤？如果是，Prompt Chaining 是最简洁可靠的选择。

```python
"""
示例：三步内容生成流水线
步骤1: 生成文章大纲
步骤2: 基于大纲生成正文
步骤3: 润色优化
"""

from google.adk.agents import SequentialAgent, LlmAgent
from google.adk.sessions import InMemorySessionService
from google.adk.runners import Runner
from google.genai import types

# 步骤1: 大纲生成 Agent
outline_agent = LlmAgent(
    name="OutlineAgent",
    model="gemini-2.0-flash",
    instruction="""你是一位专业的内容策划师。
    用户会给你一个主题，请生成一份结构清晰的文章大纲（3-5个主要章节）。
    输出格式：仅输出大纲内容，每个章节标题单独一行。""",
    output_key="article_outline"  # 输出写入 session state
)

# 步骤2: 正文生成 Agent
draft_agent = LlmAgent(
    name="DraftAgent",
    model="gemini-2.0-flash",
    instruction="""你是一位专业的文章撰写师。
    请基于 session state 中 'article_outline' 的大纲，撰写完整的文章正文。
    要求：每个章节至少200字，语言流畅自然。""",
    output_key="article_draft"
)

# 步骤3: 润色 Agent
polish_agent = LlmAgent(
    name="PolishAgent",
    model="gemini-2.0-flash",
    instruction="""你是一位资深文字编辑。
    请对 session state 中 'article_draft' 的文章进行润色：
    1. 改善句子流畅度
    2. 增强表达的专业性
    3. 确保段落衔接自然
    输出润色后的最终版本。""",
    output_key="final_article"
)

# 组合为 SequentialAgent
pipeline = SequentialAgent(
    name="ContentPipeline",
    sub_agents=[outline_agent, draft_agent, polish_agent]
)

# 运行
session_service = InMemorySessionService()
runner = Runner(agent=pipeline, session_service=session_service, app_name="content_app")

session = session_service.create_session(app_name="content_app", user_id="user_1")
response = runner.run(
    user_id="user_1",
    session_id=session.id,
    new_message=types.Content(parts=[types.Part(text="多智能体系统的未来发展")])
)

print("=== 生成结果 ===")
print(session_service.get_session(
    app_name="content_app", user_id="user_1", session_id=session.id
).state.get("final_article"))
```

### 4.2 Parallelization(Fan-Out Gather)

#### 模式定义

将任务分发给多个 Agent 同时处理，最后聚合所有结果。从"一个 Agent 串行处理 N 件事"变成"N 个 Agent 并行处理"，速度提升理论上与 Agent 数量成正比。

```
                ┌─→ [Agent_A: 分析维度1] ─┐
[Input] → Split ├─→ [Agent_B: 分析维度2] ─┤→ Aggregate → [Output]
                └─→ [Agent_C: 分析维度3] ─┘
```

#### 适用场景

- **多源信息检索**：同时搜索 Google、Wikipedia、内部知识库
- **多维度评估**：从性能、安全、可维护性三个角度同时评审代码
- **投票机制**：三个 LLM 独立给出答案，取多数票减少偏差

```python
"""
示例：三维度并行产品评估
同时从「用户体验」「技术可行性」「商业价值」三个维度评估产品方案
"""

from google.adk.agents import ParallelAgent, LlmAgent, SequentialAgent
from google.adk.sessions import InMemorySessionService
from google.adk.runners import Runner
from google.genai import types

# 三个并行的专家 Agent
ux_expert = LlmAgent(
    name="UXExpert",
    model="gemini-2.0-flash",
    instruction="""你是一位资深用户体验专家。
    请从用户体验角度评估以下产品方案（100字以内）：
    评估维度：易用性、学习曲线、用户满意度预期。
    输出格式：[UX评分: X/10] 评估内容""",
    output_key="ux_assessment"
)

tech_expert = LlmAgent(
    name="TechExpert",
    model="gemini-2.0-flash",
    instruction="""你是一位资深技术架构师。
    请从技术可行性角度评估以下产品方案（100字以内）：
    评估维度：实现难度、技术风险、架构合理性。
    输出格式：[技术评分: X/10] 评估内容""",
    output_key="tech_assessment"
)

business_expert = LlmAgent(
    name="BusinessExpert",
    model="gemini-2.0-flash",
    instruction="""你是一位商业分析师。
    请从商业价值角度评估以下产品方案（100字以内）：
    评估维度：市场潜力、盈利模式、竞争优势。
    输出格式：[商业评分: X/10] 评估内容""",
    output_key="business_assessment"
)

# 结果聚合 Agent
aggregator = LlmAgent(
    name="Aggregator",
    model="gemini-2.0-flash",
    instruction="""你是一位产品总监。
    请综合以下三份评估报告（来自 session state 的 ux_assessment、
    tech_assessment、business_assessment），给出综合评估结论和建议。
    输出：综合评分、主要风险、关键建议（各不超过3条）""",
    output_key="final_assessment"
)

# 并行评估 + 串行聚合
evaluation_pipeline = SequentialAgent(
    name="EvaluationPipeline",
    sub_agents=[
        ParallelAgent(
            name="ParallelEvaluation",
            sub_agents=[ux_expert, tech_expert, business_expert]
        ),
        aggregator
    ]
)

# 运行
session_service = InMemorySessionService()
runner = Runner(
    agent=evaluation_pipeline,
    session_service=session_service,
    app_name="eval_app"
)

session = session_service.create_session(app_name="eval_app", user_id="user_1")
product_proposal = """
产品方案：AI 驱动的个人财务管理助手
- 自动分类记账，学习用户消费习惯
- 智能预算建议，提前预警超支风险
- 投资组合优化建议（非执行）
- 目标用户：25-40岁城市白领
"""

runner.run(
    user_id="user_1",
    session_id=session.id,
    new_message=types.Content(parts=[types.Part(text=product_proposal)])
)

session_data = session_service.get_session(
    app_name="eval_app", user_id="user_1", session_id=session.id
)
print("=== 综合评估结果 ===")
print(session_data.state.get("final_assessment"))
```

### 4.3 Routing

#### 模式定义

根据输入的类型或意图，将请求分发到专门处理该类型的 Agent。核心是一个"分诊台"——不是所有问题都要排同一个队。

```
                      ┌─→ [TechSupportAgent]
[Input] → [Router] ───┼─→ [BillingAgent]
                      └─→ [GeneralInfoAgent]
```

#### 适用场景

- **客服分诊**：技术问题 → 技术支持，账单问题 → 财务，投诉 → 客户成功
- **多领域问答**：法律问题 → 法律 Agent，医疗问题 → 医疗 Agent
- **意图路由**：写代码 → CodeAgent，写文案 → CopywritingAgent

```python
"""
示例：智能客服分诊系统
基于用户意图路由到不同专业 Agent
"""

from google.adk.agents import LlmAgent
from google.adk.sessions import InMemorySessionService
from google.adk.runners import Runner
from google.genai import types

# 专业 Agent
tech_agent = LlmAgent(
    name="TechSupportAgent",
    model="gemini-2.0-flash",
    instruction="""你是技术支持专家。请解答用户的技术问题，
    提供具体的操作步骤和解决方案。"""
)

billing_agent = LlmAgent(
    name="BillingAgent",
    model="gemini-2.0-flash",
    instruction="""你是财务客服专员。请处理用户的账单、退款、
    订阅相关问题，提供准确的账单说明。"""
)

general_agent = LlmAgent(
    name="GeneralInfoAgent",
    model="gemini-2.0-flash",
    instruction="""你是通用客服代表。回答用户关于产品功能、
    政策和一般性问题。"""
)

# 路由 Coordinator
coordinator = LlmAgent(
    name="CustomerServiceCoordinator",
    model="gemini-2.0-flash",
    instruction="""你是智能客服调度中心。分析用户问题并路由到合适的专员：
    - 技术问题（安装、Bug、功能异常）→ TechSupportAgent
    - 账单问题（付款、退款、订阅）→ BillingAgent
    - 其他问题 → GeneralInfoAgent
    
    使用 transfer_to_agent 工具完成路由，不要自己回答用户问题。""",
    sub_agents=[tech_agent, billing_agent, general_agent]
)
```

### 4.4 Orchestrator-Worker

#### 模式定义

这是多智能体系统中最强大、也最常用的模式。中心 Orchestrator Agent 负责：
1. 理解复杂任务目标
2. 动态分解为子任务
3. 并行分发给专业 Worker Agent
4. 综合所有结果生成最终输出

```
                    ┌─→ [Worker A: 子任务1] ─┐
[Goal] → [Orchestrator: 任务分解] ├─→ [Worker B: 子任务2] ─┤→ [Synthesizer] → Output
                    └─→ [Worker C: 子任务3] ─┘
```

与 Parallelization 的关键区别：**分解本身是动态的，由 LLM 决定如何拆分**，而非预定义。

#### 适用场景

- **复杂报告生成**：调研报告需要数据收集、竞品分析、市场预测等多个并行工作流
- **多步骤研究**：研究课题的不同方面可以并行推进
- **代码审查**：安全审查、性能分析、代码风格检查可以同时进行

```python
"""
示例：深度竞品分析系统
Orchestrator 动态决定分析维度，Workers 并行执行
"""

from google.adk.agents import LlmAgent
from google.adk.tools import agent_tool
from google.adk.sessions import InMemorySessionService
from google.adk.runners import Runner
from google.genai import types

# Worker Agents（被封装为 AgentTool）
product_feature_analyst = LlmAgent(
    name="ProductFeatureAnalyst",
    model="gemini-2.0-flash",
    instruction="""你是产品功能分析专家。
    分析给定产品/公司的核心功能特性、差异化竞争点。
    输出结构化的功能分析报告。"""
)

pricing_analyst = LlmAgent(
    name="PricingAnalyst",
    model="gemini-2.0-flash",
    instruction="""你是定价策略专家。
    分析给定产品/公司的定价模型、价格梯度、目标客群定价策略。
    输出定价策略分析。"""
)

market_position_analyst = LlmAgent(
    name="MarketPositionAnalyst",
    model="gemini-2.0-flash",
    instruction="""你是市场定位专家。
    分析给定产品/公司在市场中的定位、品牌认知、用户口碑。
    输出市场定位分析。"""
)

tech_stack_analyst = LlmAgent(
    name="TechStackAnalyst",
    model="gemini-2.0-flash",
    instruction="""你是技术分析专家。
    分析给定产品/公司的技术架构、AI能力、技术护城河。
    输出技术栈分析。"""
)

# 将 Workers 封装为工具
feature_tool = agent_tool.AgentTool(agent=product_feature_analyst)
pricing_tool = agent_tool.AgentTool(agent=pricing_analyst)
market_tool = agent_tool.AgentTool(agent=market_position_analyst)
tech_tool = agent_tool.AgentTool(agent=tech_stack_analyst)

# Orchestrator Agent
orchestrator = LlmAgent(
    name="CompetitiveIntelligenceOrchestrator",
    model="gemini-2.0-flash",
    instruction="""你是竞争情报分析调度专家。
    
    当用户要求分析某个竞争对手时，你需要：
    1. 调用以下工具并行收集各维度分析（一次性全部调用以提高效率）：
       - ProductFeatureAnalyst: 产品功能分析
       - PricingAnalyst: 定价策略分析
       - MarketPositionAnalyst: 市场定位分析
       - TechStackAnalyst: 技术栈分析
    
    2. 综合所有分析，生成完整的竞品报告，包括：
       - 执行摘要（核心发现，3条）
       - 四维分析详情
       - 对我方的战略建议（2-3条）
    
    报告要有深度，避免泛泛而谈。""",
    tools=[feature_tool, pricing_tool, market_tool, tech_tool]
)

# 运行
session_service = InMemorySessionService()
runner = Runner(agent=orchestrator, session_service=session_service, app_name="ci_app")

session = session_service.create_session(app_name="ci_app", user_id="analyst_1")
runner.run(
    user_id="analyst_1",
    session_id=session.id,
    new_message=types.Content(parts=[types.Part(
        text="请对 Notion AI 进行全面的竞品分析"
    )])
)
```

### 4.5 Evaluator-Optimizer

#### 模式定义

生成器（Generator）和评估器（Evaluator/Critic）形成**反馈循环**，持续迭代直到输出满足质量标准。这是将 AI 输出质量从"够用"提升到"优秀"的关键模式。

```
[Input] → Generator → [Draft]
              ↑              ↓
         [Revise]    Evaluator/Critic
              ↑              ↓
         ← ← ← [Pass? No → Feedback]
                     ↓ Yes
                  [Output]
```

#### 适用场景

- **代码生成**：生成代码 → 测试执行 → 修复 Bug → 再测试
- **文案精修**：初稿 → 专业点评 → 修改 → 再点评（直到评分达标）
- **质量达标控制**：任何有明确质量标准的生成任务

**关键机制：Generator-Critic 双角色交互**

这不仅仅是"自我修改"。Generator 和 Critic 应该有**不同的 Prompt**，甚至可以使用不同的模型。Critic 的职责是挑剔、严格、发现不足；Generator 的职责是创作、改进。角色分离避免了"自我美化"的偏见。

```python
"""
示例：代码质量迭代优化
Generator 生成代码，Critic 评审，循环优化直到满分
"""

from google.adk.agents import LoopAgent, LlmAgent
from google.adk.sessions import InMemorySessionService
from google.adk.runners import Runner
from google.genai import types

# Generator Agent
code_generator = LlmAgent(
    name="CodeGenerator",
    model="gemini-2.0-flash",
    instruction="""你是一位 Python 工程师。

    如果 session state 中没有 'code_draft'，
    根据用户需求生成初始代码实现。
    
    如果存在 'critic_feedback'，
    根据反馈改进 'code_draft' 中的代码。
    
    输出：只输出 Python 代码，不要解释。
    将代码存入 output_key 'code_draft'。""",
    output_key="code_draft"
)

# Critic/Evaluator Agent  
code_critic = LlmAgent(
    name="CodeCritic",
    model="gemini-2.0-flash",
    instruction="""你是一位严格的代码审查专家。
    
    审查 session state 中 'code_draft' 的代码，从以下维度评分（各25分）：
    1. 正确性：逻辑是否无误
    2. 可读性：命名、注释、结构
    3. 健壮性：错误处理、边界情况
    4. 性能：算法效率
    
    输出格式（严格遵守）：
    总分: XX/100
    
    问题清单:
    - [具体问题1]
    - [具体问题2]
    
    如果总分 >= 85，在最后一行输出：APPROVED
    
    将你的评审结果存入 output_key 'critic_feedback'。
    将 'APPROVED' 或 'NEEDS_REVISION' 存入 output_key 'review_status'。""",
    output_key="critic_feedback"  # 注：实际需要自定义 tool 写多个 key
)

# 终止条件检查 Agent
termination_checker = LlmAgent(
    name="TerminationChecker",
    model="gemini-2.0-flash",
    instruction="""检查 session state 中 'critic_feedback' 是否包含 'APPROVED'。
    如果包含 APPROVED，调用 escalate() 终止循环。
    否则，不做任何操作，让循环继续。""",
)

# 循环 Agent（最多5轮迭代）
optimization_loop = LoopAgent(
    name="CodeOptimizationLoop",
    sub_agents=[code_generator, code_critic, termination_checker],
    max_iterations=5
)

# 运行
session_service = InMemorySessionService()
runner = Runner(
    agent=optimization_loop,
    session_service=session_service,
    app_name="code_optimizer"
)

session = session_service.create_session(
    app_name="code_optimizer", user_id="dev_1"
)

runner.run(
    user_id="dev_1",
    session_id=session.id,
    new_message=types.Content(parts=[types.Part(
        text="""实现一个 LRU Cache 类，支持 get(key) 和 put(key, value) 操作，
        容量满时自动淘汰最久未使用的元素。时间复杂度要求 O(1)。"""
    )])
)

final_state = session_service.get_session(
    app_name="code_optimizer", user_id="dev_1", session_id=session.id
).state

print("=== 最终代码 ===")
print(final_state.get("code_draft"))
print("\n=== 评审结果 ===")
print(final_state.get("critic_feedback"))
```

### 4.6 Human-in-the-Loop

#### 模式定义

在 Agent 决策流中的**关键节点**引入人工审核。AI 提议，人决定。这不是对 AI 的不信任，而是在不确定性高、风险大、或规则复杂的场景下，设置合理的"安全阀"。

```
[Agent 分析] → [高风险操作?] → YES → [暂停，等待人工审核]
                                              ↓ 批准/拒绝
                              NO → [继续执行] ← ← ← ← ← ←
```

#### 适用场景

- **高风险操作**：删除数据、发送批量邮件、执行生产环境变更
- **审批流程**：合同签署、预算申请、权限变更
- **复杂判断**：医疗诊断建议、法律风险评估、重大投资决策

#### 基于策略引擎的确认机制

**进阶模式**：不是所有操作都需要人工确认——这会导致"确认疲劳"。更好的做法是引入 **Policy Engine**，根据预定义规则自动决定哪些操作需要人工介入。

```
操作请求 → PolicyEngine.evaluate(action) → 
    LOW_RISK: 自动执行
    MEDIUM_RISK: 记录日志，执行
    HIGH_RISK: 暂停，人工审核
    CRITICAL: 强制拒绝
```

```python
"""
Human-in-the-Loop 示例：策略引擎驱动的智能审批系统
低风险操作自动执行，高风险操作暂停等待人工确认
"""

import asyncio
from dataclasses import dataclass
from enum import Enum
from typing import Any, Optional
from google.adk.agents import LlmAgent
from google.adk.tools import FunctionTool
from google.adk.sessions import InMemorySessionService
from google.adk.runners import Runner
from google.genai import types

# ===== 策略引擎 =====
class RiskLevel(Enum):
    LOW = "low"
    MEDIUM = "medium"
    HIGH = "high"
    CRITICAL = "critical"

@dataclass
class PolicyDecision:
    risk_level: RiskLevel
    requires_approval: bool
    reason: str

class PolicyEngine:
    """基于规则的风险评估引擎"""
    
    RULES = {
        "delete": RiskLevel.HIGH,
        "send_email_batch": RiskLevel.HIGH,
        "update_production": RiskLevel.CRITICAL,
        "read_data": RiskLevel.LOW,
        "generate_report": RiskLevel.LOW,
        "send_notification": RiskLevel.MEDIUM,
        "update_config": RiskLevel.MEDIUM,
    }
    
    @classmethod
    def evaluate(cls, action: str, context: dict) -> PolicyDecision:
        base_risk = cls.RULES.get(action, RiskLevel.MEDIUM)
        
        # 上下文调整：影响人数越多，风险越高
        affected_users = context.get("affected_users", 0)
        if affected_users > 1000 and base_risk == RiskLevel.MEDIUM:
            base_risk = RiskLevel.HIGH
        
        requires_approval = base_risk in [RiskLevel.HIGH, RiskLevel.CRITICAL]
        
        return PolicyDecision(
            risk_level=base_risk,
            requires_approval=requires_approval,
            reason=f"操作'{action}'风险等级: {base_risk.value}"
        )

# ===== 人工确认模拟器 =====
class HumanApprovalGateway:
    """
    生产环境中，这里对接企业审批系统（钉钉、飞书、Slack）
    演示环境使用命令行交互
    """
    
    @staticmethod
    async def request_approval(
        action: str,
        details: dict,
        risk_level: str
    ) -> bool:
        print(f"\n{'='*50}")
        print(f"需要人工审核")
        print(f"操作: {action}")
        print(f"风险等级: {risk_level.upper()}")
        print(f"详情: {details}")
        print(f"{'='*50}")
        
        response = input("批准此操作? (yes/no): ").strip().lower()
        return response in ["yes", "y", "是", "批准"]

# ===== 工具定义 =====
approval_gateway = HumanApprovalGateway()
policy_engine = PolicyEngine()

async def execute_action(
    action: str,
    parameters: dict,
    description: str
) -> dict:
    """
    通用操作执行工具，内置策略引擎和人工审核
    
    Args:
        action: 操作类型（如 delete, send_email_batch, read_data）
        parameters: 操作参数
        description: 操作的人可读描述
    
    Returns:
        执行结果
    """
    # 策略评估
    decision = policy_engine.evaluate(action, parameters)
    
    if decision.risk_level == RiskLevel.CRITICAL:
        return {
            "status": "blocked",
            "reason": f"操作被策略引擎强制拒绝: {decision.reason}"
        }
    
    if decision.requires_approval:
        # 请求人工审核
        approved = await approval_gateway.request_approval(
            action=action,
            details={"description": description, "parameters": parameters},
            risk_level=decision.risk_level.value
        )
        
        if not approved:
            return {
                "status": "rejected",
                "reason": "操作被人工审核拒绝"
            }
    
    # 模拟执行（实际场景中调用真实 API）
    print(f"执行操作: {description}")
    return {
        "status": "success",
        "action": action,
        "message": f"操作'{description}'已成功执行",
        "auto_approved": not decision.requires_approval
    }

# ===== Agent 配置 =====
secure_agent = LlmAgent(
    name="SecureOperationsAgent",
    model="gemini-2.0-flash",
    instruction="""你是一位企业 IT 运营助手。
    
    当用户要求执行操作时，使用 execute_action 工具来完成。
    
    操作类型参考：
    - read_data: 读取数据
    - generate_report: 生成报告
    - send_notification: 发送通知
    - update_config: 更新配置
    - delete: 删除数据
    - send_email_batch: 批量发送邮件
    - update_production: 更新生产环境
    
    始终提供清晰的 description 说明操作目的和影响范围。""",
    tools=[FunctionTool(execute_action)]
)

# ===== 运行示例 =====
async def main():
    session_service = InMemorySessionService()
    runner = Runner(
        agent=secure_agent,
        session_service=session_service,
        app_name="secure_ops"
    )
    
    session = session_service.create_session(
        app_name="secure_ops", user_id="ops_1"
    )
    
    # 测试不同风险级别的操作
    test_requests = [
        "生成过去30天的用户活跃度报告",  # LOW risk，自动执行
        "向2000名用户发送产品更新通知邮件",  # HIGH risk，需要审核
    ]
    
    for request in test_requests:
        print(f"\n{'='*60}")
        print(f"用户请求: {request}")
        
        async for event in runner.run_async(
            user_id="ops_1",
            session_id=session.id,
            new_message=types.Content(parts=[types.Part(text=request)])
        ):
            if event.is_final_response():
                print(f"Agent 回复: {event.content.parts[0].text}")

asyncio.run(main())
```

## 5. 高级模式：组合与嵌套

单一模式解决单一类别的问题。**真实世界的复杂任务需要模式的组合与嵌套。**

### 模式叠加：Sequential 内嵌 Parallel + Loop

以"高质量深度研究报告生成"为例：

```
SequentialAgent (整体流水线)
├── ParallelAgent (多源并行信息收集)
│   ├── WebSearchAgent
│   ├── DatabaseQueryAgent  
│   └── ExpertInterviewAgent
├── LoopAgent (迭代优化报告质量)
│   ├── ReportDraftAgent
│   └── QualityCriticAgent (循环直到评分 >= 90)
└── HumanReviewAgent (最终发布前人工审核)
```

```python
from google.adk.agents import SequentialAgent, ParallelAgent, LoopAgent, LlmAgent

# 构建嵌套多智能体系统
research_pipeline = SequentialAgent(
    name="ResearchPipeline",
    sub_agents=[
        # 第1阶段：并行信息收集
        ParallelAgent(
            name="InformationGathering",
            sub_agents=[web_search_agent, db_query_agent, expert_agent]
        ),
        # 第2阶段：信息综合
        synthesis_agent,
        # 第3阶段：迭代优化
        LoopAgent(
            name="QualityOptimization",
            sub_agents=[draft_agent, critic_agent],
            max_iterations=3
        ),
        # 第4阶段：人工审核（仅对外发布报告）
        human_review_agent
    ]
)
```

### 多层代理树：Hierarchical Task Decomposition

对于极其复杂的任务，可以构建多层 Orchestrator 树：

```
Root Orchestrator
├── Research Orchestrator
│   ├── Domain Expert A
│   ├── Domain Expert B
│   └── Data Analyst
├── Writing Orchestrator
│   ├── Section Writer A
│   ├── Section Writer B
│   └── Editor
└── Review Orchestrator
    ├── Fact Checker
    ├── Legal Reviewer
    └── Brand Reviewer
```

**设计原则**：
- **每层 Orchestrator 只管辖直接下级**，避免跨层调度引入的复杂性
- **叶子节点是纯粹的执行者**，不做决策
- **信息向上汇聚**，决策向下分发

### 从固定工作流到半自主协作的渐进式设计

不要一上来就设计最复杂的多智能体系统。实践经验表明，最佳路径是**渐进演化**：

```
阶段1: 单体 Agent → 验证核心能力
阶段2: Prompt Chaining → 分解串行步骤
阶段3: 引入 Routing → 处理多类型输入
阶段4: Orchestrator-Worker → 复杂任务并行化
阶段5: 加入 Evaluator-Optimizer → 提升输出质量
阶段6: Human-in-the-Loop → 高风险场景加入人工干预
```

每个阶段都要有完整的评估，确认收益大于复杂性增加的成本，再进入下一阶段。

## 6. 选型指南：如何选择合适的模式？

| 模式 | 确定性 | 并发度 | 适用复杂度 | 人工介入 | 适用场景关键词 |
|------|--------|--------|------------|----------|----------------|
| Prompt Chaining | 高 | 低 | 低 | 可选 | 流水线、顺序依赖、步骤清晰 |
| Parallelization | 中 | 高 | 中 | 可选 | 多维评估、多源检索、投票 |
| Routing | 中 | 低 | 中 | 可选 | 分类分发、意图识别、专科处理 |
| Orchestrator-Worker | 低 | 高 | 高 | 可选 | 复杂任务、动态分解、综合报告 |
| Evaluator-Optimizer | 低 | 中 | 高 | 可选 | 质量迭代、反馈循环、达标控制 |
| Human-in-the-Loop | 中 | 低 | 高 | 必须 | 高风险操作、审批流程、合规要求 |

**决策树**：

```
任务是否可以明确分解为有序步骤？
├── 是 → Prompt Chaining
└── 否 → 任务是否需要多类型处理？
         ├── 是（输入分类）→ Routing
         └── 否 → 任务是否涉及多个独立子任务？
                  ├── 是（可并行）→ Parallelization 或 Orchestrator-Worker
                  │   └── 分解逻辑是否动态？→ 是 → Orchestrator-Worker
                  └── 是（质量要求高）→ Evaluator-Optimizer
                       └── 是否有高风险操作？→ 是 → Human-in-the-Loop
```

## 7. 工程实践：构建健壮的多智能体系统

### 状态管理最佳实践（Session State 设计）

Session State 是多智能体系统的"共享内存"。糟糕的状态设计会导致 Agent 间通信混乱、数据污染、难以调试。

```python
# 糟糕的状态设计
session.state["data"] = "..."  # key 太模糊
session.state["result"] = "..."  # 会被多个 Agent 覆盖

# 良好的状态设计
session.state["pipeline:step1:raw_data"] = "..."      # 命名空间隔离
session.state["pipeline:step2:analysis"] = "..."      # 阶段清晰
session.state["pipeline:final:report"] = "..."        # 最终输出标记
session.state["pipeline:metadata:iteration_count"] = 3  # 元数据分离
```

**状态设计原则**：
- **使用命名空间**：`{pipeline_name}:{stage}:{key}` 格式
- **不可变历史**：已完成步骤的状态不应被覆盖（追加 `_v2` 而非覆盖）
- **明确所有权**：每个 key 只有一个 Agent 负责写入
- **区分输入/输出**：读写分离，避免 Agent 误读自己的上一次输出

### 错误处理与超时控制

```python
import asyncio
from typing import Optional

class RobustAgentRunner:
    """健壮的 Agent 运行器，包含超时、重试、熔断"""
    
    def __init__(self, agent, max_retries: int = 3, timeout_seconds: int = 60):
        self.agent = agent
        self.max_retries = max_retries
        self.timeout = timeout_seconds
    
    async def run_with_resilience(
        self, 
        input_data: str,
        fallback_response: Optional[str] = None
    ) -> str:
        last_error = None
        
        for attempt in range(self.max_retries):
            try:
                result = await asyncio.wait_for(
                    self.agent.run_async(input_data),
                    timeout=self.timeout
                )
                return result
                
            except asyncio.TimeoutError:
                last_error = f"超时（{self.timeout}s），尝试 {attempt + 1}/{self.max_retries}"
                print(f"{last_error}")
                await asyncio.sleep(2 ** attempt)  # 指数退避
                
            except Exception as e:
                last_error = f"错误: {str(e)}"
                print(f"{last_error}")
                
                # 非可重试错误直接返回
                if "INVALID_ARGUMENT" in str(e):
                    break
        
        # 所有重试失败
        if fallback_response:
            return fallback_response
        raise RuntimeError(f"Agent 执行失败（{self.max_retries}次重试后）: {last_error}")
```

### 调试与可观测性（Tracing & Streaming）

多智能体系统的调试难度远超单体 Agent，因为问题可能出现在任意节点间的交互中。

```python
# ADK 内置的事件流可观测性
async def run_with_full_observability(runner, user_id, session_id, message):
    """带完整事件追踪的 Agent 运行"""
    
    event_log = []
    
    async for event in runner.run_async(
        user_id=user_id,
        session_id=session_id,
        new_message=message
    ):
        # 记录每个事件
        event_info = {
            "type": event.type,
            "agent": getattr(event, "author", "unknown"),
            "timestamp": asyncio.get_event_loop().time()
        }
        
        if event.content:
            # 工具调用事件
            for part in event.content.parts:
                if hasattr(part, "function_call"):
                    event_info["tool_call"] = {
                        "name": part.function_call.name,
                        "args": dict(part.function_call.args)
                    }
                    print(f"🔧 [{event_info['agent']}] 调用工具: {part.function_call.name}")
                
                elif hasattr(part, "function_response"):
                    event_info["tool_response"] = str(part.function_response.response)[:200]
                    print(f"工具响应: {str(part.function_response.response)[:100]}...")
                
                elif hasattr(part, "text") and part.text:
                    print(f"[{event_info['agent']}]: {part.text[:200]}")
        
        event_log.append(event_info)
        
        if event.is_final_response():
            print(f"\n任务完成，共 {len(event_log)} 个事件")
            break
    
    return event_log

# 使用 LangSmith 或 Langfuse 进行生产级追踪
# pip install langfuse
from langfuse import Langfuse

langfuse = Langfuse(
    secret_key="your-secret-key",
    public_key="your-public-key", 
    host="https://cloud.langfuse.com"
)

trace = langfuse.trace(name="multi_agent_pipeline")
span = trace.span(name="orchestrator_agent")
# ... 在 Agent 调用前后记录 span
```

### 从原型到生产的关键考量

**原型阶段（1-2周）**

- 使用内存型 Session（`InMemorySessionService`）快速验证逻辑
- 用简单的 print 语句追踪状态流转
- 固定使用一个测试用例，反复调整 Prompt

**测试阶段（2-4周）**

- 引入持久化 Session（数据库）
- 建立 Golden Dataset：50-100 个有标准答案的测试用例
- 对每种模式分别测试：正常路径、边界情况、错误情况

**生产阶段**

- **成本监控**：多智能体系统的 token 消耗是单体 Agent 的 N 倍，必须设置成本预警
- **延迟 SLA**：明确每个关键路径的最大容忍延迟，对超时任务降级处理
- **人工 fallback**：任何 Agent 失败时，要有降级到人工处理的通道
- **A/B 测试**：新模式上线前，用小流量对比旧实现的质量和效率

```python
# 生产级配置示例
PRODUCTION_CONFIG = {
    "max_retries": 3,
    "timeout_per_agent": 30,  # seconds
    "max_pipeline_time": 300,  # 5 minutes
    "cost_limit_per_request": 0.50,  # USD
    "fallback_enabled": True,
    "human_escalation_threshold": "CRITICAL",
    "observability": {
        "tracing": True,
        "metrics": True,
        "log_level": "INFO"
    }
}
```

## 8. 结语：多智能体架构的未来

### 从 Pattern 到 Platform 的演进

我们今天讨论的六种模式，是当前多智能体系统设计的"最佳实践语言"。就像微服务架构的模式（Service Mesh、Circuit Breaker、Sidecar）一样，这些模式最终会被平台化——框架会自动处理通信、状态管理、错误处理，开发者只需声明"我需要一个 Orchestrator-Worker 结构"，而无需手写每一行调度代码。

ADK、LangGraph、AutoGen、CrewAI 都在朝这个方向演进。未来的多智能体平台，将像 Kubernetes 之于容器化应用一样，成为 AI 应用部署的基础设施层。

### 自主协作与可控性的平衡艺术

多智能体系统面临的终极矛盾：**越自主，越强大；越自主，越危险。**

完全自主的系统可以处理任何复杂任务，但你永远不知道它会走哪条路。完全确定的系统可以预测每一个输出，但它处理不了你没有预见到的情况。

真正健壮的多智能体系统，是**在合适的层次设置合适的控制点**：

- **操作层**：低风险操作，完全自主
- **策略层**：中风险操作，基于规则引擎自动决策
- **战略层**：高风险操作，人工介入
- **架构层**：系统整体设计，人工把控

这不是一个技术问题，而是一个工程哲学问题：**你愿意在哪个层面信任 AI，在哪个层面保留人类的最终决定权？**

在这个问题上，没有普适的答案。但有一条原则是确定的：**不要因为系统能做某件事，就让它去做。先问为什么要让它做，以及如果它做错了，你能承担什么。**

多智能体架构的未来，不是构建"全自动的 AI 帝国"，而是构建**人机协同的智慧系统**——让 AI 处理它擅长的规模化、并行化、模式匹配任务；让人类保留对目标、价值观和边界条件的最终定义权。

## 参考资源

- [Google Agent Development Kit (ADK) 官方文档](https://adk.dev/agents/multi-agents/#human-in-the-loop-pattern)
- [LangGraph 多智能体文档](https://docs.langchain.com/oss/python/langgraph/workflows-agents#agents)
- [Anthropic Multi-Agent Patterns](https://www.anthropic.com/research/building-effective-agents)
- [AutoGen 框架](https://microsoft.github.io/autogen/)
