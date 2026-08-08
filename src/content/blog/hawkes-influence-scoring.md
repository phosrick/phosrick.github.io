---
title: '用 Hawkes 算圆桌影响力'
description: '圆桌讨论的影响力不是谁说得多，而是谁的发言激发了后续讨论。用 Hawkes 自激发过程建模词级时间序列，从似然函数里反推每个说话人的自影响力和他影响力参数。'
publishDate: '2026-08-08 15:00'
tags:
  - Hawkes过程
category: 圆桌讨论·AI能力评估
---
## 影响力到底怎么量化

一场 5 - 12 个人的无领导小组讨论，持续 30 分钟，转写出来约 12000 - 20000 字。最终系统要给每位参与者16 个维度打分，其中第 4 个维度是"影响力"。

影响力是什么？它衡量的是一个人在圆桌讨论里 **能不能通过自己的发言改变讨论的走向** 。

有人说了很多，但都是自说自话，没人接话 → 在场，但没影响

有人只说了两句，但一句就把讨论方向拐到了新话题上 → 影响很大

因此，影响力 ≠ 谁说的多。它要回答的问题是：发言人的讨论有没有被接住、又没有让讨论换方向。

我需要的是一种可复算的中间表示：给定同样的转写文本，跑出来的影响力分数是确定的，而且分数背后有明确的数学定义——不是"模型觉得你有影响力"，而是"你的发言在统计上显著提高了他人后续发言的强度"。

这就是我引入 Hawkes 过程的原因。

## 什么是 Hawkes 过程

Hawkes 过程是一种自激发点过程（self-exciting point process），最初由 Alan Hawkes 在 1971 年提出，用来建模地震余震——一次主震之后，短期内余震概率会上升，然后随时间指数衰减。

它的核心假设很简洁：每个事件都会在之后一段时间内提高同类事件发生的概率，这个影响以指数速率衰减。

把这套逻辑搬到圆桌讨论里：

- 事件 = 某个说话人在某个时刻使用了某个词
- 自激发 = 说话人 A 用了"成本"这个词之后，A 在后续发言中再次使用"成本"（或语义相近的词）的概率上升
- 互激发 = 说话人 A 用了"成本"之后，说话人 B 在后续发言中使用"成本"（或语义相近的词）的概率上升

自激发衡量的是思路延续能力——你能不能在自己发言的后续里把一个话题展开。互激发衡量的是话题传播能力——你说出去的话有没有被别人接住并继续讨论。

这两个参数（α_self 和 α_other）就是影响力评分的核心输出。

## 第一个问题：什么是一个"事件"

在我们的场景里，一个事件是一个三元组：`(词, 说话人, 时间戳)`。Hawkes 过程要求事件是离散的、可重复类型的时间点——它的激发核算的是：过去某个类型的事件，有没有提高未来同类型事件的出现强度。

如果以每句话为一个事件，一句话往往同时包含多个概念，比如："我们应该控制获客成本，同时提升转化率"——这句话里既有"成本"又有"转化率"。如果把整句当一个事件，就无法分清是"成本"被接住还是"转化率"被接住。而影响力恰恰是要定位"哪个话题在传播"。

以每个字为一个事件又太细——"的""了""是"这类停用词不携带语义，会把真正的激发信号淹没在噪声里。

只有词粒度才能把话题拆开单独追踪。

因此，我使用了 jieba 对转写稿分词，保留名词、动词、形容词等实义词，过滤停用词：

```python
VALID_POS = {'n', 'v', 'a', 'vn', 'ns', 'nz', 'nt'}

def _extract_keywords(self, text: str, stopwords: set) -> List[str]:
    words = pseg.cut(text)
    return [
        word for word, flag in words
        if word not in stopwords
        and re.match(r'[\u4e00-\u9fa5]+', word)
        and flag in self.VALID_POS
    ]
```

每个事件被编码为三元组 `(word, speaker_id, timestamp)`。但这里有一个绕不开的设计决策：**时间戳从哪来**。

转写文本里有说话人的时间标记（`说话人2 03:15 ...`），但实际运行中我发现这些时间戳质量参差不齐：有的转写工具按固定间隔插入时间标记，有的在停顿时跳秒，有的干脆丢失。直接用这些时间戳做时间序列，噪声会被 Hawkes 的指数衰减核放大。

