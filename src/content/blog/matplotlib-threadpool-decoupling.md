---
title: '当 Matplotlib 遇上 ThreadPoolExecutor：AI Agent 系统中的线程锁与渲染解耦'
description: '并发生成网络拓扑与冲突热力图时发生的 Segfault 与画面交织错乱问题。详解 Matplotlib 非线程安全的底层根源、MATPLOTLIB_LOCK 全局临界区保护与“计算-渲染解耦”的架构实战。'
publishDate: '2026-07-30 14:08:00'
tags:
  - AI Agent
  - Python
  - 多线程
  - 并发
  - 系统架构
category: 圆桌讨论·AI能力评估
---

## 1. 从 Agent 评估到可视化渲染

在 V5 的圆桌讨论能力评估系统里，候选人的表现需要经过 16 个能力维度的综合打分。为了交付一份高可读性的全方位评估报告，系统不仅需要输出分数和 LLM 诊断评语，还需要动态生成可视化图表：

- **协作中心性（5_3）**：使用 NetworkX 绘制候选人之间的互动有向图，用绿边/红边标注 `support` 和 `deny` 关系；
- **冲突消解（5_2）**：使用 Seaborn 绘制话题贡献热力图，并生成各成员在发言轮次中的冲突变动折线图；
- **能力雷达图**：绘制包含 16 维能力指标与团队均分对比的雷达图。

这个评分流程的主调度引擎运行在 `scoring_runner_timed.py` 中。

为了加快评估速度，我使用 Python 的 `ThreadPoolExecutor` 开启多线程并发拉起 16 个 Scorer（评分器）：

```python
# 一开始的直觉逻辑：在线程池里并行运行每个 Scorer
with ThreadPoolExecutor(max_workers=max_workers) as executor:
    futures = {
        executor.submit(scorer.run): sid
        for sid in scorer_ids
    }
```

每个 Scorer 内部除了计算数据、调用 LLM 生成摘要，还会调用 Matplotlib 绘图并存盘：

```python
# 某个 Scorer 内部
plt.figure(figsize=(12, 8))
nx.draw_networkx_nodes(...)
plt.savefig(plot_path)
plt.close()
```

逻辑看起来非常自然：16 个维度的 Scorer 彼此独立，写入的文件路径也完全隔离开。

然而，一旦把 `max_workers` 拉大到 8 或 16，系统开始频繁出现非常诡异的现象。

---

## 2. 现象：神秘崩溃与画面“串味”

多线程并发绘图带来的不是预期中的线性提速，而是两种灾难性的失败模式：

### 现象 A：直接 Segment Fault 崩溃
程序跑到一半，控制台没有抛出任何 Python 层的异常，而是直接输出了一句底层 C 库报错后惨烈退出：

```text
Fatal Python error: Segmentation fault
Current thread 0x00007f9b8c123700 (most recent call first):
  File ".../matplotlib/backends/_backend_agg.cpython-312-x86_64-linux-gnu.so", line ...
```

或者触发 `free(): invalid pointer`，整个 Python 解释器崩溃崩溃。

### 现象 B：逻辑正常，但画面“串味”
有时候程序侥幸没有崩溃，输出了所有图片。但在校验 PDF 报告时我发现：Scorer 5_3 生成的社会网络图上面，竟然印着 Scorer 5_2 的标题；或者折线图上叠加了雷达图的网格线，甚至字体和坐标轴全部错乱地粘在一起。

在代码层，每个线程都有独立的变量作用域，写入的是不同的 `.png` 文件。**为什么数据没有冲突，渲染结果却“串味”了？**

---

## 3. 根源剖析：Matplotlib 为什么不是线程安全的

深入 Matplotlib 的源码和架构设计后，原因变得十分清晰：**Matplotlib 的 `pyplot` 状态机接口和底层 C 渲染后端，根本不是为了多线程并发设计的。**

### 1) Stateful API 与全局状态指针
Matplotlib 提供了两套 API：面向对象的 API（如 `fig, ax = plt.subplots()`）和状态机 API（如 `plt.figure()`, `plt.plot()`, `plt.title()`）。

