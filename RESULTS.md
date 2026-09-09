# 阶段性结果总结

## 1. 数据规模

- **全量PubMed语料**：125,214篇T1D相关论文（`data/all_type1_diabetes_pubmed_papers.csv.gz`）
- **人工标注GT**：91篇采样论文中的68篇，由Yuqi和Yuefei两人独立标注，经union+adjudication流程合并裁决，得到最终GT：
  - 876个实体（10类：Disease, Gene, Protein, Drug, Chemical, Biological_Process, Pathway, Cell_Type, Biomarker, Clinical_Outcome）
  - 459条关系（BioRED风格8类：Positive_Correlation, Negative_Correlation, Association, Bind, Drug_Interaction, Cotreatment, Comparison, Conversion）

> 后文704、747等数字统计口径不同：704为跨68篇论文去重后的unique实体数（用于第3节本体标准化），747为第4节模型评测用的raw entity gold（Yuqi/Yuefei标注取并集）。

## 2. 标注一致性（Inter-Annotator Agreement）

对Yuqi和Yuefei独立标注的原始结果（未经裁决前）计算一致性，按匹配严格程度分三档：

| 匹配标准 | Entity Jaccard | Relation Jaccard |
|---|---:|---:|
| 精确匹配（逐字一致） | 39.2% | 10.9% |
| 精确匹配 + 允许缩写/笔误 | 42.8% | 14.0% |

分别以Yuqi、Yuefei各自的标注总数为分母（而非并集）：

| 匹配标准 | Entity 相对Yuqi | Entity 相对Yuefei | Relation 相对Yuqi | Relation 相对Yuefei |
|---|---:|---:|---:|---:|
| 精确匹配 | 59.5% | 53.5% | 19.4% | 19.9% |
| 精确匹配 + 允许缩写/笔误 | 63.4% | 56.9% | 24.3% | 24.9% |

## 3. PanKgraph 标准化覆盖率

将GT中的704个去重实体（68篇论文汇总后）逐一标准化到公开生物医学本体（MONDO/GO/CL/HGNC/UniProt/ChEBI/HP，V2版本再加MeSH兜底），并检查是否能在外部PanKgraph知识图谱中找到对应节点：

| 版本 | PanKgraph_Accepted | External_Standardized | Local_Concept（标准化失败） |
|---|---:|---:|---:|
| V1（无MeSH） | 56 | 168 | 480 |
| V2（+MeSH兜底） | 57 | 247 | 400 |

加入MeSH后，整体标准化成功率从31.8%提升到43.2%，主要收益在疾病/生物标志物类的老式/同义命名法。

**PanKgraph自身覆盖极窄**：图谱里仅有4种node类型可能对应我们的实体schema（disease/gene/gene_ontology/anatomical_structure，对应Disease/Gene/Biological_Process/Cell_Type这4类），且disease节点全库仅3个（T1D、T2D、healthy）。该图谱本质是一个"T1D遗传学/多组学图谱"（GWAS信号、效应基因、供体样本为主），不建模疾病-并发症等临床语义关系，与文献知识图谱的目标本就不完全重合。

## 4. GPT-4o vs GPT-5 抽取效果对比

用四套不同严格程度的gold standard评测两个模型的实体/关系抽取效果：

- **raw**：Yuqi+Yuefei标注的真实并集（747实体/455关系），不做标准化过滤
- **broad_v1**：4类PanKgraph相关实体（Disease/Gene/Cell_Type/Biological_Process）里标准化成功的（295/29）
- **broad_v2**：全部10类里标准化成功的（539/89）
- **precise**：4类里标准化成功且PanKgraph真能查到节点的（92/3）

计数口径：entity数量按论文中的entity mention统计（broad_v1/broad_v2/precise为Yuqi+Yuefei两人mention的直接池化，不做跨annotator合并；raw见上一节说明）；relation上，broad_v1/broad_v2/precise只有当source和target两个端点实体都满足该版本的entity筛选条件时才保留该relation，raw的relation则直接来自两人标注关系的并集，不额外做entity归属过滤。