最终我用了一个替代方案：**用字符位置作为时间代理**。整场讨论的转写文本是一个线性流，第 N 个字符出现在第 M 个字符之前。把字符位置除以一个缩放常数（SCALE=10000），就得到了一个归一化的"伪时间"：

```python
def _gen_timeseq(self, corpus, word2group, word2num, all_word_num_list, SCALE):
    timeseq = {}
    max_time = 0
    for line in corpus:
        word, speaker, char_pos = line.split(',')
        speaker_num = int(speaker) - 2
        timestamp = (int(char_pos) + 5) / SCALE
        if timestamp >= max_time:
            max_time = timestamp
        # ...
        timeseq[(speaker_num, word_num)].append(timestamp)
    return timeseq, (max_time + 5)
```

这个选择的代价是丢失了真实时间间隔信息——两个人在 3 秒内先后说同一个词，和间隔 5 分钟后说同一个词，在字符时间轴上可能差几十个字符。

## 第二个问题：不是所有词都互相激发

如果说话人 A 说了"成本"，说话人 B 5 秒后说了"效率"，这算不算激发？

严格按词面匹配来说，不算。但在语义上，"成本"和"效率"高度相关，A 对"成本"的讨论很可能激发了对"效率"的讨论。

我需要一种方式来衡量词与词之间的语义距离，让激发核不只关注同词匹配，还能覆盖近义词。

做法分三步：

**第一步：用预训练词向量计算余弦相似度。** 加载中文 SGNS 词向量（来自中文维基百科训练），为每个词生成 embedding，计算两两余弦相似度作为距离矩阵：

```python
def _create_distance_matrix_optimized(self, num2vec):
    nodes = sorted(num2vec.keys())
    vectors = np.array([num2vec[n] for n in nodes])
    sim_matrix = cosine_similarity(vectors)
    distance_matrix = {}
    for i_idx, i_node in enumerate(nodes):
        for j_idx, j_node in enumerate(nodes):
            if i_node < j_node:
                distance_matrix[(i_node, j_node)] = sim_matrix[i_idx, j_node]
    return distance_matrix
```

**第二步：用 PMI 构建词共现网络。** 两个词在同一个滑动窗口内同时出现，说明它们在当前讨论中有上下文关联。PMI（Pointwise Mutual Information）衡量的是共现频率是否显著高于独立出现的期望：

```python
pij = (nij + equiv_concur) / (big_n + equiv_concur)
pi = (ni + equiv_concur) / (big_n + equiv_concur)
pj = (nj + equiv_concur) / (big_n + equiv_concur)
if pi * pj * pij > 0:
    pmi_val = math.log(pij**param_k / pi / pj, 2)
    if pmi_val >= mi_thres:
        edge_list_num.append([i, j])
```

这里有一个混合策略：PMI 本身只看共现，但 `equiv_concur` 用词向量余弦相似度做了加权——即使两个词在当前讨论中从未共现，如果它们的 embedding 足够接近，也会获得一个等效共现计数。这让网络不会因为单场讨论的语料稀疏而丢失语义关联。

**第三步：Louvain 社区检测。** 有了词共现网络后，用 Louvain 算法把词分成社区。同一个社区里的词属于同一个话题簇。这一步的目的是为激发核提供一个额外的"同话题加成"：如果两个词在同一个社区里，即使语义距离稍远，也认为它们之间有激发的可能性。

```python
# Louvain 社区检测
node2pst_node, new_modularity = self._outside_Louvain(
    new_edge_list_num, node_list, thres=self._louvain_early_stop
)
```

到这一步，每个词都有了三个属性：编号、所属说话人的使用时间序列、所属社区。这些就是 Hawkes 过程的全部输入。

## 第三个问题：激发核怎么设计

Hawkes 过程的核心是激发核（excitation kernel），它定义了"一个事件对后续事件的影响有多强"。在我的实现里，激发核被参数化为 α 矩阵——一个 N×N 矩阵，N 是 (speaker, word) 对的数量，矩阵元素 α[i,j] 表示 sender j 对 receiver i 的激发强度。

α 的取值取决于四个因素：

| 因素           | 参数                | 含义                        |
| -------------- | ------------------- | --------------------------- |
| 是否同一说话人 | α_sbase / α_obase | 自激发 vs 互激发            |
| 词距离阈值 1   | D_THRES1=1          | 近义词（距离≤1）：完整激发 |
| 词距离阈值 2   | D_THRES2=4          | 相关词（距离2-4）：衰减激发 |
| 同社区加成     | α_clust=0.1        | 同话题簇的额外激发          |

