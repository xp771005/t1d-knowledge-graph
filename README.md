# Type 1 Diabetes Literature Knowledge Graph

Data resources for a course project building a Type 1 Diabetes (T1D)
biomedical knowledge graph from PubMed literature.

## Contents

### `data/all_type1_diabetes_pubmed_papers.csv.gz`

Full PubMed corpus of T1D-related papers, collected via the NCBI E-utilities
API (Abstract-field search over a fixed keyword list, full available date
range, deduplicated).

- **125,214 papers**
- Columns: `pmid, pubmed_url, year, publication_date, title, abstract, journal,
  authors, doi, doi_url, keywords, mesh_terms, publication_types,
  matched_terms, has_abstract`

Decompress with `gunzip data/all_type1_diabetes_pubmed_papers.csv.gz`.

### `data/year_counts.csv`

Per-year paper counts for the full corpus above.

### `data/gt_entities_68papers.csv` / `data/gt_relations_68papers.csv`

Human-annotated ground truth built from 68 of a 91-paper sample, independently
annotated by two annotators and reconciled via a union + adjudication process
(entities/relations agreed by both annotators, agreed-upon single-annotator
additions, and adjudicated disagreements are all tiered and retained).

- `gt_entities_68papers.csv`: 876 entities across 10 types (Disease, Gene,
  Protein, Drug, Chemical, Biological_Process, Pathway, Cell_Type, Biomarker,
  Clinical_Outcome). Columns: `pmid, mention, entity_type, tier, note`.
- `gt_relations_68papers.csv`: 459 relations (BioRED-style relation types:
  Positive_Correlation, Negative_Correlation, Association, Bind,
  Drug_Interaction, Cotreatment, Comparison, Conversion). Columns:
  `pmid, source, relation, target, tier, note`.
- `tier` marks provenance: `Tier1_agree` (both annotators agreed), a
  `Tier2_*` variant (agreement reached after adjudicating a disagreement —
  e.g. `Tier2_type_disagree_*`, `Tier2_reviewed_*`, `Tier2_resolved_specific`),
  or `Tier3_single_*` (only one annotator captured it, adjudicated in).

### Four gold-standard variants used for model evaluation

Built to test whether the extraction task should be scored against the full
adjudicated annotation, or only against the subset of entities/relations that
map onto a standard biomedical ontology (and, further, onto the external
PanKgraph knowledge graph specifically). Each file has one row per paper with
columns `pmid, entity, relation` (same `mention (Type)` / `(source, relation,
target)` semicolon-joined cell format as the source annotation table).

| File | Scope | Filter | Entities | Relations |
|---|---|---|---:|---:|
| `raw_union_by_paper.xlsx` | all 10 entity types | none — the full deduplicated union of both annotators | 747 | 455 |
| `broad_v1_4types_by_paper.xlsx` | Disease/Gene/Cell_Type/Biological_Process only (the 4 types with a PanKgraph node label) | standardized to a real ontology ID (any match quality) | 295 | 29 |
| `broad_v2_10types_by_paper.xlsx` | all 10 entity types | standardized to a real ontology ID | 539 | 89 |
| `precise_4types_by_paper.xlsx` | same 4 types as broad_v1 | standardized AND the resulting ID has a matching node in PanKgraph | 92 | 3 |

Entity/relation counts here are pooled (Yuqi + Yuefei mentions kept
separately, not merged) except `raw_union_by_paper.xlsx`, which is the true
deduplicated union (see `gt_entities_68papers.csv` above) — a disagreement
pair (same mention, different type/relation label) is counted once, keeping
the Yuqi-labeled variant.

### `data/gpt4o_vs_gpt5_eval.csv`

GPT-4o vs GPT-5 entity/relation extraction (same prompt, same 91-paper
sample), scored against all four gold variants above, both under exact
normalized-string matching and under a fuzzy match (substring + string
similarity + verified synonym table, greedily matched per paper). Columns
follow the pattern `{gold_version}_{entity|relation}_{exact|fuzzy}_{P|R|F1}`.

Only the `raw` column is valid evidence for "which model extracts better" —
the other three gold sets shrink sharply from one filter to the next, which
mechanically depresses precision and inflates recall regardless of model
quality (the model's own prediction set is fixed; a smaller gold target is
simply harder to land inside). They are informative about how narrow
PanKgraph's own knowledge coverage is, not about relative model quality.

### `data/model_eval_summary.csv`

GPT-4o + 10 Qwen variants (Qwen2.5: 0.5B/7B/14B/32B; Qwen3-4B; Qwen3.5:
0.8B/2B/4B/9B/27B), entity/relation P/R/F1 against an earlier, smaller
8-paper union ground truth (`union_entities_8papers.csv` /
`union_relations_8papers.csv`, not included here) — kept separate from the
GPT-4o/GPT-5 comparison above because it uses a different (smaller, less
refined) gold standard and does not include GPT-5.
