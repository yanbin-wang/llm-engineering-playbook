# LLM Engineering Playbook · 大模型应用工程：方法边界与选型手册

> 这不是学习笔记，是一本**工程决策手册**：记录的不是"学了什么"，而是"遇到什么问题 → 选什么方案 → 边界在哪"。
> 范围是方法边界与选型：什么时候轮到提示工程、什么时候轮到微调，以及写 prompt 时采用哪套现成标准（CO-STAR 或 Anthropic 官方最佳实践）。方法部分不自创维度，直接采用这两个标准。

## 怎么用这本手册

先读[两条技术路线](#两条技术路线)和[产品落地优先级](#产品落地优先级)，决定走推理时工程还是微调，以及落地时先做哪一步。

需要写 prompt 时，再看[何时用 CO-STAR 与 Anthropic](#何时用-co-star-与-anthropic)，然后打开对应标准全文：内容创作、面向特定受众用 [CO-STAR](docs/co-star.md)；API 调用、输出被程序消费用 [Anthropic 官方最佳实践](docs/anthropic-prompt-engineering.md)。

术语对照、方法谱系、在线 API 能力边界和一句话记忆都在本页后半，用下面的目录跳转。Prompt Tuning 的机制细节、提示注入、IAG 组内调研见[附录](docs/appendix.md)。

## 目录

- [怎么用这本手册](#怎么用这本手册)
- [边界判断](#边界判断)
  - [两条技术路线](#两条技术路线)
    - [路线一：推理时工程（Inference-time Engineering）——不改权重](#路线一推理时工程inference-time-engineering不改权重)
      - [A. 纯提示层（Prompt-level，纯 API 调用可实现）](#a-纯提示层prompt-level纯-api-调用可实现)
      - [B. 系统层（System-level，需 API 外基础设施）](#b-系统层system-level需-api-外基础设施)
    - [路线二：微调（Fine-tuning）——改权重](#路线二微调fine-tuning改权重)
  - [产品落地优先级](#产品落地优先级)
  - [何时用 CO-STAR 与 Anthropic](#何时用-co-star-与-anthropic)
  - [核心概念澄清](#核心概念澄清)
    - [Prompt 相关术语](#prompt-相关术语)
  - [Prompt 工程内部的方法论](#prompt-工程内部的方法论)
    - [按示例数量分](#按示例数量分)
    - [推理增强提示（Reasoning-augmented）——均不需要微调](#推理增强提示reasoning-augmented均不需要微调)
  - [API 服务的能力边界（OpenAI / DeepSeek / GLM 等在线 API）](#api-服务的能力边界openai--deepseek--glm-等在线-api)
  - [一句话记忆](#一句话记忆)
- [维护约定](#维护约定)
- [CO-STAR 框架（中文版）](docs/co-star.md)
  - [框架定义](docs/co-star.md#框架定义)
  - [各要素说明](docs/co-star.md#各要素说明)
  - [完整示例（译自原作）](docs/co-star.md#完整示例译自原作)
  - [配套技巧：分隔符](docs/co-star.md#配套技巧分隔符)
  - [适用边界](docs/co-star.md#适用边界)
- [Anthropic 官方提示工程最佳实践（中文版）](docs/anthropic-prompt-engineering.md)
  - [1. Be clear and direct（清晰直接）](docs/anthropic-prompt-engineering.md#1-be-clear-and-direct清晰直接)
  - [2. Use examples (multishot prompting)（用示例）](docs/anthropic-prompt-engineering.md#2-use-examples-multishot-prompting用示例)
  - [3. Let Claude think (chain of thought)（让模型思考）](docs/anthropic-prompt-engineering.md#3-let-claude-think-chain-of-thought让模型思考)
  - [4. Use XML tags（用 XML 标签结构化）](docs/anthropic-prompt-engineering.md#4-use-xml-tags用-xml-标签结构化)
  - [5. Give Claude a role（给角色）](docs/anthropic-prompt-engineering.md#5-give-claude-a-role给角色)
  - [6. Prefill the response（预填充响应）](docs/anthropic-prompt-engineering.md#6-prefill-the-response预填充响应)
  - [7. Chain prompts（链式提示）](docs/anthropic-prompt-engineering.md#7-chain-prompts链式提示)
  - [8. Long context tips（长上下文技巧）](docs/anthropic-prompt-engineering.md#8-long-context-tips长上下文技巧)
  - [官方建议的通用工作流](docs/anthropic-prompt-engineering.md#官方建议的通用工作流)
- [附录：机制细节、安全项与组内调研](docs/appendix.md)
  - [Prompt Tuning 怎么做（机制，理解边界即可）](docs/appendix.md#prompt-tuning-怎么做机制理解边界即可)
  - [提示注入（安全项）](docs/appendix.md#提示注入安全项)
  - [IAG 组内调研](docs/appendix.md#iag-组内调研)

## 边界判断

用数据继续训练模型、修改模型权重，让模型"内化"格式或领域知识。与推理时工程是**两条独立路线**，不是同一件事的两个阶段。

### 两条技术路线

#### 路线一：推理时工程（Inference-time Engineering）——不改权重

绝大多数产品的主路线，又分两层：

##### A. 纯提示层（Prompt-level，纯 API 调用可实现）

- 系统消息硬性约束输出格式（用分隔符隔离指令与内容）
- **零样本提示**（Zero-shot）：只给指令，不给示例
- **少样本提示**（Few-shot）：在 prompt 里给 1~3 个"输入→期望输出"示例，让模型模仿
- 低 temperature / top_p，降低随机性
- API 结构化输出参数（`response_format` 设为 `json_object` 或 JSON Schema），从解码层杜绝格式错误
- 分步拆解、链式调用：把复杂任务拆成多次 API 调用，每次任务更窄

##### B. 系统层（System-level，需 API 外基础设施）

- **输出校验 + 自动重试**：应用层用 JSON Schema / 正则校验，不合规就重调并把错误反馈给模型
- **RAG**（检索增强生成）：先检索相关资料再让模型回答，减少胡编、提升事实准确率
- **上下文工程**：向量库、文档处理、查询重写、状态管理
- **Function Calling / 工具调用**：让模型调函数查真实数据，服务端需实现函数端点

#### 路线二：微调（Fine-tuning）——改权重

适用场景（**不是起点，是升级选项**）：

1. 格式极复杂，结构化输出参数搞不定
2. 领域术语/风格要求高，提示里塞示例太占 token
3. 延迟敏感，需要缩短提示
4. 厂商提供微调 API 且成本可接受

### 产品落地优先级

目标：每次按格式返回正确结果。按顺序做，前一步不够再上下一步：

1. **启用 API 结构化输出参数**（如支持）
2. **配低 temperature**
3. **系统消息写死格式约束**
4. **应用层加校验重试**（兜底保障）

前三步纯 API 调用可做，覆盖大部分格式稳定性问题；第四步需少量后端代码。**推理时工程 + 结构化输出 + 校验重试可覆盖绝大部分场景，不需要微调。**

### 何时用 CO-STAR 与 Anthropic

前人已经给出成熟方案，**不自创提示设计的维度**。写 prompt 时按任务类型直接采用以下两个标准（全文见 [CO-STAR 框架](docs/co-star.md) 与 [Anthropic 官方最佳实践](docs/anthropic-prompt-engineering.md)）：

- **[CO-STAR](docs/co-star.md)**：六要素 C 背景 / O 目标 / S 风格 / T 语调 / A 受众 / R 响应。适合内容创作类、输出面向特定受众的任务。
- **[Anthropic 官方最佳实践](docs/anthropic-prompt-engineering.md)**：8 条技巧——清晰直接 / 示例 / CoT / XML 标签 / 角色 / 预填充 / 链式提示 / 长上下文。适合 API 调用、输出被程序消费的工程场景。

| 场景 | 用哪个 |
|---|---|
| 内容创作、文案、面向特定受众的输出 | [CO-STAR](docs/co-star.md) |
| API 工程调用，输出被下游程序消费 | [Anthropic 清单](docs/anthropic-prompt-engineering.md)（核心三件：XML 标签隔离 + 示例 + 显式输出格式） |
| 复杂工程 prompt | 并用：[CO-STAR](docs/co-star.md) 定"要哪些要素"，[Anthropic](docs/anthropic-prompt-engineering.md) 定"怎么组织"（system prompt 角色、XML 分隔、示例数量） |

---

### 核心概念澄清

#### Prompt 相关术语

| 术语 | 英文 | 含义 |
|---|---|---|
| 提示工程 | Prompt Engineering | 系统性设计提示词与调用参数，让模型稳定、可重复、规模化地输出正确结果 |
| 提示设计 | Prompt Design | 侧重沟通与表达：写清指令、设定角色、明确格式。常与 Prompt Engineering 混用 |
| 提示微调 | Prompt Tuning | 改模型内部参数的技术：训练可学习的"软提示"向量，**不是人写的文字** |
| 前缀调优 | Prefix Tuning | Prompt Tuning 的近亲，在每层 Transformer 前加可训练前缀向量 |

**关键区分：**

> Prompt Design / Engineering 是**推理时工程**（模型权重不动）；
> Prompt Tuning / Prefix Tuning 是**参数高效微调**（改内部参数，需要模型权重访问权限）。

两条线的"怎么做"：Prompt Design 直接用现成标准，见 [CO-STAR 框架](docs/co-star.md) 与 [Anthropic 官方最佳实践](docs/anthropic-prompt-engineering.md)；Prompt Tuning 的机制见[附录](docs/appendix.md#prompt-tuning-怎么做机制理解边界即可)。

### Prompt 工程内部的方法论

#### 按示例数量分

- **零样本**（Zero-shot）：不给示例，只写指令
- **少样本**（Few-shot / In-Context Learning）：在 prompt 里给示例让模型模仿。纯文本操作，成本最低

#### 推理增强提示（Reasoning-augmented）——均不需要微调

- **Chain-of-Thought（CoT，思维链）**：让模型先输出中间推理步骤再给答案，提升复杂推理表现
- **Self-Consistency（自洽性）**：同一问题多次采样（调高 temperature 让路径不同），对最终答案多数投票。代价是 API 调用次数翻倍。适合有唯一答案的数学/逻辑题
- **ReAct**：让模型"边想边查"——循环执行 Thought（思考缺什么）→ Action（决定调哪个工具）→ Observation（系统执行查询并返回结果），直到信息足够给出答案。需要外部工具配合，不是纯 API 调用能自动完成的

提示注入（安全项）见[附录](docs/appendix.md#提示注入安全项)。

### API 服务的能力边界（OpenAI / DeepSeek / GLM 等在线 API）

| 能做什么 | 不能做什么 |
|---|---|
| Prompt Design / Engineering | 真正的 Prompt Tuning（需注入软提示向量） |
| 零样本/少样本提示、CoT、Self-Consistency | Prefix Tuning |
| 结构化输出参数、低温度采样 | — |
| 构建 RAG（厂商多提供兼容 Embedding 接口） | 访问模型权重与训练流程 |
| 调用厂商微调 API（如有，本质是改权重） | — |

IAG 组内调研的能做与不能做见[附录](docs/appendix.md#iag-组内调研)。

### 一句话记忆

- Prompt Design = 写给人看的提示词；Prompt Tuning = 训给机器用的提示向量
- Self-Consistency = 闭卷考多遍取平均分；ReAct = 开卷考，允许翻书查资料
- 推理增强提示不需要微调；微调是独立路线，**不是起点**

---

## 维护约定

- 只收"经过实践检验或有明确文献依据"的判断；方法类内容不自创维度，直接译介 CO-STAR 与 Anthropic 两个标准。
- 入口页只留边界判断。CO-STAR、Anthropic 的全文，以及机制细节、安全项、组内调研，分文件放在 `docs/`。
- 页面公开，欢迎学生指正——发现判断过时直接提 issue。

* maintained by [@yanbin-wang](https://github.com/yanbin-wang)