核心逻辑在 Numba JIT 加速的矩阵预计算函数里：

```python
@jit(nopython=True, parallel=True, fastmath=True, cache=True)
def _calculate_alpha_matrix_numba(receivers_p, receivers_n, distance_flat,
                                   n_receivers, param_matrix, word2group_arr,
                                   D_THRES1, D_THRES2):
    alpha_matrix = np.zeros((n_receivers, n_receivers))
    for i in prange(n_receivers):
        p1, n1 = receivers_p[i], receivers_n[i]
        alpha_sbase = param_matrix[p1, 0]
        alpha_obase = param_matrix[p1, 1]
        alpha_clust = param_matrix[p1, 2]
        diver = param_matrix[p1, 3]
        for j in range(n_receivers):
            p2, n2 = receivers_p[j], receivers_n[j]
            # 词距离
            if n1 == n2:
                distance = 0
            else:
                distance = distance_flat.get(key, 10000)
            # 激发强度
            retval = 0.0
            if distance <= D_THRES1:
                if p1 == p2:
                    retval += alpha_sbase
                else:
                    retval += alpha_obase
            elif D_THRES1 < distance <= D_THRES2:
                if p1 == p2:
                    retval += alpha_sbase * diver
                else:
                    retval += alpha_obase * diver
            # 同社区加成
            if word2group_arr[n1] == word2group_arr[n2] and word2group_arr[n1] >= 0:
                retval += alpha_clust
            alpha_matrix[i, j] = retval
    return alpha_matrix
```

这里有一个关键设计：**只有 α_sbase 和 α_obase 是需要学习的参数**，α_clust（社区加成）和 diver（远距离衰减因子）是固定值。

```python
FIXED_PARAMS = [0.1, 0.01]  # alpha_clust, diver
```

这个选择是一种正则化：如果四个参数全部学习，每个说话人需要优化 4 个参数，8 人讨论就是 32 维优化空间，数据量根本不够。固定其中两个，把优化维度压到 2×说话人数，让似然函数更容易收敛。

另一个值得注意的细节是 diver=0.01：距离在 2-4 之间的相关词，激发强度只有近义词的 1%。这意味着模型实际上几乎只关注近义词激发，远距离的语义关联靠社区加成（0.1）来补充。这个设计有点保守，但它避免了"几乎所有词都互相激发"的过拟合。

## 第四个问题：似然函数怎么写

Hawkes 过程的对数似然函数分为两部分：

$$
\log L = \sum_{k=1}^{K} \log\left(\mu + \sum_{j} \alpha_{ij} R_j(t_k)\right) - \sum_{j} \frac{\alpha_{ij}}{\beta} \int_0^T g(t) \, dt
$$

第一部分是事件发生时的瞬时强度对数和，第二部分是整个观测窗口内的积分项（保证强度不会无限增长）。

这里最关键的计算是 $R_j(t_k)$——sender j 在时刻 $t_k$ 之前累积的激发量。它有一个高效的递归形式：

**自激发（sender = receiver）**：

```python
@jit(nopython=True, fastmath=True, cache=True)
def _calculate_R_self_numba(receiver_times, beta):
    K = len(receiver_times)
    R_values = np.zeros(K)
    for k in range(K):
        if k == 0:
            R_values[k] = 0.0
        else:
            tkmi = receiver_times[k-1]
            tki = receiver_times[k]
            R_values[k] = (1.0 + R_values[k-1]) * np.exp(-beta * (tki - tkmi))
    return R_values
```

递推公式是 $R_k = (1 + R_{k-1}) \cdot e^{-\beta(t_k - t_{k-1})}$。每一步把上一步的累积激发量衰减一次，再加上当前事件自身的新激发（+1）。

**互激发（sender ≠ receiver）**：

```python
@jit(nopython=True, fastmath=True, cache=True)
def _calculate_R_cross_numba(receiver_times, sender_times, beta):
    K = len(receiver_times)
    R_values = np.zeros(K)
    for k in range(K):
        tkmi = receiver_times[k-1] if k > 0 else 0.0
        tki = receiver_times[k]
        # 递归部分：上一步的衰减
        prev_R = R_values[k-1] * np.exp(-beta * (tki - tkmi)) if k > 0 else 0.0
        # 累加部分：在 (t_{k-1}, t_k] 区间内的 sender 事件
        term2 = 0.0
        for st in sender_times:
            if tkmi <= st < tki:
                term2 += np.exp(-beta * (tki - st))
        R_values[k] = prev_R + term2
    return R_values
```

