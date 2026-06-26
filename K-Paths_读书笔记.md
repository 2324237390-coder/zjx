# K-Paths 读书笔记

> 论文：**K-Paths: Reasoning over Graph Paths for Drug Repurposing and Drug Interaction Prediction**  
> 作者：Tassallah Abdullahi, Ioanna Gemou, Nihal V. Nayak, Ghulam Murtaza, Stephen H. Bach, Carsten Eickhoff, Ritambhara Singh  
> 发表：**KDD 2025，Proceedings of the 31st ACM SIGKDD Conference on Knowledge Discovery and Data Mining**  
> 年份：**2025**  
> 论文链接：<https://dl.acm.org/doi/10.1145/3711896.3737011>  
> arXiv 链接：<https://arxiv.org/abs/2502.13344>  
> 代码链接：<https://github.com/rsinghlab/K-Paths>

---

## 1. 阅读目的

我当前负责的任务是：

> **调研并设计基于知识图谱的科研假设提出策略，重点在于如何探索知识图谱中的弱关联或未知关联，并基于这些关联提出有效科研假设。**

在这个任务边界下，我不负责构建知识图谱，而是假设已经存在一个 biomedical knowledge graph，然后研究如何从已有图谱中发现潜在关系、解释这些关系，并将其进一步转化为科研假设。

因此，K-Paths 这篇论文的价值在于：

1. 它不是研究如何构建知识图谱，而是在已有知识图谱上进行路径检索；
2. 它关注 drug-disease 和 drug-drug 这类未观察到关系的预测；
3. 它强调路径的结构化、多样性和可解释性；
4. 它的路径可以作为 LLM 或 GNN 的输入；
5. 它可以作为后续“图谱路径证据 → 科研假设生成”的参考方法。

---

## 2. 论文解决的问题

### 2.1 背景问题

药物发现和药物重定位是非常耗时、成本高的过程。生物医学知识图谱中已经包含了大量药物、疾病、基因、蛋白、通路等实体及其关系，但这些知识图谱通常是不完整的。

例如，图谱中可能已经有：

```text
Drug A → targets → Gene X
Gene X → associated_with → Disease B
```

但图谱中未必直接记录：

```text
Drug A → treats / affects → Disease B
```

这条 `Drug A - Disease B` 关系就是一个**未观察到的潜在关联**。

K-Paths 试图解决的问题是：

> 如何从已有生物医学知识图谱中检索出结构化、多样化、具有生物学意义的多跳路径，并利用这些路径辅助预测未观察到的 drug-disease 和 drug-drug 关系？

---

## 3. 论文核心思想

K-Paths 的核心思想可以概括为：

```text
已有 biomedical KG
        ↓
检索 K 条多跳路径
        ↓
筛选结构化、多样化、生物学有意义的路径
        ↓
将路径输入 LLM 或 GNN
        ↓
预测未观察到的 drug-disease / drug-drug 关系
```

它不是单纯依靠黑盒 embedding 或 GNN 直接预测一条边，而是先找出两端实体之间的多跳连接路径。

例如对于候选关系：

```text
Drug A 是否可能治疗 Disease B？
```

K-Paths 可能检索出：

```text
Drug A → targets → Protein X
Protein X → participates_in → Pathway Y
Pathway Y → associated_with → Disease B
```

这条路径就可以解释为什么 Drug A 和 Disease B 之间可能存在关系。

---

## 4. 关键词解释

### 4.1 Biomedical Knowledge Graph

生物医学知识图谱，即由生物医学实体和关系组成的图结构。

常见节点包括：

```text
Drug / Disease / Gene / Protein / Pathway / Symptom / Side Effect
```

常见关系包括：

```text
targets / treats / associated_with / participates_in / causes / interacts_with
```

### 4.2 Multi-hop Path

多跳路径是指实体之间不是通过一条边直接连接，而是通过多个中间节点连接。

一跳：

```text
Drug A → Disease B
```

两跳：

```text
Drug A → Gene X → Disease B
```

三跳：

```text
Drug A → Protein X → Pathway Y → Disease B
```

在科研假设发现中，多跳路径通常更有价值，因为它们可能揭示间接机制。

### 4.3 Structured Path

结构化路径不是任意节点的连接，而是包含明确的节点类型和关系类型。

例如：

```text
Drug → targets → Protein → participates_in → Pathway → associated_with → Disease
```

这类路径比普通的“节点 A 连接到节点 B”更适合生成机制解释。

### 4.4 Diverse Paths

多样化路径是指对于同一个候选关系，检索多种不同类型的路径，而不是只找最短或最相似的一条路径。

