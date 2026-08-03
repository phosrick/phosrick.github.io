---
title: '大模型响应截断自愈：JSON 正则挽救与 Schema 容错实战'
description: '在大模型提取长关系数组时遭遇 Token 截断导致 JSON 解析崩溃。详解多级解析防线、基于正则的 Pattern Rescue 字符流挽救机制以及 Agent 防御性编程架构。'
publishDate: '2026-08-03 05:15:00'
tags:
  - AI Agent
  - Python
  - 容错机制
  - LLM
category: 圆桌讨论·AI能力评估
---

## 1. 生产环境的隐形杀手：被截断的 LLM JSON 响应

在 V5 圆桌讨论能力评估系统里，我们需要通过大模型（LLM）从动辄数万字的无领导小组讨论转写文本中提取极其复杂的结构化数据。

例如在 **协作中心性评分器（Scorer 5_3）** 中，为了构建成员之间的互动有向图，模型需要按时间顺序梳理整场讨论中所有说话人之间的支持（`support`）与否定（`deny`）互动关系：

```python
prompt = f"""你是一个熟悉无领导小组讨论的专家...
你需要按照时间顺序，依次统计所有发言人之间出现的互相支持或互相否定的情况...
只返回JSON数组，不要有其他内容。
"""
```

当讨论轮次较多、互动频繁时，LLM 返回的 JSON 数组往往长达几百行，包含数十条边（Arcs）。然而，在实际运行长流水线时，我们频频遇到一个致命的生产痛点：**LLM 响应被截断**。

不管是由于达到模型的 `max_tokens` 上限、上下文窗口接近临界值，还是由于 API 网关超期强制关停生成流，模型在输出到 95% 时突然戛然而止：

```json
[
    {"source": "说话人 2", "target": "说话人 3", "relation": "support"},
    {"source": "说话人 3", "target": "说话人 4", "relation": "deny"},
    {"source": "说话人 4", "target": "说话人 2", "relation": "support"},
    {"source": "说话人 5", "target": "说话
```

当这段缺失了结尾右括号 `]` 和最后一个对象闭合双引号的“残缺字符串”传入 Python 时，标准的 `json.loads()` 会毫不留情地抛出异常：

```text
json.decoder.JSONDecodeError: Unterminated string starting at line 6 column 28 (char 245)
```

如果是在单线程脚本中，这会导致整个计算中断；如果是在 `ThreadPoolExecutor` 并发跑 16 个评分器的调度器中，某个 Worker 线程抛出末端 Decode 错误，轻则导致该维度的评分结果归零，重则引发连锁状态穿透。

---

## 2. 为什么常规策略在 Agent 系统中“通通打靶”？

面对 `JSONDecodeError`，传统的软件工程思维和初级的 Agent Prompt 技巧通常有三种处理方案，但在复杂长流水线中，它们无一例外都会失效：

### 失败方案 A：直接 `try...except` 捕获并扔掉

```python
try:
    return json.loads(response)
except json.JSONDecodeError:
    return [] # 失败就返回空
```

* **痛点**：这意味着前面生成的 95% 有效数据（比如已经精准提取出的 40 条互动边）因为末端 5% 的字符截断被完全废弃！有向图退化为空图，下游 NetworkX 无法计算任何入度与中心性，评估报告直接少了一项核心能力。

### 失败方案 B：触发 LLM 重新生成（Retry 机制）

```python
for attempt in range(max_retries):
    try:
        return parse_json(llm.chat(prompt))
    except ParseError:
        continue
```

* **痛点**：
  1. **开销翻倍**：一次提取长文本对话往往消耗数千 Input Tokens 和上千 Output Tokens，盲目重试极大增加了 API 费用与延时；
  2. **陷入死循环式截断**：由于输入的对话转写文本长度未变、Prompt 相同，LLM 输出的内容长度也是高度确定的。在相同的 `max_tokens` 限制下，重试 3 次大概率依然会在同样的位置被截断。

### 失败方案 C：在 Prompt 里强调“请严格输出合法的 JSON”

```text
警告：你必须返回合法的 JSON，绝对不能在末尾截断！
```

* **痛点**：Prompt 是用来约束模型**意图**的，无法突破底层物理与架构限制。当生成的 Token 数量达到模型硬上限（如 4096 tokens）时，生成引擎的解包逻辑会强制 Cutoff，Prompt 的告诫在硬性 Token Limit 面前无能为力。

---