### 精确匹配结果（GPT-4o vs GPT-5，四个Gold版本）

**GPT-4o**

| Gold版本 | 实体数 | 关系数 | Entity P | Entity R | Entity F1 | Relation P | Relation R | Relation F1 |
|---|---:|---:|---:|---:|---:|---:|---:|---:|
| raw | 747 | 455 | 0.512 | 0.285 | 0.366 | 0.254 | 0.116 | 0.160 |
| broad_v1 | 295 | 29 | 0.197 | 0.278 | 0.231 | 0.033 | 0.241 | 0.059 |
| broad_v2 | 539 | 89 | 0.380 | 0.293 | 0.331 | 0.072 | 0.169 | 0.101 |
| precise | 92 | 3 | 0.067 | 0.304 | 0.110 | 0.010 | 0.667 | 0.019 |

**GPT-5**

| Gold版本 | 实体数 | 关系数 | Entity P | Entity R | Entity F1 | Relation P | Relation R | Relation F1 |
|---|---:|---:|---:|---:|---:|---:|---:|---:|
| raw | 747 | 455 | 0.472 | 0.448 | 0.460 | 0.192 | 0.182 | 0.187 |
| broad_v1 | 295 | 29 | 0.141 | 0.339 | 0.199 | 0.021 | 0.310 | 0.039 |
| broad_v2 | 539 | 89 | 0.295 | 0.388 | 0.335 | 0.051 | 0.247 | 0.084 |
| precise | 92 | 3 | 0.049 | 0.380 | 0.087 | 0.007 | 1.000 | 0.014 |

## 5. 为什么Relation效果明显差于Entity

拆解GPT-5的432条relation预测（raw gold）：

- **62%**：至少一个端点实体本身就不在人工标注中（entity识别错误直接传导放大到relation上）
- **38%**：两个端点实体都识别正确，但其中：
  - 约82%为人工GT中未标注的额外关联，其中不少从原文语义看属于合理关系（未逐条人工复核确认，非系统性验证结论）
  - 约18%是关系标签本身判断错误，其中高度集中在GPT-5把"Positive_Correlation"误标为更笼统的"Association"这一种系统性模式

## 6. Qwen模型规模对比（另一套评测，基于早期8篇论文GT，样本较小）

GPT-4o + Qwen2.5(0.5B/7B/14B/32B) + Qwen3-4B + Qwen3.5(0.8B/2B/4B/9B/27B)，共11个模型，详见`data/model_eval_summary.csv`。因样本量（8篇）远小于上述68篇GT评测，不与GPT-4o/GPT-5的68篇评测直接比较。

| 模型 | Entity F1 | Relation F1 |
|---|---:|---:|
| Qwen2.5-32B | 0.283 | 0.166 |
| GPT-4o | 0.268 | 0.117 |
| Qwen2.5-7B | 0.244 | 0.114 |
| Qwen2.5-14B | 0.224 | 0.156 |
| Qwen3.5-4B | 0.223 | 0.083 |
| Qwen3.5-9B | 0.217 | 0.094 |
| Qwen3-4B | 0.214 | 0.090 |
| Qwen3.5-27B | 0.192 | 0.166 |
| Qwen3.5-2B | 0.132 | 0.055 |
| Qwen2.5-0.5B | 0.091 | 0.000 |
| Qwen3.5-0.8B | 0.012 | 0.000 |

**结论**：
1. **表现最好的是Qwen2.5-32B**，Entity和Relation F1均超过GPT-4o，说明本地部署的开源模型在这个特定的生物医学信息抽取任务上，规模足够大时可以达到甚至超过GPT-4o的水平。
2. **旧一代（Qwen2.5）整体不输新一代（Qwen3.5/Qwen3）**——同等或更小尺寸下，Qwen2.5系列的F1普遍持平或更高，说明模型代际更新未必带来这个具体任务上的提升。
3. **存在一个明显的能力门槛**——0.5B和0.8B这两个最小模型的Relation F1都是0，Entity F1也远低于其他模型，说明在这个任务复杂度下，模型规模小于某个阈值会完全丧失结构化抽取能力，而不是"效果差一点"。
