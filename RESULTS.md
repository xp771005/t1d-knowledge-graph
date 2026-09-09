# 阶段性结果总结

## 1. 数据规模

- **全量PubMed语料**：125,214篇T1D相关论文（`data/all_type1_diabetes_pubmed_papers.csv.gz`）
- **人工标注GT**：91篇采样论文中的68篇，由Yuqi和Yuefei两人独立标注，经union+adjudication流程合并裁决，得到最终GT：
  - 876个实体（10类：Disease, Gene, Protein, Drug, Chemical, Biological_Process, Pathway, Cell_Type, Biomarker, Clinical_Outcome）
  - 459条关系（BioRED风格8类：Positive_Correlation, Negative_Correlation, Association, Bind, Drug_Interaction, Cotreatment, Comparison, Conversion）

## 2. 标注一致性（Inter-Annotator Agreement）

对Yuqi和Yuefei独立标注的原始结果（未经裁决前）计算一致性，按匹配严格程度分三档：

| 匹配标准 | Entity Jaccard | Relation Jaccard |
|---|---:|---:|
| 精确匹配（逐字一致） | 39.2% | 10.9% |
| 精确匹配 + 允许缩写/笔误 | 42.8% | 14.0% |
| 完整模糊匹配（允许子串/相似表述/同义词） | 61.8% | 32.1% |

分别以Yuqi、Yuefei各自的标注总数为分母（而非并集）：

| 匹配标准 | Entity 相对Yuqi | Entity 相对Yuefei | Relation 相对Yuqi | Relation 相对Yuefei |
|---|---:|---:|---:|---:|
| 精确匹配 | 59.5% | 53.5% | 19.4% | 19.9% |
| 精确匹配 + 允许缩写/笔误 | 63.4% | 56.9% | 24.3% | 24.9% |
| 完整模糊匹配 | 80.7% | 72.5% | 48.0% | 49.2% |

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

### 精确匹配结果（GPT-5）

| Gold版本 | Entity P | Entity R | Entity F1 | Relation P | Relation R | Relation F1 |
|---|---:|---:|---:|---:|---:|---:|
| raw | 0.472 | 0.448 | 0.460 | 0.192 | 0.182 | 0.187 |
| broad_v1 | 0.141 | 0.339 | 0.199 | 0.021 | 0.310 | 0.039 |
| broad_v2 | 0.295 | 0.388 | 0.335 | 0.051 | 0.247 | 0.084 |
| precise | 0.049 | 0.380 | 0.087 | 0.007 | 1.000 | 0.014 |

### 模型对比（raw作为唯一有效对比基准）

| | Entity F1（精确/模糊） | Relation F1（精确/模糊） |
|---|---|---|
| GPT-4o | 0.366 / 0.526 | 0.160 / 0.322 |
| **GPT-5** | **0.460 / 0.620** | **0.187 / 0.406** |

**GPT-5在实体和关系抽取上均全面优于GPT-4o**，提升主要来自Recall（更全面），Precision略低（更"敢抽"）。

**重要提醒**：broad_v1/broad_v2/precise三列不能用于判断模型优劣——模型的预测结果是固定的，gold集合越收越窄，命中的目标必然越来越少，Precision暴跌是数学上的必然结果，这三列真正的用途是揭示PanKgraph图谱知识覆盖的局限性。

## 5. 为什么Relation效果明显差于Entity

拆解GPT-5的432条relation预测（raw gold）：

- **62%**：至少一个端点实体本身就不在人工标注中（entity识别错误直接传导放大到relation上）
- **38%**：两个端点实体都识别正确，但其中：
  - 约82%是人工没有标注的真实关联（GPT-5多找出来的，非模型错误）
  - 约18%是关系标签本身判断错误，其中高度集中在**GPT-5把"Positive_Correlation"误标为更笼统的"Association"**这一种系统性模式

**结论**：Relation效果差，主因是entity识别不准的复合误差（占大头），次因是relation分类本身也存在难度，且GPT-5在"具体相关性 vs 笼统关联"上有明显的保守化倾向。

## 6. Qwen模型规模对比（另一套评测，基于早期8篇论文GT，样本较小）

GPT-4o + Qwen2.5(0.5B/7B/14B/32B) + Qwen3-4B + Qwen3.5(0.8B/2B/4B/9B/27B)，共11个模型，详见`data/model_eval_summary.csv`。因样本量（8篇）远小于上述68篇GT评测，结论仅供参考，不与GPT-4o/GPT-5的68篇评测直接比较。