## 3. 思考转变：将 LLM 响应视为“半结构化字符流”而非“标准 API”

要彻底解决这个问题，必须在架构设计层面完成一次思想转变：

> **在 Agent 防御性编程中，LLM 输出不能被视为严格的 RESTful API 响应，而应该被视为一种带有某种模式（Pattern）的半结构化字符流。**

即便截断破坏了全局 JSON 的语法结构（破坏了最外层的 `[]` 或末端 `}`），但在被截断点之前，那些**已经完整打印出来的 JSON 子对象，其内部的键值对依然是 100% 正确且合法的领域数据**。

既然数据本身是正确的，我们为什么一定要因为最外层少了一个 `]`，就否定前面几十个完整提取的有效节点呢？

由此，我们在 V5 评分器中设计了一套 **多级自愈解析防线（Multi-Tier Parsing Defense Pipeline）**，重点引入基于正则抽离的 **Pattern Rescue（字符流挽救）** 机制。

---

## 4. 多级自愈解析防线设计与实战

在 V5 的`s5_2_conflict_resolution.py` 和`s5_3_collaboration_centrality.py` 中，我们将 JSON 的解析过程拆解为四级自愈防线：

```text
              LLM 原始响应字符流
                     │
                     ▼
  ┌──────────────────────────────────────────────┐
  │ 第一道防线 (Tier 1): Markdown 清理 + 标准解析    │ ──成功──→ 返回完整 JSON 数据
  └──────────────────────────────────────────────┘
                     │ (JSONDecodeError)
                     ▼
  ┌──────────────────────────────────────────────┐
  │ 第二道防线 (Tier 2): 贪婪提取最外层闭合 JSON 块   │ ──成功──→ 返回有效 JSON 数据
  └──────────────────────────────────────────────┘
                     │ (JSONDecodeError / Match Missing)
                     ▼
  ┌──────────────────────────────────────────────┐
  │ 第三道防线 (Tier 3): Pattern Rescue 正则救援    │ ──成功──→ 挽救 90%+ 已生成有效对象
  └──────────────────────────────────────────────┘
                     │ (No Matches Found)
                     ▼
  ┌──────────────────────────────────────────────┐
  │ 第四道防线 (Tier 4): Safe Fallback & 容错填充   │ ──降级──→ 返回空对象/默认补零，保活流程
  └──────────────────────────────────────────────┘
```

### 第一道防线（Tier 1）：Markdown 格式清理与标准解析

LLM 经常会在 JSON 外层包裹 ````json ... ```` 代码块标记，或者附带前导空格与换行。第一步进行规范化剥离：

```python
cleaned = response.strip()
cleaned = re.sub(r'^```json\s*', '', cleaned)
cleaned = re.sub(r'\s*```$', '', cleaned)
cleaned = cleaned.strip()

# 尝试直接解析
try:
    return json.loads(cleaned)
except json.JSONDecodeError:
    pass
```

### 第二道防线（Tier 2）：贪婪匹配最外层闭合 JSON 块

如果 LLM 在 JSON 前后输出了一些解释性废话（例如 "Here is your JSON:"），导致标准 `json.loads` 失败，但内部的数组是完整的，我们通过正则表达式提取最外层的 `[...]` 或 `{...}` 块：

```python
match = re.search(r'\[[\s\S]*\]', cleaned)
if match:
    try:
        return json.loads(match.group())
    except json.JSONDecodeError:
        pass
```

### 第三道防线（Tier 3 Core Rescue）：基于模式匹配的 Pattern Rescue（正则救援）

当 Tier 1 和 Tier 2 均因末端截断报 `JSONDecodeError` 宣告失败时，程序会自动滑入 Tier 3。

这是防线的核心：因为我们提前定义好了提取的 Schema（即 `{"source": ..., "target": ..., "relation": ...}`），我们可以**绕过全局 JSON 语法规则，直接针对单个合法的子对象构造严格的正则表达式**。

在 `s5_3_collaboration_centrality.py` 中的具体实现如下：

```python
# ⭐ 处理截断的 JSON：提取所有完整的对象
print(f"[{self.SCORER_ID}] JSON 可能被截断，尝试提取完整对象...")

# 匹配所有完整的 {"source": ..., "target": ..., "relation": ...} 对象
pattern = r'\{\s*"source"\s*:\s*"([^"]+)"\s*,\s*"target"\s*:\s*"([^"]+)"\s*,\s*"relation"\s*:\s*"(support|deny)"\s*\}'
matches = re.findall(pattern, cleaned)