互激发的递推多了一个累加项：在 $[t_{k-1}, t_k)$ 区间内，sender 的每个事件都会贡献一个衰减后的激发量。

β 是衰减速率，当前固定为 92。这个值控制激发影响的持续时间——β 越大，影响衰减越快，一次发言只在很短时间内有效；β 越小，影响持续越久。92 这个值是通过实验选定的：在字符时间轴上，SCALE=10000 意味着整场讨论的伪时间范围大约是 0 到 2-3，β=92 对应的半衰期大约是 0.0075 个伪时间单位，约等于 75 个字符的间距。也就是说，一次发言的影响大约在后续 75 个字符内显著存在，之后基本衰减殆尽。

这个数字在直觉上合理吗？一句中文发言大约 20-50 个字符，75 个字符大约对应 1-3 句话。也就是说，如果 A 说了一个词，B 在 1-3 句话内接住了，算激发有效；如果 B 隔了 10 句话才说，激发基本消失。这个尺度对无领导小组讨论来说是一个合理的近似。

完整的似然计算把积分项和对数项合在一起：

```python
def _calculate_genre_likelihood_part_jit(self, receiver_idx, receiver,
        active_receivers, timeseq_array, max_time, beta, alpha_matrix,
        node2backrate):
    receiver_times = timeseq_array[receiver]
    K = len(receiver_times)
    genre_like = 0.0

    # 第一部分：积分项
    for sender_idx, sender in enumerate(active_receivers):
        alpha = alpha_matrix[receiver_idx, sender_idx]
        if alpha > 0:
            integral_sum = _calculate_integral_numba(sender_times, max_time, beta)
            genre_like -= (alpha / beta) * integral_sum

    # 第二部分：log-sum 项
    log_sum_terms = np.zeros(K)
    for sender_idx, sender in enumerate(active_receivers):
        alpha = alpha_matrix[receiver_idx, sender_idx]
        if alpha <= 0:
            continue
        is_self = (sender == receiver)
        if is_self:
            R_values = _calculate_R_self_numba(receiver_times, beta)
        else:
            R_values = _calculate_R_cross_numba(receiver_times, sender_times, beta)
        log_sum_terms += alpha * R_values

    backrate = node2backrate.get(receiver[1], 1e-9)
    log_sum_terms += backrate
    genre_like += np.sum(np.log(log_sum_terms))
    return genre_like
```

`backrate` 是背景强度 μ，代表不受任何激发影响时事件自发发生的概率。当前实现里用词频占比来估计：

```python
totalfreq = sum(word2freq.get(word, 0) for word in word2group.keys())
num2freqprop = {word2num[word]: word2freq.get(word, 0) / totalfreq
                for word in word2group.keys() if word in word2num}
node2backrate = {node: num2freqprop[node] * BACKRATE_SUM for node in num2freqprop.keys()}
```

高频词的背景强度更高，低频词的背景强度更低。BACKRATE_SUM=10 是一个缩放常数，确保背景强度和激发强度在同一数量级。

## 第五个问题：怎么从似然函数到分数

有了似然函数，下一步是找到使似然最大的 α 参数。这是一个非线性优化问题。

每个说话人有 2 个待估参数：α_self 和 α_other。8 人讨论就是 16 维优化。初始值设为 `[10, 2.5] * 8`——自激发初始强度是互激发的 4 倍，对应"自己延续自己的话题比影响别人更容易"的先验。

```python
initial_param = [10, 2.5] * SIZE  # SIZE = 说话人数

res = minimize(
    self._cal_single_likelihood_parallel,
    initial_param,
    args=args_for_minimize,
    method='Powell',
    options={
        'disp': True,
        'ftol': 1,
        'maxiter': self._minimize_maxiter,  # 30
        'maxfev': self._minimize_maxfev      # 300
    }
)
param = res.x
```

这里有几个有意识的折中：

**为什么用 Powell 而不是 L-BFGS-B？** Powell 方法不需要梯度，而似然函数里有 log 和递归 R，手写梯度容易出错。代价是收敛速度慢，且在高维空间里可能陷入局部最优。

