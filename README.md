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

### `data/clean_candidate_pool_pmids.csv`

Rule-based (no AI) quality/relevance filter applied to the full 125,178-paper
corpus, producing a clean pool of **75,780 papers** suitable as a source for
future annotation sampling. Columns: `pmid, year, title`. A paper is excluded
if:
- it has no abstract (21,666 papers), or
- `matched_terms` is empty in the full corpus file — i.e. none of the
  diabetes keywords actually appear in the paper's own title/abstract text,
  meaning it was only pulled in via PubMed's MeSH auto-term-mapping rather
  than genuinely discussing diabetes (27,552 papers; this is exactly how a
  confirmed false positive, "Penile necrosis secondary to an indwelling
  Foley catheter" pmid 3669177, slipped into the original 91-paper sample —
  it carries a "Diabetes Mellitus, Type 1" MeSH tag but never mentions
  diabetes in its abstract), or
- its abstract is under 200 characters (180 papers).

### `data/sampled_t1d_papers_for_labeling_191.csv`

The original 91-paper annotation sample (rows 1-91) plus 100 newly sampled
papers (rows 92-191, drawn at random, seed 42, from the clean candidate pool
above, excluding papers already in the original 91) appended directly after
it — ready to hand off for the next round of manual annotation. Same 9-column
schema as the original sample file (`pmid, pubmed_url, year, title, abstract,
journal, keywords, mesh_terms, publication_types`).

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

### `data/annotated_papers_with_labels_68papers.csv`

One row per annotated paper (68 rows) joining PubMed metadata with the final
GT labels, so each paper's source text and its annotations can be viewed
together without cross-referencing separate files.

Columns: `pmid, pubmed_url, year, title, journal, abstract, keywords,
mesh_terms, publication_types, entities, relations` — `entities` and
`relations` are semicolon-joined in the same `mention (Type)` /
`(source, relation, target)` format as elsewhere, sourced from
`gt_entities_68papers.csv` / `gt_relations_68papers.csv` (the final
adjudicated GT, not the raw per-annotator columns).

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