if matches:
    result = [{"source": m[0], "target": m[1], "relation": m[2]} for m in matches]
    print(f"[{self.SCORER_ID}] 从截断响应中提取到 {len(result)} 条关系")
    return result

print(f"[{self.SCORER_ID}] 未找到有效的关系数据")
print(f"[{self.SCORER_ID}] 响应前200字符: {cleaned[:200]}")
return []
```

#### 正则救援的技术细节解析：
1. **子模式完整性约束**：正则表达式 `\{\s*"source"..."\s*\}` 必须包含起始的 `{` 和结尾的 `}`。被截断在半截的尾部残缺对象（如 `{"source": "说话人 5", "target": "说话`）由于不匹配尾括号，会被自动忽略。
2. **捕获组精准提炼**：通过小括号 `([^"]+)` 和 `(support|deny)` 精准捕获字段内容，直接重构成标准的 Python 字典对象列表。
3. **零 overhead 容错**：在 Python 内存中执行一次 `re.findall` 耗时仅需不到 1 毫秒，瞬间完成了对已生成有效数据的挽救。

### 第四道防线（Tier 4）：Schema 容错与默认值补零

在 `s5_2_conflict_resolution.py` 这种涉及 Ridge 线性回归与概率矩阵计算的评分器中，解析出的 JSON 往往要转化为 Pandas DataFrame：

```python
def _parse_json(self, text: str) -> dict:
    """解析JSON字符串通用兜底"""
    cleaned = re.sub(r'```json\s*', '', text)
    cleaned = re.sub(r'```\s*', '', cleaned)
    cleaned = cleaned.strip()
    try:
        return json.loads(cleaned)
    except json.JSONDecodeError:
        match = re.search(r'\{.*\}', cleaned, re.S)
        if match:
            try:
                return json.loads(match.group())
            except:
                pass
        return {}
```

即使某次极端截断导致返回的是空字典 `{}`，下游的数据处理层也通过防御性设计保证计算链路不中断：

```python
# 构建矩阵并填充 NaN（LLM 返回不完整或截断时产生）
X = df.iloc[:, :len(topic)]
Y = df.iloc[:, len(topic):len(topic)+len(speakers)]

X = X.fillna(0)
Y = Y.fillna(0)

# 遇到有效变化样本不足时优雅退化，赋零向量
if y.nunique() <= 1:
    speaker_coefficients[s] = np.zeros(X.shape[1])
    continue
```

---

## 5. 生产环境运行实测效果对比

引入多级自愈解析与正则救援机制后，我们在 V5-Speed 系统中对 50 组复杂无领导小组长文转写（平均每组包含 8 名发言人、约 2.5 万字）进行了端到端评测：

| 评估维度 | 优化前（仅靠标准 json.loads + Retry） | 优化后（多级解析 + Pattern Rescue） |
| :--- | :--- | :--- |
| **流水线整体崩溃率** | 14.0%（因 JSONDecodeError 导致 Worker 终止） | **0.0%**（完全自愈） |
| **截断发生时的关系数据挽救率** | 0%（全盘丢弃，节点关系丢失） | **94.2%**（提取出截至截断点前所有有效边） |
| **平均 API 耗时与重试开销** | 每次截断触发重试，额外增加 15~40 秒 | **0 秒额外开销**（毫秒级正则抽离） |
| **有向图中心性计算有效性** | 频繁出现孤立节点或零度数 | 关系拓扑完整，图特征保持高稳定度 |

---

## 6. 总结

回顾解决 LLM 输出截断问题的工程过程，在复杂 AI Agent 系统开发中，我总结了以下三条防御性编程法则：

1. **Prompt 里的 Schema 是“期望”，而非“保证”**
   永远不要把 LLM 当成严格恪守协议的微服务。在系统边界处，必须针对 LLM 响应编写具备自愈能力的 Parser。
2. **追求“局部成功（Partial Success）”，拒绝全盘崩溃**
   在长流水线和多 Agent 协同中，得到 90% 的有效数据远比抛出 Exception 导致系统归零要好得多。只要单条数据具备独立语义（如数组中的项、图中的边），就应该在字符流粒度进行挽救。
3. **解析层与控制流解耦**
   JSON 解析失败应该被局限在数据提取函数内部解决，绝不能让 `JSONDecodeError` 沿着调用栈向上扩散，破坏主线程池调度或触发耗时的非必要重试。