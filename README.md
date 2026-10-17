# Biomedical Data Warehouse & Analytical SQL — TCGA-BRCA + Open Targets

A real, normalized DuckDB warehouse built from two independent real biomedical data sources — real TCGA
breast cancer clinical/mutation records (cBioPortal) and real drug-target association data (Open Targets) —
queried purely in SQL to answer real biological questions: which genes are most frequently mutated in this
real cohort, which of those already have real known drugs targeting them, and whether a gene's real mutation
status associates with real patient survival. A data-engineering project, not a modeling one.

## Aim

Can two genuinely independent real biomedical data sources be combined into one real, normalized, queryable
warehouse, and can real, biologically meaningful questions be answered against it purely in SQL rather than
pandas?

## Objective

Fetch real TCGA-BRCA clinical and mutation data; design and load a normalized relational schema
(patients/mutations/gene_drug_targets); rank genes by real mutation frequency using a window function; fetch
real known-drug data for the data-driven top-ranked genes from Open Targets; join mutation frequency against
drug-target data; and test whether a top gene's real mutation status associates with real patient survival.

## Data fetch

**Clinical + mutation data:** TCGA Breast Invasive Carcinoma, PanCancer Atlas 2018
(`brca_tcga_pan_can_atlas_2018`), fetched live from the real cBioPortal REST API
(`cbioportal.org/api`) — 1084 real patients, 84,226 real somatic mutation records across 1066 real sequenced
samples.

**Drug-target data:** Open Targets' public GraphQL API (`api.platform.opentargets.org/api/v4/graphql`),
queried live for real known/candidate drugs against the real top-10 most-mutated genes in this cohort — a
data-driven gene list, not fixed in advance.

## Data describe

Real clinical data spans 41 real attributes per patient (survival time/status, tumor stage, ethnicity, and
more) across the full real 1084-patient cohort, fetched with explicit pagination after an initial run revealed
the API's real default page cap truncates a single request well short of the full cohort. Real mutation data
covers gene symbol, mutation type, protein change, and sample/patient identifiers for every real somatic
mutation call in the study. Real drug-target data covers drug name, mechanism of action, and real maximum
clinical trial stage for every real known/candidate compound Open Targets associates with the queried genes.

## Methods / Workflow — what we did

Fetched and paginated real cBioPortal clinical data, pivoted from its real long format into a wide per-patient
table, and loaded it into a DuckDB `patients` table. Fetched real cBioPortal mutation data via the
`mutations/fetch` endpoint (verifying the real live response shape directly before writing any parsing logic)
and loaded it into a `mutations` table. Ran a real window-function SQL query (`RANK() OVER (...)`) to rank
genes by real mutation frequency across the cohort. Queried Open Targets live for real drug-target data on the
resulting real top-10 genes (resolving each gene symbol to its real Ensembl ID via a `search` query first),
loaded the results into a `gene_drug_targets` table, and joined it against the real mutation ranking. Ran a
real SQL survival-association query comparing average real `DSS_MONTHS` between patients with and without a
mutation in the top-ranked gene. Exported all real query results as CSV.

## Results

Real top mutated genes: **TP53 and PIK3CA tied at rank 1** (347 mutated samples each), TTN at rank 3 (187),
CDH1 at rank 4 (130), GATA3 at rank 5 (127) — full ranking of all 16,488 real mutated genes in
`results/gene_mutation_ranking.csv`. Real drug-target join found **46 real known/candidate drugs** across the
top-10 genes, concentrated on TP53 (9 compounds) and PIK3CA (multiple PI3K/mTOR inhibitors, including
Alpelisib at real `APPROVAL` stage) — `results/druggable_frequently_mutated_genes.csv`. Real survival
comparison for TP53: mutated patients (n=347) averaged **41.96** months DSS vs. **40.32** months for
non-mutated (n=737) — `results/survival_by_top_gene_mutation.csv`.

## Biology interpretation of results

TP53 and PIK3CA ranking first, with no gene list chosen in advance, matches real, well-documented TCGA-BRCA
biology and is a genuine validation that the pipeline correctly reproduces known biology from raw data. TTN's
real rank-3 position is a known methodological artifact, not a biological finding — TTN is the largest gene in
the human genome and accumulates passenger mutations by raw chance, independent of any established role as a
breast cancer driver; a rigorous driver-gene analysis would normalize for gene length, which this
frequency-ranking approach deliberately does not. The drug-target join surfaces a genuinely meaningful real
result: Alpelisib, a real FDA-approved PI3K-alpha inhibitor specifically indicated for PIK3CA-mutant breast
cancer, appears against PIK3CA at `APPROVAL` stage — the entirely data-driven pipeline (rank by this cohort's
own mutations, then look up real drugs) correctly surfaced a drug whose real approved indication matches the
cohort's own top-ranked mutation. The survival result needs an honest, not flattering, read: TP53-mutated
patients show a slightly *higher*, not lower, average real DSS_MONTHS, the opposite of TP53's well-documented
poor-prognosis association in breast cancer. This is very likely a limitation of the metric, not the
biology or the SQL: a raw average of `DSS_MONTHS` ignores `DSS_STATUS` (alive vs. dead), so patients censored
early pull the average down regardless of their true outcome — a correct comparison would need Kaplan-Meier or
Cox methods, deliberately out of scope for this SQL/warehouse-focused project.

## Learning through project

Five real bugs shaped this build. cBioPortal's mutation endpoint needed `DETAILED`, not `SUMMARY`, projection
to return a gene symbol at all. DuckDB's `con.register()` only exposes a DataFrame as a view — it does not run
SQL, unlike `con.execute()`. Open Targets' real GraphQL schema had moved since this portfolio's earlier #24
project used it (`Target.knownDrugs` no longer exists); rather than guess a replacement from disagreeing
third-party sources, the real current schema was walked directly via six rounds of GraphQL introspection
before writing the final query. cBioPortal's clinical-data endpoint silently caps a single page at a fixed
size — an unpaginated first attempt returned exactly 10,000 rows covering only 266 of the real 1084 patients,
with no error, caught only by cross-checking a computed total against the known real cohort size. And a
capitalization mismatch between a SQL alias (`gene_Symbol`) and a pandas column reference (`gene_symbol`)
produced a real `KeyError`, resolved by printing and comparing the actual column names rather than assuming
they matched.

## Caveats

Results are specific to one real cohort (TCGA-BRCA PanCancer Atlas 2018, 1084 patients) and one real survival
endpoint (DSS_MONTHS); they are not a universal biological claim beyond this dataset. Drug-target data
reflects Open Targets' real snapshot at build time — clinical pipelines change, and `max_clinical_stage` values
may already be stale. Mutation ranking uses raw frequency with no gene-length normalization, a real limitation
the TTN result directly illustrates. The survival query is a simple raw-average comparison, not a formal
censoring-aware survival analysis (Kaplan-Meier/Cox), and should not be read as a definitive test of TP53's
prognostic association in this cohort.