**为什么 maxiter 只有 30？** 完整收敛通常需要 50-100 次迭代，但每次迭代都要计算一次完整似然，8 人讨论一轮就要几秒。30 次迭代在精度和耗时之间做了平衡。优化前的完整运行约 572 秒，压缩到 30 次迭代后约 100-150 秒。

优化完成后，每个说话人拿到两个参数：

```python
score1_list = [param[i*2] for i in range(SIZE)]      # α_self
score2_list = [param[i*2+1] for i in range(SIZE)]    # α_other
```

这两个参数直接做了 min-max 归一化，然后相加得到最终分数：

```python
if max_score1 != min_score1:
    norm_score1 = (score1 - min_score1) / (max_score1 - min_score1)
else:
    norm_score1 = 0.5

final_score = norm_score1 + norm_score2
```

分数范围是 [0, 2]，其中自影响力和他影响力各占 [0, 1]。

## 572 秒到 150 秒：性能优化怎么做的

这个评分器最初跑一个 8 人讨论组需要约 572 秒，其中 90% 的时间花在似然函数的计算上。瓶颈有两处：R 值的递归计算和 α 矩阵的预计算。

**R 值优化：Numba JIT。** `_calculate_R_self_numba` 和 `_calculate_R_cross_numba` 是纯数值循环，没有 Python 对象操作，非常适合 Numba 的 nopython 模式。加上 `fastmath=True` 允许浮点重排，`cache=True` 避免重复编译：

```python
@jit(nopython=True, fastmath=True, cache=True)
def _calculate_R_self_numba(receiver_times, beta):
    # ... 纯数值循环
```

**α 矩阵优化：Numba 并行。** α 矩阵是 N×N 的双重循环，每个元素的计算互相独立，可以用 `prange` 做并行：

```python
@jit(nopython=True, parallel=True, fastmath=True, cache=True)
def _calculate_alpha_matrix_numba(...):
    for i in prange(n_receivers):  # 并行外层循环
        for j in range(n_receivers):
            # ...
```

**似然计算：线程池并行。** 每个 receiver 的似然计算互相独立，用 ThreadPoolExecutor 并行：

```python
if self._use_parallel and len(active_receivers) > 4:
    with ThreadPoolExecutor(max_workers=self._parallel_workers) as executor:
        futures = [
            executor.submit(
                self._calculate_genre_likelihood_part_jit,
                i, r, active_receivers, timeseq_array,
                max_time, beta, alpha_matrix, node2backrate
            ) for i, r in enumerate(active_receivers)
        ]
```

**Louvain 提前终止。** 社区检测的模块度增量低于 0.01 时直接停止，避免在收敛末尾浪费迭代：

```python
if new_modularity - pst_modularity <= self._louvain_early_stop:
    print(f"Louvain early stop at iteration {iteration}")
    break
```

**跳过可达性计算。** 原始实现包含一个 O(N³) 的图可达性分析，用于过滤孤立节点。实测发现它对最终分数几乎无影响，但耗时可观，于是加了一个开关直接跳过：

```python
self._skip_accessibility = True

def _accessibility(self, ...):
    if self._skip_accessibility:
        node2accnum = {node: 10 for node in node2deg.keys()}
        return {}, node2accnum
```

这些优化合在一起，把单组讨论的评分时间从约 572 秒压到约 100-150 秒。

## 写在最后

主导这个项目之前，我写了七年 Java。我的舒适区是接口、并发、GC 调优，是把需求变成能跑、能扛量的服务。点过程、似然函数、指数衰减核——这些词对我来说，相当陌生。

这篇文章里的内容，是我为了把这个评分器跑通、调稳，一点一点啃出来的最小必要知识。Hawkes 过程的数学内核——似然函数为什么长那样、Powell 在 16 维非凸空间里到底停在了哪、β 和 α 的耦合为什么让联合优化变难——我至今只停留在"知道它存在、知道它影响结果"的皮毛层面。正因为我只停在皮毛，当前这个评分器里必然还藏着一些我自己看不出来的不合理设计——只是以我现在的功力，暂时发现不了它们。

不过先把它当黑盒用起来、产生价值，比等完全搞懂再动手更现实；深层次的理解可以后面再补。不懂，不等于不能上手；不能推导，不等于不能用好。真正拖慢一个项目的，往往不是哪里不会，而是不敢在不会的地方先动起来。
