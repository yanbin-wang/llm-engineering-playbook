# LLM Engineering Playbook · 大模型应用工程

这是一本 LLM 应用工程 playbook，讲常用工程路线，以及每条路线怎么做。

默认顺序 **A → B → C**。A 和 B 不改权重，是绝大多数产品的主路线；多数任务不必上 C。

| 路线 | 改权重 | 需要什么 | 什么时候进入 |
|---|---|---|---|
| **A 纯提示层** | 否 | 只有 API 调用 | 默认起点 |
| **B 系统增强的推理时工程** | 否 | API 之外的基础设施：校验重试 / RAG / 工具 / 上下文 | A 之后格式仍不稳；依赖外部资料或真实数据 |
| **C 微调**（含 PEFT） | 是 | 模型权重访问权限 + 训练数据与成本 | A、B 都到边界才考虑 |

**正文**：[A 纯提示层](#路线-a-纯提示层) · [B 系统增强](#路线-b-系统增强的推理时工程) · [C 微调](#路线-c-微调) · [术语与边界](#术语与边界)

**附录**：[CO-STAR 框架](docs/co-star.md) · [Anthropic 官方最佳实践](docs/anthropic-prompt-engineering.md) · [机制细节 · 安全项 · 组内调研](docs/appendix.md)

## 路线 A 纯提示层

### 特点

权重不动（Prompt-level）。格式、风格和步骤都落在提示和调用参数上。

### 使用场景

不依赖外部资料和工具，指令加示例就能交付。

### 怎么做

- 系统消息写死格式，用分隔符隔离指令与内容
- 零样本：只给指令。少样本：给 1~3 个「输入→期望输出」示例（纯文本，成本最低）
- 调低 `temperature` / `top_p`，压低随机性
- `response_format` 设为 `json_object` 或 JSON Schema，从解码层杜绝格式错误
- 复杂任务拆成多次 API 调用，每次更窄
- CoT：先输出推理步骤再给答案
- Self-Consistency：调高 `temperature` 多次采样，对最终答案投票。调用次数翻倍，适合有唯一答案的数学/逻辑题

**升级信号**：格式不稳 → 结构化输出 → 调低 `temperature` → 系统消息锁格式；仍不稳就上 B 的校验重试（需少量后端代码）。前三步纯 API 可做。

写 prompt 不自创维度：

- [CO-STAR](docs/co-star.md)：C 背景 / O 目标 / S 风格 / T 语调 / A 受众 / R 响应。内容创作、面向特定受众。
- [Anthropic 官方最佳实践](docs/anthropic-prompt-engineering.md)：清晰直接 / 示例 / CoT / XML 标签 / 角色 / 预填充 / 链式提示 / 长上下文。API 调用、输出被程序消费。核心三件：XML 标签隔离 + 示例 + 显式输出格式。
- 复杂工程 prompt 并用：CO-STAR 定要素，Anthropic 定组织（system prompt 角色、XML 分隔、示例数量）。

**升级信号**：需要查外部资料或调真实数据时，进入 B（RAG / Function Calling / ReAct）。

## 路线 B 系统增强的推理时工程

### 特点

权重不动，要 API 之外的基础设施（System-level）。

### 使用场景

A 之后格式仍不稳；知识在模型外面、资料会变；必须调函数查真实数据。

### 怎么做

- 输出校验 + 自动重试：JSON Schema / 正则校验，不合规就重调，并把错误反馈给模型
- RAG：先检索相关资料再回答，减少胡编、提升事实准确率
- 上下文工程：向量库、文档处理、查询重写、状态管理
- Function Calling：模型调函数查真实数据，服务端实现函数端点
- ReAct：循环 Thought（缺什么）→ Action（调哪个工具）→ Observation（系统返回结果），直到能作答。需要外部工具，纯 API 不能自动完成

提示注入见[附录](docs/appendix.md#提示注入安全项)。

## 路线 C 微调

### 特点

用数据继续训练、修改权重，让模型内化格式或领域知识（Fine-tuning，含 PEFT）。

### 使用场景

1. 格式极复杂，结构化输出参数搞不定
2. 领域术语/风格要求高，提示里塞示例太占 token
3. 延迟敏感，需要缩短提示
4. 厂商提供微调 API 且成本可接受

### 怎么做

走厂商微调 API（本质是改权重）。Prompt Tuning 训练软提示向量，Prefix Tuning 在每层 Transformer 前加可训练前缀；都不是人写的文字，都要能访问模型权重。在线 API 没有这个入口，机制见[附录](docs/appendix.md#prompt-tuning-怎么做机制理解边界即可)。

## 术语与边界

| 术语 | 是什么 | 落在 |
|---|---|---|
| **Prompt Engineering / Prompt Design** | 系统性设计人写的提示与调用参数（常混用） | A；要工具或检索时进 B |
| **Prompt Tuning / Prefix Tuning** | 训出来的软提示/前缀向量，不是人写的文字，需访问模型权重 | C |

在线 API（OpenAI / DeepSeek / GLM）：

| 能做 | 不能做 |
|---|---|
| Prompt Design / Engineering | 真正的 Prompt Tuning（需注入软提示向量） |
| 零样本/少样本提示、CoT、Self-Consistency | Prefix Tuning |
| 结构化输出参数、低温度采样 | — |
| 构建 RAG（厂商多提供兼容 Embedding 接口） | 访问模型权重与训练流程 |
| 调用厂商微调 API（如有，本质是改权重） | — |

Self-Consistency = 闭卷考多遍取平均分；ReAct = 开卷考，允许翻书查资料。Prompt Design = 写给人看的提示词；Prompt Tuning = 训给机器用的提示向量。

IAG 组内调研见[附录](docs/appendix.md#iag-组内调研)。

## 维护约定

- 只收经过实践检验或有明确文献依据的判断；方法不自创维度，直接译介 CO-STAR 与 Anthropic。
- 入口页只写三条路线。CO-STAR、Anthropic 全文，以及机制细节、安全项、组内调研，放在 `docs/`。
- 页面公开，欢迎学生指正——发现判断过时直接提 issue。

* maintained by [@yanbin-wang](https://github.com/yanbin-wang)