即便在代码里尽量使用面向对象方式，但在调用 `plt.figure()` 或 `plt.savefig()` 时，Matplotlib 内部都会访问一个全局帮助类 `matplotlib._pylab_helpers.Gcf`。它维护着一个全局的“当前活跃 Figure”指针：

```python
# Matplotlib 底层伪代码逻辑
class Gcf:
    active_figure = None  # 全局唯一指针！

def figure():
    fig = Figure(...)
    Gcf.active_figure = fig  # 多线程下产生严重的竞争条件！
    return fig
```

当线程 A 执行 `plt.figure()` 创建了 Figure 1，还未来得及绘制，线程 B 刚好也执行了 `plt.figure()` 创建了 Figure 2，此时全局指针 `Gcf.active_figure` 被线程 B 改写为 Figure 2。接下来线程 A 执行 `plt.title("线程A的标题")` 时，这个标题就会被画到线程 B 的 Figure 2 上！这就是画面“串味”的根本原因。

### 2) C 底层后端与字体解析器无锁
Matplotlib 的图形绘制依赖 C/C++ 编写的底层 Backend（如 Agg、Cairo）以及 FreeType 字体渲染库。这些底层 C 库在内存分配、字体缓存 (`FontManager`) 管理时都没有加 C 级别的 Mutex 锁。

当 16 个线程同时调用 FreeType 去解析 `WenQuanYi Micro Hei` 字体并向一块 C 缓冲区写入像素时，会导致内存非法踩踏（Buffer Overflow / Memory Corruption），直接引发操作系统级别的 Segmentation Fault。

---

## 4. 架构演进：从“暴力加锁”到“计算-渲染解耦”

知道了根源，解决方案有两种演进路线。

### 方案 1：暴力全局临界区锁（MATPLOTLIB_LOCK）

最直接的做法是在全局创建一个互斥锁，任何线程在调用 Matplotlib 绘图时都必须先获取这把锁。

在 `src/scoring/base.py` 中，我定义了全局锁：

```python
import threading

# matplotlib 全局锁：保护任何渲染与存盘操作
MATPLOTLIB_LOCK = threading.Lock()
```

在具体评分器（如 `s5_3_collaboration_centrality.py`）绘图时使用 `with` 语法保护：

```python
def save_network(self, filename, title="Collaboration Network"):
    if not HAS_MATPLOTLIB:
        return
    
    # ⭐ 必须放入临界区，防止多线程竞争 Matplotlib 全局 Gcf 指针与 Agg C 缓冲区
    with MATPLOTLIB_LOCK:
        plt.figure(figsize=(12, 8))
        pos = nx.spring_layout(self.G, seed=42)
        ...
        plt.title(title)
        plt.savefig(filename, bbox_inches='tight', dpi=300)
        plt.close()
```

**方案 1 的评估**：
- **优点**：改造成本低，彻底解决了崩溃与画面串味问题。
- **缺点**：锁粒度太粗。当某个 Scorer 正在绘制包含上百节点的复杂图表时，其他 15 个线程如果执行到绘图逻辑就会被迫阻塞挂起。CPU 绘图和 IO 存盘拖慢了线程池中高并发 LLM 调用的吞吐效率。

---

### 方案 2：计算与渲染两阶段解耦 (Compute-Render Decoupled Architecture)

为了追求极致的执行速度，我将整个 Scoring Pipeline 拆分为完全解耦的两个阶段：

```text
ThreadPoolExecutor (16 Workers)
┌──────────────────────────────────────────────────────────┐
│ 阶段 1：纯计算与 LLM 诊断 (高并发, generate_plot=False)      │
│   ├── Scorer 1_1: 逻辑计算 + LLM Summary → 存内存/文件      │
│   ├── Scorer 5_3: 图拓扑计算 + 交互分析 → 写入 _plot_cache   │
│   └── ...                                                │
└──────────────────────────────────────────────────────────┘
                            │
                            ▼
Main Thread (Sequential Loop)
┌──────────────────────────────────────────────────────────┐
│ 阶段 2：安全绘图渲染 (单线程/加锁, generate_plots_only)       │
│   ├── 读取 Scorer 5_3 的 _plot_cache 绘图                  │
│   ├── 读取 Scorer 5_2 的 _plot_cache 绘图                  │
│   └── ...                                                │
└──────────────────────────────────────────────────────────┘
```