例如对于 `Drug A - Disease B`，可以有：

```text
Path 1: Drug A → Gene X → Disease B
Path 2: Drug A → Pathway Y → Disease B
Path 3: Drug A → Side Effect Z → Disease B
```

多样化路径可以提供多个角度的证据。

### 4.5 Explainable Path

可解释路径是指路径不仅用于预测，还能解释为什么预测成立。

普通 link prediction 可能只输出：

```text
Drug A - Disease B: score = 0.87
```

而可解释路径会输出：

```text
Drug A → targets Protein X
Protein X → participates in Pathway Y
Pathway Y → associated with Disease B
```

这对科研假设提出尤其重要，因为科研假设需要机制解释和证据链。

### 4.6 Zero-shot Reasoning

Zero-shot reasoning 指的是不专门训练 LLM，只给 LLM 输入任务说明和图谱路径，让它直接进行推理。

例如：

```text
Input:
Drug A targets Protein X.
Protein X participates in Pathway Y.
Pathway Y is associated with Disease B.

Question:
Could Drug A be repurposed for Disease B?
```

LLM 根据路径直接判断并生成解释，这就是 zero-shot reasoning。

---

## 5. 方法流程理解

### 5.1 输入

K-Paths 的输入包括：

1. 一个已有的 biomedical KG；
2. 一个待判断的实体对，例如 drug-disease 或 drug-drug；
3. 目标任务，例如 drug repurposing 或 drug-drug interaction prediction。

### 5.2 路径检索

论文使用一种 diversity-aware 的路径检索方法，从两个实体之间寻找 K 条 loopless paths，即没有环的路径。

简单理解：

```text
给定实体 A 和实体 B
    ↓
在 KG 中搜索 A 到 B 的多条路径
    ↓
保留较短、较相关、类型多样的路径
```

### 5.3 路径结构化

检索出来的路径会被转化成适合模型处理的结构化格式。

例如：

```text
Drug A --targets--> Protein X --participates_in--> Pathway Y --associated_with--> Disease B
```

这种格式可以被 LLM 读入，也可以被 GNN 转成子图结构。

### 5.4 用于 LLM

K-Paths 将路径转成自然语言或结构化文本，输入给 LLM，让 LLM 判断候选关系是否成立。

这一步的重点是：

> 用知识图谱路径约束 LLM 推理，减少 LLM 凭空猜测。

### 5.5 用于 GNN

K-Paths 还可以把检索出的路径构造成较小的子图，让 GNN 在更小、更相关的图结构上训练和预测。

这样可以减少完整 KG 的规模，同时保留与任务相关的信息。

---

## 6. 实验任务

论文主要评估两个任务：

### 6.1 Drug Repurposing

药物重定位，即预测已有药物是否可以用于新的疾病。

任务形式：

```text
Input: Drug + Disease
Output: 该 drug 是否可能用于该 disease
```

对应当前任务中的：

```text
未知 drug-disease 关联发现
```

### 6.2 Drug-Drug Interaction Prediction

药物相互作用预测，即预测两个药物之间是否存在相互作用，或判断相互作用严重程度。

任务形式：

```text
Input: Drug A + Drug B
Output: 两个药物之间是否存在 interaction / interaction severity
```

---

## 7. 主要结果理解

论文的实验表明：

1. K-Paths 可以提升 LLM 的 zero-shot 推理效果；
2. K-Paths 可以帮助 GNN 在更小的子图上保持较强预测性能；
3. K-Paths 生成的路径具有解释性，可以为预测结果提供 rationale；
4. 它在 drug repurposing 和 drug-drug interaction prediction 两类任务中都有作用。

从当前任务角度看，最重要的不是具体提升了多少指标，而是：

> K-Paths 证明了“图谱路径”可以作为未知关联预测和解释的有效中间表示。

---

## 8. 与普通 Link Prediction 的区别

普通 link prediction 往往直接输出一个预测分数：

```text
Drug A - Disease B: 0.87
```

但这种结果不够适合科研假设生成，因为它缺少解释。

K-Paths 更强调：

```text
Drug A 可能与 Disease B 有关
因为存在以下路径：
Drug A → Protein X → Pathway Y → Disease B
```

因此，K-Paths 更适合支持科研假设提出。

---

## 9. 与当前任务的关系

当前任务是：

> 基于知识图谱探索弱关联或未知关联，并提出有效科研假设。

K-Paths 对应其中的前半部分：

```text
已有知识图谱
    ↓
检索多跳路径
    ↓
发现潜在未知关联
    ↓
提供证据链
```

