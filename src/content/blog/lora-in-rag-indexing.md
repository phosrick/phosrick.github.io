---
title: 'LoRA 应用：在 RAG 里只建索引'
description: '复盘 LoRA 在这个项目里是什么、解决了什么'
publishDate: '2026-07-27 21:10:00'
tags:
  - LoRA
  - RAG
category: 圆桌讨论·AI能力评估
---
## 一、我是怎么知道项目用到 LoRA 的

接手这个项目时，没人告诉我它用了 LoRA。我是在代码里看到的。

 `src/finetune/` 目录，有两个训练脚本：`qwen_lora.py` 和 `deepseek_lora.py`。翻 `checkpoints/`，有 `qwen_lora/final_model` 这样的训练产物。翻 `src/pipeline/node_agents.py`，有一个 `LocalFinetunedExtractor` 类，`__init__` 里写着：

```python
def __init__(self, model_path="./checkpoints/qwen_lora/final_model",
             base_model="/home/ubuntu/.cache/modelscope/hub/Qwen/Qwen2.5-14B",
             model_type="qwen"):
```

基座 Qwen2.5-14B，LoRA adapter 挂在 `checkpoints/qwen_lora/final_model`。

再翻 CLI：

```text
--extractor {local,api,both}   Node extractor type  (default: local)
```

默认 `local`。一开始我想当然的认为，本地部署`Qwen2.5-14B`大模型后，所有 LLM 调用都用本地 LoRA 跑、就可以把 API 成本压到 0。

但完整跑下来后我发现并不是这么一回事，在 `src/pipeline/runner.py` ——Phase 2 的"判断候选人发言中是否提到某个参考节点"，直接使用 `OpenAI` client 调 `qwen-plus`，与 `--extractor` 无关：

```python
response = client.chat.completions.create(
    model="qwen-plus",   # 写死
    messages=[{"role": "user", "content": prompt}],
    ...
)
```

`--extractor` 只控制 Phase 1 的一个工位：把参考语料的 chunk 抽成结构化节点（信息节点 c / 决策节点 d / 效用节点 u）。而下游的节点命中判断、边抽取、边验证、质检、分类——全是 API。

**LoRA 在这条流水线里只服务一个工位：把参考语料编译成节点索引。**

## 二、LoRA 是什么

**基座 + adapter 的组合。** `LocalFinetunedExtractor._load_model`（node_agents.py:312）做了两件事：先加载基座 `Qwen2.5-14B`，再用 `PeftModel.from_pretrained` 把 LoRA adapter 挂上去：

```python
base_model = AutoModelForCausalLM.from_pretrained(
    self.base_model, torch_dtype=torch.bfloat16,
    device_map="cuda:0", trust_remote_code=True)
self.model = PeftModel.from_pretrained(base_model, self.model_path)
self.model.eval()
```

基座 14B，4bit NF4 量化（见 `qwen_lora.py` 的 `BitsAndBytesConfig`），config 标注 ~40GB 显存。`device_map="cuda:0"` 单卡部署——评估场景下模型只服务一个工位，不需要跨卡 shard。

训练脚本 `qwen_lora.py` 的 `FineTuneConfig` 里，LoRA 的配置是 `r=16, alpha=32, dropout=0.05`，target modules 覆盖了 attention 的 `q_proj/k_proj/v_proj/o_proj` 和 MLP 的 `gate_proj/up_proj/down_proj`。意思是：冻全模型（基座 14B 权重整体锁死、训练时一个数值都不动），只在那些投影层旁边插一个秩 16 的小矩阵，训练时只更新小矩阵。这就是"Low-Rank Adaptation"——用很少的参数（相对全量微调）让模型学会一个特定任务。

**这个项目里 LoRA 学的特定任务是：从一段参考语料里抽出信息/决策/效用三类节点。** 训练数据是 JSONL，每条样本是 `{instruction, input, output}`——instruction 是固定的节点定义说明，input 是一段文本，output 是节点列表。训练 10 个 epoch，`do_sample=False` 保证确定性。

## 三、LoRA 解决了什么

**LoRA 在这个项目里到底解决了什么？**

参考语料在**设计意图**上只处理一次——pre 阶段建图、post 阶段复用。LoRA 的训练 + 部署成本可以摊到整批候选人头上：

| 维度       | 本地 LoRA（Phase 1 抽节点）  | 云端 API（其余所有 LLM 任务）    |
| ---------- | ---------------------------- | -------------------------------- |
| 调用频次   | 参考语料全场一次             | 每位候选人都跑一遍               |
| 成本形态   | 一次性训练 + GPU 持有        | 按 token 持续计费                |
| 输入敏感度 | 参考语料可能含业务机密       | 候选人发言也敏感，但已脱敏       |
| 可重放性   | `do_sample=False` 完全确定 | `temperature=0` 仍受服务端影响 |

但既然本地 Qwen2.5-14B 都下载跑起来了，直接用 base Qwen2.5-14B 加 prompt 就能做边抽取、验证、打分、写报告——和调 qwen-plus 一样是 prompting，不是 fine-tuning，下游为啥不顺便用 Qwen2.5-14B，非得调外部 API 呢？我自己的分析是：

**1. 能力大小。** base 14B 在复杂因果推理、细致打分上弱于 qwen-plus/max，而下游那些开放式判断正好吃这部分能力。

**2. 上下文窗口。** 本地 4bit 14B 在单卡上的实用上下文窗口本就小于 qwen-plus/max，这是事实。

**3. 并发 / 吞吐。** 本地模型单卡、串行前向、一次加载。下游是每位候选人 × 多遍调用——API 天然并发、自动扩缩，白送的并行度；全压到本地单卡 14B，整批吞吐卡死。下游是每位候选人 × 多遍的高频调用——如果这些全走本地 14B，就要为每次候选调用付 GPU 时间，还被串行瓶颈拖死；API 按 token 计费在这种体量下反而更合适。所以当前设计是把"贵但低频、需合规"放本地，"高频、需强模型、已脱敏"放 API，各自放到更划算的地方。

**4. 合规边界不同。** Phase 1 用本地 LoRA 的主因是参考语料可能含未脱敏业务文本、不能出域。下游处理的是已脱敏的候选人发言。

## 四、小结

LoRA 和 API 不是二选一，是两种不同岗位的分工——LoRA 干 indexing，API 干 generation。

什么时候该用 LoRA？LoRA 有固定成本（训练 + GPU 持有 + 维护），**项目 LoRA 只在 Phase 1 抽参考语料节点——全场只跑一次——从纯成本看其实不划算。** 选 LoRA 的真实驱动力是合规（参考语料不出域）和确定性（评估系统要求可复现），不是省钱。

此外，LoRA 学的是特定任务的输出模式，任务变了——节点定义改了、输出 schema 变了、instruction 换了——就要重新训练。API 改 prompt 是字符串级修改，5 分钟搞定；**团队有 MLOps 治理能力、调用频次够覆盖固定成本、任务短期内不会变——这三个都符合就可以考虑 LoRA**。
