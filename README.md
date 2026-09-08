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