但 K-Paths 本身还没有完全解决：

```text
图谱路径 → 科研假设 → 实验验证建议
```

所以可以将 K-Paths 作为一个图谱路径检索与证据链构建模块，再在其基础上加入 Graph-to-Hypothesis 步骤。

---

## 10. 可以如何融入项目

最简单的融入方式不是大改 K-Paths，而是把它作为一个图谱证据检索工具。

最小闭环：

```text
已有知识图谱
    ↓
K-Paths / 路径检索模块
    ↓
输出 drug-disease 或 drug-drug 的多跳路径
    ↓
LLM 将路径转化为科研假设
    ↓
SuperScientist / SuperJudge 评价和排序
```

在项目中可以命名为：

```text
Graph Evidence Retriever
```

它的职责是：

1. 给定候选实体对；
2. 检索多跳路径；
3. 输出结构化证据链；
4. 为后续假设生成提供图谱上下文。

---

## 11. 对复现的启发

完整复现 K-Paths 可能包括：

1. 路径抽取；
2. LLM zero-shot 推理；
3. GNN 子图训练；
4. 多任务评估。

但对于当前阶段，不一定需要完整复现全部流程。

更适合的最小复现目标是：

```text
输入：已有 biomedical KG 或样例图谱
任务：给定 drug-disease 实体对
方法：检索 2-hop / 3-hop 多跳路径
输出：
1. 候选未知关联
2. 证据路径
3. 科研假设文本
```

如果官方代码和论文实现存在差异，可以采用模块级复现：

> 不追求完整复现论文 benchmark，而是复现 K-Paths 中与任务最相关的核心思想：多跳路径检索与证据链生成。

---

## 12. 局限性与注意点

### 12.1 依赖已有 KG 的质量

K-Paths 不构建知识图谱，因此它的效果依赖输入 KG 的覆盖范围和关系质量。

如果 KG 中缺少关键节点或边，路径检索也很难发现有价值的关系。

### 12.2 路径存在不等于因果关系

即使 Drug A 和 Disease B 之间存在多跳路径，也不代表它们一定存在真实因果关系。

路径只能说明：

```text
存在潜在关联线索
```

而不是：

```text
关系已经被证明成立
```

### 12.3 路径可能包含噪声

大型医学 KG 中可能存在很多弱相关、文本共现或低置信度关系。路径检索需要结合关系类型、证据强度、路径长度和生物学合理性进行过滤。

### 12.4 需要进一步转化为科研假设

K-Paths 输出的是路径和预测结果，不是完整科研假设。

对于当前任务，还需要增加：

```text
Graph-to-Hypothesis
Hypothesis Evaluation
SuperJudge Ranking
```

---

## 13. 阶段性理解

K-Paths 对当前任务最重要的启发是：

> 基于知识图谱提出科研假设时，不应该只预测一条未知边，而应该同时给出支持这条未知边的多跳路径证据。

因此，后续可以设计如下流程：

```text
已有 KG
    ↓
路径检索：找到 Drug / Gene / Pathway / Disease 之间的多跳路径
    ↓
弱关联识别：判断哪些路径对应当前 KG 中尚未直接记录的关系
    ↓
证据链构建：整理路径中的实体、关系和证据
    ↓
假设生成：用 LLM 把路径转化为研究问题、机制解释和验证实验
    ↓
假设评价：由 SuperJudge 从新颖性、机制合理性、证据一致性、可验证性等维度排序
```

---

## 14. 可用于口头汇报的简短总结

K-Paths 是 KDD 2025 的一篇会议论文，主要研究如何在已有生物医学知识图谱上检索结构化、多样化、可解释的多跳路径，并用这些路径辅助 LLM 或 GNN 预测未观察到的 drug-disease 和 drug-drug 关系。它和我的任务比较匹配，因为我不负责构建知识图谱，而是关注如何基于已有图谱发现弱关联或未知关联。K-Paths 可以作为图谱路径检索和证据链构建的参考方法，后续我可以在它的基础上进一步设计“图谱路径到科研假设”的生成策略。

---

## 15. 后续阅读问题

继续阅读或复现时，需要重点关注以下问题：

1. K-Paths 如何定义一条路径是否 biologically meaningful？
2. 它如何保证路径的 diversity？
3. 它如何将路径转化为 LLM 可读输入？
4. 它的路径检索是否可以单独复现？
5. 如果不复现完整代码，如何做一个最小路径检索 demo？
6. 如何把 K-Paths 输出路径转化为科研假设？
7. 如何评价生成假设的新颖性、机制合理性和可验证性？