#### 1) 内存缓存设计（In-Memory Plot Cache）
在各 Scorer（如 `CollaborationCentralityScorer`）内部，阶段 1 并行计算时只把关系图序列化为纯 Python 字典存入 `_plot_cache`，绝不触发任何 Matplotlib 调用：

```python
# 阶段 1：纯数据抽取与计算，不画图
cn = CollaborationNetwork()
for item in arcs:
    cn.add_interaction(item['source'], item['target'], item['relation'])

# 将计算完的图结构存入内存 Cache
group_id = filename[:-4]
self._plot_cache[group_id] = cn.to_dict()
```

#### 2) 两阶段调度器落地
在 `scoring_runner_timed.py` 的 `run_all_scorers` 函数中，显式将执行流程一分为二：

```python
# ========== 第一阶段：并行计算 + LLM 摘要 ==========
print("\n📊 阶段1: 并行计算分数和生成摘要...")
with ThreadPoolExecutor(max_workers=max_workers) as executor:
    futures = {
        executor.submit(
            run_single_scorer,
            sid,
            dm,
            llm,
            generate_summary,
            False,  # ⭐ 强制关闭第一阶段的绘图！保持纯粹的线程池高并发
        ): sid
        for sid in scorer_ids
    }
    for future in as_completed(futures):
        all_results[futures[future]] = future.result()

# ========== 第二阶段：串行/安全生成图片 ==========
if generate_plot:
    print("\n🎨 阶段2: 串行生成图片...")
    plot_scorer_ids = [
        sid for sid in scorer_ids 
        if SCORERS.get(sid) and getattr(SCORERS[sid], 'HAS_PLOT', True)
    ]
    
    for scorer_id in plot_scorer_ids:
        scorer = SCORERS[scorer_id](dm, llm, generate_plot=True)
        # 读取第一阶段缓存在内存中的数据，安全作画
        scorer.generate_plots_only(all_results.get(scorer_id, {}))
```

---

## 5. 实战对比与性能收益

通过引入“计算与渲染解耦”的架构重构，V5 系统在稳定性与性能上获得了显著提升：

| 指标 | 原始多线程直出 | 暴力全局锁 (方案1) | 两阶段计算-渲染解耦 (方案2) |
| :--- | :--- | :--- | :--- |
| **崩溃概率 (Segfault)** | ~35% 高频崩溃 | **0%** | **0%** |
| **图片错乱率 (串味)** | ~60% 随机错乱 | **0%** | **0%** |
| **16 Scorer 耗时** | 无法稳定跑完 | 185 秒 (受锁阻塞) | **112 秒** (吞吐提升 39%) |
| **架构扩展性** | 极差 (无法增加并发) | 一般 (线程受制于绘图) | **优秀** (计算与渲染职责分离) |

---

## 6. 总结与 Agent 系统设计思考

在构建复杂的 AI Agent 系统时，我们往往很容易关注 LLM Prompt 的设计或 Agent 的协同逻辑，却忽略了底层工程基础设施的“副作用与异构性”。

1. **异构任务不要硬塞进同一个并发池**：
   Agent 流水线中的任务可以分为三类：
   - **网络 IO 密集型**（如调用 OpenAI/Qwen API）；
   - **CPU 运算密集型**（如 NetworkX 图算法、Ridge 回归矩阵运算）；
   - **有状态副作用渲染型**（如 Matplotlib/Seaborn 作图、PDF 渲染）。
   把带 C 依赖和全局状态的渲染任务与无状态的网络 IO 任务混在同一个 `ThreadPoolExecutor` 里，是引发系统不稳定的根源。

2. **遵从 Compute-Render Decoupling 原则**：
   在系统设计上，将“根据 Prompt 抽取与计算中间状态”和“将中间状态可视化为图形/PDF”明确划分为两个阶段。计算阶段追求高并发与零副作用，渲染阶段追求线程安全与幂等性。

这种将计算逻辑与渲染表现分离的思考，不仅适用于 Matplotlib 绘图，也是构建生产级、高并发 AI Agent 系统时必备的架构工程习惯。
